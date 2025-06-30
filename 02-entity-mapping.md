# 2. 엔티티 매핑

## 📚 개요

엔티티 매핑은 **객체와 테이블, 필드와 컬럼을 매핑하는 것**이다. JPA에서 가장 중요한 일은 엔티티와 테이블을 정확히 매핑하는 것이다.

### 매핑 어노테이션 분류

1. **객체와 테이블 매핑**: `@Entity`, `@Table`
2. **기본 키 매핑**: `@Id`
3. **필드와 컬럼 매핑**: `@Column`
4. **연관관계 매핑**: `@ManyToOne`, `@JoinColumn`

## 🏢 @Entity, @Table

### @Entity

**@Entity가 붙은 클래스는 JPA가 관리하는 엔티티**라고 한다.

```java
@Entity
public class Member {
    @Id
    private Long id;
    private String name;
    
    // JPA는 기본 생성자가 필수 (public 또는 protected)
    protected Member() {}
    
    public Member(String name) {
        this.name = name;
    }
    
    // getter, setter...
}
```

#### @Entity 사용시 주의사항

1. **기본 생성자는 필수** (public 또는 protected)
2. **final 클래스, enum, interface, inner 클래스에는 사용할 수 없다**
3. **저장할 필드에 final을 사용하면 안 된다**

#### @Entity 속성

```java
@Entity(name = "Member")  // JPA에서 사용할 엔티티 이름 지정
public class Member {
    // 기본값: 클래스 이름을 그대로 사용
    // 같은 클래스 이름이 없으면 가급적 기본값 사용
}
```

### @Table

**@Table은 엔티티와 매핑할 테이블을 지정**한다.

```java
@Entity
@Table(name = "MEMBER")  // 매핑할 테이블 이름
public class Member {
    // 기본값: 엔티티 이름을 사용
}
```

#### @Table 속성

```java
@Entity
@Table(
    name = "MEMBER",                    // 매핑할 테이블 이름
    catalog = "mycatalog",              // 데이터베이스 catalog 매핑
    schema = "myschema",                // 데이터베이스 schema 매핑
    uniqueConstraints = {               // DDL 생성 시 유니크 제약조건
        @UniqueConstraint(
            name = "NAME_AGE_UNIQUE",
            columnNames = {"NAME", "AGE"}
        )
    }
)
public class Member {
    @Id
    private Long id;
    
    private String name;
    private int age;
}
```

## 🔑 기본 키 매핑

### @Id

```java
@Entity
public class Member {
    @Id
    private Long id;  // 직접 할당
    
    private String name;
}

// 사용
Member member = new Member();
member.setId(1L);  // 직접 id 값 설정
em.persist(member);
```

### @GeneratedValue

**기본 키 생성을 데이터베이스에 위임**한다.

#### 기본 키 생성 전략

```java
public enum GenerationType {
    AUTO,       // 방언에 따라 자동 지정 (기본값)
    IDENTITY,   // 데이터베이스에 위임 (MySQL AUTO_INCREMENT)
    SEQUENCE,   // 데이터베이스 시퀀스 사용 (Oracle, PostgreSQL)
    TABLE       // 키 생성용 테이블 사용
}
```

#### 1. IDENTITY 전략

```java
@Entity
public class Member {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String name;
}

// MySQL AUTO_INCREMENT 사용
// CREATE TABLE MEMBER (
//     ID BIGINT NOT NULL AUTO_INCREMENT,
//     NAME VARCHAR(255),
//     PRIMARY KEY (ID)
// )
```

**IDENTITY 전략의 특징:**
- 기본 키 생성을 데이터베이스에 위임
- **주로 MySQL, PostgreSQL, SQL Server, DB2에서 사용**
- JPA는 보통 트랜잭션 커밋 시점에 INSERT SQL 실행
- **IDENTITY 전략은 em.persist() 시점에 즉시 INSERT SQL 실행하고 DB에서 식별자를 조회**

```java
Member member = new Member();
member.setName("회원1");

System.out.println("=== BEFORE ===");
em.persist(member);  // 이 시점에 INSERT SQL 실행!
System.out.println("member.id = " + member.getId());  // ID 값이 설정됨
System.out.println("=== AFTER ===");

tx.commit();  // 추가 SQL 실행 없음
```

#### 2. SEQUENCE 전략

