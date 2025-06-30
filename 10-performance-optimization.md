# 10. 성능 최적화

## 🚨 N+1 문제

### N+1 문제란?

연관 관계가 설정된 엔티티를 조회할 경우에 조회된 데이터 갯수(n) 만큼 연관관계의 조회 쿼리가 추가로 발생하여 데이터를 읽어오게 되는 현상이다.

#### 즉시 로딩에서 발생하는 N+1 문제

```java
@Entity
public class Member {
    @Id @GeneratedValue
    private Long id;
    
    private String name;
    
    @ManyToOne(fetch = FetchType.EAGER)
    @JoinColumn(name = "team_id")
    private Team team;
    
    // ...
}

@Entity
public class Team {
    @Id @GeneratedValue
    private Long id;
    
    private String name;
    
    @OneToMany(mappedBy = "team", fetch = FetchType.EAGER)
    private List<Member> members = new ArrayList<>();
    
    // ...
}

// 문제 발생 코드
List<Member> members = em.createQuery("select m from Member m", Member.class)
                        .getResultList();

// 실행되는 SQL
// 1. select * from Member
// 2. select * from Team where id = ? (첫 번째 회원의 팀)
// 3. select * from Team where id = ? (두 번째 회원의 팀)
// 4. select * from Team where id = ? (세 번째 회원의 팀)
// ...
// 회원이 N명이면 팀을 조회하는 쿼리가 N번 추가 실행!
```

#### 지연 로딩에서 발생하는 N+1 문제

```java
@Entity
public class Member {
    @Id @GeneratedValue
    private Long id;
    
    private String name;
    
    @ManyToOne(fetch = FetchType.LAZY) // 지연 로딩 설정
    @JoinColumn(name = "team_id")
    private Team team;
    
    // ...
}

// 문제 발생 코드
List<Member> members = em.createQuery("select m from Member m", Member.class)
                        .getResultList();

// 각 회원의 팀 정보에 접근할 때 쿼리 발생
for (Member member : members) {
    System.out.println(member.getTeam().getName()); // 지연 로딩으로 쿼리 실행
}

// 실행되는 SQL
// 1. select * from Member
// 2. select * from Team where id = ? (첫 번째 회원의 팀)
// 3. select * from Team where id = ? (두 번째 회원의 팀)
// 4. select * from Team where id = ? (세 번째 회원의 팀)
// ...
```

### N+1 문제 해결 방법

#### 1. 페치 조인(Fetch Join) 사용

```java
// JPQL 페치 조인
List<Member> members = em.createQuery(
    "select m from Member m join fetch m.team", Member.class)
    .getResultList();

// 실행되는 SQL (한 번의 쿼리로 해결)
// select m.*, t.* from Member m inner join Team t on m.team_id = t.id

// 스프링 데이터 JPA에서 사용
public interface MemberRepository extends JpaRepository<Member, Long> {
    
    @Query("select m from Member m join fetch m.team")
    List<Member> findMemberFetchJoin();
}
```

#### 2. @EntityGraph 사용

```java
public interface MemberRepository extends JpaRepository<Member, Long> {
    
    // 공통 메소드 오버라이드
    @Override
    @EntityGraph(attributePaths = {"team"})
    List<Member> findAll();
    
    // 메소드 이름으로 쿼리
    @EntityGraph(attributePaths = {"team"})
    List<Member> findEntityGraphByName(String name);
    
    // JPQL + EntityGraph
    @EntityGraph(attributePaths = {"team"})
    @Query("select m from Member m")
    List<Member> findMemberEntityGraph();
}
```

#### 3. @BatchSize 사용

```java
@Entity
public class Member {
    @Id @GeneratedValue
    private Long id;
    
    private String name;
    
    @BatchSize(size = 100) // 한 번에 100개씩 IN 쿼리로 조회
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "team_id")
    private Team team;
    
    // ...
}

// 또는 연관 컬렉션에 적용
@Entity
public class Team {
    @Id @GeneratedValue
    private Long id;
    
    private String name;
    
    @BatchSize(size = 100)
    @OneToMany(mappedBy = "team")
    private List<Member> members = new ArrayList<>();
    
    // ...
}

// 실행되는 SQL
// 1. select * from Member
// 2. select * from Team where id in (?, ?, ?, ...) -- 최대 100개씩
```

