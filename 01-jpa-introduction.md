# 1. JPA 소개 및 개념

## 📖 JPA란 무엇인가?

### JPA (Java Persistence API)

JPA는 **자바 플랫폼에서 관계형 데이터베이스와 객체 지향 프로그래밍 언어 간의 패러다임 불일치를 해결하기 위한 API**이다.

#### JPA의 정의

- **Java Persistence API**의 줄임말
- 자바 진영의 **ORM(Object-Relational Mapping) 기술 표준**
- 인터페이스의 모음 (구현체가 아님)
- **EJB 3.0**에서 처음 도입되어 현재 **JPA 2.2**까지 발전

#### JPA의 역사

```
EJB → Hibernate → JPA
```

1. **EJB 엔티티 빈** (복잡하고 사용하기 어려웠음)
2. **Hibernate** 등장 (오픈소스 ORM 프레임워크)
3. **JPA** 탄생 (Hibernate를 기반으로 한 자바 표준)

### JPA 구현체

JPA는 인터페이스이므로 구현체가 필요하다.

#### 주요 JPA 구현체

```java
// 1. Hibernate (가장 많이 사용)
// - 가장 성숙한 ORM 프레임워크
// - JPA 표준을 가장 많이 구현
// - 레퍼런스 구현체

// 2. EclipseLink
// - 오라클에서 제공
// - JPA 표준 스펙 리드

// 3. DataNucleus
// - 다양한 데이터베이스 지원
```

#### JPA 버전별 주요 기능

| 버전 | 주요 기능 |
|------|-----------|
| JPA 1.0 (2006) | 기본적인 ORM 기능, 엔티티 매핑, JPQL |
| JPA 2.0 (2009) | Criteria API, 컬렉션 매핑 향상, @OrderColumn |
| JPA 2.1 (2013) | 컨버터, 엔티티 그래프, 저장 프로시저 |
| JPA 2.2 (2017) | Stream API 지원, @Repeatable 지원 |

## 🔄 ORM과 JPA의 관계

### ORM (Object-Relational Mapping)

#### ORM이란?

- **객체와 관계형 데이터베이스의 데이터를 자동으로 매핑해주는 기술**
- 객체 지향 프로그래밍과 관계형 데이터베이스 간의 호환되지 않는 데이터를 변환하는 프로그래밍 기법

#### 패러다임의 불일치

```java
// 객체 지향적 코드
public class Member {
    private String name;
    private Team team;  // 객체 참조
    
    public Team getTeam() {
        return team;
    }
}

public class Team {
    private String name;
    private List<Member> members;  // 컬렉션
}
```

```sql
-- 관계형 데이터베이스
CREATE TABLE MEMBER (
    MEMBER_ID BIGINT NOT NULL,
    NAME VARCHAR(255),
    TEAM_ID BIGINT,  -- 외래 키
    PRIMARY KEY (MEMBER_ID)
);

CREATE TABLE TEAM (
    TEAM_ID BIGINT NOT NULL,
    NAME VARCHAR(255),
    PRIMARY KEY (TEAM_ID)
);
```

#### 객체와 관계형 데이터베이스의 차이

1. **상속**
   - 객체: 상속 관계 존재
   - 관계형 DB: 상속 관계 없음 (테이블 슈퍼타입 서브타입으로 표현)

2. **연관관계**
   - 객체: 참조(reference) 사용, 단방향성
   - 관계형 DB: 외래 키(Foreign Key) 사용, 양방향성

3. **객체 그래프 탐색**
   - 객체: 자유로운 객체 그래프 탐색
   - 관계형 DB: 처음 실행한 SQL에 따라 탐색 범위 결정

4. **비교**
   - 객체: 동일성(`==`)과 동등성(`equals()`) 비교
   - 관계형 DB: 기본 키(Primary Key)로 구분

### JPA와 ORM

```java
// JPA 없이 JDBC 직접 사용시
public Member findMember(Long memberId) {
    String sql = "SELECT * FROM MEMBER WHERE MEMBER_ID = ?";
    // JDBC API를 사용해서 SQL 실행
    // 결과를 Member 객체로 매핑
    return member;
}

// JPA 사용시
@Entity
public class Member {
    @Id
    private Long id;
    private String name;
    
    @ManyToOne
    @JoinColumn(name = "TEAM_ID")
    private Team team;
}

// JPA EntityManager 사용
Member member = em.find(Member.class, memberId);  // 자동으로 SQL 생성 및 매핑
```

## ⚡ JPA의 장단점

### JPA의 장점

#### 1. 생산성 향상

