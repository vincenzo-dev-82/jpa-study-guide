# 3. 연관관계 매핑

## 📖 개요

**엔티티들은 대부분 다른 엔티티와 연관관계가 있다.** 객체지향 설계의 목표는 자율적인 객체들의 **협력 공동체**를 만드는 것이다.

### 연관관계가 필요한 이유

#### 예제 시나리오

- 회원과 팀이 있다
- 회원은 하나의 팀에만 소속될 수 있다
- 회원과 팀은 다대일 관계이다

#### 객체를 테이블에 맞추어 모델링 (연관관계가 없는 객체)

```java
@Entity
public class Member {
    @Id @GeneratedValue
    @Column(name = "MEMBER_ID")
    private Long id;
    
    @Column(name = "USERNAME")
    private String username;
    
    @Column(name = "TEAM_ID")
    private Long teamId;  // 외래 키를 그대로 필드로
    
    // getter, setter...
}

@Entity
public class Team {
    @Id @GeneratedValue
    @Column(name = "TEAM_ID")
    private Long id;
    
    private String name;
    
    // getter, setter...
}
```

#### 문제점 1: 참조 대신에 외래 키를 그대로 사용

```java
// 팀 저장
Team team = new Team();
team.setName("TeamA");
em.persist(team);

// 회원 저장
Member member = new Member();
member.setUsername("member1");
member.setTeamId(team.getId());  // 외래 키 식별자를 직접 다룸
em.persist(member);
```

#### 문제점 2: 식별자로 다시 조회 (객체 지향적인 방법이 아님)

```java
// 조회
Member findMember = em.find(Member.class, member.getId());

// 연관된 팀을 찾기 위해 외래 키로 다시 조회
Long teamId = findMember.getTeamId();
Team findTeam = em.find(Team.class, teamId);  // 객체지향적이지 않음
```

**객체를 테이블에 맞추어 데이터 중심으로 모델링하면, 협력 관계를 만들 수 없다.**
- 테이블은 외래 키로 조인을 사용해서 연관된 테이블을 찾는다
- 객체는 참조를 사용해서 연관된 객체를 찾는다
- 테이블과 객체 사이에는 이런 큰 간격이 있다

## 🔗 단방향 연관관계

### 객체 지향 모델링 (객체 연관관계 사용)

```java
@Entity
public class Member {
    @Id @GeneratedValue
    @Column(name = "MEMBER_ID")
    private Long id;
    
    @Column(name = "USERNAME")
    private String username;
    
    @ManyToOne  // 다대일 관계
    @JoinColumn(name = "TEAM_ID")  // 외래 키 매핑
    private Team team;  // 객체 참조
    
    // getter, setter...
}

@Entity
public class Team {
    @Id @GeneratedValue
    @Column(name = "TEAM_ID")
    private Long id;
    
    private String name;
    
    // getter, setter...
}
```

### 연관관계 저장

```java
// 팀 저장
Team team = new Team();
team.setName("TeamA");
em.persist(team);

// 회원 저장
Member member = new Member();
member.setUsername("member1");
member.setTeam(team);  // 단방향 연관관계 설정, 참조 저장
em.persist(member);
```

### 참조로 연관관계 조회 - 객체 그래프 탐색

```java
// 조회
Member findMember = em.find(Member.class, member.getId());

// 참조를 사용해서 연관관계 조회
Team findTeam = findMember.getTeam();
System.out.println("팀 이름: " + findTeam.getName());  // 객체지향적!
```

### 연관관계 수정

```java
// 새로운 팀B
Team teamB = new Team();
teamB.setName("TeamB");
em.persist(teamB);

// 회원1에 새로운 팀B 설정
Member findMember = em.find(Member.class, member.getId());
findMember.setTeam(teamB);  // 연관관계 수정
```

## ↔️ 양방향 연관관계와 연관관계의 주인

### 양방향 매핑

```java
@Entity
public class Member {
    @Id @GeneratedValue
    @Column(name = "MEMBER_ID")
    private Long id;
    
    @Column(name = "USERNAME")
    private String username;
    
    @ManyToOne
    @JoinColumn(name = "TEAM_ID")
    private Team team;
    
    // getter, setter...
}

@Entity
public class Team {
    @Id @GeneratedValue
    @Column(name = "TEAM_ID")
    private Long id;
    
    private String name;
    
    @OneToMany(mappedBy = "team")  // 일대다 매핑, 연관관계의 주인이 아님
    private List<Member> members = new ArrayList<>();
    
    // getter, setter...
}
```

### 반대 방향으로 객체 그래프 탐색