#### 4. 글로벌 BatchSize 설정

```yaml
# application.yml
spring:
  jpa:
    properties:
      hibernate:
        default_batch_fetch_size: 100
```

#### 5. @Fetch(FetchMode.SUBSELECT) 사용

```java
@Entity
public class Team {
    @Id @GeneratedValue
    private Long id;
    
    private String name;
    
    @Fetch(FetchMode.SUBSELECT)
    @OneToMany(mappedBy = "team")
    private List<Member> members = new ArrayList<>();
    
    // ...
}

// 실행되는 SQL
// 1. select * from Team
// 2. select * from Member where team_id in (
//        select team_id from Team where ...
//    )
```

### N+1 문제 해결 방법 비교

| 방법 | 장점 | 단점 |
|------|------|------|
| 페치 조인 | - 한 번의 쿼리로 해결<br>- 최적의 성능 | - 페이징 불가능<br>- 컬렉션 페치 조인시 중복 데이터<br>- 2개 이상의 컬렉션 페치 조인 불가능 |
| @EntityGraph | - 페치 조인의 간편 버전<br>- 어노테이션으로 쉽게 적용 | - 페치 조인과 동일한 한계 |
| @BatchSize | - 페이징 가능<br>- 컬렉션도 최적화<br>- 여러 테이블에 적용 가능 | - 쿼리 수 감소하지만 완전히 1개로 줄이지는 못함 |
| @Fetch(SUBSELECT) | - 2번의 쿼리로 해결 | - 즉시 로딩으로 변경됨<br>- 메모리 사용량 증가 |

## 🔗 페치 조인(Fetch Join)

### 페치 조인의 특징과 한계

#### 페치 조인 대상에는 별칭을 줄 수 없다

```java
// 잘못된 예 - 하이버네이트에서는 가능하지만 JPA 표준에서는 지원하지 않음
String jpql = "select m from Member m join fetch m.team t where t.name = 'teamA'";

// 올바른 예
String jpql = "select m from Member m join fetch m.team where m.team.name = 'teamA'";
```

#### 둘 이상의 컬렉션은 페치 조인할 수 없다

```java
// 잘못된 예 - 카테시안 곱으로 인한 데이터 폭증
String jpql = "select t from Team t join fetch t.members join fetch t.orders";

// 해결 방법 1: 컬렉션 하나만 페치 조인
String jpql = "select t from Team t join fetch t.members";

// 해결 방법 2: @BatchSize 사용
@Entity
public class Team {
    @BatchSize(size = 100)
    @OneToMany(mappedBy = "team")
    private List<Member> members = new ArrayList<>();
    
    @BatchSize(size = 100)
    @OneToMany(mappedBy = "team")
    private List<Order> orders = new ArrayList<>();
}
```

#### 컬렉션을 페치 조인하면 페이징 API를 사용할 수 없다

```java
// 문제가 되는 코드
@Query("select t from Team t join fetch t.members")
Page<Team> findTeamWithMembers(Pageable pageable); // 경고 발생!

// 해결 방법 1: ToOne 관계는 페치 조인, 컬렉션은 @BatchSize
@Query("select m from Member m join fetch m.team")
Page<Member> findMemberWithTeam(Pageable pageable); // OK

// 해결 방법 2: @BatchSize 사용
@Entity
public class Team {
    @BatchSize(size = 100)
    @OneToMany(mappedBy = "team")
    private List<Member> members = new ArrayList<>();
}

Page<Team> teams = teamRepository.findAll(pageable); // 페이징 가능
```

### 페치 조인 vs 일반 조인

```java
// 일반 조인 - 연관된 엔티티를 함께 조회하지 않음
String jpql = "select m from Member m join m.team t where t.name = 'teamA'";
List<Member> members = em.createQuery(jpql, Member.class).getResultList();

for (Member member : members) {
    System.out.println(member.getTeam().getName()); // 지연 로딩으로 추가 쿼리 실행
}

// 페치 조인 - 연관된 엔티티까지 함께 조회
String jpql = "select m from Member m join fetch m.team t where t.name = 'teamA'";
List<Member> members = em.createQuery(jpql, Member.class).getResultList();

for (Member member : members) {
    System.out.println(member.getTeam().getName()); // 추가 쿼리 실행 안함
}
```

