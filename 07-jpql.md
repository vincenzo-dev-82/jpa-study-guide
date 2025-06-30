# 7. 객체지향 쿼리 언어 (JPQL)

## 📖 JPQL 소개

### JPA는 다양한 쿼리 방법을 지원

- **JPQL**
- JPA Criteria
- **QueryDSL**
- 네이티브 SQL
- JDBC API 직접 사용, MyBatis, SpringJdbcTemplate 함께 사용

### JPQL의 필요성

- 가장 단순한 조회 방법
  - EntityManager.find()
  - 객체 그래프 탐색(a.getB().getC())
- **나이가 18살 이상인 회원을 모두 검색하고 싶다면?**

### JPQL

- JPA를 사용하면 엔티티 객체를 중심으로 개발
- 문제는 검색 쿼리
- 검색을 할 때도 **테이블이 아닌 엔티티 객체를 대상으로 검색**
- 모든 DB 데이터를 객체로 변환해서 검색하는 것은 불가능
- 애플리케이션이 필요한 데이터만 DB에서 불러오려면 결국 검색 조건이 포함된 SQL이 필요

- JPA는 SQL을 추상화한 JPQL이라는 객체 지향 쿼리 언어 제공
- SQL과 문법 유사, SELECT, FROM, WHERE, GROUP BY, HAVING, JOIN 지원
- **JPQL은 엔티티 객체**를 대상으로 쿼리
- **SQL은 데이터베이스 테이블**을 대상으로 쿼리

### JPQL 예시

```java
String jpql = "select m from Member m where m.age > 18";

List<Member> result = em.createQuery(jpql, Member.class)
        .getResultList();
```

- 테이블이 아닌 객체를 대상으로 검색하는 객체 지향 쿼리
- SQL을 추상화해서 특정 데이터베이스 SQL에 의존X
- JPQL을 한마디로 정의하면 객체 지향 SQL

## 🛠️ 기본 문법과 기능

### JPQL 문법

```sql
select_문 :: = 
    select_절
    from_절
    [where_절]
    [groupby_절]
    [having_절]
    [orderby_절]

update_문 :: = update_절 [where_절]
delete_문 :: = delete_절 [where_절]
```

### select 문

```sql
SELECT m FROM Member AS m WHERE m.username = 'Hello'
```

- 엔티티와 속성은 대소문자 구분O (Member, username)
- JPQL 키워드는 대소문자 구분X (SELECT, FROM, where)
- 엔티티 이름 사용, 테이블 이름이 아님(Member)
- **별칭은 필수(m)** (as는 생략가능)

### TypeQuery, Query

- **TypeQuery**: 반환 타입이 명확할 때 사용
- **Query**: 반환 타입이 명확하지 않을 때 사용

```java
TypedQuery<Member> query = 
    em.createQuery("SELECT m FROM Member m", Member.class);

Query query = 
    em.createQuery("SELECT m.username, m.age from Member m");
```

### 결과 조회 API

- **query.getResultList()**: 결과가 하나 이상일 때, 리스트 반환
  - 결과가 없으면 빈 리스트 반환
- **query.getSingleResult()**: 결과가 정확히 하나, 단일 객체 반환
  - 결과가 없으면: javax.persistence.NoResultException
  - 둘 이상이면: javax.persistence.NonUniqueResultException

```java
TypedQuery<Member> query = em.createQuery("SELECT m FROM Member m", Member.class);

List<Member> resultList = query.getResultList();

for (Member member1 : resultList) {
    System.out.println("member1 = " + member1);
}

Member result = em.createQuery("SELECT m FROM Member m where m.id = 10L", Member.class)
                  .getSingleResult();
System.out.println("result = " + result);
```

### 파라미터 바인딩 - 이름 기준, 위치 기준

#### 이름 기준 (권장)
```java
TypedQuery<Member> query = em.createQuery(
    "SELECT m FROM Member m where m.username = :username", Member.class);

query.setParameter("username", "member1");
Member singleResult = query.getSingleResult();
System.out.println("singleResult = " + singleResult.getUsername());
```

#### 위치 기준 (비추천)
```java
TypedQuery<Member> query = em.createQuery(
    "SELECT m FROM Member m where m.username = ?1", Member.class);

query.setParameter(1, "member1");
Member singleResult = query.getSingleResult();
```

### 메소드 체이닝
```java
Member result = em.createQuery(
    "SELECT m FROM Member m where m.username = :username", Member.class)
    .setParameter("username", "member1")
    .getSingleResult();
```

## 📊 프로젝션

