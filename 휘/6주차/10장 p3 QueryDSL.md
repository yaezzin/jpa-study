# 10장 p3. QueryDSL

**목표**

JPQL을 자바 코드로 작성하도록 도와주는 빌더 클래스인 Criteria와 QueryDSL의 사용 방법을 알아보자.

**목차**

1. 기본 쿼리 생성
2. 검색 조건
3. 결과 조회
4. 페이징과 정렬
5. 그룹 : groupby(), having()
6. 조인
7. 서브쿼리
8. 프로젝션과 결과 반환
9. 수정, 삭제 배치 쿼리
10. 동적 쿼리
11. 메소드 위임

**정리**

jpql 대신 빌더 클래스를 사용하면 쿼리를 문자가 아닌 코드로 안전하게 작성할 수 있다. Criteria를 사용하면 너무 코드가 복잡해서 사용하기 어렵다. QueryDSL은 쉽고 단순해서 사용하기 좋다. 그리고 둘다 복잡한 동적 쿼리를 해결할 수 있다.

# QueryDSL

Criteria는 너무 복잡하고 장황해서 사용하기 어렵다. 깔끔하고 사용하기 쉬운 QueryDSL을 우선 알아보자.

## 기본 쿼리 생성

1. JPAQuery 객체 생성
2. 쿼리 타입 생성, 생성자에 별칭 부여
3. 쿼리 실행 및 결과 반환

```java
EntityManager em = emf.createEntityManager();

JPAQuery query = new JPAQuery(em);
QMember member = new QMember("m");

List<Member> members = 
		query.from(member)
		.where(member.username.eq("kim"))
		.list(member);
```

## 검색 조건

- where(조건절)
    - 값.between(시작값, 끝값)
    - 값.contains()
    - 값.startsWith()
- and()나 or()을 사용할 수 있다.

```java
JPAQuery query = new JPAQuery(em);
QItem item = QItem.item;
List<Item> list = query.from(item)
	**.where(item.name.eq("좋은상품").and(item.price.gt(20000))**
	.list(item); // 조회 할 프로젝션 지정
```

## 결과 조회

- uniqueResult() : 조회 결과가 한 건일 때 사용. 없으면 null 반환, 하나 이상이면 NonUniqueResultException 예외 발생
- singleResult() : 조회 결과가 한 건일 때 사용. 하나 이상이면 처음 데이터 반환
- list() : 결과가 하나 이상일 때 사용, 결과 없을시 빈 컬렉션 반환

## 페이징과 정렬

- orderBy(정렬조건+)
- offset(), limit()

```java
query.from(item)
	.where(item.price.gt(20000))
	**.orderBy(item.price.desc(), item.stockQuantity.asc()) //정렬
	.offset(10).limit(20) // 페이징**
	.list(item);
```

- listResults() : 전체 데이터 수 조회
- 전체 데이터 조회를 위한 count 쿼리를 한번 더 실행한다.

```java
SearchResults<Item> result =
	query.from(item)
	.where(item.price.gt(10000))
	.offset(10).limit(20)
	**.listResults(item);**
```

## 그룹

- groupby() : 그룹화
- having() : 결과 제한

```java
query.from(item)
	.groupBy(item.price)
	.having(item.price.gt(10000)) //조건
	.list(item);
```

## 조인

- innerJoin, leftJoin, rightJoin, fullJoin
- JPQL의 성능 최적화를 위한 fetch 조인, on

```java
// 기본 조인
QOrder order = QOrder.order;
QMember member = QMember.member;
QOrderItem orderItem = QOrderItem.orderItem;

query.from(order)
	**.join(order.member, member) // 조인대상, 쿼리타입
	.leftJoin(order.orderItems, orderItem)**
	.list(order);
```

```java
// on : 조건
query.from(order)
	.leftJoin(order.orderItems, orderItem)
	**.on(orderItem.count.gt(2))**
	.list(order);
```

