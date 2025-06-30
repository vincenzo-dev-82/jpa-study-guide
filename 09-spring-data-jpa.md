# 9. 스프링 데이터 JPA

## 📖 스프링 데이터 JPA 소개

### 스프링 데이터 JPA란?

스프링 데이터 JPA는 스프링 프레임워크에서 JPA를 편리하게 사용할 수 있도록 지원하는 프로젝트다. 

#### 기존 JPA 사용의 문제점

```java
// 기존 JPA 방식의 반복되는 코드
@Repository
public class MemberJpaRepository {
    
    @PersistenceContext
    private EntityManager em;
    
    public Member save(Member member) {
        em.persist(member);
        return member;
    }
    
    public Member findOne(Long id) {
        return em.find(Member.class, id);
    }
    
    public List<Member> findAll() {
        return em.createQuery("select m from Member m", Member.class)
                .getResultList();
    }
    
    public List<Member> findByName(String name) {
        return em.createQuery("select m from Member m where m.name = :name", Member.class)
                .setParameter("name", name)
                .getResultList();
    }
    
    public long count() {
        return em.createQuery("select count(m) from Member m", Long.class)
                .getSingleResult();
    }
    
    public void delete(Member member) {
        em.remove(member);
    }
}
```

**문제점:**
- 반복되는 CRUD 코드
- 단순한 조건 검색도 JPQL을 작성해야 함
- 개발자가 작성해야 할 코드량이 많음

#### 스프링 데이터 JPA의 해결책

```java
// 스프링 데이터 JPA 사용
public interface MemberRepository extends JpaRepository<Member, Long> {
    
    // 메소드 이름으로 쿼리 자동 생성
    List<Member> findByName(String name);
    
    List<Member> findByNameAndAge(String name, int age);
    
    List<Member> findByNameContaining(String name);
    
    // 커스텀 쿼리도 가능
    @Query("select m from Member m where m.name = :name")
    List<Member> findMembersByName(@Param("name") String name);
}
```

**장점:**
- 인터페이스만 선언하면 구현체를 자동으로 생성
- CRUD 메소드들이 기본 제공
- 메소드 이름으로 쿼리 자동 생성
- 반복되는 코드 제거

### 스프링 데이터 프로젝트

스프링 데이터 JPA는 스프링 데이터 프로젝트의 하위 프로젝트 중 하나다.

**주요 스프링 데이터 프로젝트:**
- Spring Data JPA: JPA에 특화
- Spring Data MongoDB: MongoDB용
- Spring Data Redis: Redis용
- Spring Data Elasticsearch: Elasticsearch용

## 🏗️ 공통 인터페이스 기능

### 주요 인터페이스 구조

```java
// JpaRepository 상속 구조
public interface Repository<T, ID> {}

public interface CrudRepository<T, ID> extends Repository<T, ID> {
    <S extends T> S save(S entity);
    Optional<T> findById(ID id);
    boolean existsById(ID id);
    Iterable<T> findAll();
    Iterable<T> findAllById(Iterable<ID> ids);
    long count();
    void deleteById(ID id);
    void delete(T entity);
    void deleteAllById(Iterable<? extends ID> ids);
    void deleteAll(Iterable<? extends T> entities);
    void deleteAll();
}

public interface PagingAndSortingRepository<T, ID> extends CrudRepository<T, ID> {
    Iterable<T> findAll(Sort sort);
    Page<T> findAll(Pageable pageable);
}

public interface JpaRepository<T, ID> extends PagingAndSortingRepository<T, ID>, QueryByExampleExecutor<T> {
    List<T> findAll();
    List<T> findAll(Sort sort);
    List<T> findAllById(Iterable<ID> ids);
    <S extends T> List<S> saveAll(Iterable<S> entities);
    void flush();
    <S extends T> S saveAndFlush(S entity);
    <S extends T> List<S> saveAllAndFlush(Iterable<S> entities);
    void deleteAllInBatch(Iterable<T> entities);
    void deleteAllByIdInBatch(Iterable<ID> ids);
    void deleteAllInBatch();
    T getOne(ID id);
    T getById(ID id);
    T getReferenceById(ID id);
}
```

