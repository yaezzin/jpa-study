# 10장 p2. JPQL

**목표**

엔티티를 쿼리하는 가장 기본적인 방법은 JPQL이다. JPQL의 사용 방법을 알아보자.

**목차**

1. 기본 문법과 쿼리 API
2. 파라미터 바인딩
3. 프로젝션
4. 페이징 API
5. 집합과 정렬
6. JPQL 조인
7. 페치 조인
8. 경로 표현식
9. 서브 쿼리
10. 조건식
11. 다형성 쿼리
12. 사용자 정의 함수 호출
13. 기타 정리
14. 엔티티 직접 사용
15. Named 쿼리 (정적 쿼리)

### JPQL의 특징

1. JPQL은 엔티티 객체를 대상으로 쿼리한다.
2. JPQL은 SQL을 추상화해서 특정 SQL에 의존하지 않는다.
3. JPQL은 결국 SQL로 변환된다.

## 기본 문법과 쿼리 API

### SELECT

```sql
SELECT m FROM **Member** AS m WHERE m.username = 'KIM'
```

1. 엔티티와 속성은 대소문자를 구분한다.
2. 클래스 명이 아니라 엔티티 명을 적어야 한다.
3. 별칭을 필수로 사용해야 한다.

### TypeQuery와 Query

- TypeQuery : 반환할 타입을 명확하게 지정할 수 있을 경우
- Query : 반환할 타입을 명확하게 지정할 수 없거나, 여러 엔티티나 컬럼을 선택할 경우

```java
**TypedQuery**<Member> query =
     em.createQuery("SELECT m FROM Member m", Member.class);

List<Member> resultList = query.getResultList();
for (Member member : resultList) {
    System.out.println("member = " + member);
}
```

```java
Query query =
     em.createQuery("SELECT m.username, m.age FROM Member m"); 

List resultList = query.getResultList();
for (Object o : resultList) {
    Object[] result = (Object[]) o; // 결과가 둘 이상일 경우 Object[]    
		System.out.println("username = " + result[0]);
    System.out.println("age = " + result[1]);
}
```

### 결과 조회

- query.getResultList() : 결과를 예제로 반환한다. 결과가 없을 경우 빈 컬렉션 반환
- query.getSingleResult() : 결과가 정확히 하나일 때 사용한다.
    - 결과가 없을 경우 javax.persistence.NoResultException 예외 발생
    - 결과가 1보다 많으면javax.persistence.NonUniqueResultException 예외 발생

## 파라미터 바인딩

- 이름 기준 파라미터 : 파라미터를 이름으로 구분한다. 더 가독성이 좋다.
- 위치 기준 파라미터 : ? 다음에 위치 값을 지정한다.

```java
// 이름 기준 바인딩
String usernameParam = "User1"; 
TypedQuery<Member> query =
     em.createQuery("SELECT m FROM Member m where m.username = :username", Member.class);
                                                            // 파라미터 정의
query.setParameter("username", usernameParam); // 파라미터 바인딩
List<Member> resultList = query.getResultList(); // 아래와 같이 작성 가능(메소드 체인)
List<Member> members =
     em.createQuery("SELECT m FROM Member m where m.username = :username", Member.class)
    .setParameter("username", usernameParam)
    .getResultList();
```

```java
// 위치 기준 바인딩
List<Member> member =
    em.createQuery("SELECT m FROM Member m where m.username = ?1", Member.class)
    .setParameter(1, usernameParam)
    .getResultList();
```

## 프로젝션

SELECT 절에 조회할 대상을 지정할 수 있다.

- 엔티티 프로젝션 : 원하는 객체를 바로 조회한다. 영속성 컨텍스트가 관리한다.
- 임베디드 타입 프로젝션 : 엔티티와 달리 조회의 시작점이 될 수 없다. 영속성 컨텍스트에서 관리되지 않는다.
- 스칼라 타입(기본값) 프로젝션
- 여러 값 조회 : 엔티티 중 필요한 데이터들만 선택. TypeQuery가 아닌 Query를 사용한다.
- new 명령어 : 자바 코드로 객체 변환 작업을 해주지 않아도 된다. new 명령어 다음에 반환받을 클래스 지정

```sql
# 엔티티 프로젝션
SELECT m FROM Member m        // 회원
SELECT m.team FROM Member m   // 팀
```

```java
// 임베디드 타입 프로젝션
String query = "SELECT o.address FROM Order o";
List<Address> addresses = em.createQuery(query, Address.class)
```

```java
// 스칼라 타입 프로젝션
List<String> username =
     em.createQuery("SELECT username FROM Member m", String.class)
      .getResultList();
```

```java
// 여러 값 조회
List<Object[]> resultList =
     em.createQuery("SELECT o.member, o.product, o.orderAmount FROM Order o")
     .getResultList();
for (Object[] row : resultList) {
    Member member = (Member) row[0];    // 엔티티
    Product product = (Product) row[1];    // 엔티티
    int orderAmount = (Integer) row[2];    // 스칼라
}
```

