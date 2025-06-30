# 4. 고급 매핑

## 📖 개요

이 장에서는 상속관계 매핑, @MappedSuperclass, 복합 키와 식별 관계 매핑, 조인 테이블, 엔티티 하나에 여러 테이블 매핑 등 고급 매핑을 다룬다.

## 🌳 상속관계 매핑

### 상속관계 매핑이란?

- 관계형 데이터베이스는 상속 관계가 없다
- 슈퍼타입 서브타입 관계라는 모델링 기법이 객체 상속과 유사하다
- 상속관계 매핑: 객체의 상속 구조와 DB의 슈퍼타입 서브타입 관계를 매핑

### 상속관계 매핑 전략

슈퍼타입 서브타입 논리 모델을 실제 물리 모델로 구현하는 방법은 3가지이다.

#### 1. 조인 전략 (JOINED)

**각각을 테이블로 변환 → 조인으로 조회**

```java
@Entity
@Inheritance(strategy = InheritanceType.JOINED)
@DiscriminatorColumn(name = "DTYPE")  // 구분 컬럼
public abstract class Item {
    @Id @GeneratedValue
    @Column(name = "ITEM_ID")
    private Long id;
    
    private String name;
    private int price;
    
    // getter, setter...
}

@Entity
@DiscriminatorValue("A")  // 구분 값
public class Album extends Item {
    private String artist;
    
    // getter, setter...
}

@Entity
@DiscriminatorValue("M")
public class Movie extends Item {
    private String director;
    private String actor;
    
    // getter, setter...
}

@Entity
@DiscriminatorValue("B")
public class Book extends Item {
    private String author;
    private String isbn;
    
    // getter, setter...
}
```

**생성되는 테이블:**

```sql
-- 부모 테이블
CREATE TABLE ITEM (
    ITEM_ID BIGINT NOT NULL,
    NAME VARCHAR(255),
    PRICE INTEGER,
    DTYPE VARCHAR(31),  -- 구분 컬럼
    PRIMARY KEY (ITEM_ID)
);

-- 자식 테이블들
CREATE TABLE ALBUM (
    ITEM_ID BIGINT NOT NULL,
    ARTIST VARCHAR(255),
    PRIMARY KEY (ITEM_ID),
    FOREIGN KEY (ITEM_ID) REFERENCES ITEM(ITEM_ID)
);

CREATE TABLE MOVIE (
    ITEM_ID BIGINT NOT NULL,
    DIRECTOR VARCHAR(255),
    ACTOR VARCHAR(255),
    PRIMARY KEY (ITEM_ID),
    FOREIGN KEY (ITEM_ID) REFERENCES ITEM(ITEM_ID)
);

CREATE TABLE BOOK (
    ITEM_ID BIGINT NOT NULL,
    AUTHOR VARCHAR(255),
    ISBN VARCHAR(255),
    PRIMARY KEY (ITEM_ID),
    FOREIGN KEY (ITEM_ID) REFERENCES ITEM(ITEM_ID)
);
```

**조인 전략 사용:**

```java
// 저장
Movie movie = new Movie();
movie.setName("바람과 함께 사라지다");
movie.setPrice(10000);
movie.setDirector("감독1");
movie.setActor("배우1");

em.persist(movie);

// 조회
Movie findMovie = em.find(Movie.class, movie.getId());

// 실행되는 SQL
// SELECT m.ITEM_ID, i.NAME, i.PRICE, m.DIRECTOR, m.ACTOR
// FROM MOVIE m
// INNER JOIN ITEM i ON m.ITEM_ID = i.ITEM_ID
// WHERE m.ITEM_ID = ?
```

**장점:**
- 테이블 정규화
- 외래 키 참조 무결성 제약조건 활용가능
- 저장공간 효율화

**단점:**
- 조회시 조인을 많이 사용, 성능 저하
- 조회 쿼리가 복잡함
- 데이터 저장시 INSERT SQL 2번 호출

#### 2. 단일 테이블 전략 (SINGLE_TABLE)

