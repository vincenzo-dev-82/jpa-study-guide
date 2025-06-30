# 5. 프록시와 연관관계 관리

## 📖 프록시

### 프록시 기초

#### Member를 조회할 때 Team도 함께 조회해야 할까?
```java
// 회원과 팀 함께 출력
public void printUserAndTeam(String memberId) {
    Member member = em.find(Member.class, memberId);
    Team team = member.getTeam();
    System.out.println("회원 이름: " + member.getUsername());
    System.out.println("소속팀: " + team.getName()); // 팀도 사용
}

// 회원만 출력
public void printUser(String memberId) {
    Member member = em.find(Member.class, memberId);
    System.out.println("회원 이름: " + member.getUsername()); // 팀은 사용하지 않음
}
```

**문제점:** 비즈니스 로직에 따라 Team을 사용하지 않을 수도 있는데 항상 함께 조회하는 것은 효율적이지 않음

### em.find() vs em.getReference()

- **em.find()**: 데이터베이스를 통해서 실제 엔티티 객체 조회
- **em.getReference()**: **데이터베이스 조회를 미루는 가짜(프록시) 엔티티 객체 조회**

```java
// 실제 엔티티 조회
Member member = em.find(Member.class, 1L);
System.out.println("member = " + member.getClass()); // class hellojpa.Member

// 프록시 객체 조회
Member refMember = em.getReference(Member.class, 1L);
System.out.println("refMember = " + refMember.getClass()); // class hellojpa.Member$HibernateProxy$...
```

### 프록시 특징

#### 프록시 객체의 구조
- 실제 클래스를 상속 받아서 만들어짐
- 실제 클래스와 겉 모양이 같음
- 사용하는 입장에서는 진짜 객체인지 프록시 객체인지 구분하지 않고 사용하면 됨(이론상)

#### 프록시 객체의 초기화
```java
Member member = em.getReference(Member.class, 1L);
member.getName(); // 프록시 객체 초기화
```

**프록시 초기화 과정:**
1. `member.getName()` 호출
2. 프록시 객체에 target이 없으면 영속성 컨텍스트에 초기화 요청
3. 영속성 컨텍스트가 DB 조회
4. 실제 Entity 생성
5. 프록시 객체의 target이 실제 Entity를 참조
6. `target.getName()` 호출해서 결과 반환

### 프록시의 특징 정리

1. **프록시 객체는 처음 사용할 때 한 번만 초기화**
2. **프록시 객체를 초기화 할 때, 프록시 객체가 실제 엔티티로 바뀌는 것은 아님**
   - 초기화되면 프록시 객체를 통해서 실제 엔티티에 접근 가능
3. **프록시 객체는 원본 엔티티를 상속받음**
   - 따라서 타입 체크시 주의해야함 (== 비교 실패, 대신 instance of 사용)
4. **영속성 컨텍스트에 찾는 엔티티가 이미 있으면 em.getReference()를 호출해도 실제 엔티티 반환**
5. **영속성 컨텍스트의 도움을 받을 수 없는 준영속 상태일 때, 프록시를 초기화하면 문제 발생**

### 프록시 확인

```java
// 프록시 초기화 여부 확인
PersistenceUnitUtil.isLoaded(Object entity);

// 프록시 클래스 확인 방법
entity.getClass().getName(); // ..javasist.. or HibernateProxy...

// 프록시 강제 초기화
Hibernate.initialize(entity);

// JPA 표준은 강제 초기화 없음
// member.getName() 이런 식으로 강제 호출
```

## ⚡ 즉시 로딩과 지연 로딩

### 지연 로딩 LAZY

```java
@Entity
public class Member {
    
    @Id @GeneratedValue
    private Long id;
    
    @Column(name = "USERNAME")
    private String name;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "TEAM_ID")
    private Team team;
    
    // getter, setter...
}
```

**지연 로딩 사용 예:**
```java
Member member = em.find(Member.class, 1L);
System.out.println("member = " + member.getTeam().getClass()); // 프록시

System.out.println("==================");
member.getTeam().getName(); // 실제 team을 사용하는 시점에 초기화(DB 조회)
System.out.println("==================");
```

### 즉시 로딩 EAGER

```java
@Entity
public class Member {
    
    @Id @GeneratedValue
    private Long id;
    
    @Column(name = "USERNAME")
    private String name;
    
    @ManyToOne(fetch = FetchType.EAGER)
    @JoinColumn(name = "TEAM_ID")
    private Team team;
    
    // getter, setter...
}
```