```java
@Entity
@SequenceGenerator(
    name = "MEMBER_SEQ_GENERATOR",
    sequenceName = "MEMBER_SEQ",  // 매핑할 데이터베이스 시퀀스 이름
    initialValue = 1,             // DDL 생성 시에만 사용됨
    allocationSize = 1            // 시퀀스 한 번 호출에 증가하는 수 (성능 최적화에 사용)
)
public class Member {
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE,
                    generator = "MEMBER_SEQ_GENERATOR")
    private Long id;
    
    private String name;
}

// Oracle, PostgreSQL, DB2, H2 데이터베이스에서 사용
// CREATE SEQUENCE MEMBER_SEQ START WITH 1 INCREMENT BY 1;
```

**SEQUENCE 전략의 특징:**
- 데이터베이스 시퀀스는 유일한 값을 순서대로 생성하는 특별한 데이터베이스 오브젝트
- **주로 Oracle, PostgreSQL, DB2, H2에서 사용**

#### SEQUENCE 성능 최적화

```java
@Entity
@SequenceGenerator(
    name = "MEMBER_SEQ_GENERATOR",
    sequenceName = "MEMBER_SEQ",
    initialValue = 1,
    allocationSize = 50  // 한 번에 50개씩 미리 할당
)
public class Member {
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE,
                    generator = "MEMBER_SEQ_GENERATOR")
    private Long id;
}
```

**allocationSize 최적화:**
- 기본값: 50
- 한 번에 50개씩 시퀀스를 미리 할당받아서 메모리에서 식별자를 할당
- 네트워크를 한 번만 타면 되므로 성능이 향상됨

#### 3. TABLE 전략

```java
@Entity
@TableGenerator(
    name = "MEMBER_SEQ_GENERATOR",
    table = "MY_SEQUENCES",
    pkColumnValue = "MEMBER_SEQ",
    allocationSize = 1
)
public class Member {
    @Id
    @GeneratedValue(strategy = GenerationType.TABLE,
                    generator = "MEMBER_SEQ_GENERATOR")
    private Long id;
    
    private String name;
}

// 키 생성 전용 테이블을 하나 만들어서 데이터베이스 시퀀스를 흉내내는 전략
// CREATE TABLE MY_SEQUENCES (
//     sequence_name VARCHAR(255) NOT NULL,
//     next_val BIGINT,
//     PRIMARY KEY (sequence_name)
// )
```

**TABLE 전략의 특징:**
- 키 생성 전용 테이블을 하나 만들어서 데이터베이스 시퀀스를 흉내내는 전략
- 모든 데이터베이스에 적용 가능
- **성능이 떨어짐** (테이블을 직접 사용하므로)

#### 4. AUTO 전략

```java
@Entity
public class Member {
    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)  // 기본값
    private Long id;
    
    private String name;
}
```

**AUTO 전략의 특징:**
- 방언에 따라 자동 지정 (기본값)
- MySQL: IDENTITY
- Oracle: SEQUENCE
- **운영 환경에서는 구체적인 전략을 명시하는 것이 좋음**

### 권장하는 식별자 전략

#### 기본 키 제약 조건

1. **null이면 안 된다**
2. **유일해야 한다**
3. **변하면 안 된다**

#### 미래까지 이 조건을 만족하는 자연키는 찾기 어렵다

```java
// 나쁜 예: 자연키 사용
@Entity
public class Member {
    @Id
    private String ssn;  // 주민등록번호 (자연키)
    // 주민등록번호도 변경될 수 있고, 개인정보이므로 부적절
}

// 좋은 예: 대리키(대체키) 사용
@Entity
public class Member {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;  // 대리키
    
    private String ssn;  // 주민등록번호는 일반 필드로
}
```

#### 권장 전략

**Long형 + 대체키 + 키 생성전략 사용**

```java
@Entity
public class Member {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;  // Long 타입 권장 (Integer는 20억 정도에서 끝남)
    
    private String name;
}
```

## 📋 필드와 컬럼 매핑

### @Column

**필드와 컬럼을 매핑**한다.

```java
@Entity
public class Member {
    @Id
    private Long id;
    
    @Column(name = "name")  // 객체의 필드를 테이블의 컬럼에 매핑
    private String username;
    
    private Integer age;  // @Column 생략 가능
}
```

#### @Column 속성

