# 6. 값 타입

## 📖 JPA의 데이터 타입 분류

### 엔티티 타입
- @Entity로 정의하는 객체
- 데이터가 변해도 식별자로 지속해서 추적 가능
- 예) 회원 엔티티의 키나 나이 값을 변경해도 식별자로 인식 가능

### 값 타입
- int, Integer, String처럼 단순히 값으로 사용하는 자바 기본 타입이나 객체
- 식별자가 없고 값만 있으므로 변경시 추적 불가
- 예) 숫자 100을 200으로 변경하면 완전히 다른 값으로 대체

### 값 타입 분류

1. **기본값 타입**
   - 자바 기본 타입(int, double)
   - 래퍼 클래스(Integer, Long)
   - String

2. **임베디드 타입**(embedded type, 복합 값 타입)

3. **컬렉션 값 타입**(collection value type)

## 🏗️ 기본값 타입

### 예시
```java
@Entity
public class Member {
    
    @Id @GeneratedValue
    private Long id;
    
    private String name;
    private int age;
    
    // getter, setter...
}
```

### 특징
- 생명주기를 엔티티에 의존
  - 회원을 삭제하면 이름, 나이 필드도 함께 삭제
- 값 타입은 공유하면 안됨
  - 회원 이름 변경시 다른 회원의 이름도 함께 변경되면 안됨

### 참고: 자바의 기본 타입은 절대 공유X
```java
int a = 20;
int b = a; // 20이라는 값을 복사
a = 10;
System.out.println(a); // 10
System.out.println(b); // 20
```

- int, double 같은 기본 타입(primitive type)은 절대 공유X
- 기본 타입은 항상 값을 복사함
- Integer같은 래퍼 클래스나 String 같은 특수한 클래스는 공유 가능한 객체이지만 변경X

## 📦 임베디드 타입(복합 값 타입)

### 개요
- 새로운 값 타입을 직접 정의할 수 있음
- JPA는 임베디드 타입(embedded type)이라 함
- 주로 기본 값 타입을 모아서 만들어서 복합 값 타입이라고도 함
- int, String과 같은 값 타입

### 예시
```java
// 기존 방식
@Entity
public class Member {
    
    @Id @GeneratedValue
    private Long id;
    
    private String name;
    
    // 근무 기간
    private LocalDateTime startDate;
    private LocalDateTime endDate;
    
    // 집 주소
    private String city;
    private String street;
    private String zipcode;
    
    // getter, setter...
}
```

```java
// 임베디드 타입 사용
@Entity
public class Member {
    
    @Id @GeneratedValue
    private Long id;
    
    private String name;
    
    @Embedded
    private Period workPeriod; // 근무 기간
    
    @Embedded
    private Address homeAddress; // 집 주소
    
    // getter, setter...
}

@Embeddable
public class Period {
    
    private LocalDateTime startDate;
    private LocalDateTime endDate;
    
    public Period() {}
    
    public Period(LocalDateTime startDate, LocalDateTime endDate) {
        this.startDate = startDate;
        this.endDate = endDate;
    }
    
    // 의미있는 메소드 작성 가능
    public boolean isWork() {
        // 근무중인지 판단하는 로직
        return LocalDateTime.now().isAfter(startDate) && 
               LocalDateTime.now().isBefore(endDate);
    }
    
    // getter, setter...
}

@Embeddable
public class Address {
    
    private String city;
    private String street;
    private String zipcode;
    
    public Address() {}
    
    public Address(String city, String street, String zipcode) {
        this.city = city;
        this.street = street;
        this.zipcode = zipcode;
    }
    
    // 의미있는 메소드 작성 가능
    public String fullAddress() {
        return city + " " + street + " " + zipcode;
    }
    
    // getter, setter...
}
```

### 임베디드 타입 사용법
- @Embeddable: 값 타입을 정의하는 곳에 표시
- @Embedded: 값 타입을 사용하는 곳에 표시
- 기본 생성자 필수