### 프로젝션이란?
- SELECT 절에 조회할 대상을 지정하는 것
- 프로젝션 대상: 엔티티, 임베디드 타입, 스칼라 타입(숫자, 문자등 기본 데이터 타입)

### 엔티티 프로젝션
```sql
SELECT m FROM Member m -- 엔티티 프로젝션
SELECT m.team FROM Member m -- 엔티티 프로젝션
```

```java
List<Member> result = em.createQuery("select m from Member m", Member.class)
                       .getResultList();

Member findMember = result.get(0);
findMember.setAge(20);

// 영속성 컨텍스트에서 관리됨 - 변경 감지 동작
```

### 임베디드 타입 프로젝션
```java
List<Address> addresses = em.createQuery("select o.address from Order o", Address.class)
                           .getResultList();
```

### 스칼라 타입 프로젝션
```sql
SELECT m.username, m.age FROM Member m
```

### 여러 값 조회

#### 1. Query 타입으로 조회
```java
List resultList = em.createQuery("select m.username, m.age from Member m")
                   .getResultList();

Object o = resultList.get(0);
Object[] result = (Object[]) o;
System.out.println("username = " + result[0]);
System.out.println("age = " + result[1]);
```

#### 2. Object[] 타입으로 조회
```java
List<Object[]> resultList = em.createQuery("select m.username, m.age from Member m")
                             .getResultList();

Object[] result = resultList.get(0);
System.out.println("username = " + result[0]);
System.out.println("age = " + result[1]);
```

#### 3. new 명령어로 조회
```java
// DTO 클래스
public class MemberDTO {
    
    private String username;
    private int age;
    
    public MemberDTO(String username, int age) {
        this.username = username;
        this.age = age;
    }
    
    // getter, setter...
}

// JPQL
List<MemberDTO> result = em.createQuery(
    "select new jpql.MemberDTO(m.username, m.age) from Member m", MemberDTO.class)
    .getResultList();

MemberDTO memberDTO = result.get(0);
System.out.println("memberDTO = " + memberDTO.getUsername());
System.out.println("memberDTO = " + memberDTO.getAge());
```

**new 명령어 주의사항:**
- 패키지 명을 포함한 전체 클래스 명 입력
- 순서와 타입이 일치하는 생성자 필요

## 📃 페이징 API

JPA는 페이징을 다음 두 API로 추상화

- **setFirstResult(int startPosition)**: 조회 시작 위치(0부터 시작)
- **setMaxResults(int maxResult)**: 조회할 데이터 수

### 페이징 API 예제

```java
for(int i = 0; i < 100; i++) {
    Member member = new Member();
    member.setUsername("member" + i);
    member.setAge(i);
    em.persist(member);
}

em.flush();
em.clear();

// 페이징 쿼리
List<Member> result = em.createQuery("select m from Member m order by m.age desc", Member.class)
                       .setFirstResult(10)
                       .setMaxResults(20)
                       .getResultList();

System.out.println("result.size = " + result.size());
for (Member member1 : result) {
    System.out.println("member1 = " + member1);
}
```

### 데이터베이스 방언

- MySQL: LIMIT
- Oracle: ROWNUM
- 등등... JPA가 자동으로 처리

## 🔗 조인

### 내부 조인
```sql
SELECT m FROM Member m [INNER] JOIN m.team t
```

### 외부 조인
```sql
SELECT m FROM Member m LEFT [OUTER] JOIN m.team t
```

### 세타 조인
```sql
select count(m) from Member m, Team t where m.username = t.name
```

### 조인 예제

```java
String query = "select m from Member m inner join m.team t";
List<Member> result = em.createQuery(query, Member.class)
                       .getResultList();
```

### ON 절 (JPA 2.1부터 지원)

#### 1. 조인 대상 필터링
```sql
-- JPQL
SELECT m, t FROM Member m LEFT JOIN m.team t on t.name = 'A'

-- SQL
SELECT m.*, t.* FROM Member m LEFT JOIN Team t ON m.TEAM_ID=t.id and t.name='A'
```

#### 2. 연관관계 없는 엔티티 외부 조인
```sql
-- JPQL
SELECT m, t FROM Member m LEFT JOIN Team t on m.username = t.name

-- SQL
SELECT m.*, t.* FROM Member m LEFT JOIN Team t ON m.username = t.name
```

## 🔍 서브 쿼리

### 지원 함수

- **[NOT] EXISTS (subquery)**: 서브쿼리에 결과가 존재하면 참
- **{ALL | ANY | SOME} (subquery)**
  - ALL 모두 만족하면 참
  - ANY, SOME: 같은 의미, 조건을 하나라도 만족하면 참
- **[NOT] IN (subquery)**: 서브쿼리의 결과 중 하나라도 같은 것이 있으면 참