```java
// 조회
Team findTeam = em.find(Team.class, team.getId());
int memberSize = findTeam.getMembers().size();  // 역방향 조회

// 팀에서 회원으로 접근
List<Member> members = findTeam.getMembers();
for (Member member : members) {
    System.out.println("member: " + member.getUsername());
}
```

### 연관관계의 주인과 mappedBy

#### 객체와 테이블간에 연관관계를 맺는 차이를 이해해야 한다

- **객체 연관관계 = 2개**
  - 회원 → 팀 연관관계 1개 (단방향)
  - 팀 → 회원 연관관계 1개 (단방향)
  
- **테이블 연관관계 = 1개**
  - 회원 ↔ 팀의 연관관계 1개 (양방향)

```sql
-- 테이블은 외래 키 하나로 두 테이블의 연관관계를 관리
SELECT *
FROM MEMBER M
JOIN TEAM T ON M.TEAM_ID = T.TEAM_ID;

SELECT *
FROM TEAM T
JOIN MEMBER M ON T.TEAM_ID = M.TEAM_ID;
```

#### 둘 중 하나로 외래 키를 관리해야 한다

```java
// Member의 team을 변경했을 때 TEAM_ID 외래 키가 변경되어야 할까?
Member member = em.find(Member.class, 1L);
member.setTeam(newTeam);

// Team의 members를 변경했을 때 TEAM_ID 외래 키가 변경되어야 할까?
Team team = em.find(Team.class, 1L);
team.getMembers().add(newMember);
```

### 연관관계의 주인 (Owner)

#### 양방향 매핑 규칙

- 객체의 두 관계중 하나를 연관관계의 주인으로 지정
- **연관관계의 주인만이 외래 키를 관리(등록, 수정)**
- **주인이 아닌쪽은 읽기만 가능**
- 주인은 mappedBy 속성 사용 X
- 주인이 아니면 mappedBy 속성으로 주인 지정

#### 누구를 주인으로?

**외래 키가 있는 곳을 주인으로 정해라**

```java
@Entity
public class Member {
    @Id @GeneratedValue
    private Long id;
    
    @ManyToOne
    @JoinColumn(name = "TEAM_ID")
    private Team team;  // 연관관계의 주인 (외래 키가 있는 곳)
}

@Entity
public class Team {
    @Id @GeneratedValue
    private Long id;
    
    @OneToMany(mappedBy = "team")  // 주인이 아님
    private List<Member> members = new ArrayList<>();
}
```

- 여기서는 **Member.team이 연관관계의 주인**
- DB 테이블의 다대일, 일대다 관계에서는 항상 **다(N) 쪽이 외래 키를 가짐**
- 따라서 @ManyToOne은 항상 연관관계의 주인이 되므로 mappedBy 속성이 없음

## ⚠️ 양방향 연관관계의 주의점

### 가장 많이 하는 실수

**연관관계의 주인에 값을 입력하지 않음**

```java
Team team = new Team();
team.setName("TeamA");
em.persist(team);

Member member = new Member();
member.setUsername("member1");

// 역방향(주인이 아닌 방향)만 연관관계 설정
team.getMembers().add(member);

em.persist(member);
```

**결과: 외래 키 값이 null이 됨**

| MEMBER_ID | USERNAME | TEAM_ID |
|-----------|----------|---------|
| 1         | member1  | null    |

### 올바른 방법

**연관관계의 주인에 값을 입력해야 한다**

```java
Team team = new Team();
team.setName("TeamA");
em.persist(team);

Member member = new Member();
member.setUsername("member1");

team.getMembers().add(member);
member.setTeam(team);  // 연관관계의 주인에 값 설정 ★

em.persist(member);
```

**결과**

| MEMBER_ID | USERNAME | TEAM_ID |
|-----------|----------|---------|
| 1         | member1  | 1       |

### 순수한 객체 관계를 고려하면 항상 양쪽 다 값을 입력해야 한다

```java
@Test
public void testORM_순수한객체_양방향() {
    // 팀1
    Team team1 = new Team();
    team1.setName("TeamA");
    
    Member member1 = new Member();
    member1.setUsername("member1");
    
    Member member2 = new Member();
    member2.setUsername("member2");
    
    member1.setTeam(team1);  // 연관관계의 주인에 값 설정
    member2.setTeam(team1);  // 연관관계의 주인에 값 설정
    
    team1.getMembers().add(member1);  // 주인이 아닌 곳에도 값 설정
    team1.getMembers().add(member2);  // 주인이 아닌 곳에도 값 설정
    
    List<Member> members = team1.getMembers();
    System.out.println("members.size = " + members.size());  // 2
}
```