**통합 테이블로 변환**

```java
@Entity
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name = "DTYPE")
public abstract class Item {
    @Id @GeneratedValue
    @Column(name = "ITEM_ID")
    private Long id;
    
    private String name;
    private int price;
    
    // getter, setter...
}

@Entity
@DiscriminatorValue("A")
public class Album extends Item {
    private String artist;
    
    // getter, setter...
}

@Entity
@DiscriminatorValue("M")
public class Movie extends Item {
    private String director;
    private String actor;
    
    // getter, setter...
}

@Entity
@DiscriminatorValue("B")
public class Book extends Item {
    private String author;
    private String isbn;
    
    // getter, setter...
}
```

**생성되는 테이블:**

```sql
CREATE TABLE ITEM (
    ITEM_ID BIGINT NOT NULL,
    DTYPE VARCHAR(31) NOT NULL,  -- 구분 컬럼 (필수)
    NAME VARCHAR(255),
    PRICE INTEGER,
    -- Album 필드들
    ARTIST VARCHAR(255),
    -- Movie 필드들
    DIRECTOR VARCHAR(255),
    ACTOR VARCHAR(255),
    -- Book 필드들
    AUTHOR VARCHAR(255),
    ISBN VARCHAR(255),
    PRIMARY KEY (ITEM_ID)
);
```

**단일 테이블 전략 사용:**

```java
// 저장
Movie movie = new Movie();
movie.setName("바람과 함께 사라지다");
movie.setPrice(10000);
movie.setDirector("감독1");
movie.setActor("배우1");

em.persist(movie);

// 조회
Movie findMovie = em.find(Movie.class, movie.getId());

// 실행되는 SQL (단순함!)
// SELECT ITEM_ID, DTYPE, NAME, PRICE, DIRECTOR, ACTOR
// FROM ITEM
// WHERE ITEM_ID = ? AND DTYPE = 'M'
```

**장점:**
- 조인이 필요 없으므로 일반적으로 조회 성능이 빠름
- 조회 쿼리가 단순함

**단점:**
- 자식 엔티티가 매핑한 컬럼은 모두 null 허용
- 단일 테이블에 모든 것을 저장하므로 테이블이 커질 수 있음
- 상황에 따라서 조회 성능이 오히려 느려질 수 있음

#### 3. 구현 클래스마다 테이블 전략 (TABLE_PER_CLASS)

**서브타입 테이블로 변환**

```java
@Entity
@Inheritance(strategy = InheritanceType.TABLE_PER_CLASS)
public abstract class Item {
    @Id @GeneratedValue
    @Column(name = "ITEM_ID")
    private Long id;
    
    private String name;
    private int price;
    
    // getter, setter...
}

@Entity
public class Album extends Item {
    private String artist;
    
    // getter, setter...
}

@Entity
public class Movie extends Item {
    private String director;
    private String actor;
    
    // getter, setter...
}

@Entity
public class Book extends Item {
    private String author;
    private String isbn;
    
    // getter, setter...
}
```

**생성되는 테이블:**

```sql
-- 부모 테이블 없음!

CREATE TABLE ALBUM (
    ITEM_ID BIGINT NOT NULL,
    NAME VARCHAR(255),     -- 부모 필드들이 각 테이블에 포함
    PRICE INTEGER,
    ARTIST VARCHAR(255),
    PRIMARY KEY (ITEM_ID)
);

CREATE TABLE MOVIE (
    ITEM_ID BIGINT NOT NULL,
    NAME VARCHAR(255),     -- 부모 필드들이 각 테이블에 포함
    PRICE INTEGER,
    DIRECTOR VARCHAR(255),
    ACTOR VARCHAR(255),
    PRIMARY KEY (ITEM_ID)
);

CREATE TABLE BOOK (
    ITEM_ID BIGINT NOT NULL,
    NAME VARCHAR(255),     -- 부모 필드들이 각 테이블에 포함
    PRICE INTEGER,
    AUTHOR VARCHAR(255),
    ISBN VARCHAR(255),
    PRIMARY KEY (ITEM_ID)
);
```