### 컬렉션 페치 조인시 중복 제거

```java
// 문제: 컬렉션 페치 조인시 중복된 엔티티가 조회될 수 있음
String jpql = "select t from Team t join fetch t.members where t.name = 'teamA'";
List<Team> teams = em.createQuery(jpql, Team.class).getResultList();
System.out.println("teams = " + teams.size()); // 중복된 팀이 조회될 수 있음

// 해결 방법 1: DISTINCT 사용
String jpql = "select distinct t from Team t join fetch t.members where t.name = 'teamA'";
List<Team> teams = em.createQuery(jpql, Team.class).getResultList();

// 해결 방법 2: LinkedHashSet 사용
String jpql = "select t from Team t join fetch t.members where t.name = 'teamA'";
List<Team> teams = em.createQuery(jpql, Team.class).getResultList();
Set<Team> uniqueTeams = new LinkedHashSet<>(teams);
```

## 📊 엔티티 그래프

### @NamedEntityGraph

```java
@Entity
@NamedEntityGraph(
    name = "Member.all",
    attributeNodes = {
        @NamedAttributeNode("team"),
        @NamedAttributeNode(value = "orders", subgraph = "orders")
    },
    subgraphs = @NamedSubgraph(name = "orders", attributeNodes = @NamedAttributeNode("product"))
)
public class Member {
    @Id @GeneratedValue
    private Long id;
    
    private String name;
    
    @ManyToOne(fetch = FetchType.LAZY)
    private Team team;
    
    @OneToMany(mappedBy = "member")
    private List<Order> orders = new ArrayList<>();
}

// 사용
public interface MemberRepository extends JpaRepository<Member, Long> {
    
    @EntityGraph("Member.all")
    @Query("select m from Member m")
    List<Member> findMemberEntityGraph();
}
```

### 동적 엔티티 그래프

```java
// EntityManager 사용
Map<String, Object> properties = new HashMap<>();
properties.put("javax.persistence.fetchgraph", 
    em.getEntityGraph("Member.all"));

Member member = em.find(Member.class, id, properties);

// 또는
EntityGraph<Member> graph = em.createEntityGraph(Member.class);
graph.addAttributeNodes("team");
Subgraph<Order> orderGraph = graph.addSubgraph("orders");
orderGraph.addAttributeNodes("product");

TypedQuery<Member> query = em.createQuery("select m from Member m", Member.class);
query.setHint("javax.persistence.fetchgraph", graph);
List<Member> members = query.getResultList();
```

### 페치 그래프 vs 로드 그래프

```java
// FETCH 그래프 (기본값)
// 명시된 attribute는 EAGER로 가져오고, 명시되지 않은 attribute는 LAZY로 가져옴
@EntityGraph(attributePaths = {"team"}, type = EntityGraph.EntityGraphType.FETCH)
List<Member> findByName(String name);

// LOAD 그래프
// 명시된 attribute는 EAGER로 가져오고, 명시되지 않은 attribute는 기본 fetch 전략 따름
@EntityGraph(attributePaths = {"team"}, type = EntityGraph.EntityGraphType.LOAD)
List<Member> findByName(String name);
```

## 🏭 배치 처리

### 대용량 데이터 수정

#### 잘못된 예 - 메모리 부족 위험

```java
// 10만건의 데이터를 모두 메모리에 올려서 수정
List<Member> members = em.createQuery("select m from Member m", Member.class)
                        .getResultList(); // OutOfMemoryError 위험!

for (Member member : members) {
    member.setAge(member.getAge() + 1);
}
```

#### 해결 방법 1: 페이징 배치 처리

```java
int pageSize = 100;
int totalCount = getTotalCount();

for (int i = 0; i < totalCount; i += pageSize) {
    List<Member> members = em.createQuery("select m from Member m order by m.id", Member.class)
                            .setFirstResult(i)
                            .setMaxResults(pageSize)
                            .getResultList();
    
    for (Member member : members) {
        member.setAge(member.getAge() + 1);
    }
    
    em.flush();
    em.clear(); // 영속성 컨텍스트 초기화
}
```