```java
@Entity
public class Member {
    @Id
    private Long id;
    
    @Column(
        name = "name",              // 필드와 매핑할 테이블의 컬럼 이름
        insertable = true,          // 등록 가능 여부 (기본값: true)
        updatable = true,           // 변경 가능 여부 (기본값: true)
        nullable = false,           // null 값의 허용 여부 (기본값: true)
        unique = false,             // 유니크 제약조건 (기본값: false)
        columnDefinition = "varchar(100) default 'EMPTY'",  // 컬럼 정보 직접 입력
        length = 400,               // 문자 길이 제약조건 (String 타입에만 사용, 기본값: 255)
        precision = 19,             // BigDecimal 타입에서 사용 (전체 자리수)
        scale = 2                   // BigDecimal 타입에서 사용 (소수점 자리수)
    )
    private String name;
    
    @Column(updatable = false)  // 수정 불가
    private LocalDateTime createdDate;
    
    @Column(insertable = false) // 등록 불가
    private LocalDateTime lastModifiedDate;
}
```

### @Enumerated

**자바 enum 타입을 매핑**할 때 사용한다.

```java
public enum RoleType {
    ADMIN, USER
}

@Entity
public class Member {
    @Id
    private Long id;
    
    @Enumerated(EnumType.STRING)  // 권장
    private RoleType roleType;
    
    // @Enumerated(EnumType.ORDINAL)  // 사용 금지!
}
```

#### @Enumerated 속성

```java
public enum RoleType {
    USER,   // ORDINAL: 0
    ADMIN   // ORDINAL: 1
}

@Entity
public class Member {
    // ORDINAL 사용 시 문제점
    @Enumerated(EnumType.ORDINAL)  // 기본값이지만 사용하면 안됨!
    private RoleType roleType1;
    // USER=0, ADMIN=1로 저장됨
    // 나중에 enum 순서가 바뀌거나 새로운 값이 추가되면 문제 발생!
    
    // STRING 사용 (권장)
    @Enumerated(EnumType.STRING)
    private RoleType roleType2;
    // USER="USER", ADMIN="ADMIN"으로 저장됨
    // 데이터베이스에 저장되는 데이터 크기가 ORDINAL보다 크지만 안전함
}
```

**⚠️ 주의사항: ORDINAL 사용 금지!**

```java
// 처음 설계
public enum RoleType {
    USER,   // 0
    ADMIN   // 1
}

// 나중에 GUEST 추가
public enum RoleType {
    GUEST,  // 0  ← 기존 USER 데이터와 충돌!
    USER,   // 1
    ADMIN   // 2
}
```

### @Temporal

**날짜 타입을 매핑**할 때 사용한다.

```java
@Entity
public class Member {
    @Id
    private Long id;
    
    @Temporal(TemporalType.DATE)        // 날짜
    private Date date;                  // 2024-01-01
    
    @Temporal(TemporalType.TIME)        // 시간
    private Date time;                  // 12:11:20
    
    @Temporal(TemporalType.TIMESTAMP)   // 날짜와 시간
    private Date timestamp;             // 2024-01-01 12:11:20
    
    // Java 8의 LocalDate, LocalDateTime을 사용할 때는 생략 가능
    private LocalDate localDate;       // @Temporal 불필요
    private LocalDateTime localDateTime; // @Temporal 불필요
}
```

#### 최신 자바 시간 API 사용 (권장)

```java
@Entity
public class Member {
    @Id
    private Long id;
    
    private LocalDate date;          // 날짜
    private LocalTime time;          // 시간
    private LocalDateTime timestamp; // 날짜 + 시간
    
    // 하이버네이트 5.2.10 이상부터 지원
    // 별도 어노테이션 불필요
}
```

### @Lob

**데이터베이스 BLOB, CLOB 타입과 매핑**한다.

```java
@Entity
public class Member {
    @Id
    private Long id;
    
    @Lob
    private String lobString;  // CLOB으로 매핑 (문자 타입)
    
    @Lob
    private byte[] lobByte;    // BLOB으로 매핑 (나머지 타입)
}
```

**@Lob 매핑 규칙:**
- 문자 타입: CLOB 매핑 (`String`, `char[]`, `java.sql.CLOB`)
- 나머지 타입: BLOB 매핑 (`byte[]`, `java.sql.BLOB`)

### @Transient

**필드 매핑을 제외**한다.

```java
@Entity
public class Member {
    @Id
    private Long id;
    
    private String name;
    
    @Transient
    private int temp;  // 매핑하지 않음
    
    // 메모리에서만 임시로 어떤 값을 보관하고 싶을 때 사용
}
```

## 🏗️ 데이터베이스 스키마 자동 생성

### hibernate.hbm2ddl.auto

**애플리케이션 실행 시점에 DDL을 자동 생성**한다.

```yaml
# application.yml
spring:
  jpa:
    hibernate:
      ddl-auto: create  # 옵션 설정
```

#### 옵션

