# JPA (Java Persistence API) 학습 가이드

## 📚 목차

### 1. [JPA 소개 및 개념](./01-jpa-introduction.md)
- JPA란 무엇인가?
- ORM과 JPA의 관계
- JPA의 장단점
- 환경 설정

### 2. [엔티티 매핑](./02-entity-mapping.md)
- @Entity, @Table
- 기본 키 매핑
- 필드와 컬럼 매핑
- 데이터베이스 스키마 자동 생성

### 3. [연관관계 매핑](./03-association-mapping.md)
- 단방향 연관관계
- 양방향 연관관계
- 일대일 관계
- 일대다 관계
- 다대일 관계
- 다대다 관계

### 4. [고급 매핑](./04-advanced-mapping.md)
- 상속관계 매핑
- @MappedSuperclass
- 복합 키와 식별 관계 매핑
- 조인 테이블

### 5. [프록시와 연관관계 관리](./05-proxy-and-lazy-loading.md)
- 프록시
- 즉시 로딩과 지연 로딩
- 영속성 전이: CASCADE
- 고아 객체

### 6. [값 타입](./06-value-types.md)
- 기본값 타입
- 임베디드 타입
- 값 타입과 불변 객체
- 값 타입의 비교
- 값 타입 컬렉션

### 7. [객체지향 쿼리 언어 (JPQL)](./07-jpql.md)
- JPQL 소개
- 기본 문법과 기능
- 페이징 API
- 집합과 정렬
- 조인
- 서브 쿼리
- 조건식
- 다형성 쿼리
- 엔티티 직접 사용
- Named 쿼리

### 8. [웹 애플리케이션과 영속성 관리](./08-web-application-persistence.md)
- 트랜잭션 범위의 영속성 컨텍스트
- 준영속 상태와 지연 로딩
- OSIV(Open Session In View)
- 성능 최적화

### 9. [스프링 데이터 JPA](./09-spring-data-jpa.md)
- 스프링 데이터 JPA 소개
- 공통 인터페이스 기능
- 쿼리 메소드 기능
- 확장 기능
- 스프링 데이터 JPA 분석

### 10. [성능 최적화](./10-performance-optimization.md)
- N+1 문제
- 페치 조인
- 엔티티 그래프
- 배치 처리
- SQL 힌트
- 트랜잭션을 지원하는 쓰기 지연

---

## 🎯 학습 목표

이 가이드를 통해 다음과 같은 내용을 학습할 수 있습니다:

- JPA의 기본 개념과 동작 원리 이해
- 엔티티 설계 및 연관관계 매핑
- JPQL을 활용한 객체지향 쿼리 작성
- 스프링 데이터 JPA를 활용한 실무 개발
- JPA 성능 최적화 기법

## 📖 참고 자료

- Java Persistence API 공식 문서
- 김영한의 자바 ORM 표준 JPA 프로그래밍
- Spring Data JPA 공식 문서
- Hibernate 공식 문서

---

**Happy Learning! 🚀**