### 기본 CRUD 기능

```java
@Service
@Transactional
public class MemberService {
    
    @Autowired
    private MemberRepository memberRepository;
    
    public void crudExample() {
        // 저장
        Member member = new Member("memberA", 20);
        Member savedMember = memberRepository.save(member);
        
        // 단건 조회
        Optional<Member> findMember = memberRepository.findById(savedMember.getId());
        
        // 전체 조회
        List<Member> allMembers = memberRepository.findAll();
        
        // 개수 조회
        long count = memberRepository.count();
        
        // 삭제
        memberRepository.delete(savedMember);
        
        // ID로 삭제
        memberRepository.deleteById(savedMember.getId());
        
        // 존재 확인
        boolean exists = memberRepository.existsById(savedMember.getId());
    }
}
```

### 주요 메소드 분석

#### save() 메소드

```java
@Transactional
public <S extends T> S save(S entity) {
    Assert.notNull(entity, "Entity must not be null.");
    
    if (entityInformation.isNew(entity)) {
        em.persist(entity);
        return entity;
    } else {
        return em.merge(entity);
    }
}
```

**동작 방식:**
- 새로운 엔티티면 `persist()`
- 기존 엔티티면 `merge()`
- 새로운 엔티티 판단 기준: 식별자가 null이거나 0

```java
// 새로운 엔티티 예제
Member newMember = new Member("memberA", 20); // ID가 null
memberRepository.save(newMember); // persist() 호출

// 기존 엔티티 예제  
Member existingMember = memberRepository.findById(1L).get();
existingMember.setName("memberB");
memberRepository.save(existingMember); // merge() 호출
```

#### findById() vs getById()

```java
// findById() - 권장
Optional<Member> member = memberRepository.findById(1L);
if (member.isPresent()) {
    // 안전하게 사용
    Member foundMember = member.get();
}

// getById() - 지연 로딩 프록시 반환
Member member = memberRepository.getById(1L); // 프록시 객체 반환
member.getName(); // 실제 데이터베이스 접근 시점
```

## 🔍 쿼리 메소드 기능

### 메소드 이름으로 쿼리 생성

스프링 데이터 JPA는 메소드 이름을 분석해서 JPQL 쿼리를 생성한다.

#### 기본 사용법

```java
public interface MemberRepository extends JpaRepository<Member, Long> {
    
    // select m from Member m where m.name = ?1
    List<Member> findByName(String name);
    
    // select m from Member m where m.name = ?1 and m.age = ?2
    List<Member> findByNameAndAge(String name, int age);
    
    // select m from Member m where m.age > ?1
    List<Member> findByAgeGreaterThan(int age);
    
    // select m from Member m where m.name like ?1
    List<Member> findByNameLike(String name);
    
    // select m from Member m where m.name like %?1%
    List<Member> findByNameContaining(String name);
    
    // select m from Member m where m.name in ?1
    List<Member> findByNameIn(Collection<String> names);
    
    // select m from Member m where m.age between ?1 and ?2
    List<Member> findByAgeBetween(int startAge, int endAge);
    
    // select m from Member m where m.name is null
    List<Member> findByNameIsNull();
    
    // select m from Member m order by m.name desc
    List<Member> findByAgeOrderByNameDesc(int age);
    
    // 상위 3개 조회
    List<Member> findTop3ByAge(int age);
    List<Member> findFirst3ByAge(int age);
}
```

#### 스프링 데이터 JPA가 제공하는 쿼리 메소드 필터 조건