**즉시 로딩 사용 예:**
```java
Member member = em.find(Member.class, 1L);
System.out.println("member = " + member.getTeam().getClass()); // 실제 Team 객체
```

### 프록시와 즉시로딩 주의

#### 1. 가급적 지연 로딩만 사용(특히 실무에서)
#### 2. 즉시 로딩을 적용하면 예상하지 못한 SQL이 발생
#### 3. 즉시 로딩은 JPQL에서 N+1 문제를 일으킨다

```java
// JPQL
List<Member> members = em.createQuery("select m from Member m", Member.class)
                        .getResultList();

// SQL: select * from Member
// SQL: select * from Team where TEAM_ID = xxx (첫 번째 회원의 팀 조회)
// SQL: select * from Team where TEAM_ID = xxx (두 번째 회원의 팀 조회)
// SQL: select * from Team where TEAM_ID = xxx (세 번째 회원의 팀 조회)
// ...
```

#### 4. @ManyToOne, @OneToOne은 기본이 즉시 로딩
→ LAZY로 설정

#### 5. @OneToMany, @ManyToMany는 기본이 지연 로딩

### N+1 문제 해결 방법

#### 1. 페치 조인 사용
```java
List<Member> members = em.createQuery(
    "select m from Member m join fetch m.team", Member.class)
    .getResultList();
```

#### 2. 엔티티 그래프 사용
```java
@Entity
@NamedEntityGraph(name = "Member.all", attributeNodes = @NamedAttributeNode("team"))
public class Member {
    // ...
}

// 사용
List<Member> members = em.createQuery("select m from Member m", Member.class)
    .setHint("javax.persistence.fetchgraph", em.getEntityGraph("Member.all"))
    .getResultList();
```

#### 3. 배치 사이즈 조정
```xml
<property name="hibernate.default_batch_fetch_size" value="100"/>
```

## 🔄 영속성 전이: CASCADE

### CASCADE란?
특정 엔티티를 영속 상태로 만들 때 연관된 엔티티도 함께 영속 상태로 만들고 싶을 때 사용

### CASCADE 종류

- **ALL**: 모든 CASCADE 적용
- **PERSIST**: 영속
- **REMOVE**: 삭제
- **MERGE**: 병합
- **REFRESH**: REFRESH
- **DETACH**: DETACH

### CASCADE.PERSIST

```java
@Entity
public class Parent {
    
    @OneToMany(mappedBy = "parent", cascade = CascadeType.PERSIST)
    private List<Child> childList = new ArrayList<>();
    
    public void addChild(Child child) {
        childList.add(child);
        child.setParent(this);
    }
    
    // getter, setter...
}

// 사용
Child child1 = new Child();
Child child2 = new Child();

Parent parent = new Parent();
parent.addChild(child1);
parent.addChild(child2);

em.persist(parent); // parent만 persist 해도 child들도 함께 저장됨
```

### CASCADE.REMOVE

```java
@Entity
public class Parent {
    
    @OneToMany(mappedBy = "parent", cascade = CascadeType.ALL)
    private List<Child> childList = new ArrayList<>();
    
    // getter, setter...
}

// 사용
Parent findParent = em.find(Parent.class, id);
em.remove(findParent); // parent 삭제 시 child들도 함께 삭제됨
```

### CASCADE 주의사항

1. **영속성 전이는 연관관계를 매핑하는 것과 아무 관련이 없음**
2. **엔티티를 영속화할 때 연관된 엔티티도 함께 영속화하는 편의를 제공할 뿐**
3. **CASCADE는 소유자가 하나일 때만 사용해야 함**
4. **Parent와 Child의 라이프사이클이 거의 유사할 때 사용**
5. **단일 소유자일 때 사용 (특정 엔티티가 개인 소유할 때)**

## 🗑️ 고아 객체

### 고아 객체 제거
부모 엔티티와 연관관계가 끊어진 자식 엔티티를 자동으로 삭제

```java
@Entity
public class Parent {
    
    @OneToMany(mappedBy = "parent", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Child> childList = new ArrayList<>();
    
    // getter, setter...
}

// 사용
Parent parent1 = em.find(Parent.class, id);
parent1.getChildList().remove(0); // 자식 엔티티를 컬렉션에서 제거

// DELETE FROM CHILD WHERE ID = ?
```

### 고아 객체 주의사항