```java
// 기존 JDBC 방식
public void save(Member member) {
    String sql = "INSERT INTO MEMBER(MEMBER_ID, NAME, TEAM_ID) VALUES(?, ?, ?)";
    PreparedStatement pstmt = connection.prepareStatement(sql);
    pstmt.setLong(1, member.getId());
    pstmt.setString(2, member.getName());
    pstmt.setLong(3, member.getTeam().getId());
    pstmt.executeUpdate();
}

// JPA 방식
public void save(Member member) {
    em.persist(member);  // 단순!
}
```

#### 2. 유지보수성

```java
// 필드 추가시
@Entity
public class Member {
    @Id
    private Long id;
    private String name;
    private int age;  // 새 필드 추가
    
    @ManyToOne
    @JoinColumn(name = "TEAM_ID")
    private Team team;
}

// JPA가 자동으로 SQL을 수정해줌
// INSERT INTO MEMBER(MEMBER_ID, NAME, AGE, TEAM_ID) VALUES(?, ?, ?, ?)
```

#### 3. 패러다임 불일치 해결

```java
// 연관관계
Member member = em.find(Member.class, memberId);
Team team = member.getTeam();  // 객체 그래프 탐색 가능

// 비교
Member member1 = em.find(Member.class, 1L);
Member member2 = em.find(Member.class, 1L);
System.out.println(member1 == member2);  // true (같은 인스턴스 반환)
```

#### 4. 성능 최적화

```java
// 1차 캐시
Member member1 = em.find(Member.class, 1L);  // SQL 실행
Member member2 = em.find(Member.class, 1L);  // 캐시에서 조회 (SQL 실행 안함)

// 지연 로딩
@Entity
public class Member {
    @ManyToOne(fetch = FetchType.LAZY)  // 지연 로딩
    private Team team;
}

Member member = em.find(Member.class, 1L);  // Member만 조회
Team team = member.getTeam();  // 이 시점에 Team 조회
String teamName = team.getName();  // 실제 사용 시점에 초기화
```

#### 5. 데이터 접근 추상화와 벤더 독립성

```java
// 방언(Dialect) 설정으로 다양한 데이터베이스 지원
// Oracle: Oracle12cDialect
// MySQL: MySQL8Dialect
// PostgreSQL: PostgreSQL95Dialect

// 페이징 처리
// Oracle: ROWNUM
// MySQL: LIMIT
// JPA에서는 동일한 코드로 처리
TypedQuery<Member> query = em.createQuery("SELECT m FROM Member m", Member.class);
query.setFirstResult(10);
query.setMaxResults(20);
```

### JPA의 단점

#### 1. 학습 곡선

```java
// 복잡한 매핑 관계 이해 필요
@Entity
public class Order {
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> orderItems = new ArrayList<>();
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "MEMBER_ID")
    private Member member;
}
```

#### 2. 성능 이슈

```java
// N+1 문제 발생 가능
List<Member> members = em.createQuery("SELECT m FROM Member m", Member.class)
                        .getResultList();

for (Member member : members) {
    System.out.println(member.getTeam().getName());  // 각 멤버마다 Team 조회 쿼리 실행
}

// 해결: Fetch Join
List<Member> members = em.createQuery(
    "SELECT m FROM Member m JOIN FETCH m.team", Member.class)
    .getResultList();
```

#### 3. 복잡한 쿼리의 한계

```java
// 복잡한 통계 쿼리나 네이티브 쿼리가 필요한 경우
@Query(value = "SELECT * FROM (SELECT m.*, ROW_NUMBER() OVER (PARTITION BY m.team_id ORDER BY m.created_date DESC) as rn FROM member m) WHERE rn = 1", nativeQuery = true)
List<Member> findLatestMemberPerTeam();
```

#### 4. 설계의 중요성

```java
// 잘못된 연관관계 설계시 성능 문제
@Entity
public class Member {
    @OneToMany(mappedBy = "member", fetch = FetchType.EAGER)  // 즉시 로딩으로 인한 성능 문제
    private List<Order> orders = new ArrayList<>();
}
```

## 🛠️ 환경 설정

### 의존성 추가

#### Maven 설정

```xml
<!-- pom.xml -->
<dependencies>
    <!-- JPA 하이버네이트 -->
    <dependency>
        <groupId>org.hibernate</groupId>
        <artifactId>hibernate-entitymanager</artifactId>
        <version>5.6.14.Final</version>
    </dependency>
    
    <!-- H2 데이터베이스 -->
    <dependency>
        <groupId>com.h2database</groupId>
        <artifactId>h2</artifactId>
        <version>2.1.214</version>
    </dependency>
    
    <!-- JPA API -->
    <dependency>
        <groupId>javax.persistence</groupId>
        <artifactId>javax.persistence-api</artifactId>
        <version>2.2</version>
    </dependency>
</dependencies>
```

#### Gradle 설정