**양쪽 다 값을 입력해야 하는 이유:**
1. **순수한 객체 상태를 고려**
2. **테스트 케이스 작성시**
3. **JPA를 사용하지 않는 순수한 객체에서도 동작**

### 연관관계 편의 메소드를 생성하자

```java
@Entity
public class Member {
    @Id @GeneratedValue
    private Long id;
    
    @Column(name = "USERNAME")
    private String username;
    
    @ManyToOne
    @JoinColumn(name = "TEAM_ID")
    private Team team;
    
    // 연관관계 편의 메소드
    public void setTeam(Team team) {
        this.team = team;
        team.getMembers().add(this);  // 양방향 연관관계 한번에 처리
    }
    
    // getter, setter...
}
```

**사용**

```java
Team team = new Team();
team.setName("TeamA");
em.persist(team);

Member member = new Member();
member.setUsername("member1");
member.setTeam(team);  // 이 한줄로 양방향 연관관계 설정!

em.persist(member);

List<Member> members = team.getMembers();
System.out.println("members.size = " + members.size());  // 1
```

### 연관관계 편의 메소드 작성 시 주의사항

```java
@Entity
public class Member {
    // 기존 팀과의 관계를 제거하는 로직 추가
    public void setTeam(Team team) {
        // 기존 팀과의 관계 제거
        if (this.team != null) {
            this.team.getMembers().remove(this);
        }
        
        this.team = team;
        if (team != null) {
            team.getMembers().add(this);
        }
    }
}
```

### 양방향 매핑시에 무한 루프를 조심하자

#### toString(), lombok, JSON 생성 라이브러리

```java
@Entity
public class Member {
    // ...
    
    @Override
    public String toString() {
        return "Member{" +
                "id=" + id +
                ", username='" + username + '\'' +
                ", team=" + team +  // team.toString() 호출
                '}';
    }
}

@Entity
public class Team {
    // ...
    
    @Override
    public String toString() {
        return "Team{" +
                "id=" + id +
                ", name='" + name + '\'' +
                ", members=" + members +  // members.toString() 호출
                '}';
    }
}

// 무한 루프 발생!
// Member.toString() → Team.toString() → Member.toString() → ...
```

**해결 방법:**
- toString()에서 연관관계 필드 제외
- JSON 직렬화시 @JsonIgnore 사용
- **컨트롤러에서는 엔티티를 절대 반환하지 말고 DTO 사용**

## 📋 양방향 연관관계 정리

### 양방향 매핑 정리

- **단방향 매핑만으로도 이미 연관관계 매핑은 완료**
- 양방향 매핑은 반대 방향으로 조회(객체 그래프 탐색) 기능이 추가된 것 뿐
- JPQL에서 역방향으로 탐색할 일이 많음
- 단방향 매핑을 잘 하고 양방향은 필요할 때 추가해도 됨 (테이블에 영향을 주지 않음)

### 연관관계의 주인을 정하는 기준

- 비즈니스 로직을 기준으로 연관관계의 주인을 선택하면 안됨
- **연관관계의 주인은 외래 키의 위치를 기준으로 정해야함**

### 실무 조언

```java
// 1. 단방향 매핑으로 설계 완료
@Entity
public class Member {
    @ManyToOne
    @JoinColumn(name = "TEAM_ID")
    private Team team;
}

// 2. 필요시 양방향 매핑 추가 (테이블에 영향 없음)
@Entity
public class Team {
    @OneToMany(mappedBy = "team")
    private List<Member> members = new ArrayList<>();
}
```

## 🔄 일대일 관계

### 일대일 관계의 특징

- 일대일 관계는 그 반대도 일대일
- 주 테이블이나 대상 테이블 중에 외래 키 선택 가능
- 외래 키에 데이터베이스 유니크(UNI) 제약조건 추가

### 주 테이블에 외래 키

#### 단방향

```java
@Entity
public class Member {
    @Id @GeneratedValue
    @Column(name = "MEMBER_ID")
    private Long id;
    
    private String username;
    
    @OneToOne
    @JoinColumn(name = "LOCKER_ID")
    private Locker locker;
}

@Entity
public class Locker {
    @Id @GeneratedValue
    @Column(name = "LOCKER_ID")
    private Long id;
    
    private String name;
}
```

#### 양방향

```java
@Entity
public class Member {
    @Id @GeneratedValue
    @Column(name = "MEMBER_ID")
    private Long id;
    
    private String username;
    
    @OneToOne
    @JoinColumn(name = "LOCKER_ID")
    private Locker locker;
}

@Entity
public class Locker {
    @Id @GeneratedValue
    @Column(name = "LOCKER_ID")
    private Long id;
    
    private String name;
    
    @OneToOne(mappedBy = "locker")  // 연관관계의 주인이 아님
    private Member member;
}
```