**구현 클래스마다 테이블 전략의 문제점:**

```java
// 부모 타입으로 조회시 UNION SQL 사용
Item item = em.find(Item.class, movie.getId());

// 실행되는 SQL (복잡함!)
// SELECT ITEM_ID, NAME, PRICE, null as ARTIST, null as DIRECTOR, null as ACTOR, null as AUTHOR, null as ISBN
// FROM ALBUM
// WHERE ITEM_ID = ?
// UNION
// SELECT ITEM_ID, NAME, PRICE, null as ARTIST, DIRECTOR, ACTOR, null as AUTHOR, null as ISBN
// FROM MOVIE
// WHERE ITEM_ID = ?
// UNION
// SELECT ITEM_ID, NAME, PRICE, null as ARTIST, null as DIRECTOR, null as ACTOR, AUTHOR, ISBN
// FROM BOOK
// WHERE ITEM_ID = ?
```

**장점:**
- 서브 타입을 명확하게 구분해서 처리할 때 효과적
- not null 제약조건 사용 가능

**단점:**
- 여러 자식 테이블을 함께 조회할 때 성능이 느림(UNION SQL 필요)
- 자식 테이블을 통합해서 쿼리하기 어려움

**⚠️ 이 전략은 데이터베이스 설계자와 ORM 전문가 둘 다 추천 X**

### 주요 어노테이션

#### @Inheritance

상속 매핑 전략을 지정한다.

```java
@Inheritance(strategy = InheritanceType.JOINED)      // 조인 전략
@Inheritance(strategy = InheritanceType.SINGLE_TABLE) // 단일 테이블 전략
@Inheritance(strategy = InheritanceType.TABLE_PER_CLASS) // 구현 클래스마다 테이블 전략
```

#### @DiscriminatorColumn

부모 클래스에 구분 컬럼을 지정한다.

```java
@DiscriminatorColumn(
    name = "DTYPE",           // 구분 컬럼명 (기본값: DTYPE)
    discriminatorType = DiscriminatorType.STRING,  // 구분 컬럼 타입 (기본값: STRING)
    columnDefinition = "VARCHAR(100)"  // 컬럼 정의
)
```

#### @DiscriminatorValue

자식 클래스에 구분 컬럼 값을 지정한다.

```java
@DiscriminatorValue("XXX")  // 기본값: 엔티티 이름
```

## 🗂️ @MappedSuperclass

### @MappedSuperclass란?

- **공통 매핑 정보가 필요할 때 사용** (id, name)
- 상속관계 매핑이 아님
- 엔티티가 아니므로 테이블과 매핑되지 않음
- 부모 클래스를 상속 받는 **자식 클래스에 매핑 정보만 제공**
- 조회, 검색 불가 (em.find(BaseEntity) 불가)
- 직접 생성해서 사용할 일이 없으므로 **추상 클래스 권장**

### 공통 정보를 관리하는 부모 클래스

```java
@MappedSuperclass
public abstract class BaseEntity {
    
    @Column(name = "INSERT_MEMBER")
    private String createdBy;
    
    private LocalDateTime createdDate;
    
    @Column(name = "UPDATE_MEMBER")
    private String lastModifiedBy;
    
    private LocalDateTime lastModifiedDate;
    
    // getter, setter...
}

@Entity
public class Member extends BaseEntity {
    @Id @GeneratedValue
    @Column(name = "MEMBER_ID")
    private Long id;
    
    private String name;
    
    // BaseEntity의 필드들이 Member 테이블에 포함됨
}

@Entity
public class Team extends BaseEntity {
    @Id @GeneratedValue
    @Column(name = "TEAM_ID")
    private Long id;
    
    private String name;
    
    // BaseEntity의 필드들이 Team 테이블에 포함됨
}
```

**생성되는 테이블:**