#### 해결 방법 2: 벌크 연산 사용

```java
// JPA 벌크 연산
int resultCount = em.createQuery("update Member m set m.age = m.age + 1")
                   .executeUpdate();

em.clear(); // 영속성 컨텍스트 초기화

// 스프링 데이터 JPA
public interface MemberRepository extends JpaRepository<Member, Long> {
    
    @Modifying(clearAutomatically = true)
    @Query("update Member m set m.age = m.age + 1")
    int bulkAgePlus();
}
```

#### 해결 방법 3: 하이버네이트의 무상태 세션 사용

```java
SessionFactory sessionFactory = entityManagerFactory.unwrap(SessionFactory.class);
StatelessSession session = sessionFactory.openStatelessSession();
Transaction tx = session.beginTransaction();

ScrollableResults scroll = session.createQuery("select m from Member m")
                                 .scroll(ScrollMode.FORWARD_ONLY);

while (scroll.next()) {
    Member member = (Member) scroll.get(0);
    member.setAge(member.getAge() + 1);
    session.update(member);
}

tx.commit();
session.close();
```

### 대용량 데이터 조회

#### JPA 커서 기반 조회

```java
SessionFactory sessionFactory = entityManagerFactory.unwrap(SessionFactory.class);
Session session = sessionFactory.openSession();

ScrollableResults scroll = session.createQuery("select m from Member m")
                                 .setReadOnly(true)
                                 .setFetchSize(100)
                                 .scroll(ScrollMode.FORWARD_ONLY);

while (scroll.next()) {
    Member member = (Member) scroll.get(0);
    // 처리 로직
    processOne(member);
}

scroll.close();
session.close();
```

#### 스트림 API 사용

```java
// 스프링 데이터 JPA
public interface MemberRepository extends JpaRepository<Member, Long> {
    
    @Query("select m from Member m")
    Stream<Member> findAllByCustom();
}

// 사용 (try-with-resources 필수)
@Transactional(readOnly = true)
public void processAllMembers() {
    try (Stream<Member> stream = memberRepository.findAllByCustom()) {
        stream.forEach(this::processOne);
    }
}
```

## 🔧 SQL 힌트

### 하이버네이트 쿼리 힌트

```java
public interface MemberRepository extends JpaRepository<Member, Long> {
    
    // 읽기 전용 힌트
    @QueryHints(value = @QueryHint(name = "org.hibernate.readOnly", value = "true"))
    @Query("select m from Member m")
    List<Member> findReadOnlyMembers();
    
    // 페치 크기 힌트
    @QueryHints(value = @QueryHint(name = "org.hibernate.fetchSize", value = "100"))
    @Query("select m from Member m")
    List<Member> findMembersWithFetchSize();
    
    // 캐시 힌트
    @QueryHints(value = {
        @QueryHint(name = "org.hibernate.cacheable", value = "true"),
        @QueryHint(name = "org.hibernate.cacheRegion", value = "member")
    })
    @Query("select m from Member m where m.id = :id")
    Member findCacheableMember(@Param("id") Long id);
}
```

### 네이티브 SQL 힌트

```java
public interface MemberRepository extends JpaRepository<Member, Long> {
    
    // 오라클 힌트 예제
    @Query(value = "select /*+ INDEX(m, idx_member_name) */ * from member m where m.name = ?1", 
           nativeQuery = true)
    List<Member> findByNameWithHint(String name);
    
    // MySQL 힌트 예제
    @Query(value = "select * from member m use index (idx_member_name) where m.name = ?1", 
           nativeQuery = true)
    List<Member> findByNameWithMySQLHint(String name);
}
```

## 💾 쓰기 지연과 성능 최적화

### 트랜잭션을 지원하는 쓰기 지연

```java
@Entity
public class Member {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String name;
    private int age;
}

// 배치 삽입 최적화
@Transactional
public void saveMembersInBatch() {
    for (int i = 0; i < 100; i++) {
        Member member = new Member("member" + i, 20 + i);
        memberRepository.save(member);
        
        // 20개씩 배치로 처리
        if (i % 20 == 0) {
            em.flush();
            em.clear();
        }
    }
}
```