```java
// new 명령어 사용
public class UserDTO {
     private String username;
     private int age;
     public UserDTO(String username, int age) {
        this.username = username;
        this.age = age;
    }
    //...
} 
TypeQuery<UserDTO> query =
    em.createQuery("SELECT **new** test.jpql.UserDTO(m.username, m.age)
                        FROM Member m", UserDTO.class);
 List<UserDTO> resultList = query.getResultList();
```

## 페이징 API

데이터 방언 때문에 DB마다 다른 페이징 처리를 동일한 API로 처리할 수 있다.

- setFirstResult (int startPosition) : 조회 시작 위치(zero base)
- setMaxResults (int maxResult) : 조회할 데이터 수

```java
TypeQuery<Member> query =
     em.createQuery("SELECT m FROM Member m ORDER BY m.username DESC", Member.class); 
// 11~30번 데이터 조회
query.setFirstResult(10);    // 조회 시작 위치
query.setMaxResults(20);    // 조회할 데이터 수
query.getResultList();
```

## 집합과 정렬

- 집합 함수 : COUNT, MAX, MIN, AVG, DU<
- GROUP BY, HAVING : 통계 데이터를 구할 때 특정 그룹끼리 묶어줌
- ORDER BY : DESC, ASC

```java
Query query = em.createQuery(
	"select  count(m), sum(m.age), avg(m.age) 
		from Member m left join m.team t **group by t.name having avg(m.age) >= 10");**
```

## JPQL 조인

JPQL도 조인을 지원하는데 SQL 조인과 기능은 같고 문법만 약간 다르다.