```sql
CREATE TABLE MEMBER (
    MEMBER_ID BIGINT NOT NULL,
    NAME VARCHAR(255),
    INSERT_MEMBER VARCHAR(255),
    CREATED_DATE TIMESTAMP,
    UPDATE_MEMBER VARCHAR(255),
    LAST_MODIFIED_DATE TIMESTAMP,
    PRIMARY KEY (MEMBER_ID)
);

CREATE TABLE TEAM (
    TEAM_ID BIGINT NOT NULL,
    NAME VARCHAR(255),
    INSERT_MEMBER VARCHAR(255),
    CREATED_DATE TIMESTAMP,
    UPDATE_MEMBER VARCHAR(255),
    LAST_MODIFIED_DATE TIMESTAMP,
    PRIMARY KEY (TEAM_ID)
);
```

### 속성 재정의

```java
@Entity
@AttributeOverrides({
    @AttributeOverride(name = "createdBy", column = @Column(name = "MEMBER_INSERT_BY")),
    @AttributeOverride(name = "createdDate", column = @Column(name = "MEMBER_INSERT_DATE"))
})
public class Member extends BaseEntity {
    @Id @GeneratedValue
    private Long id;
    
    private String name;
}
```

### @MappedSuperclass 특징

- 테이블과 관계 없고, 단순히 엔티티가 공통으로 사용하는 매핑 정보를 모으는 역할
- 주로 등록일, 수정일, 등록자, 수정자 같은 전체 엔티티에서 공통으로 적용하는 정보를 모을 때 사용

### 실전 예제

```java
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class BaseTimeEntity {
    
    @CreatedDate
    @Column(updatable = false)
    private LocalDateTime createdDate;
    
    @LastModifiedDate
    private LocalDateTime lastModifiedDate;
    
    // getter, setter...
}

@MappedSuperclass
public abstract class BaseEntity extends BaseTimeEntity {
    
    @CreatedBy
    @Column(updatable = false)
    private String createdBy;
    
    @LastModifiedBy
    private String lastModifiedBy;
    
    // getter, setter...
}

// 실제 엔티티들
@Entity
public class Member extends BaseEntity {
    // Member 고유 필드들...
}

@Entity
public class Order extends BaseEntity {
    // Order 고유 필드들...
}
```

## 🔑 복합 키와 식별 관계 매핑

### 식별 관계 vs 비식별 관계

#### 식별 관계 (Identifying Relationship)

- 부모 테이블의 기본 키를 내려받아서 자식 테이블의 **기본 키 + 외래 키**로 사용하는 관계

```sql
-- 식별 관계 예제
CREATE TABLE PARENT (
    ID VARCHAR(255) NOT NULL,
    NAME VARCHAR(255),
    PRIMARY KEY (ID)
);

CREATE TABLE CHILD (
    PARENT_ID VARCHAR(255) NOT NULL,  -- 기본 키이면서 외래 키
    CHILD_ID VARCHAR(255) NOT NULL,
    NAME VARCHAR(255),
    PRIMARY KEY (PARENT_ID, CHILD_ID),
    FOREIGN KEY (PARENT_ID) REFERENCES PARENT(ID)
);

CREATE TABLE GRANDCHILD (
    PARENT_ID VARCHAR(255) NOT NULL,     -- 기본 키이면서 외래 키
    CHILD_ID VARCHAR(255) NOT NULL,      -- 기본 키이면서 외래 키
    GRANDCHILD_ID VARCHAR(255) NOT NULL,
    NAME VARCHAR(255),
    PRIMARY KEY (PARENT_ID, CHILD_ID, GRANDCHILD_ID),
    FOREIGN KEY (PARENT_ID, CHILD_ID) REFERENCES CHILD(PARENT_ID, CHILD_ID)
);
```

#### 비식별 관계 (Non-Identifying Relationship)

- 부모 테이블의 기본 키를 받아서 자식 테이블의 **외래 키로만** 사용하는 관계