### JDBC 배치 설정

```yaml
# application.yml
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          batch_size: 20 # 배치 크기
          batch_versioned_data: true # 버전 관리 엔티티도 배치 처리
        order_inserts: true # INSERT 순서 최적화
        order_updates: true # UPDATE 순서 최적화
```

### IDENTITY 전략과 배치 처리

```java
// IDENTITY 전략은 쓰기 지연이 동작하지 않음
@Entity
public class Member {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) // 배치 처리 불가
    private Long id;
}

// 해결 방법: SEQUENCE 전략 사용
@Entity
public class Member {
    @Id @GeneratedValue(strategy = GenerationType.SEQUENCE,
                        generator = "member_seq_generator")
    @SequenceGenerator(name = "member_seq_generator",
                      sequenceName = "member_seq",
                      allocationSize = 50) // 성능 최적화
    private Long id;
}
```

## 🔄 지연 로딩 최적화

### 프록시 초기화 최적화

```java
// 문제: 여러 프록시를 개별적으로 초기화
@Transactional(readOnly = true)
public void printMemberAndTeam() {
    List<Member> members = memberRepository.findAll();
    
    for (Member member : members) {
        System.out.println(member.getName());
        System.out.println(member.getTeam().getName()); // 각각 쿼리 실행
    }
}

// 해결 방법 1: 배치 페치 사이즈
@Entity
public class Member {
    @BatchSize(size = 100)
    @ManyToOne(fetch = FetchType.LAZY)
    private Team team;
}

// 해결 방법 2: 페치 조인
@Transactional(readOnly = true)
public void printMemberAndTeamOptimized() {
    List<Member> members = memberRepository.findMemberFetchJoin();
    
    for (Member member : members) {
        System.out.println(member.getName());
        System.out.println(member.getTeam().getName()); // 추가 쿼리 없음
    }
}
```

### DTO 직접 조회로 최적화

```java
// 필요한 데이터만 조회하는 DTO
@Data
@AllArgsConstructor
public class MemberTeamDto {
    private Long memberId;
    private String memberName;
    private String teamName;
}

public interface MemberRepository extends JpaRepository<Member, Long> {
    
    // 필요한 필드만 조회
    @Query("select new com.example.dto.MemberTeamDto(m.id, m.name, t.name) " +
           "from Member m join m.team t")
    List<MemberTeamDto> findMemberTeamDto();
}

// 컬렉션 조회 최적화
@Query("select new com.example.dto.TeamDto(t.id, t.name) from Team t")
List<TeamDto> findTeamDto();

@Query("select new com.example.dto.MemberDto(m.id, m.name, m.teamId) " +
       "from Member m where m.team.id in :teamIds")
List<MemberDto> findMemberDtoByTeamIds(@Param("teamIds") List<Long> teamIds);
```

## 📈 성능 모니터링

### JPA 통계 정보

```yaml
# application.yml
spring:
  jpa:
    properties:
      hibernate:
        generate_statistics: true # 통계 정보 수집
        session:
          events:
            log:
              LOG_QUERIES_SLOWER_THAN_MS: 1000 # 1초 이상 걸리는 쿼리 로깅
logging:
  level:
    org.hibernate.stat: DEBUG # 통계 로그 출력
    org.hibernate.SQL: DEBUG # SQL 로그 출력
    org.hibernate.type.descriptor.sql.BasicBinder: TRACE # 파라미터 로그
```

### P6Spy를 활용한 쿼리 로깅

```groovy
// build.gradle
implementation 'com.github.gavlyukovskiy:p6spy-spring-boot-starter:1.8.1'
```

```yaml
# application.yml
decorator:
  datasource:
    p6spy:
      enable-logging: true
      multiline: true
      logging: slf4j
```

### 성능 측정 코드