```groovy
// build.gradle
dependencies {
    implementation 'org.hibernate:hibernate-entitymanager:5.6.14.Final'
    implementation 'com.h2database:h2:2.1.214'
    implementation 'javax.persistence:javax.persistence-api:2.2'
}
```

### 스프링 부트 설정

```yaml
# application.yml
spring:
  jpa:
    hibernate:
      ddl-auto: create-drop  # 애플리케이션 시작/종료시 테이블 생성/삭제
    properties:
      hibernate:
        dialect: org.hibernate.dialect.H2Dialect  # 데이터베이스 방언
        format_sql: true  # SQL 포맷팅
        show_sql: true    # SQL 출력
    defer-datasource-initialization: true
    
  datasource:
    url: jdbc:h2:mem:testdb
    driver-class-name: org.h2.Driver
    username: sa
    password:
    
  h2:
    console:
      enabled: true  # H2 콘솔 활성화

logging:
  level:
    org.hibernate.SQL: DEBUG          # SQL 로그
    org.hibernate.type: TRACE         # 파라미터 로그
```

### 순수 JPA 설정

#### persistence.xml

```xml
<!-- src/main/resources/META-INF/persistence.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<persistence version="2.2"
             xmlns="http://java.sun.com/xml/ns/persistence"
             xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xsi:schemaLocation="http://java.sun.com/xml/ns/persistence 
             http://java.sun.com/xml/ns/persistence/persistence_2_2.xsd">
    
    <persistence-unit name="hello">
        <properties>
            <!-- 필수 속성 -->
            <property name="javax.persistence.jdbc.driver" value="org.h2.Driver"/>
            <property name="javax.persistence.jdbc.user" value="sa"/>
            <property name="javax.persistence.jdbc.password" value=""/>
            <property name="javax.persistence.jdbc.url" value="jdbc:h2:mem:testdb"/>
            <property name="hibernate.dialect" value="org.hibernate.dialect.H2Dialect"/>
            
            <!-- 옵션 -->
            <property name="hibernate.show_sql" value="true"/>
            <property name="hibernate.format_sql" value="true"/>
            <property name="hibernate.use_sql_comments" value="true"/>
            <property name="hibernate.hbm2ddl.auto" value="create"/>
        </properties>
    </persistence-unit>
    
</persistence>
```

### 기본 JPA 사용법

#### 엔티티 클래스 정의

```java
@Entity
@Table(name = "MEMBER")
public class Member {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "MEMBER_ID")
    private Long id;
    
    @Column(name = "USERNAME", nullable = false, length = 100)
    private String username;
    
    @Column(name = "AGE")
    private Integer age;
    
    // 기본 생성자 필수
    protected Member() {}
    
    public Member(String username, Integer age) {
        this.username = username;
        this.age = age;
    }
    
    // Getter, Setter
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    
    public String getUsername() { return username; }
    public void setUsername(String username) { this.username = username; }
    
    public Integer getAge() { return age; }
    public void setAge(Integer age) { this.age = age; }
}
```

#### JPA 애플리케이션 기본 구조

```java
public class JpaMain {
    
    public static void main(String[] args) {
        
        // EntityManagerFactory는 하나만 생성해서 애플리케이션 전체에서 공유
        EntityManagerFactory emf = Persistence.createEntityManagerFactory("hello");
        
        // EntityManager는 쓰레드간에 공유하면 안됨 (사용하고 버려야 함)
        EntityManager em = emf.createEntityManager();
        
        // JPA의 모든 데이터 변경은 트랜잭션 안에서 실행
        EntityTransaction tx = em.getTransaction();
        tx.begin();
        
        try {
            // 저장
            Member member = new Member("홍길동", 25);
            em.persist(member);
            
            // 조회
            Member findMember = em.find(Member.class, member.getId());
            System.out.println("조회한 회원: " + findMember.getUsername());
            
            // 수정 (Dirty Checking)
            findMember.setUsername("홍길동2");
            
            // 삭제
            em.remove(findMember);
            
            tx.commit();
            
        } catch (Exception e) {
            tx.rollback();
            e.printStackTrace();
        } finally {
            em.close();
        }
        
        emf.close();
    }
}
```

#### 스프링 부트에서의 사용

```java
@Service
@Transactional
public class MemberService {
    
    @PersistenceContext
    private EntityManager em;
    
    public Long save(Member member) {
        em.persist(member);
        return member.getId();
    }
    
    public Member findOne(Long id) {
        return em.find(Member.class, id);
    }
    
    public List<Member> findAll() {
        return em.createQuery("SELECT m FROM Member m", Member.class)
                .getResultList();
    }
    
    public List<Member> findByName(String name) {
        return em.createQuery("SELECT m FROM Member m WHERE m.username = :name", Member.class)
                .setParameter("name", name)
                .getResultList();
    }
}
```