| 옵션 | 설명 |
|------|------|
| **create** | 기존테이블 삭제 후 다시 생성 (DROP + CREATE) |
| **create-drop** | create와 같으나 종료시점에 테이블 DROP |
| **update** | 변경분만 반영 (운영DB에는 사용하면 안됨) |
| **validate** | 엔티티와 테이블이 정상 매핑되었는지만 확인 |
| **none** | 사용하지 않음 |

#### 주의사항

```java
// 개발 초기 단계
spring.jpa.hibernate.ddl-auto=create

// 테스트 서버
spring.jpa.hibernate.ddl-auto=update
spring.jpa.hibernate.ddl-auto=validate

// 스테이징과 운영 서버
spring.jpa.hibernate.ddl-auto=validate
spring.jpa.hibernate.ddl-auto=none
```

**⚠️ 운영 장비에는 절대 create, create-drop, update 사용하면 안된다!**

### DDL 생성 기능

```java
@Entity
public class Member {
    @Id
    private Long id;
    
    @Column(nullable = false, length = 10)  // DDL: not null, varchar(10)
    private String name;
    
    @Column(unique = true)  // DDL: unique 제약조건
    private String email;
}

@Entity
@Table(uniqueConstraints = {
    @UniqueConstraint(
        name = "NAME_AGE_UNIQUE",
        columnNames = {"NAME", "AGE"}
    )
})
public class Member {
    // 복합 unique 제약조건
}
```

**DDL 생성 기능의 특징:**
- **DDL 생성에만 영향을 주고, JPA의 실행 로직에는 영향을 주지 않는다**
- 스키마 자동 생성을 사용할 때만 의미가 있음
- 개발자가 엔티티만 보고도 제약조건을 파악할 수 있어 가독성이 좋음

## 💡 실전 예제

### 요구사항 분석

1. 회원은 상품을 주문할 수 있다
2. 주문 시 여러 종류의 상품을 선택할 수 있다

### 도메인 모델 분석

- **회원과 주문**의 관계: 회원은 여러 번 주문할 수 있다 (일대다)
- **주문과 상품**의 관계: 주문할 때 여러 상품을 선택할 수 있고, 같은 상품도 여러 번 주문될 수 있다 (다대다) → 주문상품이라는 엔티티 추가해서 다대다 관계를 일대다, 다대일 관계로 풀어냄

### 테이블 설계

```sql
CREATE TABLE MEMBER (
    MEMBER_ID BIGINT NOT NULL,
    NAME VARCHAR(255),
    CITY VARCHAR(255),
    STREET VARCHAR(255),
    ZIPCODE VARCHAR(255),
    PRIMARY KEY (MEMBER_ID)
);

CREATE TABLE ORDERS (
    ORDER_ID BIGINT NOT NULL,
    MEMBER_ID BIGINT,
    ORDERDATE TIMESTAMP,
    STATUS VARCHAR(255),
    PRIMARY KEY (ORDER_ID)
);

CREATE TABLE ORDER_ITEM (
    ORDER_ITEM_ID BIGINT NOT NULL,
    ORDER_ID BIGINT,
    ITEM_ID BIGINT,
    ORDERPRICE INTEGER,
    COUNT INTEGER,
    PRIMARY KEY (ORDER_ITEM_ID)
);

CREATE TABLE ITEM (
    ITEM_ID BIGINT NOT NULL,
    NAME VARCHAR(255),
    PRICE INTEGER,
    STOCKQUANTITY INTEGER,
    PRIMARY KEY (ITEM_ID)
);
```

### 엔티티 설계와 매핑

#### 회원 엔티티

```java
@Entity
@Table(name = "MEMBER")
public class Member {
    
    @Id @GeneratedValue(strategy = GenerationType.AUTO)
    @Column(name = "MEMBER_ID")
    private Long id;
    
    private String name;
    private String city;
    private String street;
    private String zipcode;
    
    // 기본 생성자
    protected Member() {}
    
    // 생성자
    public Member(String name, String city, String street, String zipcode) {
        this.name = name;
        this.city = city;
        this.street = street;
        this.zipcode = zipcode;
    }
    
    // Getter, Setter
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    
    public String getCity() { return city; }
    public void setCity(String city) { this.city = city; }
    
    public String getStreet() { return street; }
    public void setStreet(String street) { this.street = street; }
    
    public String getZipcode() { return zipcode; }
    public void setZipcode(String zipcode) { this.zipcode = zipcode; }
}
```

#### 주문 엔티티