```sql
-- 비식별 관계 예제
CREATE TABLE PARENT (
    ID BIGINT NOT NULL,
    NAME VARCHAR(255),
    PRIMARY KEY (ID)
);

CREATE TABLE CHILD (
    ID BIGINT NOT NULL,        -- 자체 기본 키
    PARENT_ID BIGINT,          -- 외래 키
    NAME VARCHAR(255),
    PRIMARY KEY (ID),
    FOREIGN KEY (PARENT_ID) REFERENCES PARENT(ID)
);
```

**비식별 관계는 외래 키에 NULL을 허용하는지에 따라 나뉜다:**
- **필수적 비식별 관계**: 외래 키에 NULL을 허용하지 않음 (NotNull)
- **선택적 비식별 관계**: 외래 키에 NULL을 허용 (Nullable)

### 복합 키: 비식별 관계 매핑

#### @IdClass

```java
// 식별자 클래스
public class MemberId implements Serializable {
    private String id1;
    private String id2;
    
    public MemberId() {}
    
    public MemberId(String id1, String id2) {
        this.id1 = id1;
        this.id2 = id2;
    }
    
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        MemberId memberId = (MemberId) o;
        return Objects.equals(id1, memberId.id1) && 
               Objects.equals(id2, memberId.id2);
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(id1, id2);
    }
    
    // getter, setter...
}

// 엔티티 클래스
@Entity
@IdClass(MemberId.class)
public class Member {
    @Id
    private String id1;
    
    @Id
    private String id2;
    
    private String username;
    
    // getter, setter...
}
```

**@IdClass 사용:**

```java
// 저장
Member member = new Member();
member.setId1("myId1");
member.setId2("myId2");
member.setUsername("회원1");

em.persist(member);

// 조회
MemberId memberId = new MemberId("myId1", "myId2");
Member findMember = em.find(Member.class, memberId);
```

**@IdClass 조건:**
- 식별자 클래스의 속성명과 엔티티에서 사용하는 식별자의 속성명이 같아야 함
- Serializable 인터페이스를 구현해야 함
- equals, hashCode를 구현해야 함
- 기본 생성자가 있어야 함
- 식별자 클래스는 public이어야 함

#### @EmbeddedId

```java
// 식별자 클래스
@Embeddable
public class MemberId implements Serializable {
    private String id1;
    private String id2;
    
    public MemberId() {}
    
    public MemberId(String id1, String id2) {
        this.id1 = id1;
        this.id2 = id2;
    }
    
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        MemberId memberId = (MemberId) o;
        return Objects.equals(id1, memberId.id1) && 
               Objects.equals(id2, memberId.id2);
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(id1, id2);
    }
    
    // getter, setter...
}

// 엔티티 클래스
@Entity
public class Member {
    @EmbeddedId
    private MemberId id;
    
    private String username;
    
    // getter, setter...
}
```

**@EmbeddedId 사용:**

```java
// 저장
MemberId memberId = new MemberId("myId1", "myId2");
Member member = new Member();
member.setId(memberId);
member.setUsername("회원1");

em.persist(member);

// 조회
MemberId memberId = new MemberId("myId1", "myId2");
Member findMember = em.find(Member.class, memberId);
```

**@EmbeddedId 조건:**
- @Embeddable 어노테이션을 붙여주어야 함
- Serializable 인터페이스를 구현해야 함
- equals, hashCode를 구현해야 함
- 기본 생성자가 있어야 함
- 식별자 클래스는 public이어야 함

### 복합 키: 식별 관계 매핑

#### @IdClass와 식별 관계