### 서브 쿼리 예제

#### 1. 나이가 평균보다 많은 회원
```sql
select m from Member m
where m.age > (select avg(m2.age) from Member m2)
```

#### 2. 한 건이라도 주문한 고객
```sql
select m from Member m
where (select count(o) from Order o where m = o.member) > 0
```

#### 3. 팀A 소속인 회원
```sql
select m from Member m
where exists (select t from m.team t where t.name = '팀A')
```

#### 4. 전체 상품 각각의 재고보다 주문량이 많은 주문들
```sql
select o from Order o
where o.orderAmount > ALL (select p.stockAmount from Product p)
```

#### 5. 어떤 팀이든 팀에 소속된 회원
```sql
select m from Member m
where m.team = ANY (select t from Team t)
```

### JPA 서브 쿼리 한계

- JPA는 WHERE, HAVING 절에서만 서브 쿼리 사용 가능
- SELECT 절도 가능(하이버네이트에서 지원)
- **FROM 절의 서브 쿼리는 현재 JPQL에서 불가능**
  - 조인으로 풀 수 있으면 풀어서 해결

## 📋 JPQL 타입 표현과 기타식

### JPQL 타입 표현

- **문자**: 'HELLO', 'She\'s'
- **숫자**: 10L(Long), 10D(Double), 10F(Float)
- **Boolean**: TRUE, FALSE
- **ENUM**: jpabook.MemberType.Admin (패키지명 포함)
- **엔티티 타입**: TYPE(m) = Member (상속 관계에서 사용)

### ENUM 사용 예제

```java
String query = "select m.username, 'HELLO', TRUE from Member m " +
               "where m.type = jpql.MemberType.ADMIN";

List<Object[]> result = em.createQuery(query)
                         .getResultList();

for (Object[] objects : result) {
    System.out.println("objects = " + objects[0]);
    System.out.println("objects = " + objects[1]);
    System.out.println("objects = " + objects[2]);
}

// 파라미터 바인딩 사용 (권장)
String query = "select m.username, 'HELLO', TRUE from Member m " +
               "where m.type = :userType";

List<Object[]> result = em.createQuery(query)
                         .setParameter("userType", MemberType.ADMIN)
                         .getResultList();
```

### 기타 표현

- SQL과 문법이 같은 식
- EXISTS, IN
- AND, OR, NOT
- =, >, >=, <, <=, <>
- BETWEEN, LIKE, IS NULL

## 🔧 조건식 - CASE 식

### 기본 CASE 식

```sql
select
    case when m.age <= 10 then '학생요금'
         when m.age >= 60 then '경로요금'
         else '일반요금'
    end
from Member m
```

### 단순 CASE 식

```sql
select
    case t.name
        when '팀A' then '인센티브110%'
        when '팀B' then '인센티브120%'
        else '인센티브105%'
    end
from Team t
```

### COALESCE

사용자 이름이 없으면 이름 없는 회원을 반환

```sql
select coalesce(m.username,'이름 없는 회원') from Member m
```

### NULLIF

사용자 이름이 '관리자'면 null을 반환하고 나머지는 본인의 이름을 반환

```sql
select NULLIF(m.username, '관리자') from Member m
```

## 🎯 JPQL 기본 함수

### JPQL이 제공하는 표준 함수

- **CONCAT**: 문자열 합치기
- **SUBSTRING**: 부분 문자열
- **TRIM**: 공백 제거
- **LOWER, UPPER**: 대소문자 변환
- **LENGTH**: 문자열 길이
- **LOCATE**: 문자열 위치
- **ABS, SQRT, MOD**: 수학 함수
- **SIZE, INDEX(JPA 용도)**: 컬렉션 함수

### 기본 함수 예제

```java
String query = "select concat('a', 'b') From Member m";
String query = "select substring(m.username, 2, 3) From Member m";
String query = "select locate('de', 'abcdefg') From Member m";
String query = "select size(t.members) From Team t";
```

### 사용자 정의 함수 호출

하이버네이트는 사용전 방언에 추가해야 한다.

- 사용하는 DB 방언을 상속받고, 사용자 정의 함수를 등록한다.

```sql
select function('group_concat', i.name) from Item i
```

```java
public class MyH2Dialect extends H2Dialect {
    
    public MyH2Dialect() {
        registerFunction("group_concat", new StandardSQLFunction("group_concat", StandardBasicTypes.STRING));
    }
}
```

```xml
<!-- persistence.xml -->
<property name="hibernate.dialect" value="dialect.MyH2Dialect"/>
```

## ⚡ 페치 조인(fetch join)