### 임베디드 타입의 장점
- 재사용성
- 높은 응집도
- Period.isWork()처럼 해당 값 타입만 사용하는 의미 있는 메소드를 만들 수 있음
- 임베디드 타입을 포함한 모든 값 타입은, 값 타입을 소유한 엔티티에 생명주기를 의존함

### 임베디드 타입과 테이블 매핑
- 임베디드 타입은 엔티티의 값일 뿐이다
- 임베디드 타입을 사용하기 전과 후에 **매핑하는 테이블은 같다**
- 객체와 테이블을 아주 세밀하게(find-grained) 매핑하는 것이 가능
- 잘 설계한 ORM 애플리케이션은 매핑한 테이블의 수보다 클래스의 수가 더 많음

### @AttributeOverride: 속성 재정의

한 엔티티에서 같은 값 타입을 사용하면?

```java
@Entity
public class Member {
    
    @Id @GeneratedValue
    private Long id;
    
    private String name;
    
    @Embedded
    private Address homeAddress;
    
    @Embedded
    @AttributeOverrides({
        @AttributeOverride(name="city", column=@Column(name = "WORK_CITY")),
        @AttributeOverride(name="street", column=@Column(name = "WORK_STREET")),
        @AttributeOverride(name="zipcode", column=@Column(name = "WORK_ZIPCODE"))
    })
    private Address workAddress;
    
    // getter, setter...
}
```

### 임베디드 타입과 null
- 임베디드 타입의 값이 null이면 매핑한 컬럼 값은 모두 null

```java
member.setAddress(null); // null 입력
em.persist(member);
// 결과: CITY, STREET, ZIPCODE 모두 null
```

## 🔗 값 타입과 불변 객체

### 값 타입 공유 참조

임베디드 타입 같은 값 타입을 여러 엔티티에서 공유하면 위험함

```java
Address address = new Address("city", "street", "10000");

Member member1 = new Member();
member1.setName("member1");
member1.setHomeAddress(address);
em.persist(member1);

Member member2 = new Member();
member2.setName("member2");
member2.setHomeAddress(address);
em.persist(member2);

member1.getHomeAddress().setCity("newCity"); // member1의 주소만 변경하려 했지만

// 결과: member1, member2 둘 다 주소가 변경됨
```

**문제점:**
- 회원1의 주소만 "newCity"로 변경하고 싶었지만
- 회원1, 회원2의 주소가 모두 "newCity"로 변경됨
- 이런 부작용(side effect)을 막으려면 값을 복사해서 사용해야 함

### 값 타입 복사

```java
Address address = new Address("city", "street", "10000");

Member member1 = new Member();
member1.setName("member1");
member1.setHomeAddress(address);
em.persist(member1);

Address copyAddress = new Address(address.getCity(), address.getStreet(), address.getZipcode());

Member member2 = new Member();
member2.setName("member2");
member2.setHomeAddress(copyAddress); // 복사한 주소 사용
em.persist(member2);

member1.getHomeAddress().setCity("newCity");

// 결과: member1의 주소만 변경됨
```

### 객체 타입의 한계

- 항상 값을 복사해서 사용하면 공유 참조로 인한 부작용을 피할 수 있다
- 문제는 임베디드 타입처럼 **직접 정의한 값 타입은 자바의 기본 타입이 아니라 객체 타입**이다
- 자바 기본 타입에 값을 대입하면 값을 복사한다
- **객체 타입은 참조 값을 직접 대입하는 것을 막을 방법이 없다**
- **객체의 공유 참조는 피할 수 없다**

### 불변 객체

객체 타입을 수정할 수 없게 만들면 **부작용을 원천 차단**

**값 타입은 불변 객체(immutable object)로 설계해야함**
- 불변 객체: 생성 시점 이후 절대 값을 변경할 수 없는 객체
- 생성자로만 값을 설정하고 수정자(Setter)를 만들지 않으면 됨
- 참고: Integer, String은 자바가 제공하는 대표적인 불변 객체