```java
// 부모
@Entity
public class Parent {
    @Id @Column(name = "PARENT_ID")
    private String id;
    
    private String name;
    
    // getter, setter...
}

// 자식 - 식별자 클래스
public class ChildId implements Serializable {
    private String parent;  // Child.parent 매핑
    private String childId; // Child.childId 매핑
    
    // equals, hashCode, 기본생성자...
}

// 자식
@Entity
@IdClass(ChildId.class)
public class Child {
    @Id
    @ManyToOne
    @JoinColumn(name = "PARENT_ID")
    public Parent parent;
    
    @Id @Column(name = "CHILD_ID")
    private String childId;
    
    private String name;
    
    // getter, setter...
}

// 손자 - 식별자 클래스
public class GrandChildId implements Serializable {
    private ChildId child;  // GrandChild.child 매핑
    private String id;      // GrandChild.id 매핑
    
    // equals, hashCode, 기본생성자...
}

// 손자
@Entity
@IdClass(GrandChildId.class)
public class GrandChild {
    @Id
    @ManyToOne
    @JoinColumns({
        @JoinColumn(name = "PARENT_ID"),
        @JoinColumn(name = "CHILD_ID")
    })
    private Child child;
    
    @Id @Column(name = "GRANDCHILD_ID")
    private String id;
    
    private String name;
    
    // getter, setter...
}
```

#### @EmbeddedId와 식별 관계

```java
// 부모
@Entity
public class Parent {
    @Id @Column(name = "PARENT_ID")
    private String id;
    
    private String name;
    
    // getter, setter...
}

// 자식 - 식별자 클래스
@Embeddable
public class ChildId implements Serializable {
    private String parentId;  // @MapsId("parentId")로 매핑
    
    @Column(name = "CHILD_ID")
    private String id;
    
    // equals, hashCode, 기본생성자...
}

// 자식
@Entity
public class Child {
    @EmbeddedId
    private ChildId id;
    
    @MapsId("parentId")  // ChildId.parentId 매핑
    @ManyToOne
    @JoinColumn(name = "PARENT_ID")
    public Parent parent;
    
    private String name;
    
    // getter, setter...
}
```

### 비식별 관계로 구현

복잡한 식별 관계보다는 **비식별 관계를 사용하고 기본 키는 Long 타입의 대리 키를 사용**하는 것이 좋다.

```java
// 부모
@Entity
public class Parent {
    @Id @GeneratedValue
    @Column(name = "PARENT_ID")
    private Long id;
    
    private String name;
    
    // getter, setter...
}

// 자식
@Entity
public class Child {
    @Id @GeneratedValue
    @Column(name = "CHILD_ID")
    private Long id;
    
    @ManyToOne
    @JoinColumn(name = "PARENT_ID")
    private Parent parent;
    
    private String name;
    
    // getter, setter...
}

// 손자
@Entity
public class GrandChild {
    @Id @GeneratedValue
    @Column(name = "GRANDCHILD_ID")
    private Long id;
    
    @ManyToOne
    @JoinColumn(name = "CHILD_ID")
    private Child child;
    
    private String name;
    
    // getter, setter...
}
```

### 일대일 식별 관계

```java
// 부모
@Entity
public class Board {
    @Id @GeneratedValue
    @Column(name = "BOARD_ID")
    private Long id;
    
    private String title;
    
    @OneToOne(mappedBy = "board")
    private BoardDetail boardDetail;
    
    // getter, setter...
}

// 자식
@Entity
public class BoardDetail {
    @Id
    private Long boardId;
    
    @MapsId  // BoardDetail.boardId 매핑
    @OneToOne
    @JoinColumn(name = "BOARD_ID")
    private Board board;
    
    private String content;
    
    // getter, setter...
}
```

**사용:**

```java
Board board = new Board();
board.setTitle("제목");
em.persist(board);

BoardDetail boardDetail = new BoardDetail();
boardDetail.setContent("내용");
boardDetail.setBoard(board);
em.persist(boardDetail);
```

## 🔗 조인 테이블

### 조인 테이블이란?

데이터베이스 테이블의 연관관계를 설계하는 방법은 크게 2가지다.

#### 조인 컬럼 사용 (외래 키)

```sql
CREATE TABLE MEMBER (
    MEMBER_ID BIGINT NOT NULL,
    TEAM_ID BIGINT,  -- 외래 키
    USERNAME VARCHAR(255),
    PRIMARY KEY (MEMBER_ID)
);

CREATE TABLE TEAM (
    TEAM_ID BIGINT NOT NULL,
    NAME VARCHAR(255),
    PRIMARY KEY (TEAM_ID)
);
```