| 키워드 | 예제 | JPQL 예제 |
|--------|------|-----------|
| And | findByLastnameAndFirstname | where x.lastname = ?1 and x.firstname = ?2 |
| Or | findByLastnameOrFirstname | where x.lastname = ?1 or x.firstname = ?2 |
| Is, Equals | findByFirstname, findByFirstnameIs, findByFirstnameEquals | where x.firstname = ?1 |
| Between | findByStartDateBetween | where x.startDate between ?1 and ?2 |
| LessThan | findByAgeLessThan | where x.age < ?1 |
| LessThanEqual | findByAgeLessThanEqual | where x.age <= ?1 |
| GreaterThan | findByAgeGreaterThan | where x.age > ?1 |
| GreaterThanEqual | findByAgeGreaterThanEqual | where x.age >= ?1 |
| After | findByStartDateAfter | where x.startDate > ?1 |
| Before | findByStartDateBefore | where x.startDate < ?1 |
| IsNull, Null | findByAge(Is)Null | where x.age is null |
| IsNotNull, NotNull | findByAge(Is)NotNull | where x.age not null |
| Like | findByFirstnameLike | where x.firstname like ?1 |
| NotLike | findByFirstnameNotLike | where x.firstname not like ?1 |
| StartingWith | findByFirstnameStartingWith | where x.firstname like ?1% |
| EndingWith | findByFirstnameEndingWith | where x.firstname like %?1 |
| Containing | findByFirstnameContaining | where x.firstname like %?1% |
| OrderBy | findByAgeOrderByLastnameDesc | where x.age = ?1 order by x.lastname desc |
| Not | findByLastnameNot | where x.lastname <> ?1 |
| In | findByAgeIn(Collection<Age> ages) | where x.age in ?1 |
| NotIn | findByAgeNotIn(Collection<Age> ages) | where x.age not in ?1 |
| True | findByActiveTrue() | where x.active = true |
| False | findByActiveFalse() | where x.active = false |
| IgnoreCase | findByFirstnameIgnoreCase | where UPPER(x.firstname) = UPPER(?1) |

### JPA NamedQuery

```java
// 엔티티에 NamedQuery 정의
@Entity
@NamedQuery(
    name = "Member.findByName",
    query = "select m from Member m where m.name = :name"
)
public class Member {
    // ...
}

// 리포지토리에서 사용
public interface MemberRepository extends JpaRepository<Member, Long> {
    
    // NamedQuery 호출 (관례: 도메인클래스.메소드이름)
    List<Member> findByName(@Param("name") String name);
}
```

### @Query 어노테이션

#### 메소드에 JPQL 정의

```java
public interface MemberRepository extends JpaRepository<Member, Long> {
    
    @Query("select m from Member m where m.name = :name and m.age = :age")
    List<Member> findMembersByNameAndAge(@Param("name") String name, @Param("age") int age);
    
    // 위치 기반 파라미터도 가능
    @Query("select m from Member m where m.name = ?1 and m.age = ?2")
    List<Member> findMembersByNameAndAge(String name, int age);
    
    // 값 타입만 조회
    @Query("select m.name from Member m")
    List<String> findNameList();
    
    // DTO로 직접 조회
    @Query("select new study.datajpa.dto.MemberDto(m.id, m.name, t.name) " +
           "from Member m join m.team t")
    List<MemberDto> findMemberDto();
}
```

#### 네이티브 SQL 사용

```java
public interface MemberRepository extends JpaRepository<Member, Long> {
    
    @Query(value = "select * from member where name = ?1", nativeQuery = true)
    List<Member> findByNativeQuery(String name);
    
    // 네이티브 SQL + Projection
    @Query(value = "select m.member_id as id, m.name, t.name as teamName " +
                   "from member m left join team t",
           countQuery = "select count(*) from member",
           nativeQuery = true)
    Page<MemberProjection> findByNativeProjection(Pageable pageable);
}
```

### 반환 타입

스프링 데이터 JPA는 유연한 반환 타입을 지원한다.

```java
public interface MemberRepository extends JpaRepository<Member, Long> {
    
    // 컬렉션
    List<Member> findByName(String name);
    Set<Member> findSetByName(String name);
    
    // 단건
    Member findMemberByName(String name);
    Optional<Member> findOptionalByName(String name);
    
    // 스트림 (try-with-resources 사용 권장)
    @Query("select m from Member m")
    Stream<Member> findAllByCustom();
}

// 스트림 사용 예제
try (Stream<Member> stream = memberRepository.findAllByCustom()) {
    stream.map(Member::getName)
          .forEach(System.out::println);
}
```

**주의사항:**
- 단건 조회시 결과가 2개 이상이면 `NonUniqueResultException`
- 단건 조회시 결과가 없으면 `null` 반환 (Optional 사용 권장)

## 📄 페이징과 정렬