- 내부 조인  : INNER JOIN, 연관 필드를 꼭 적어줘야 한다.
    
    → select m from Member m inner join m.team t where [t.name](http://t.name/) = :teamName
    
- 외부 조인 : OUTER는 생략 가능해서 LEFT JOIN으로 적는다.
    
    → select m from Member m left join m.team t where [t.name](http://t.name/) = :teamName
    
- 컬렉션 조인 : 일대다나 다대다 관계처럼 컬렉션을 사용하는 곳에 조인한다.
    
    → select t from Team t left join t.members m
    
- 세타 조인 : 전혀 관계 없는 엔티티를 조인할 수 있다. 내부 조인만 지원한다.
    
    → select m from Member m, Team t where [m.name](http://m.name/) = [t.name](http://t.name/)
    
- JOIN ON : 조인 대상을 필터링하고 조인한다. 외부 조인에서만 사용한다.
    
    → select m, t from Member m left join [m.team](http://m.team) on t[.name](http://t.name/) = ‘A’
    

## 페치 조인

페치 조인은 JPQL에서 성능 최적화를 위해 제공하는 기능이다. 연관된 엔티티나 컬렉션을 한번에 같이 조회하는 기능인데, join fetch 명령어로 사용할 수 있다.

- 엔티티 fetch join : 회원 엔티티를 조회하면서 연관된 팀 엔티티도 같이 조회한다. 별칭을 사용할 수 없다. 연관 객체를 실제 조회하므로 지연 로딩이 일어나지 않는다.
    
    ```sql
    select m from Member m join fetch m.team
    ```
    
- 컬렉션 패치 조인 : 일대다 관계인 컬렉션을 페치 조인할 수 있다. 이때 일대다는 결과가 증가할 수 있으므로 Distinct 키워드를 함께 사용해야 한다.
    
    ```sql
    select distinct t from Team t join fetch t.members
    ```
    

### 페치 조인의 특징과 한계

페치 조인을 사용하면 SQL 한번으로 연관된 엔티티들을 함게 조회할 수 있어서 SQL 호출 횟수를 줄여 성능을 최적화할 수 있다.  글로벌 로딩전략을 즉시 로딩으로 설정하면 애플리케이션 전체에 항상 즉시 로딩이 일어난다.  일부는 빠를 수 있지만 전체로 보면 사용하지 않는 엔티티를 자주 로딩하므로 오히려 성능에 악영향을 미칠 수 있다. **따라서 글로벌 로딩 전략은 될 수 있으면 지연 로딩을 사용하고 최적화가 필요하면 페치 조인을 적용하는 것이 효과적이다.** 

## 경로 표현식

- 상태 필드 : 단순히 값을 저장하기 위한 필드
- 연관 필드 : 연관관계를 위한 필드. 임베디드 타입, 엔티티, 컬렉션. 경로 탐색을 하면 내부 조인이 발생한다.
    - 단일 값 연관 필드 : @ManyToOne, @OneToOne, 대상이 엔티티
    - 컬렉션 값 연관 필드 : @OneToMany, @ManyToMany, 대상이 컬렉션

```sql
-- 단일 값 연관 경로 탐색 : 내부 조인 세번(member, team, product)
select o.member.team
from Order o
where o.product.name = 'productA' and o.address.city = 'JINJU'
```

```sql
-- 컬렉션 값 연관 경로 탐색 : 컬렉션에서 경로 탐색을 시작할 수 없다
select t.members.username from Team t // 실패
select m.username from Team t join t.members m // 성공
```

## 서브 쿼리

JPQL도 SQL처럼 서브 쿼리를 지원한다. 몇가지 제약이 있는데 서브쿼리를 WHERE, HAVING 절에서만 사용할 수 있고 SELECT, FROM 절에는 사용할 수 없다.

- [NOT] EXISTS (subquery)
- { ALL | ANY | SOME } (subquery)
- [NOT] IN (subquery)

```sql
-- 팀 A소속 회원
select m
from Member m
where exists 
(select t from m.team t where t.name = '팀A')
```

```sql
-- 전체 상품 각각의 재고보다 주문량이 많은 주문들
select o
from Order o
where o.orderAmount > ALL (select p.stockAmount from Product p)
```

```sql
-- 20세 이상을 보유한 팀
select t
from Team t
where t IN (select t2 from Team t2 JOIN t2.members m2 where m2.age >= 20)
```

## 조건식

### 타입 표현

- 문자 : ‘’, “”, ‘문자’’문자’
- 숫자 : L, D, F
- 날짜 : DATE, TIME, DATETIME
- Boolean : TRUE, FALSE
- Enum : 패키지 명을 포함한 전체 이름, test.MemberType.Enum
- 엔티티 타입 : TYPE(m) = Member

### 연산자 우선 순위

1. 경로 탐색 연산: '.'
2. 수학 연산: +, -, *, /
3. 비교 연산: >=, >, <=, <, <>, BETWEEN, LIKE, IN, IS NULL, IS EMPTY, MEMBER, EXIST
4. 논리연산: NOT, AND, OR

### 논리 연산과 비교식

- 논리 : AND, OR, NOT
- 비교식 : >=, >, <=, <, <>

### Between, IN, Like, NULL

- Between : X [NOT] BETWEEN A AND B
- IN : X [NOT] IN (예제)
- Like : 문자표현식 [NOT] LIKE 패턴값 [ESCAPE 이스케이프문자]
- NULL : [단일값 경로 | 입력 파라미터] IS [NOT] NULL

### 컬렉션 식

- 빈 컬렉션 비교 식 : IS [NOT] EMPTY
- 컬렉션의 멤버 식 : [엔티티나 값] [NPT] MEMBER [OF] [컬렉션 값 연관 경로]

### 스칼라 식

- 문자 함수 : CONCAT, SUBSTRING, LOWER, UPPER, LENGTH, LOCATE
- 수학 함수 : ABS, SORT, MOD, SIZE, INDEX
- 날짜 함수 : CURRENT_DATE, CURRENT_TEIM, CURRENT_TIMESTAMP

### CASE 식

기본 CASE, 심플 CASE, COALESCE, NULLIF

## 다형성 쿼리

엔티티의 상속 관계에서 엔티티를 캐스팅할 때 사용한다.

- TYPE : 엔티티의 상속 구조에서 조회 대상을 특정 자식 타입으로 한정할 때 사용한다.
    
    ```sql
    -- Item 중 Book, Movie를 조회
    select i
    from Item i
    where type(i) IN (Book, Movie)
    ```
    
- TREAT : 상속 구조에서 부모 타입을 특정 자식 타입으로 캐스팅할 때 사용한다.
    
    ```sql
    select i
    from Item i
    where treat(i as Book).author = 'kim'
    ```
    

## 엔티티 직접 사용

- 기본 키 값 : 엔티티 객체를 직접 사용하면 SQL에서는 엔티티의 기본값을 사용한다.
- 외래 키 값 : 상태 필드의 경로를 검색해 외래키와 식별자 값을 비교한다.

```java

// 기본 키 값
String ql = "select m from Member m where m.team = :team";

// 외래 키 값
Strign ql = "select m from Member m where m.team.id = :teamId";
```

## Named 쿼리 (정적 쿼리)

Named 쿼리로 정적 쿼리를 만들 수 있다. 정적 쿼리는 미리 정의한 쿼리에 이름을 부여해서 필요할 때 사용한다. 한번 정의하면 변경할 수 없는 정적인 쿼리이다. Named 쿼리를 사용하면 애플리케이션을 로딩할 때 JPQL 문법을 체크할 수 있고, 사용 시점에 파싱된 결과를 재사용할 수 있고 DB도 한번 사용한 쿼리를 재사용할 수 있으므로 성능상 이점이 있다. Annotation으로 자바 코드에 작성하거나 XMl 문서에 작성할 수 있는데, 자바로 멀티 라인을 다루는 것은 귀찮은 일이므로 xml에 작성하는 것이 편하다.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<entity-mappings
        xmlns="http://java.sun.com/xml/ns/persistence/orm" version="2.1">
    <named-query name="Member.findByUsername">
        <query><![CDATA[
             select m
             from Member m
             where m.username = :username
        ]]></query>
    </named-query>
     <named-query name="Member.count">
        <query>
            select count(m)
            from Member m
        </query>
    </named-query> 
</entity-mappings>
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<persistence xmlns="http://xmlns.jcp.org/xml/ns/persistence" version="2.1">
     <persistence-unit name="test">
        <mapping-file> META-INF/Member.xml </mapping-file>
    </persistence-unit>
</persistence>
```

xml 파일을 작성하고, 이 파일을 인식할 수 있도록 persistence.xml에 등록했다. 참고로 xml이 Annotation보다 우선권을 가진다.