#### 조인 테이블 사용 (테이블 사용)

```sql
CREATE TABLE MEMBER (
    MEMBER_ID BIGINT NOT NULL,
    USERNAME VARCHAR(255),
    PRIMARY KEY (MEMBER_ID)
);

CREATE TABLE TEAM (
    TEAM_ID BIGINT NOT NULL,
    NAME VARCHAR(255),
    PRIMARY KEY (TEAM_ID)
);

CREATE TABLE MEMBER_TEAM (  -- 조인 테이블
    MEMBER_ID BIGINT NOT NULL,
    TEAM_ID BIGINT NOT NULL,
    PRIMARY KEY (MEMBER_ID, TEAM_ID)
);
```

### 일대일 조인 테이블

```java
// 부모
@Entity
public class Parent {
    @Id @GeneratedValue
    @Column(name = "PARENT_ID")
    private Long id;
    
    private String name;
    
    @OneToOne
    @JoinTable(name = "PARENT_CHILD",
               joinColumns = @JoinColumn(name = "PARENT_ID"),
               inverseJoinColumns = @JoinColumn(name = "CHILD_ID"))
    private Child child;
    
    // getter, setter...
}

// 자식
@Entity
public class Child {
    @Id @GeneratedValue
    @Column(name = "CHILD_ID")
    private Long id;
    
    private String name;
    
    @OneToOne(mappedBy = "child")
    private Parent parent;
    
    // getter, setter...
}
```

### 일대다 조인 테이블

```java
// 부모
@Entity
public class Parent {
    @Id @GeneratedValue
    @Column(name = "PARENT_ID")
    private Long id;
    
    private String name;
    
    @OneToMany
    @JoinTable(name = "PARENT_CHILD",
               joinColumns = @JoinColumn(name = "PARENT_ID"),
               inverseJoinColumns = @JoinColumn(name = "CHILD_ID"))
    private List<Child> child = new ArrayList<>();
    
    // getter, setter...
}

// 자식
@Entity
public class Child {
    @Id @GeneratedValue
    @Column(name = "CHILD_ID")
    private Long id;
    
    private String name;
    
    // getter, setter...
}
```

### 다대일 조인 테이블

```java
// 부모
@Entity
public class Parent {
    @Id @GeneratedValue
    @Column(name = "PARENT_ID")
    private Long id;
    
    private String name;
    
    @OneToMany(mappedBy = "parent")
    private List<Child> child = new ArrayList<>();
    
    // getter, setter...
}

// 자식
@Entity
public class Child {
    @Id @GeneratedValue
    @Column(name = "CHILD_ID")
    private Long id;
    
    private String name;
    
    @ManyToOne(optional = false)
    @JoinTable(name = "PARENT_CHILD",
               joinColumns = @JoinColumn(name = "CHILD_ID"),
               inverseJoinColumns = @JoinColumn(name = "PARENT_ID"))
    private Parent parent;
    
    // getter, setter...
}
```

### 다대다 조인 테이블

```java
// 부모
@Entity
public class Parent {
    @Id @GeneratedValue
    @Column(name = "PARENT_ID")
    private Long id;
    
    private String name;
    
    @ManyToMany
    @JoinTable(name = "PARENT_CHILD",
               joinColumns = @JoinColumn(name = "PARENT_ID"),
               inverseJoinColumns = @JoinColumn(name = "CHILD_ID"))
    private List<Child> child = new ArrayList<>();
    
    // getter, setter...
}

// 자식
@Entity
public class Child {
    @Id @GeneratedValue
    @Column(name = "CHILD_ID")
    private Long id;
    
    private String name;
    
    @ManyToMany(mappedBy = "child")
    private List<Parent> parents = new ArrayList<>();
    
    // getter, setter...
}
```