### Pageable 인터페이스 사용

```java
public interface MemberRepository extends JpaRepository<Member, Long> {
    
    // 페이징과 정렬 지원
    Page<Member> findByAge(int age, Pageable pageable);
    
    // Slice 반환 (다음 페이지 존재 여부만 확인)
    Slice<Member> findSliceByAge(int age, Pageable pageable);
    
    // List 반환 (카운트 쿼리 없음)
    List<Member> findListByAge(int age, Pageable pageable);
    
    // 카운트 쿼리 분리
    @Query(value = "select m from Member m left join m.team t",
           countQuery = "select count(m) from Member m")
    Page<Member> findMemberAllCountBy(Pageable pageable);
}
```

### 페이징 사용 예제

```java
@Test
public void pagingTest() {
    // given
    memberRepository.save(new Member("member1", 10));
    memberRepository.save(new Member("member2", 10));
    memberRepository.save(new Member("member3", 10));
    memberRepository.save(new Member("member4", 10));
    memberRepository.save(new Member("member5", 10));
    
    int age = 10;
    
    // PageRequest.of(페이지, 사이즈, 정렬조건)
    PageRequest pageRequest = PageRequest.of(0, 3, Sort.by(Sort.Direction.DESC, "name"));
    
    // when
    Page<Member> page = memberRepository.findByAge(age, pageRequest);
    
    // then
    List<Member> content = page.getContent(); // 조회된 데이터
    long totalElements = page.getTotalElements(); // 전체 데이터 수
    
    assertThat(content.size()).isEqualTo(3); // 조회된 데이터 수
    assertThat(totalElements).isEqualTo(5); // 전체 데이터 수
    assertThat(page.getNumber()).isEqualTo(0); // 페이지 번호
    assertThat(page.getTotalPages()).isEqualTo(2); // 전체 페이지 번호
    assertThat(page.isFirst()).isTrue(); // 첫번째 항목인가?
    assertThat(page.hasNext()).isTrue(); // 다음 페이지가 있는가?
}
```

### DTO 변환

```java
// Page 내용을 DTO로 변환
Page<Member> page = memberRepository.findByAge(age, pageRequest);
Page<MemberDto> dtoPage = page.map(member -> new MemberDto(member.getId(), member.getName()));
```

### Top, First 사용

```java
public interface MemberRepository extends JpaRepository<Member, Long> {
    
    List<Member> findTop3ByAge(int age);
    List<Member> findFirst10ByAge(int age);
    
    // 정렬과 함께 사용
    List<Member> findTop3ByAgeOrderByNameDesc(int age);
}
```

## 🔧 벌크성 수정 쿼리

### @Modifying 어노테이션

```java
public interface MemberRepository extends JpaRepository<Member, Long> {
    
    @Modifying
    @Query("update Member m set m.age = m.age + 1 where m.age >= :age")
    int bulkAgePlus(@Param("age") int age);
    
    // clearAutomatically 옵션으로 영속성 컨텍스트 자동 클리어
    @Modifying(clearAutomatically = true)
    @Query("delete from Member m where m.age > :age")
    int deleteBulkByAge(@Param("age") int age);
}
```

### 벌크 연산 주의사항

```java
@Test
public void bulkUpdateTest() {
    // given
    memberRepository.save(new Member("member1", 10));
    memberRepository.save(new Member("member2", 19));
    memberRepository.save(new Member("member3", 20));
    memberRepository.save(new Member("member4", 21));
    memberRepository.save(new Member("member5", 40));
    
    // when
    int resultCount = memberRepository.bulkAgePlus(20);
    
    // 벌크 연산 후 영속성 컨텍스트 클리어 필요
    em.flush();
    em.clear();
    
    // 또는 clearAutomatically = true 옵션 사용
    
    // then
    List<Member> result = memberRepository.findByName("member5");
    Member member5 = result.get(0);
    assertThat(member5.getAge()).isEqualTo(41); // 40 -> 41
}
```

**주의사항:**
- 벌크 연산은 영속성 컨텍스트를 무시하고 실행된다
- 벌크 연산 직후에는 영속성 컨텍스트를 초기화하는 것이 안전하다

## 🔗 @EntityGraph