```java
@Component
@Slf4j
public class PerformanceMonitor {
    
    public void measureQueryPerformance() {
        Statistics stats = entityManagerFactory.unwrap(SessionFactory.class)
                                             .getStatistics();
        
        stats.setStatisticsEnabled(true);
        
        // 비즈니스 로직 실행
        businessLogic();
        
        log.info("쿼리 실행 횟수: {}", stats.getQueryExecutionCount());
        log.info("쿼리 실행 시간: {}ms", stats.getQueryExecutionMaxTime());
        log.info("2차 캐시 히트율: {}%", 
                stats.getSecondLevelCacheHitCount() * 100.0 / 
                stats.getSecondLevelCacheRequestCount());
        
        stats.clear();
    }
}
```

## ⚡ 캐싱 전략

### 1차 캐시 최적화

```java
// 영속성 컨텍스트 범위에서 동작하는 1차 캐시
@Transactional
public void firstLevelCacheExample() {
    Member member1 = memberRepository.findById(1L).get(); // DB 조회
    Member member2 = memberRepository.findById(1L).get(); // 1차 캐시에서 조회
    
    assertThat(member1 == member2).isTrue(); // 동일한 인스턴스
}
```

### 2차 캐시 설정

```java
// 엔티티 레벨 캐시
@Entity
@Cacheable
@org.hibernate.annotations.Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Member {
    // ...
}

// 컬렉션 캐시
@Entity
public class Team {
    @OneToMany(mappedBy = "team")
    @org.hibernate.annotations.Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
    private List<Member> members = new ArrayList<>();
}

// 쿼리 캐시
public interface MemberRepository extends JpaRepository<Member, Long> {
    
    @QueryHints(value = @QueryHint(name = "org.hibernate.cacheable", value = "true"))
    @Query("select m from Member m where m.age > :age")
    List<Member> findByAgeGreaterThan(@Param("age") int age);
}
```

```yaml
# application.yml - EhCache 설정
spring:
  jpa:
    properties:
      hibernate:
        cache:
          use_second_level_cache: true
          use_query_cache: true
          region:
            factory_class: org.hibernate.cache.jcache.JCacheRegionFactory
        javax:
          cache:
            provider: org.ehcache.jsr107.EhcacheCachingProvider
```

### Redis 캐시 활용

```java
@Service
@RequiredArgsConstructor
public class MemberCacheService {
    
    private final MemberRepository memberRepository;
    private final RedisTemplate<String, Object> redisTemplate;
    
    @Cacheable(value = "members", key = "#id")
    public Member findMember(Long id) {
        return memberRepository.findById(id)
                             .orElseThrow(() -> new EntityNotFoundException("Member not found"));
    }
    
    @CacheEvict(value = "members", key = "#member.id")
    public Member updateMember(Member member) {
        return memberRepository.save(member);
    }
    
    @CacheEvict(value = "members", allEntries = true)
    public void clearAllCache() {
        // 모든 캐시 삭제
    }
}
```

## 🎯 실무 최적화 패턴

### 조회 전용 기능 최적화

```java
// 조회 전용 서비스 분리
@Service
@Transactional(readOnly = true)
@RequiredArgsConstructor
public class MemberQueryService {
    
    private final EntityManager em;
    
    // DTO 직접 조회
    public List<MemberListDto> findMemberList() {
        return em.createQuery(
            "select new com.example.dto.MemberListDto(m.id, m.name, t.name) " +
            "from Member m join m.team t", 
            MemberListDto.class)
            .getResultList();
    }
    
    // 페이징 + 카운트 쿼리 최적화
    public Page<MemberDto> findMemberPage(Pageable pageable) {
        List<MemberDto> content = em.createQuery(
            "select new com.example.dto.MemberDto(m.id, m.name, t.name) " +
            "from Member m join m.team t", 
            MemberDto.class)
            .setFirstResult((int) pageable.getOffset())
            .setMaxResults(pageable.getPageSize())
            .getResultList();
        
        // 카운트 쿼리 최적화 (조인 제거)
        Long total = em.createQuery(
            "select count(m) from Member m", 
            Long.class)
            .getSingleResult();
        
        return new PageImpl<>(content, pageable, total);
    }
}
```

### 커맨드와 쿼리 분리 (CQRS 패턴)