### 페치 조인의 특징

- SQL 조인 종류X
- JPQL에서 **성능 최적화**를 위해 제공하는 기능
- 연관된 엔티티나 컬렉션을 **SQL 한 번에 함께 조회**하는 기능
- join fetch 명령어 사용
- 페치 조인 ::= [ LEFT [OUTER] | INNER ] JOIN FETCH 조인경로

### 엔티티 페치 조인

회원을 조회하면서 연관된 팀도 함께 조회(SQL 한 번에)

```sql
[JPQL]
select m from Member m join fetch m.team

[SQL]
SELECT M.*, T.* FROM MEMBER M
INNER JOIN TEAM T ON M.TEAM_ID=T.ID
```

### 페치 조인 사용 코드

```java
String jpql = "select m from Member m join fetch m.team";

List<Member> members = em.createQuery(jpql, Member.class)
                        .getResultList();

for (Member member : members) {
    // 페치 조인으로 회원과 팀을 함께 조회해서 지연 로딩X
    System.out.println("username = " + member.getUsername() + ", " +
                      "teamName = " + member.getTeam().name());
}
```

### 컬렉션 페치 조인

일대다 관계, 컬렉션 페치 조인

```sql
[JPQL]
select t from Team t join fetch t.members where t.name = '팀A'

[SQL]
SELECT T.*, M.* FROM TEAM T
INNER JOIN MEMBER M ON T.ID=M.TEAM_ID
WHERE T.NAME = '팀A'
```

### 페치 조인과 DISTINCT

SQL의 DISTINCT는 중복된 결과를 제거하는 명령

```sql
select distinct t from Team t join fetch t.members where t.name = '팀A'
```

**JPQL의 DISTINCT 2가지 기능:**
1. SQL에 DISTINCT를 추가
2. 애플리케이션에서 엔티티 중복 제거

### 페치 조인과 일반 조인의 차이

#### 일반 조인
```sql
[JPQL]
select t from Team t join t.members m where t.name = '팀A'

[SQL]
SELECT T.* FROM TEAM T
INNER JOIN MEMBER M ON T.ID=M.TEAM_ID
WHERE T.NAME = '팀A'
```

- JPQL은 결과를 반환할 때 연관관계 고려X
- 단지 SELECT 절에 지정한 엔티티만 조회할 뿐
- 여기서는 팀 엔티티만 조회하고, 회원 엔티티는 조회X

#### 페치 조인
```sql
[JPQL]
select t from Team t join fetch t.members where t.name = '팀A'

[SQL]
SELECT T.*, M.* FROM TEAM T
INNER JOIN MEMBER M ON T.ID=M.TEAM_ID
WHERE T.NAME = '팀A'
```

- 페치 조인을 사용할 때만 연관된 엔티티도 함께 **조회(즉시 로딩)**
- **페치 조인은 객체 그래프를 SQL 한번에 조회하는 개념**

### 페치 조인의 한계

- **페치 조인 대상에는 별칭을 줄 수 없다**
  - 하이버네이트는 가능, 가급적 사용X
- **둘 이상의 컬렉션은 페치 조인 할 수 없다**
- **컬렉션을 페치 조인하면 페이징 API(setFirstResult, setMaxResults)를 사용할 수 없다**
  - 일대일, 다대일 같은 단일 값 연관 필드들은 페치 조인해도 페이징 가능
  - 하이버네이트는 경고 로그를 남기고 메모리에서 페이징(매우 위험)

### 페치 조인 정리

- 모든 것을 페치 조인으로 해결할 수는 없음
- 페치 조인은 객체 그래프를 유지할 때 사용하면 효과적
- 여러 테이블을 조인해서 엔티티가 가진 모양이 아닌 전혀 다른 결과를 내야 하면, 페치 조인 보다는 일반 조인을 사용하고 필요한 데이터들만 조회해서 DTO로 반환하는 것이 효과적

## 📊 다형성 쿼리

### TYPE

조회 대상을 특정 자식으로 한정

```sql
[JPQL]
select i from Item i where type(i) IN (Book, Movie)

[SQL]
select i from i where i.DTYPE in ('B','M')
```

### TREAT(JPA 2.1)

자바의 타입 캐스팅과 유사

```sql
[JPQL]
select i from Item i where treat(i as Book).auther = 'kim'

[SQL]
select i.* from Item i where i.DTYPE = 'B' and i.auther = 'kim'
```

## 🏷️ 엔티티 직접 사용

### 기본 키 값

JPQL에서 엔티티를 직접 사용하면 SQL에서 해당 엔티티의 기본 키 값을 사용