### 연관된 엔티티들을 SQL 한번에 조회하는 방법

```java
// Member 엔티티
@Entity
@NamedEntityGraph(name = "Member.all", attributeNodes = @NamedAttributeNode("team"))
public class Member {
    
    @Id @GeneratedValue
    private Long id;
    
    private String name;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "team_id")
    private Team team;
    
    // ...
}

// Repository에서 사용
public interface MemberRepository extends JpaRepository<Member, Long> {
    
    // 공통 메소드 오버라이드
    @Override
    @EntityGraph(attributePaths = {"team"})
    List<Member> findAll();
    
    // JPQL + EntityGraph
    @EntityGraph(attributePaths = {"team"})
    @Query("select m from Member m")
    List<Member> findMemberEntityGraph();
    
    // 메소드 이름으로 쿼리에서 특별히 EntityGraph 설정
    @EntityGraph(attributePaths = {"team"})
    List<Member> findByName(String name);
    
    // NamedEntityGraph 사용
    @EntityGraph("Member.all")
    List<Member> findEntityGraphByName(String name);
}
```

### EntityGraph 타입

```java
public interface MemberRepository extends JpaRepository<Member, Long> {
    
    // FETCH 조인 사용 (기본값)
    @EntityGraph(attributePaths = {"team"}, type = EntityGraph.EntityGraphType.FETCH)
    List<Member> findFetchByName(String name);
    
    // LOAD: EntityGraph에 명시된 attribute는 EAGER로, 나머지는 LAZY로
    @EntityGraph(attributePaths = {"team"}, type = EntityGraph.EntityGraphType.LOAD)  
    List<Member> findLoadByName(String name);
}
```

## 🎯 사용자 정의 리포지토리

### 사용자 정의 리포지토리가 필요한 경우

- JPA 직접 사용 (EntityManager)
- 스프링 JDBC Template 사용
- MyBatis 사용
- 데이터베이스 커넥션 직접 사용
- Querydsl 사용

### 사용자 정의 리포지토리 구현

#### 1. 사용자 정의 인터페이스 작성

```java
public interface MemberRepositoryCustom {
    List<Member> findMemberCustom();
}
```

#### 2. 사용자 정의 인터페이스 구현

```java
@RequiredArgsConstructor
public class MemberRepositoryImpl implements MemberRepositoryCustom {
    
    private final EntityManager em;
    
    @Override
    public List<Member> findMemberCustom() {
        return em.createQuery("select m from Member m")
                .getResultList();
    }
}
```

#### 3. 사용자 정의 인터페이스 상속

```java
public interface MemberRepository 
        extends JpaRepository<Member, Long>, MemberRepositoryCustom {
    
    List<Member> findByName(String name);
}
```

#### 4. 사용

```java
@Test
public void customRepositoryTest() {
    List<Member> result = memberRepository.findMemberCustom();
}
```

### 사용자 정의 구현 클래스 이름 규칙

스프링 데이터 JPA는 사용자 정의 구현 클래스를 찾을 때 다음 규칙을 사용한다:

1. **리포지토리 인터페이스 이름 + Impl** (기본값)
   - `MemberRepository` → `MemberRepositoryImpl`

2. **사용자 정의 인터페이스 이름 + Impl**
   - `MemberRepositoryCustom` → `MemberRepositoryCustomImpl`

3. **JavaConfig 설정으로 변경 가능**

```java
@EnableJpaRepositories(repositoryImplementationPostfix = "Impl")
```

### 실무 팁

```java
// 항상 사용자 정의 리포지토리가 필요한 것은 아니다
// 단순한 경우 별도의 클래스로 분리하는 것도 좋은 방법

@Repository
@RequiredArgsConstructor
public class MemberQueryRepository {
    
    private final EntityManager em;
    
    public List<Member> findAllMembers() {
        return em.createQuery("select m from Member m")
                .getResultList();
    }
}

@Service
@RequiredArgsConstructor
public class MemberService {
    
    private final MemberRepository memberRepository; // 스프링 데이터 JPA
    private final MemberQueryRepository memberQueryRepository; // 사용자 정의
    
    public void businessLogic() {
        // 각각의 용도에 맞게 사용
        memberRepository.save(new Member());
        List<Member> members = memberQueryRepository.findAllMembers();
    }
}
```