```java
// 커맨드용 서비스
@Service
@Transactional
@RequiredArgsConstructor
public class MemberCommandService {
    
    private final MemberRepository memberRepository;
    
    public Member createMember(MemberCreateRequest request) {
        Member member = Member.builder()
                            .name(request.getName())
                            .age(request.getAge())
                            .build();
        
        return memberRepository.save(member);
    }
    
    public Member updateMember(Long id, MemberUpdateRequest request) {
        Member member = memberRepository.findById(id)
                                      .orElseThrow(() -> new EntityNotFoundException());
        
        member.update(request.getName(), request.getAge());
        return member;
    }
}

// 쿼리용 서비스
@Service
@Transactional(readOnly = true)
@RequiredArgsConstructor
public class MemberQueryService {
    
    private final EntityManager em;
    
    public MemberDetailDto findMemberDetail(Long id) {
        return em.createQuery(
            "select new com.example.dto.MemberDetailDto(" +
            "   m.id, m.name, m.age, t.name, " +
            "   (select count(o) from Order o where o.member = m)" +
            ") " +
            "from Member m " +
            "left join m.team t " +
            "where m.id = :id", 
            MemberDetailDto.class)
            .setParameter("id", id)
            .getSingleResult();
    }
}
```

### 동적 쿼리 최적화 전략

```java
@Repository
@RequiredArgsConstructor
public class MemberQueryRepository {
    
    private final EntityManager em;
    
    // 동적 쿼리 최적화
    public List<Member> findMembersOptimized(MemberSearchCondition condition) {
        StringBuilder jpql = new StringBuilder("select m from Member m");
        
        boolean hasWhere = false;
        
        if (StringUtils.hasText(condition.getName())) {
            jpql.append(hasWhere ? " and" : " where");
            jpql.append(" m.name like :name");
            hasWhere = true;
        }
        
        if (condition.getAgeGoe() != null) {
            jpql.append(hasWhere ? " and" : " where");
            jpql.append(" m.age >= :ageGoe");
            hasWhere = true;
        }
        
        if (condition.getAgeLoe() != null) {
            jpql.append(hasWhere ? " and" : " where");
            jpql.append(" m.age <= :ageLoe");
            hasWhere = true;
        }
        
        TypedQuery<Member> query = em.createQuery(jpql.toString(), Member.class);
        
        if (StringUtils.hasText(condition.getName())) {
            query.setParameter("name", "%" + condition.getName() + "%");
        }
        if (condition.getAgeGoe() != null) {
            query.setParameter("ageGoe", condition.getAgeGoe());
        }
        if (condition.getAgeLoe() != null) {
            query.setParameter("ageLoe", condition.getAgeLoe());
        }
        
        return query.getResultList();
    }
}
```

---

## 📝 정리

### N+1 문제 해결 전략

1. **페치 조인 우선 고려** - 한 번의 쿼리로 해결하지만 페이징 불가
2. **@BatchSize로 보완** - 페이징 가능하고 컬렉션 최적화에 유리
3. **@EntityGraph 활용** - 스프링 데이터 JPA에서 간편하게 사용

### 페이징 최적화

1. **ToOne 관계는 페치 조인** 사용해도 페이징에 영향 없음
2. **컬렉션은 @BatchSize** 로 최적화
3. **카운트 쿼리를 별도로 최적화** - 조인 제거하여 성능 향상

### 배치 처리

1. **대용량 데이터는 페이징으로 나누어 처리**
2. **영속성 컨텍스트 주기적 초기화** 필수
3. **벌크 연산 활용** - 대량 수정/삭제시 성능 향상

### 조회 성능 최적화

1. **DTO 직접 조회** - 필요한 데이터만 선택적으로 조회
2. **읽기 전용 트랜잭션** 사용
3. **캐시 적절히 활용** - 1차 캐시, 2차 캐시, 애플리케이션 캐시

### 실무 패턴

1. **CQRS 패턴** - 커맨드와 쿼리 분리
2. **조회 전용 서비스** 별도 구성
3. **성능 모니터링** 도구 활용

### 우선순위

1. **기본은 지연 로딩** 으로 설정
2. **성능 최적화가 필요한 곳에만 페치 조인** 적용
3. **@BatchSize는 글로벌 설정**으로 기본 적용
4. **복잡한 쿼리는 DTO 직접 조회** 고려

---

[← 이전: 스프링 데이터 JPA](./09-spring-data-jpa.md) | [목차로 돌아가기](./README.md)