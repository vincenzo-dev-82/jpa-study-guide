# 8. 웹 애플리케이션과 영속성 관리

## 📖 트랜잭션 범위의 영속성 컨텍스트

### 스프링 컨테이너의 기본 전략

스프링 컨테이너는 **트랜잭션 범위의 영속성 컨텍스트** 전략을 기본으로 사용한다.

#### 트랜잭션 범위의 영속성 컨텍스트란?

- 트랜잭션을 시작할 때 영속성 컨텍스트를 생성하고, 트랜잭션이 끝날 때 영속성 컨텍스트를 종료한다
- 같은 트랜잭션 안에서는 항상 같은 영속성 컨텍스트에 접근한다

```java
@Service
@Transactional
public class MemberService {
    
    @Autowired
    private MemberRepository memberRepository;
    
    public void hello() {
        // 1. 트랜잭션 시작 → 영속성 컨텍스트 생성
        
        Member member = memberRepository.findOne(1L);
        member.setName("hello");
        
        // 2. 트랜잭션 종료 → 영속성 컨텍스트 종료, 플러시 발생
    }
}
```

### 트랜잭션이 같으면 같은 영속성 컨텍스트를 사용한다

![트랜잭션과 영속성 컨텍스트 관계](https://imgur.com/transaction-context.png)

- 여러 곳에서 EntityManager를 주입받아도 같은 트랜잭션 범위에 있으면 같은 영속성 컨텍스트를 사용한다

```java
@Service
@Transactional
public class MemberService {
    
    @PersistenceContext
    private EntityManager em;
    
    @Autowired
    private MemberRepository memberRepository;
    
    public void business() {
        Member member1 = em.find(Member.class, 1L);
        Member member2 = memberRepository.findOne(1L);
        
        System.out.println(member1 == member2); // true
        // 같은 트랜잭션 범위에 있으므로 같은 영속성 컨텍스트를 사용
        // 1차 캐시에서 조회하므로 member1과 member2는 같은 인스턴스
    }
}
```

### 트랜잭션이 다르면 다른 영속성 컨텍스트를 사용한다

![다른 트랜잭션, 다른 영속성 컨텍스트](https://imgur.com/diff-transaction.png)

- 여러 스레드에서 동시에 요청이 와서 같은 EntityManager를 사용해도 트랜잭션에 따라 접근하는 영속성 컨텍스트가 다르다
- 따라서 멀티스레드 상황에서 안전하다

## 🔄 준영속 상태와 지연 로딩

### 준영속 상태의 지연 로딩 문제

스프링이나 J2EE 컨테이너의 기본 전략인 "트랜잭션 범위의 영속성 컨텍스트"를 사용하면 트랜잭션이 없는 프레젠테이션 계층에서 엔티티는 준영속 상태다. 따라서 변경 감지와 지연 로딩이 동작하지 않는다.

```java
@Controller
public class MemberController {
    
    @Autowired
    private MemberService memberService;
    
    public String viewMember(Long memberId) {
        Member member = memberService.getMember(memberId); // Service에서 조회
        // 트랜잭션이 끝났으므로 member는 준영속 상태
        
        member.getOrders().size(); // 지연 로딩 시도
        // LazyInitializationException 발생!
        
        return "member/memberView";
    }
}

@Service
@Transactional
public class MemberService {
    
    @Autowired
    private MemberRepository memberRepository;
    
    public Member getMember(Long memberId) {
        return memberRepository.findOne(memberId);
        // 트랜잭션 종료, 영속성 컨텍스트 종료
        // 반환된 member 엔티티는 준영속 상태가 됨
    }
}
```

### 준영속 상태의 지연 로딩 해결 방안

#### 1. 뷰가 필요한 엔티티를 미리 로딩해두는 방법

##### 1-1. 글로벌 페치 전략 수정

```java
@Entity
public class Member {
    
    @Id @GeneratedValue
    private Long id;
    
    @OneToMany(mappedBy = "member", fetch = FetchType.EAGER)
    private List<Order> orders = new ArrayList<>();
    
    // ...
}
```

**문제점:**
- 사용하지 않는 엔티티를 로딩한다
- N+1 문제가 발생할 수 있다

##### 1-2. JPQL 페치 조인

```java
@Repository
public class MemberRepository {
    
    public Member findMemberWithOrders(Long memberId) {
        return em.createQuery(
            "select m from Member m " +
            "join fetch m.orders " +
            "where m.id = :memberId", Member.class)
            .setParameter("memberId", memberId)
            .getSingleResult();
    }
}
```

**장점:**
- 필요한 엔티티만 선택적으로 페치 조인할 수 있다
- N+1 문제가 발생하지 않는다

**단점:**
- 무분별하게 사용하면 뷰와 리포지토리 간에 논리적인 의존성이 발생한다

##### 1-3. 강제로 초기화

```java
@Service
@Transactional
public class MemberService {
    
    public Member getMember(Long memberId) {
        Member member = memberRepository.findOne(memberId);
        member.getOrders().size(); // 강제 초기화
        return member;
    }
}
```

**문제점:**
- 프레젠테이션 계층이 서비스 계층을 침범한다

#### 2. FACADE 계층 추가

```java
@Service
@Transactional
public class MemberFacade {
    
    @Autowired
    private MemberService memberService;
    
    public Member getMemberWithOrders(Long memberId) {
        Member member = memberService.getMember(memberId);
        member.getOrders().size(); // 프록시 초기화
        return member;
    }
}
```

- 프레젠테이션 계층과 서비스 계층 사이에 FACADE 계층을 하나 더 둔다
- 뷰를 위한 프록시 초기화를 이곳에서 담당한다
- 프레젠테이션 계층과 비즈니스 로직 사이의 논리적 의존성을 분리할 수 있다

#### 3. 준영속 상태에서 지연 로딩을 가능하게 하는 방법

##### OSIV (Open Session In View)

## 🔍 OSIV (Open Session In View)

### OSIV란?

- 영속성 컨텍스트를 뷰 렌더링이 끝나는 시점까지 열어두는 것
- 뷰에서도 지연 로딩이 가능하다

### 과거 OSIV: 요청 당 트랜잭션

![과거 OSIV](https://imgur.com/old-osiv.png)

**문제점:**
- 컨트롤러나 뷰 같은 프레젠테이션 계층이 엔티티를 변경할 수 있다
- 프레젠테이션 계층에서 엔티티를 수정하고 비즈니스 로직을 수행하면 엔티티가 수정될 수 있다

```java
@Controller
public class MemberController {
    
    public String viewMember(Long memberId) {
        Member member = memberService.getMember(memberId);
        member.setName("XXX"); // 보안상의 이유로 고객 이름을 XXX로 변경
        
        memberService.biz(); // 비즈니스 로직 실행
        // 트랜잭션 커밋 시 member의 name이 XXX로 수정됨!
        
        return "member/memberView";
    }
}
```

### 스프링 OSIV: 비즈니스 계층 트랜잭션

![스프링 OSIV](https://imgur.com/spring-osiv.png)

**스프링 프레임워크가 제공하는 OSIV의 특징:**

1. **비즈니스 계층에서만 트랜잭션을 유지**한다
2. **프레젠테이션 계층에서는 트랜잭션이 없다**
3. **영속성 컨텍스트는 뷰까지 열려있다**

#### 스프링 OSIV 동작 원리

1. 클라이언트의 요청이 들어오면 서블릿 필터나, 스프링 인터셉터에서 영속성 컨텍스트를 생성한다. 단 이때 트랜잭션은 시작하지 않는다.

2. 서비스 계층에서 @Transactional로 트랜잭션을 시작할 때 1번에서 미리 생성해둔 영속성 컨텍스트를 찾아와서 트랜잭션을 시작한다.

3. 서비스 계층이 끝나면 트랜잭션을 커밋하고 영속성 컨텍스트를 플러시한다. 이때 트랜잭션은 끝내지만 영속성 컨텍스트는 종료하지 않는다.

4. 컨트롤러와 뷰까지 영속성 컨텍스트가 유지되므로 조회한 엔티티는 영속 상태를 유지한다.

5. 서블릿 필터나, 스프링 인터셉터로 요청이 끝날 때 영속성 컨텍스트를 종료한다. 이때 플러시를 호출하지 않고 바로 종료한다.

#### 트랜잭션 없이는 읽기만 가능

```java
@Controller
public class MemberController {
    
    public String viewMember(Long memberId) {
        Member member = memberService.getMember(memberId);
        
        // 프레젠테이션 계층에서는 트랜잭션이 없으므로
        member.setName("XXX"); // 영속성 컨텍스트에서는 변경되지만
        
        // 트랜잭션이 없으므로 데이터베이스에 반영되지 않는다
        
        return "member/memberView";
    }
}
```

#### 지연 로딩은 가능

```java
@Controller
public class MemberController {
    
    public String viewMember(Long memberId) {
        Member member = memberService.getMember(memberId);
        
        // 영속성 컨텍스트가 살아있으므로 지연 로딩 가능
        List<Order> orders = member.getOrders();
        orders.size(); // 지연 로딩 초기화
        
        return "member/memberView";
    }
}
```

### 스프링 OSIV 설정

#### 스프링 부트를 사용하는 경우

```yaml
# application.yml
spring:
  jpa:
    open-in-view: true # 기본값 true
```

#### 스프링 부트를 사용하지 않는 경우

```xml
<!-- web.xml -->
<filter>
    <filter-name>OpenEntityManagerInViewFilter</filter-name>
    <filter-class>org.springframework.orm.jpa.support.OpenEntityManagerInViewFilter</filter-class>
</filter>
<filter-mapping>
    <filter-name>OpenEntityManagerInViewFilter</filter-name>
    <url-pattern>/*</url-pattern>
</filter-mapping>
```

### OSIV 주의사항

#### 1. 같은 영속성 컨텍스트에서 여러 트랜잭션 사용시 주의사항

```java
@Controller
public class MemberController {
    
    @Autowired
    private MemberService memberService;
    
    public String viewMember(Long memberId) {
        Member member = memberService.getMember(memberId); // 트랜잭션 1에서 조회
        member.setName("XXX");
        
        memberService.updateMember(memberId, "YYY"); // 트랜잭션 2에서 수정
        // member 엔티티의 name이 "XXX"로 변경되어 저장될 수 있음!
        
        return "member/memberView";
    }
}
```

**해결방법:**
- 트랜잭션이 롤백되면 영속성 컨텍스트도 함께 초기화하는 것이 안전하다
- 비즈니스 로직은 서비스 계층에서 끝내고 프레젠테이션 계층에서는 데이터를 단순히 출력만 담당한다

#### 2. 프레젠테이션 계층에서 엔티티 수정 예방

```java
// 읽기 전용 인터페이스 제공
public interface MemberView {
    String getName();
    String getEmail();
}

@Entity
public class Member implements MemberView {
    // ...
}

@Controller
public class MemberController {
    
    public String viewMember(Long memberId) {
        MemberView member = memberService.getMember(memberId);
        // MemberView 인터페이스를 통해서만 접근 가능
        
        return "member/memberView";
    }
}
```

#### 3. DTO 사용

```java
public class MemberDTO {
    private String name;
    private String email;
    
    public MemberDTO(Member member) {
        this.name = member.getName();
        this.email = member.getEmail();
    }
    
    // getter만 제공
}

@Service
@Transactional(readOnly = true)
public class MemberService {
    
    public MemberDTO getMemberDTO(Long memberId) {
        Member member = memberRepository.findOne(memberId);
        return new MemberDTO(member); // DTO로 변환
    }
}
```

## ⚡ 성능 최적화

### N+1 문제

#### 문제 상황

```java
@Entity
public class Member {
    
    @OneToMany(mappedBy = "member", fetch = FetchType.EAGER)
    private List<Order> orders = new ArrayList<>();
    
    // ...
}

// JPQL 실행
List<Member> members = em.createQuery("select m from Member m", Member.class)
                        .getResultList();

// 실행되는 SQL
// 1. select * from Member
// 2. select * from Orders where MEMBER_ID = 1 (첫 번째 회원)
// 3. select * from Orders where MEMBER_ID = 2 (두 번째 회원)
// 4. select * from Orders where MEMBER_ID = 3 (세 번째 회원)
// ...
// 회원이 N명이면 N번의 추가 쿼리가 실행됨 (N+1 문제)
```

#### 해결책 1: 페치 조인 사용

```java
List<Member> members = em.createQuery(
    "select m from Member m join fetch m.orders", Member.class)
    .getResultList();

// 실행되는 SQL
// select m.*, o.* from Member m inner join Orders o on m.id = o.member_id
```

#### 해결책 2: 하이버네이트 @BatchSize

```java
@Entity
public class Member {
    
    @BatchSize(size = 5)
    @OneToMany(mappedBy = "member")
    private List<Order> orders = new ArrayList<>();
    
    // ...
}

// 실행되는 SQL
// 1. select * from Member
// 2. select * from Orders where MEMBER_ID in (1, 2, 3, 4, 5)
```

#### 해결책 3: 하이버네이트 @Fetch(FetchMode.SUBSELECT)

```java
@Entity
public class Member {
    
    @Fetch(FetchMode.SUBSELECT)
    @OneToMany(mappedBy = "member")
    private List<Order> orders = new ArrayList<>();
    
    // ...
}

// 실행되는 SQL
// 1. select * from Member
// 2. select * from Orders where MEMBER_ID in (
//        select MEMBER_ID from Member
//    )
```

### 읽기 전용 쿼리의 성능 최적화

#### 스칼라 타입으로 조회

```java
// 엔티티로 조회 (영속성 컨텍스트에서 관리)
List<Member> members = em.createQuery(
    "select m from Member m", Member.class)
    .getResultList();

// 스칼라 타입으로 조회 (영속성 컨텍스트에서 관리하지 않음)
List<String> names = em.createQuery(
    "select m.name from Member m", String.class)
    .getResultList();
```

#### 읽기 전용 트랜잭션 사용

```java
@Service
@Transactional(readOnly = true)
public class MemberService {
    
    public List<Member> findMembers() {
        return em.createQuery("select m from Member m", Member.class)
                .getResultList();
    }
}
```

**읽기 전용 트랜잭션의 이점:**
- 플러시를 작동하지 않도록 설정 (성능 향상)
- 하이버네이트 세션에서 읽기 전용 모드로 설정

#### 읽기 전용 힌트

```java
@Repository
public class MemberRepository {
    
    public List<Member> findReadOnlyMembers() {
        return em.createQuery("select m from Member m", Member.class)
                .setHint("org.hibernate.readOnly", true)
                .getResultList();
    }
}
```

### 배치 처리

#### 대용량 데이터 처리 시 주의점

```java
// 잘못된 예: 메모리 부족 위험
List<Member> members = em.createQuery(
    "select m from Member m", Member.class)
    .getResultList(); // 10만건의 데이터를 모두 메모리에 로딩

for (Member member : members) {
    member.setAge(member.getAge() + 1);
}
```

#### 배치 처리 해결책

##### 1. 페이징 배치 처리

```java
int pageSize = 100;
for (int i = 0; i < 10000; i += pageSize) {
    List<Member> members = em.createQuery(
        "select m from Member m order by m.id", Member.class)
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

##### 2. 커서 기반 배치 처리 (하이버네이트 scroll)

```java
SessionFactory sessionFactory = entityManagerFactory.unwrap(SessionFactory.class);
Session session = sessionFactory.openSession();
Transaction tx = session.beginTransaction();

ScrollableResults scroll = session.createQuery("select m from Member m")
                                 .scroll(ScrollMode.FORWARD_ONLY);

int count = 0;
while (scroll.next()) {
    Member member = (Member) scroll.get(0);
    member.setAge(member.getAge() + 1);
    
    if (++count % 100 == 0) {
        session.flush();
        session.clear();
    }
}

tx.commit();
session.close();
```

---

## 📝 정리

### 트랜잭션 범위의 영속성 컨텍스트
1. **스프링 컨테이너는 트랜잭션 범위의 영속성 컨텍스트 전략을 기본으로 사용**
2. **트랜잭션이 같으면 같은 영속성 컨텍스트를 사용**
3. **트랜잭션이 다르면 다른 영속성 컨텍스트를 사용**

### 준영속 상태와 지연 로딩
1. **컨트롤러나 뷰 같은 프레젠테이션 계층에서는 엔티티가 준영속 상태**
2. **준영속 상태에서는 변경 감지와 지연 로딩이 동작하지 않음**
3. **뷰가 필요한 엔티티를 미리 로딩하거나 OSIV를 사용해서 해결**

### OSIV (Open Session In View)
1. **스프링의 OSIV는 비즈니스 계층에서만 트랜잭션을 유지하고 프레젠테이션 계층까지 영속성 컨텍스트를 유지**
2. **프레젠테이션 계층에서는 트랜잭션이 없으므로 엔티티 수정 불가**
3. **프레젠테이션 계층에서 지연 로딩 가능**
4. **OSIV를 사용하면 같은 영속성 컨텍스트에서 여러 트랜잭션 사용시 주의**

### 성능 최적화
1. **N+1 문제는 페치 조인으로 해결**
2. **읽기 전용 쿼리에는 읽기 전용 트랜잭션과 힌트 사용**
3. **배치 처리시 영속성 컨텍스트 초기화 주의**
4. **대용량 데이터 처리시 페이징이나 커서 사용**

### 권장사항
- **비즈니스 로직은 서비스 계층에서 끝내고 프레젠테이션 계층에서는 데이터 출력만 담당**
- **프레젠테이션 계층에서 엔티티 수정을 예방하려면 읽기 전용 인터페이스나 DTO 사용**
- **복잡한 화면을 위한 쿼리는 전용 조회 서비스나 DAO를 만들어서 처리**

---

[← 이전: 객체지향 쿼리 언어 (JPQL)](./07-jpql.md) | [목차로 돌아가기](./README.md) | [다음: 스프링 데이터 JPA →](./09-spring-data-jpa.md)