### 대상 테이블에 외래 키

#### 단방향

**JPA에서는 지원하지 않음**

#### 양방향

```java
@Entity
public class Member {
    @Id @GeneratedValue
    @Column(name = "MEMBER_ID")
    private Long id;
    
    private String username;
    
    @OneToOne(mappedBy = "member")  // 연관관계의 주인이 아님
    private Locker locker;
}

@Entity
public class Locker {
    @Id @GeneratedValue
    @Column(name = "LOCKER_ID")
    private Long id;
    
    private String name;
    
    @OneToOne
    @JoinColumn(name = "MEMBER_ID")  // 외래 키가 있는 곳이 연관관계의 주인
    private Member member;
}
```

### 일대일 관계 정리

#### 주 테이블에 외래 키

- 주 객체가 대상 객체의 참조를 가지는 것처럼 주 테이블에 외래 키를 두고 대상 테이블을 찾음
- 객체지향 개발자 선호
- JPA 매핑 편리
- 장점: 주 테이블만 조회해도 대상 테이블에 데이터가 있는지 확인 가능
- 단점: 값이 없으면 외래 키에 null 허용

#### 대상 테이블에 외래 키

- 대상 테이블에 외래 키가 존재
- 전통적인 데이터베이스 개발자 선호
- 장점: 주 테이블과 대상 테이블을 일대일에서 일대다 관계로 변경할 때 테이블 구조 유지
- 단점: 프록시 기능의 한계로 **지연 로딩으로 설정해도 항상 즉시 로딩됨**

## 🔢 일대다 관계

### 일대다 단방향

```java
@Entity
public class Team {
    @Id @GeneratedValue
    @Column(name = "TEAM_ID")
    private Long id;
    
    private String name;
    
    @OneToMany
    @JoinColumn(name = "TEAM_ID")  // Team이 연관관계의 주인
    private List<Member> members = new ArrayList<>();
}

@Entity
public class Member {
    @Id @GeneratedValue
    @Column(name = "MEMBER_ID")
    private Long id;
    
    @Column(name = "USERNAME")
    private String username;
    
    // Team에 대한 참조가 없음
}
```

#### 일대다 단방향 매핑의 단점

- 엔티티가 관리하는 외래 키가 다른 테이블에 있음
- 연관관계 관리를 위해 추가로 UPDATE SQL 실행

```java
Member member = new Member();
member.setUsername("member1");

em.persist(member);

Team team = new Team();
team.setName("teamA");
team.getMembers().add(member);

em.persist(team);

// 실행되는 SQL
// INSERT INTO MEMBER (USERNAME, MEMBER_ID) VALUES ('member1', 1)
// INSERT INTO TEAM (NAME, TEAM_ID) VALUES ('teamA', 1)
// UPDATE MEMBER SET TEAM_ID=1 WHERE MEMBER_ID=1  -- 추가 UPDATE!
```

**일대다 단방향 매핑보다는 다대일 양방향 매핑을 사용하자**

### 일대다 양방향

```java
@Entity
public class Team {
    @Id @GeneratedValue
    @Column(name = "TEAM_ID")
    private Long id;
    
    private String name;
    
    @OneToMany
    @JoinColumn(name = "TEAM_ID")
    private List<Member> members = new ArrayList<>();
}

@Entity
public class Member {
    @Id @GeneratedValue
    @Column(name = "MEMBER_ID")
    private Long id;
    
    @Column(name = "USERNAME")
    private String username;
    
    @ManyToOne
    @JoinColumn(name = "TEAM_ID", insertable = false, updatable = false)
    private Team team;  // 읽기 전용
}
```

**이런 매핑은 공식적으로 존재하지 않음**
- @JoinColumn(insertable=false, updatable=false)
- 읽기 전용 필드를 사용해서 양방향처럼 사용하는 방법
- **다대일 양방향을 사용하자**

## 🔀 다대다 관계

### 다대다 관계의 한계

- 관계형 데이터베이스는 정규화된 테이블 2개로 다대다 관계를 표현할 수 없음
- 연결 테이블을 추가해서 일대다, 다대일 관계로 풀어내야함

### @ManyToMany 사용