```java
@Embeddable
public class Address {
    
    private String city;
    private String street;
    private String zipcode;
    
    public Address() {}
    
    public Address(String city, String street, String zipcode) {
        this.city = city;
        this.street = street;
        this.zipcode = zipcode;
    }
    
    // Getter만 제공, Setter는 제공하지 않음
    public String getCity() { return city; }
    public String getStreet() { return street; }
    public String getZipcode() { return zipcode; }
}
```

**불변이라는 작은 제약으로 부작용이라는 큰 재앙을 막을 수 있다**

## ⚖️ 값 타입의 비교

값 타입: 인스턴스가 달라도 그 안에 값이 같으면 같은 것으로 봐야 함

```java
int a = 10;
int b = 10;
System.out.println(a == b); // true

Address a = new Address("서울시", "종로구", "1번지");
Address b = new Address("서울시", "종로구", "1번지");
System.out.println(a == b); // false
```

### 동일성(identity) 비교와 동등성(equivalence) 비교

- **동일성(identity) 비교**: 인스턴스의 참조 값을 비교, == 사용
- **동등성(equivalence) 비교**: 인스턴스의 값을 비교, equals() 사용
- 값 타입은 a.equals(b)를 사용해서 동등성 비교를 해야 함
- 값 타입의 equals() 메소드를 적절하게 재정의(주로 모든 필드 사용)

```java
@Embeddable
public class Address {
    
    private String city;
    private String street;
    private String zipcode;
    
    // 생성자, getter...
    
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        
        Address address = (Address) o;
        
        return Objects.equals(city, address.city) &&
               Objects.equals(street, address.street) &&
               Objects.equals(zipcode, address.zipcode);
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(city, street, zipcode);
    }
}
```

**주의:** equals()를 재정의하면 hashCode()도 재정의해야 함

## 📚 값 타입 컬렉션

### 개념
- 값 타입을 하나 이상 저장할 때 사용
- @ElementCollection, @CollectionTable 사용
- 데이터베이스는 컬렉션을 같은 테이블에 저장할 수 없다
- 컬렉션을 저장하기 위한 별도의 테이블이 필요함

### 예시
```java
@Entity
public class Member {
    
    @Id @GeneratedValue
    private Long id;
    
    @Column(name = "USERNAME")
    private String name;
    
    @Embedded
    private Address homeAddress;
    
    @ElementCollection
    @CollectionTable(name = "FAVORITE_FOOD", 
                     joinColumns = @JoinColumn(name = "MEMBER_ID"))
    @Column(name = "FOOD_NAME")
    private Set<String> favoriteFoods = new HashSet<>();
    
    @ElementCollection
    @CollectionTable(name = "ADDRESS", 
                     joinColumns = @JoinColumn(name = "MEMBER_ID"))
    private List<Address> addressHistory = new ArrayList<>();
    
    // getter, setter...
}
```

### 값 타입 컬렉션 사용

#### 저장
```java
Member member = new Member();
member.setName("member1");
member.setHomeAddress(new Address("homeCity", "street", "10000"));

member.getFavoriteFoods().add("치킨");
member.getFavoriteFoods().add("족발");
member.getFavoriteFoods().add("피자");

member.getAddressHistory().add(new Address("old1", "street", "10000"));
member.getAddressHistory().add(new Address("old2", "street", "10000"));

em.persist(member);
```

#### 조회
```java
Member findMember = em.find(Member.class, member.getId());

// homeAddress
Address homeAddress = findMember.getHomeAddress();

// favoriteFoods
Set<String> favoriteFoods = findMember.getFavoriteFoods(); // LAZY

for (String favoriteFood : favoriteFoods) {
    System.out.println("favoriteFood = " + favoriteFood);
}

// addressHistory
List<Address> addressHistory = findMember.getAddressHistory(); // LAZY

for (Address address : addressHistory) {
    System.out.println("address = " + address.getCity());
}
```