### JPA 핵심 개념

#### 1. EntityManagerFactory와 EntityManager

```java
// EntityManagerFactory: 애플리케이션 전체에서 하나만 생성하여 공유
EntityManagerFactory emf = Persistence.createEntityManagerFactory("hello");

// EntityManager: 쓰레드간 공유 금지 (사용 후 close)
EntityManager em = emf.createEntityManager();
```

#### 2. 영속성 컨텍스트 (Persistence Context)

```java
// 엔티티를 영구 저장하는 환경
Member member = new Member("회원1", 20);

// 1차 캐시에 저장
em.persist(member);

// 1차 캐시에서 조회
Member findMember = em.find(Member.class, member.getId());

// 동일성 보장
System.out.println(member == findMember);  // true
```

#### 3. 엔티티 생명주기

```java
// 비영속 (new/transient): 영속성 컨텍스트와 전혀 관계가 없는 상태
Member member = new Member("회원1", 20);

// 영속 (managed): 영속성 컨텍스트에 저장된 상태
em.persist(member);

// 준영속 (detached): 영속성 컨텍스트에서 분리된 상태
em.detach(member);

// 삭제 (removed): 삭제된 상태
em.remove(member);
```

#### 4. 플러시 (Flush)

```java
Member member = new Member("회원1", 20);
em.persist(member);

// 영속성 컨텍스트의 변경 내용을 데이터베이스에 반영
em.flush();  // INSERT SQL이 데이터베이스에 전송됨

// 트랜잭션 커밋 시에도 자동으로 플러시 발생
tx.commit();
```

## 🎯 주요 어노테이션

### 기본 어노테이션

```java
@Entity  // JPA 엔티티임을 선언
@Table(name = "MEMBER")  // 매핑할 테이블 이름 지정
public class Member {
    
    @Id  // 기본 키 매핑
    @GeneratedValue(strategy = GenerationType.IDENTITY)  // 기본 키 생성 전략
    private Long id;
    
    @Column(name = "USERNAME", nullable = false, length = 100)  // 컬럼 매핑
    private String username;
    
    @Temporal(TemporalType.TIMESTAMP)  // 날짜 타입 매핑
    private Date createdDate;
    
    @Enumerated(EnumType.STRING)  // Enum 타입 매핑
    private MemberType type;
    
    @Lob  // 대용량 데이터 매핑
    private String description;
    
    @Transient  // 매핑 제외
    private int temp;
}
```

### 연관관계 어노테이션

```java
@Entity
public class Member {
    @Id @GeneratedValue
    private Long id;
    
    @ManyToOne  // 다대일 관계
    @JoinColumn(name = "TEAM_ID")
    private Team team;
}

@Entity
public class Team {
    @Id @GeneratedValue
    private Long id;
    
    @OneToMany(mappedBy = "team")  // 일대다 관계 (연관관계의 주인이 아님)
    private List<Member> members = new ArrayList<>();
}
```

---

## 📝 정리

### JPA의 핵심 개념

1. **JPA는 자바 진영의 ORM 기술 표준**이다
2. **객체와 관계형 데이터베이스 간의 패러다임 불일치를 해결**한다
3. **생산성, 유지보수성, 성능, 데이터 접근 추상화** 등의 장점을 제공한다

### JPA의 주요 구성 요소

1. **EntityManagerFactory**: 애플리케이션 전체에서 하나만 생성하여 공유
2. **EntityManager**: 쓰레드간 공유 금지, 사용 후 반드시 close
3. **영속성 컨텍스트**: 엔티티를 영구 저장하는 환경
4. **트랜잭션**: JPA의 모든 데이터 변경은 트랜잭션 안에서 실행

### 엔티티 생명주기

1. **비영속(new)**: 영속성 컨텍스트와 관계없는 상태
2. **영속(managed)**: 영속성 컨텍스트에 저장된 상태
3. **준영속(detached)**: 영속성 컨텍스트에서 분리된 상태
4. **삭제(removed)**: 삭제된 상태

### 영속성 컨텍스트의 특징

1. **1차 캐시**: 성능 최적화
2. **동일성 보장**: 같은 식별자의 엔티티는 같은 인스턴스 반환
3. **쓰기 지연**: 트랜잭션 커밋 시점에 SQL 전송
4. **변경 감지(Dirty Checking)**: 엔티티 변경 시 자동으로 UPDATE SQL 생성
5. **지연 로딩**: 연관된 엔티티를 실제 사용 시점에 로딩

---

[목차로 돌아가기](./README.md) | [다음: 엔티티 매핑 →](./02-entity-mapping.md)