## 🧪 Auditing

### 엔티티를 생성, 변경할 때 변경한 사람과 시간을 추적

#### 순수 JPA 사용

```java
@MappedSuperclass
@Getter
public class JpaBaseEntity {
    
    @Column(updatable = false)
    private LocalDateTime createdDate;
    private LocalDateTime updatedDate;
    
    @PrePersist
    public void prePersist() {
        LocalDateTime now = LocalDateTime.now();
        createdDate = now;
        updatedDate = now;
    }
    
    @PreUpdate
    public void preUpdate() {
        updatedDate = LocalDateTime.now();
    }
}

@Entity
public class Member extends JpaBaseEntity {
    // ...
}
```

#### 스프링 데이터 JPA 사용

```java
// 1. @EnableJpaAuditing 스프링 부트 설정 클래스에 적용
@EnableJpaAuditing
@SpringBootApplication
public class DataJpaApplication {
    public static void main(String[] args) {
        SpringApplication.run(DataJpaApplication.class, args);
    }
}

// 2. 엔티티에 적용
@EntityListeners(AuditingEntityListener.class)
@MappedSuperclass
@Getter
public class BaseEntity {
    
    @CreatedDate
    @Column(updatable = false)
    private LocalDateTime createdDate;
    
    @LastModifiedDate
    private LocalDateTime lastModifiedDate;
    
    @CreatedBy
    @Column(updatable = false)
    private String createdBy;
    
    @LastModifiedBy
    private String lastModifiedBy;
}

// 3. 등록자, 수정자를 처리해주는 AuditorAware 스프링 빈 등록
@Bean
public AuditorAware<String> auditorProvider() {
    return () -> Optional.of(UUID.randomUUID().toString());
    // 실무에서는 세션 정보나 스프링 시큐리티 로그인 정보에서 ID를 받음
}

// 4. 실제 엔티티에 적용
@Entity
public class Member extends BaseEntity {
    // ...
}
```

#### 시간만 필요한 경우

```java
@EntityListeners(AuditingEntityListener.class)
@MappedSuperclass
@Getter
public class BaseTimeEntity {
    
    @CreatedDate
    @Column(updatable = false)
    private LocalDateTime createdDate;
    
    @LastModifiedDate
    private LocalDateTime lastModifiedDate;
}

@Getter
public class BaseEntity extends BaseTimeEntity {
    
    @CreatedBy
    @Column(updatable = false)
    private String createdBy;
    
    @LastModifiedBy
    private String lastModifiedBy;
}
```

### 실무 팁

```java
// 실무에서는 대부분의 엔티티에 등록시간, 수정시간이 필요하다
// 하지만 등록자, 수정자는 필요 없을 수도 있다
// Base 타입을 분리하고, 원하는 타입을 선택해서 상속받자

@EntityListeners(AuditingEntityListener.class)
@MappedSuperclass
public class BaseTimeEntity {
    @CreatedDate
    @Column(updatable = false)
    private LocalDateTime createdDate;
    
    @LastModifiedDate
    private LocalDateTime lastModifiedDate;
}

public class BaseEntity extends BaseTimeEntity {
    @CreatedBy
    @Column(updatable = false)
    private String createdBy;
    
    @LastModifiedBy
    private String lastModifiedBy;
}
```

## 🚀 Web 확장 - 도메인 클래스 컨버터

### 도메인 클래스 컨버터 사용 전

```java
@RestController
@RequiredArgsConstructor
public class MemberController {
    
    private final MemberRepository memberRepository;
    
    @GetMapping("/members/{id}")
    public String findMember(@PathVariable("id") Long id) {
        Member member = memberRepository.findById(id).get();
        return member.getName();
    }
}
```

### 도메인 클래스 컨버터 사용 후

```java
@RestController
@RequiredArgsConstructor
public class MemberController {
    
    private final MemberRepository memberRepository;
    
    @GetMapping("/members/{id}")
    public String findMember(@PathVariable("id") Member member) {
        return member.getName();
    }
}
```