```java
@Entity
public class Member {
    @Id @GeneratedValue
    @Column(name = "MEMBER_ID")
    private Long id;
    
    private String username;
    
    @ManyToMany
    @JoinTable(name = "MEMBER_PRODUCT")
    private List<Product> products = new ArrayList<>();
}

@Entity
public class Product {
    @Id @GeneratedValue
    @Column(name = "PRODUCT_ID")
    private Long id;
    
    private String name;
    
    @ManyToMany(mappedBy = "products")
    private List<Member> members = new ArrayList<>();
}
```

#### @ManyToMany 한계

- **편리해 보이지만 실무에서 사용 X**
- 연결 테이블이 단순히 연결만 하고 끝나지 않음
- 주문시간, 수량 같은 데이터가 들어올 수 있음

```sql
-- 실제 필요한 연결 테이블
CREATE TABLE MEMBER_PRODUCT (
    MEMBER_ID BIGINT,
    PRODUCT_ID BIGINT,
    ORDER_COUNT INT,      -- 추가 필드
    ORDER_PRICE INT,      -- 추가 필드
    ORDER_DATE TIMESTAMP, -- 추가 필드
    PRIMARY KEY (MEMBER_ID, PRODUCT_ID)
);
```

### @ManyToMany 한계 극복

**연결 테이블용 엔티티 추가 (연결 테이블을 엔티티로 승격)**

```java
@Entity
public class Member {
    @Id @GeneratedValue
    @Column(name = "MEMBER_ID")
    private Long id;
    
    private String username;
    
    @OneToMany(mappedBy = "member")
    private List<MemberProduct> memberProducts = new ArrayList<>();
}

@Entity
public class Product {
    @Id @GeneratedValue
    @Column(name = "PRODUCT_ID")
    private Long id;
    
    private String name;
    
    @OneToMany(mappedBy = "product")
    private List<MemberProduct> memberProducts = new ArrayList<>();
}

@Entity
public class MemberProduct {
    @Id @GeneratedValue
    private Long id;
    
    @ManyToOne
    @JoinColumn(name = "MEMBER_ID")
    private Member member;
    
    @ManyToOne
    @JoinColumn(name = "PRODUCT_ID")
    private Product product;
    
    private int orderCount;
    private int orderPrice;
    private LocalDateTime orderDate;
}
```

---

## 📝 정리

### 연관관계 매핑 핵심

1. **방향**: 단방향, 양방향
2. **다중성**: 다대일(N:1), 일대다(1:N), 일대일(1:1), 다대다(N:M)
3. **연관관계의 주인**: 양방향 연관관계에서 외래 키를 관리하는 쪽

### 연관관계 매핑시 고려사항 3가지

1. **다중성**
   - 다대일: @ManyToOne
   - 일대다: @OneToMany
   - 일대일: @OneToOne
   - 다대다: @ManyToMany (실무에서 쓰면 안됨)

2. **단방향, 양방향**
   - 테이블: 외래 키 하나로 양쪽 조인 가능 (방향이라는 개념이 없음)
   - 객체: 참조용 필드가 있는 쪽으로만 참조 가능
   - 한쪽만 참조하면 단방향, 양쪽이 서로 참조하면 양방향

3. **연관관계의 주인**
   - 테이블은 외래 키 하나로 두 테이블이 연관관계를 맺음
   - 객체 양방향 관계는 A→B, B→A처럼 참조가 2군데
   - 객체 양방향 관계는 참조가 2군데 있음. 둘중 테이블의 외래 키를 관리할 곳을 지정해야함
   - 연관관계의 주인: 외래 키를 관리하는 참조
   - 주인의 반대편: 외래 키에 영향을 주지 않음, 단순 조회만 가능

### 권장사항

1. **단방향 매핑으로 이미 연관관계 매핑은 완료**
2. **양방향 매핑은 반대 방향으로 조회 기능이 추가된 것**
3. **JPQL에서 역방향으로 탐색할 일이 많음**
4. **단방향 매핑을 잘 하고 양방향은 필요할 때 추가**
5. **연관관계의 주인은 외래 키의 위치를 기준으로 정하기**
6. **다대다 매핑은 실무에서 사용하지 말고 일대다, 다대일로 풀어서 사용**

### 실무 팁

- **가급적 단방향 매핑만으로 설계를 끝내자**
- **양방향 매핑은 JPQL에서 역방향으로 탐색할 일이 많을 때 추가**
- **양방향 매핑 시 연관관계 편의 메소드를 생성하자**
- **양방향 매핑 시 무한 루프를 조심하자** (toString, lombok, JSON 생성 라이브러리)
- **컨트롤러에서는 엔티티를 절대 반환하지 말자**

---

[← 이전: 엔티티 매핑](./02-entity-mapping.md) | [목차로 돌아가기](./README.md) | [다음: 고급 매핑 →](./04-advanced-mapping.md)