1. **참조가 제거된 엔티티는 다른 곳에서 참조하지 않는 고아 객체로 보고 삭제하는 기능**
2. **참조하는 곳이 하나일 때 사용해야 함**
3. **특정 엔티티가 개인 소유할 때 사용**
4. **@OneToOne, @OneToMany만 가능**
5. **개념적으로 부모를 제거하면 자식은 고아가 됨. 따라서 고아 객체 제거 기능을 활성화 하면, 부모를 제거할 때 자식도 함께 제거됨. 이것은 CascadeType.REMOVE처럼 동작**

## 🔄 영속성 전이 + 고아 객체, 생명주기

### CascadeType.ALL + orphanRemoval=true

```java
@Entity
public class Parent {
    
    @OneToMany(mappedBy = "parent", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Child> childList = new ArrayList<>();
    
    // getter, setter...
}
```

### 특징
- 스스로 생명주기를 관리하는 엔티티는 em.persist()로 영속화, em.remove()로 제거
- 두 옵션을 모두 활성화 하면 부모 엔티티를 통해서 자식의 생명 주기를 관리할 수 있음
- 도메인 주도 설계(DDD)의 Aggregate Root개념을 구현할 때 유용

### 실무 예제
```java
@Entity
public class Board {
    
    @Id @GeneratedValue
    private Long id;
    
    private String title;
    
    @OneToMany(mappedBy = "board", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Comment> comments = new ArrayList<>();
    
    @OneToMany(mappedBy = "board", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Attachment> attachments = new ArrayList<>();
    
    // 게시글을 삭제하면 댓글과 첨부파일도 함께 삭제
    // 댓글이나 첨부파일을 리스트에서 제거하면 DB에서도 삭제
    
    // getter, setter...
}
```

## 💡 실전 예제 - 연관관계 관리

### 글로벌 페치 전략 설정

```java
// 모든 연관관계를 지연 로딩으로
@Entity
public class Order {
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "MEMBER_ID")
    private Member member;
    
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL)
    private List<OrderItem> orderItems = new ArrayList<>();
    
    @OneToOne(fetch = FetchType.LAZY, cascade = CascadeType.ALL)
    @JoinColumn(name = "DELIVERY_ID")
    private Delivery delivery;
    
    // getter, setter...
}

@Entity
public class OrderItem {
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "ORDER_ID")
    private Order order;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "ITEM_ID")
    private Item item;
    
    // getter, setter...
}

@Entity
public class Category {
    
    @ManyToMany(fetch = FetchType.LAZY)
    @JoinTable(name = "CATEGORY_ITEM",
            joinColumns = @JoinColumn(name = "CATEGORY_ID"),
            inverseJoinColumns = @JoinColumn(name = "ITEM_ID"))
    private List<Item> items = new ArrayList<>();
    
    // getter, setter...
}
```

### 영속성 전이 설정

```java
@Entity
public class Order {
    
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL)
    private List<OrderItem> orderItems = new ArrayList<>();
    
    @OneToOne(fetch = FetchType.LAZY, cascade = CascadeType.ALL)
    @JoinColumn(name = "DELIVERY_ID")
    private Delivery delivery;
    
    // 주문 생성 시 주문상품과 배송정보도 함께 저장
    // 주문 삭제 시 주문상품과 배송정보도 함께 삭제
    
    // getter, setter...
}
```

---

## 📝 정리

### 프록시
1. **em.find()**: 실제 엔티티 객체 조회
2. **em.getReference()**: 프록시 객체 조회
3. **프록시 객체는 실제 객체의 참조(target)을 보관**
4. **프록시 객체를 호출하면 프록시 객체는 실제 객체의 메소드 호출**

### 즉시 로딩과 지연 로딩
1. **지연 로딩(LAZY)을 사용해서 프록시로 조회**
2. **즉시 로딩(EAGER)을 사용해서 함께 조회**
3. **실무에서는 가급적 지연 로딩만 사용**
4. **즉시 로딩은 JPQL에서 N+1 문제를 일으킨다**

### 영속성 전이
1. **CASCADE로 영속성 전이**
2. **고아 객체 제거: orphanRemoval = true**
3. **CascadeType.ALL + orphanRemoval=true**
4. **생명주기를 관리할 때 유용**

### 권장사항
- **모든 연관관계에 지연 로딩 사용**
- **실무에서 즉시 로딩 금지**
- **JPQL fetch 조인이나 엔티티 그래프 기능 사용**
- **영속성 전이는 소유자가 하나일 때만 사용**

---

[← 이전: 고급 매핑](./04-advanced-mapping.md) | [목차로 돌아가기](./README.md) | [다음: 값 타입 →](./06-value-types.md)