**주의사항:**
- 도메인 클래스 컨버터로 엔티티를 파라미터로 받으면, 이 엔티티는 **단순 조회용**으로만 사용해야 한다
- 트랜잭션이 없는 범위에서 엔티티를 조회했으므로, 엔티티를 변경해도 DB에 반영되지 않는다

## 📄 Web 확장 - 페이징과 정렬

### 페이징과 정렬 예제

```java
@RestController
@RequiredArgsConstructor
public class MemberController {
    
    private final MemberRepository memberRepository;
    
    @GetMapping("/members")
    public Page<Member> list(Pageable pageable) {
        return memberRepository.findAll(pageable);
    }
    
    @GetMapping("/members2")
    public Page<MemberDto> list2(Pageable pageable) {
        return memberRepository.findAll(pageable)
                .map(MemberDto::new);
    }
}
```

### 요청 파라미터

```
/members?page=0&size=3&sort=id,desc&sort=name,desc
```

- `page`: 현재 페이지, **0부터 시작**한다
- `size`: 한 페이지에 노출할 데이터 건수
- `sort`: 정렬 조건을 정의한다

### 기본값 변경

#### 글로벌 설정

```yaml
# application.yml
spring:
  data:
    web:
      pageable:
        default-page-size: 20 # 기본 페이지 사이즈
        max-page-size: 2000 # 최대 페이지 사이즈
```

#### 개별 설정

```java
@GetMapping("/members")
public Page<Member> list(@PageableDefault(size = 5, sort = "name") Pageable pageable) {
    return memberRepository.findAll(pageable);
}
```

### 접두사

페이지 정보가 둘 이상이면 접두사로 구분

```java
@GetMapping("/members")
public String list(
    @Qualifier("member") Pageable memberPageable,
    @Qualifier("order") Pageable orderPageable) {
    // ...
}
```

```
/members?member_page=0&order_page=1
```

### Page를 1부터 시작하기

#### 1. Pageable, Page를 파라미터와 응답 값으로 사용하지 않고, 직접 클래스를 만들어서 처리

#### 2. spring.data.web.pageable.one-indexed-parameters를 true로 설정

```yaml
spring:
  data:
    web:
      pageable:
        one-indexed-parameters: true
```

**주의:**
- 이 방법은 web에서 page 파라미터를 -1 처리할 뿐이다
- 따라서 응답값인 Page에 모든 page 정보는 0부터 시작한다

---

## 📝 정리

### 스프링 데이터 JPA 핵심

1. **스프링 데이터 JPA는 JPA를 편리하게 사용하도록 지원하는 프로젝트**
2. **CRUD를 처리하는 공통 인터페이스 제공**
3. **repository = 인터페이스만 작성하면 런타임시에 스프링 데이터 JPA가 구현 객체를 동적으로 생성해서 주입**

### 공통 인터페이스 기능

1. **JpaRepository 인터페이스를 통해서 기본적인 CRUD 기능 제공**
2. **findById(), save(), delete() 등의 기본 메소드 제공**
3. **공통화 가능한 기능들을 인터페이스를 통해 제공**

### 쿼리 메소드 기능

1. **메소드 이름으로 쿼리 생성**
2. **메소드 이름으로 JPA NamedQuery 호출**
3. **@Query 어노테이션을 사용해서 Repository 인터페이스에 쿼리 직접 정의**

### 확장 기능

1. **사용자 정의 리포지토리 구현**
2. **Auditing**
3. **Web 확장 - 도메인 클래스 컨버터**
4. **Web 확장 - 페이징과 정렬**

### 실무 활용 팁

1. **스프링 데이터 JPA와 Querydsl을 함께 사용하면 개발 생산성과 동시에 복잡한 쿼리 기능을 구현할 수 있다**
2. **복잡한 정적 쿼리는 Querydsl, 동적 쿼리는 Querydsl이나 MyBatis를 권장**
3. **단순한 경우 스프링 데이터 JPA, 복잡한 경우 Querydsl과 조합**

---

[← 이전: 웹 애플리케이션과 영속성 관리](./08-web-application-persistence.md) | [목차로 돌아가기](./README.md) | [다음: 성능 최적화 →](./10-performance-optimization.md)