## 🗃️ 엔티티 하나에 여러 테이블 매핑

### @SecondaryTable

잘 사용하지는 않지만 @SecondaryTable을 사용하면 한 엔티티에 여러 테이블을 매핑할 수 있다.

```java
@Entity
@Table(name = "BOARD")
@SecondaryTable(name = "BOARD_DETAIL",
    pkJoinColumns = @PrimaryKeyJoinColumn(name = "BOARD_DETAIL_ID"))
public class Board {
    @Id @GeneratedValue
    @Column(name = "BOARD_ID")
    private Long id;
    
    private String title;
    
    @Column(table = "BOARD_DETAIL")
    private String content;
    
    // getter, setter...
}
```

**생성되는 테이블:**

```sql
CREATE TABLE BOARD (
    BOARD_ID BIGINT NOT NULL,
    TITLE VARCHAR(255),
    PRIMARY KEY (BOARD_ID)
);

CREATE TABLE BOARD_DETAIL (
    BOARD_DETAIL_ID BIGINT NOT NULL,
    CONTENT CLOB,
    PRIMARY KEY (BOARD_DETAIL_ID),
    FOREIGN KEY (BOARD_DETAIL_ID) REFERENCES BOARD(BOARD_ID)
);
```

**@SecondaryTable의 단점:**
- 항상 두 테이블을 조회하므로 최적화하기 어려움
- 테이블에 컬럼이 많을 때 테이블을 분리해도 조회할 때는 두 테이블을 모두 조회

**대안:**
- 테이블당 엔티티를 각각 만들어서 일대일 매핑하는 것을 권장

---

## 📝 정리

### 상속관계 매핑

1. **조인 전략 (JOINED)**: 가장 정규화된 방법, 조회시 조인 필요
2. **단일 테이블 전략 (SINGLE_TABLE)**: 성능이 좋고 단순, 자식 엔티티 컬럼들이 null 허용이 되어야 함
3. **구현 클래스마다 테이블 전략 (TABLE_PER_CLASS)**: 추천하지 않음

**권장사항:**
- 조인 전략이 정석
- 단순하고 확장 가능성이 낮으면 단일 테이블 전략

### @MappedSuperclass

- 테이블과 매핑되지 않고 자식 클래스에 엔티티의 매핑 정보를 상속하기 위해 사용
- @MappedSuperclass로 지정한 클래스는 엔티티가 아니므로 em.find()나 JPQL에서 사용할 수 없음
- 직접 생성해서 사용할 일이 없으므로 추상 클래스로 만들 것을 권장

### 복합 키와 식별 관계 매핑

- **식별 관계는 기본 키와 외래 키를 같이 매핑**해야 하므로 식별자 클래스가 필요
- **비식별 관계에서는 기본 키와 외래 키를 따로 매핑**하므로 단순함
- 복합 키에는 @IdClass와 @EmbeddedId 두 가지 방법이 있음
- **실무에서는 비식별 관계를 사용하고, 기본 키는 Long 타입의 대리 키를 사용하는 것을 권장**

### 조인 테이블

- 테이블은 외래 키 하나로 연관관계를 맺을 수 있지만 연관관계를 관리하는 조인 테이블을 추가할 수도 있음
- 조인 테이블의 가장 큰 단점은 테이블 하나를 추가해야 한다는 점
- **기본은 조인 컬럼을 사용하고 필요시에만 조인 테이블을 사용**

### 실무 가이드라인

1. **상속관계 매핑은 조인 전략을 우선 고려**
2. **공통 속성은 @MappedSuperclass 활용**
3. **복합 키보다는 Long 타입의 대리 키 사용**
4. **식별 관계보다는 비식별 관계 사용**
5. **조인 컬럼을 우선 사용하고 필요시에만 조인 테이블 사용**

---

[← 이전: 연관관계 매핑](./03-association-mapping.md) | [목차로 돌아가기](./README.md) | [다음: 프록시와 연관관계 관리 →](./05-proxy-and-lazy-loading.md)