```java
// fetch
query.from(order)
	.innerJoin(order.member, member)**.fetch()**
	.leftJoin(order.orderItems, orderItem)**.fetch()**
	.list(order);
```

```java
// from 절에 여러 조건 사용
qQOrder order = QOrder.order;
QMember member = QMember.member;

query**.from(order, member)**
	.where(order.member.eq(member))
	.list(order);
```

## 서브쿼리

1. com.mysema.query.jpa.JPASubQuery 생성
2. 결과가 하나면 unique() , 여러 건이면 list() 사용

```java
QItem item = QItem.item;
QItem itemSub = new QItem("itemSub");

query.from(item)
	.where(item.in(
		new JPASubQuery().from(itemSub)
			.where(item.name.eq(itemSub.name))
			.list(itemSub)
	))
	**.list(item);**
```

## 프로젝션과 결과 반환

프로젝션 : select 절에 조회 대상을 지정하는 것

- 프로젝션 대상이 하나 : .list(필드)
- 여러 컬럼 반환과 튜플 : .list(필드+)
- 빈 생성 : .list(Projections.bean(DTO, 필드+)

```java
// 프로젝션 대상이 하나
QItem item = QItem.item;
List<String> result = query.from(item)**.list(item.name);**
for(String name : result) {
	System.out.println("name = " + name);
}
```

```java
// 여러 컬럼 반환과 튜플
QItem item = QItem.item;

List<Tuple> result = query.from(item)
			**.list(item.name, item.price);**

for(Tuple tuple : result) {
	System.out.println("name = " + tuple.get(item.name));
	System.out.println("price = " + tuple.get(item.price)); 
}
```

```java
// 빈 생성
QItem item = QItem.item;
List<ItemDTO> result = query.from(item).list(
	projections.bean(ItemDTO.class, item.name.as("username"), item.price); //Setter 사용

query.from(item).list(
	projections.fileds(ItemDTO.class, item.name.as("username"), item.price); //필드 직접 접근

query.from(item).list(
	projections.constructor(ItemDTO.class, item.name, item.price) // 생성자 사용
```

## 수정, 삭제 배치 쿼리

JPQL 배치처럼 영속성 컨텍스트를 무시하고 데이터베이스를 직접 쿼리한다.

```java
// 수정 배치 쿼리
QItem item = QItem.item;
**JPAUpdateClause updateClause = new JPAUpdateClause(em, item);**
long count = updateClause.where(item.name.eq("시골개발자의 JPA 책"))
	**.set(item.price, item.price.add(100))**
	.execute();
```

```java
// 삭제 배치 쿼리
QItem item = QItem.item;
**JPADeleteClause deleteClause = new JPADeleteClause(em, item);**
long count = updateClause.where(item.name.eq("시골개발자의 JPA 책"))
	.execute();
```

## 동적 쿼리

1. BooleanBuilder를 생성하여 조건을 추가
2. where 절에 bb 사용

```java
SearchParam param = new SearchParam();
param.setName("시골개발자”);
param.setPrice(10000);

QItem item = QItem.item;

**BooleanBuilder builder = new BooleanBuilder();**
if (StringUtils.hasText(param.getName())) {
    **builder.and(item.name.contains(param.getName()));**
}

****if (param.getPrice() != null) {
    **builder.and(item.price.gt(param.getPrice()));**
}

List<Item> result = query.from(item)
    **.where(builder)**
    .list(item);
```

## 메소드 위임

쿼리 타입에 검색 조건을 직접 정의할 수 있다.

@QueryDelegate(쿼리타입,필요한 파라미터..) 사용 

```java
// 정적 메소드 작성
// 쿼리 타입 클래스에 해당 기능이 추가된다
public class ItemExpression {
	@QueryDelegate(Item.class)
	public static BooleanExpression isExpensive(QItem item, Integer price) {
		return item.price.gt(price);
	}
}
```

```java
// 사용하기
query.from(item).where(item.isExpensive(30000)).list(item);
```