```java
@Entity
@Table(name = "ORDERS")
public class Order {
    
    @Id @GeneratedValue
    @Column(name = "ORDER_ID")
    private Long id;
    
    @Column(name = "MEMBER_ID")
    private Long memberId;  // 외래 키를 직접 매핑
    
    @Temporal(TemporalType.TIMESTAMP)
    private Date orderDate;
    
    @Enumerated(EnumType.STRING)
    private OrderStatus status;
    
    // 기본 생성자
    protected Order() {}
    
    // Getter, Setter
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    
    public Long getMemberId() { return memberId; }
    public void setMemberId(Long memberId) { this.memberId = memberId; }
    
    public Date getOrderDate() { return orderDate; }
    public void setOrderDate(Date orderDate) { this.orderDate = orderDate; }
    
    public OrderStatus getStatus() { return status; }
    public void setStatus(OrderStatus status) { this.status = status; }
}

public enum OrderStatus {
    ORDER, CANCEL
}
```

#### 주문상품 엔티티

```java
@Entity
@Table(name = "ORDER_ITEM")
public class OrderItem {
    
    @Id @GeneratedValue
    @Column(name = "ORDER_ITEM_ID")
    private Long id;
    
    @Column(name = "ORDER_ID")
    private Long orderId;
    
    @Column(name = "ITEM_ID")
    private Long itemId;
    
    private int orderPrice;
    private int count;
    
    // 기본 생성자
    protected OrderItem() {}
    
    // Getter, Setter
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    
    public Long getOrderId() { return orderId; }
    public void setOrderId(Long orderId) { this.orderId = orderId; }
    
    public Long getItemId() { return itemId; }
    public void setItemId(Long itemId) { this.itemId = itemId; }
    
    public int getOrderPrice() { return orderPrice; }
    public void setOrderPrice(int orderPrice) { this.orderPrice = orderPrice; }
    
    public int getCount() { return count; }
    public void setCount(int count) { this.count = count; }
}
```

#### 상품 엔티티

```java
@Entity
@Table(name = "ITEM")
public class Item {
    
    @Id @GeneratedValue
    @Column(name = "ITEM_ID")
    private Long id;
    
    private String name;
    private int price;
    private int stockQuantity;
    
    // 기본 생성자
    protected Item() {}
    
    // 생성자
    public Item(String name, int price, int stockQuantity) {
        this.name = name;
        this.price = price;
        this.stockQuantity = stockQuantity;
    }
    
    // Getter, Setter
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    
    public int getPrice() { return price; }
    public void setPrice(int price) { this.price = price; }
    
    public int getStockQuantity() { return stockQuantity; }
    public void setStockQuantity(int stockQuantity) { this.stockQuantity = stockQuantity; }
}
```

---

## 📝 정리

### 엔티티 매핑 핵심

1. **@Entity**: JPA가 관리하는 엔티티 선언
2. **@Table**: 엔티티와 매핑할 테이블 지정
3. **@Id**: 기본 키 매핑
4. **@GeneratedValue**: 기본 키 생성 전략
5. **@Column**: 필드와 컬럼 매핑

### 기본 키 매핑 전략

1. **직접 할당**: `@Id`만 사용
2. **자동 생성**: `@GeneratedValue` 사용
   - **IDENTITY**: 데이터베이스에 위임 (MySQL)
   - **SEQUENCE**: 데이터베이스 시퀀스 사용 (Oracle)
   - **TABLE**: 키 생성용 테이블 사용
   - **AUTO**: 방언에 따라 자동 지정

### 권장하는 식별자 전략

- **Long형 + 대체키 + 키 생성전략** 사용
- 비즈니스에 의존적인 자연키보다는 **대체키(대리키) 사용**
- **AUTO_INCREMENT나 SEQUENCE OBJECT** 사용

### 필드와 컬럼 매핑

1. **@Column**: 컬럼 매핑
2. **@Temporal**: 날짜 타입 매핑
3. **@Enumerated**: enum 타입 매핑 (STRING 사용 권장)
4. **@Lob**: BLOB, CLOB 매핑
5. **@Transient**: 특정 필드를 컬럼에 매핑하지 않음

### 주의사항

1. **엔티티에는 기본 생성자가 필수** (public 또는 protected)
2. **final 클래스, enum, interface, inner 클래스는 엔티티가 될 수 없음**
3. **저장할 필드에 final 사용 금지**
4. **@Enumerated에서 ORDINAL 사용 금지** (STRING 사용)
5. **운영 환경에서 DDL 자동 생성 기능 사용 금지**

---

[← 이전: JPA 소개 및 개념](./01-jpa-introduction.md) | [목차로 돌아가기](./README.md) | [다음: 연관관계 매핑 →](./03-association-mapping.md)