```sql
[JPQL]
select count(m.id) from Member m // 엔티티의 아이디를 사용
select count(m) from Member m    // 엔티티를 직접 사용

[SQL] (JPQL 둘다 같은 다음 SQL 실행)
select count(m.id) as cnt from Member m
```

### 엔티티를 파라미터로 전달

```java
String jpql = "select m from Member m where m = :member";
List resultList = em.createQuery(jpql)
                   .setParameter("member", member)
                   .getResultList();
```

### 식별자를 직접 전달

```java
String jpql = "select m from Member m where m.id = :memberId";
List resultList = em.createQuery(jpql)
                   .setParameter("memberId", memberId)
                   .getResultList();
```

### 외래 키 값

```java
Team team = em.find(Team.class, 1L);

String qlString = "select m from Member m where m.team = :team";
List resultList = em.createQuery(qlString)
                   .setParameter("team", team)
                   .getResultList();

// 위와 아래는 같은 SQL 실행
String qlString = "select m from Member m where m.team.id = :teamId";
List resultList = em.createQuery(qlString)
                   .setParameter("teamId", teamId)
                   .getResultList();
```

## 📜 Named 쿼리

### Named 쿼리 - 정적 쿼리

- 미리 정의해서 이름을 부여해두고 사용하는 JPQL
- 정적 쿼리
- 어노테이션, XML에 정의
- 애플리케이션 로딩 시점에 초기화 후 재사용
- **애플리케이션 로딩 시점에 쿼리를 검증**

### Named 쿼리 - 어노테이션

```java
@Entity
@NamedQuery(
    name = "Member.findByUsername",
    query="select m from Member m where m.username = :username")
public class Member {
    // ...
}

List<Member> resultList = 
    em.createNamedQuery("Member.findByUsername", Member.class)
      .setParameter("username", "회원1")
      .getResultList();
```

### Named 쿼리 - XML에 정의

```xml
[META-INF/persistence.xml]
<persistence-unit name="jpabook" >
    <mapping-file>META-INF/ormMember.xml</mapping-file>
```

```xml
[META-INF/ormMember.xml]
<?xml version="1.0" encoding="UTF-8"?>
<entity-mappings xmlns="http://java.sun.com/xml/ns/persistence/orm" version="2.0">
    
    <named-query name="Member.findByUsername">
        <query><![CDATA[
            select m
            from Member m
            where m.username = :username
        ]]></query>
    </named-query>
    
    <named-query name="Member.count">
        <query>select count(m) from Member m</query>
    </named-query>
    
</entity-mappings>
```

### Named 쿼리 환경에 따른 설정

- XML이 항상 우선권을 가진다
- 애플리케이션 운영 환경에 따라 다른 XML을 배포할 수 있다

---

## 📝 정리

### JPQL - 객체지향 쿼리 언어
1. **JPQL은 객체지향 쿼리 언어**다. 따라서 테이블을 대상으로 쿼리하는 것이 아니라 **엔티티 객체를 대상으로 쿼리**한다.
2. **JPQL은 SQL을 추상화**해서 특정데이터베이스 SQL에 의존하지 않는다.
3. JPQL은 결국 SQL로 변환된다.

### 기본 문법과 쿼리 API
- SELECT, FROM, WHERE, GROUP BY, HAVING, JOIN 지원
- TypedQuery, Query
- getResultList(), getSingleResult()
- 파라미터 바인딩 - 이름 기준 권장

### 프로젝션
- 엔티티, 임베디드 타입, 스칼라 타입
- new 명령어로 DTO 바로 조회

### 페이징
- setFirstResult(), setMaxResults()
- 데이터베이스 방언에 맞춰 SQL 변환

### 조인
- 내부 조인, 외부 조인, 세타 조인
- ON 절을 활용한 조인

### 서브 쿼리
- WHERE, HAVING 절에서만 사용 가능
- FROM 절의 서브 쿼리는 JPQL에서 불가능

### JPQL 타입 표현
- 문자, 숫자, Boolean, ENUM, 엔티티 타입

### 조건식
- CASE 식, COALESCE, NULLIF

### JPQL 함수
- 표준 함수와 사용자 정의 함수

### 페치 조인
- **성능 최적화를 위한 기능**
- 연관된 엔티티나 컬렉션을 한 번에 조회
- N+1 문제 해결의 핵심

### Named 쿼리
- 정적 쿼리만 가능
- 애플리케이션 로딩 시점에 쿼리 검증

---

[← 이전: 값 타입](./06-value-types.md) | [목차로 돌아가기](./README.md) | [다음: 웹 애플리케이션과 영속성 관리 →](./08-web-application-persistence.md)