#### 수정
```java
Member findMember = em.find(Member.class, member.getId());

// 1. 임베디드 값 타입 수정
// findMember.getHomeAddress().setCity("newCity"); // 이렇게 하면 안됨
Address a = findMember.getHomeAddress();
findMember.setHomeAddress(new Address("newCity", a.getStreet(), a.getZipcode()));

// 2. 기본값 타입 컬렉션 수정
Set<String> favoriteFoods = findMember.getFavoriteFoods();
favoriteFoods.remove("치킨");
favoriteFoods.add("한식");

// 3. 임베디드 값 타입 컬렉션 수정
List<Address> addressHistory = findMember.getAddressHistory();
addressHistory.remove(new Address("old1", "street", "10000")); // equals 구현 필요
addressHistory.add(new Address("newCity1", "street", "10000"));
```

### 값 타입 컬렉션의 제약사항

- 값 타입은 엔티티와 다르게 식별자 개념이 없다
- 값을 변경하면 추적이 어렵다
- 값 타입 컬렉션에 변경 사항이 발생하면, 주인 엔티티와 연관된 모든 데이터를 삭제하고, 값 타입 컬렉션에 있는 현재 값을 모두 다시 저장한다
- 값 타입 컬렉션을 매핑하는 테이블은 모든 컬럼을 묶어서 기본 키를 구성해야 함: **null 입력X, 중복 저장X**

### 값 타입 컬렉션 대안

실무에서는 상황에 따라 **값 타입 컬렉션 대신에 일대다 관계를 고려**

```java
@Entity
public class AddressEntity {
    
    @Id @GeneratedValue
    private Long id;
    
    @Embedded
    private Address address;
    
    public AddressEntity() {}
    
    public AddressEntity(String city, String street, String zipcode) {
        this.address = new Address(city, street, zipcode);
    }
    
    // getter, setter...
}

@Entity
public class Member {
    
    @Id @GeneratedValue
    private Long id;
    
    @Column(name = "USERNAME")
    private String name;
    
    @Embedded
    private Address homeAddress;
    
    @ElementCollection
    @CollectionTable(name = "FAVORITE_FOOD", 
                     joinColumns = @JoinColumn(name = "MEMBER_ID"))
    @Column(name = "FOOD_NAME")
    private Set<String> favoriteFoods = new HashSet<>();
    
    @OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)
    @JoinColumn(name = "MEMBER_ID")
    private List<AddressEntity> addressHistory = new ArrayList<>();
    
    // getter, setter...
}
```

**장점:**
- 일대다 관계를 위한 엔티티를 만들고, 여기에서 값 타입을 사용
- 영속성 전이(Cascade) + 고아 객체 제거를 사용해서 값 타입 컬렉션 처럼 사용
- EX) AddressEntity

---

## 📝 정리

### 엔티티 타입의 특징
- 식별자O
- 생명 주기 관리
- 공유

### 값 타입의 특징
- 식별자X
- 생명 주기를 엔티티에 의존
- 공유하지 않는 것이 안전(복사해서 사용)
- 불변 객체로 만드는 것이 안전

### 값 타입은 정말 값 타입이라 판단될 때만 사용
- 엔티티와 값 타입을 혼동해서 엔티티를 값 타입으로 만들면 안됨
- 식별자가 필요하고, 지속해서 값을 추적, 변경해야 한다면 그것은 값 타입이 아닌 엔티티

### 실무에서 값 타입 컬렉션은 사용하지 않는 것을 권장
- 대신 일대다 매핑을 사용하자
- 정말 단순한 경우에만 값 타입 컬렉션 사용 (예: checkbox, multiselect box)

---

[← 이전: 프록시와 연관관계 관리](./05-proxy-and-lazy-loading.md) | [목차로 돌아가기](./README.md) | [다음: 객체지향 쿼리 언어 (JPQL) →](./07-jpql.md)