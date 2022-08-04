# 12장 스프링 데이터 JPA

**목표**

CRUD 공통 부분을 처리하는 스프링 데이터 JPA를 이해하고 스프링 부트 프로젝트에 적용해보자.

**목차**

1. 스프링 데이터 JPA 소개
2. 공통 인터페이스 기능
3. 쿼리 메소드 기능
4. 명세
5. 사용자 정의 레포지토리 구현
6. Web 확장
7. 스프링 데이터 JPA가 사용하는 구현체
8. JPA 샵 예제에 적용하기
9. 스프링 데이터 JPA와 QueryDSL 통합

# 스프링 데이터 JPA 소개

스프링 데이터 JPA는 스프링 프레임워크에서 JPA를 편리하게 사용할 수 있도록 지원하는 프로젝트다. 데이터 접근 계층을 개발할 때 지루하게 반복되는 CRUD 문제를 세련된 방법을 해결한다. 

### 개발 방법

CRUD를 위한 공통 인터페이스를 제공한다. 그리고 레포지토리를 개발할 때 인터페이스만 작성하면, 실행 시점에 스프링 데이터 JPA가 구현 객체를 동적으로 생성해서 주입해준다. 구체적으로 말하자면 스프링 데이터 JPA가 애플리케이션을 실행할 때 basePackage에 있는 레포지토리 인터페이스를 찾아서 해당 인터페이스를 구현한 클래스를 동적으로 생성한 다음 스프링 빈으로 등록한다. 따라서 데이터 접근 계층을 개발할 때 구현 클래스 없이 인터페이스만 작성해도 개발을 완료할 수 있다. 

### JpaRepository 인터페이스

일반적인 CRUD 메소드는 JpaRepository 인터페이스가 공통적으로 제공하므로 문제가 없다. 그런데 엔티티의 필드를 사용해야하는 경우에는, 메소드명을 규칙에 따라 작성하면 스프링 데이터 JPA가 메소드 이름을 분석해서 JPQL을 실행한다.

# 공통 인터페이스 기능

### 스프링 데이터 JPA 사용 방법

```java
public interface MemberRepository extends JpaRepository<Member, Long> {}
```

- JpaRepository를 상속받은 인터페이스를 만든다
- 제네릭: 엔티티 클래스, 식별자 타입

### JpaRepository가 제공하는 주요 메소드

JpaRepository 공통 인터페이스를 사용하면 일반적인 CRUD를 해결할 수 있다.

- save(S) : 새로운 엔티티는 저장하고 이미 있는 엔티티는 수정한다.
- delete(T) : 엔티티 하나를 삭제한다. 내부에서 EM.remove()를 호출한다.
- findOne(ID) : 엔티티 하나를 조회한다. 내부에서 EM.find()를 호출한다.
- getOne(ID) : 엔티티를 프록시로 조회한다. 내부에서 EM.getReference()를 호출한다.
- findAll(…) : 모든 엔티티를 조회한다. Sort나 Pageable 조건을 파라미터로 제공할 수 있다.

# 쿼리 메소드 기능

스프링 데이터 JPA가 제공하는 마법 같은 기능이다. 크게 3가지 기능을 제공한다.

1. 메소드 이름으로 쿼리 생성
2. 메소드 이름으로 JPA NamedQuery 호출
3. @Query 어노테이션을 사용해서 레포지토리 인터페이스에 쿼리를 직접 정의한다.

## 메소드 이름으로 쿼리 생성

이메일과 이름으로 회원 조회

```java
public interface MemberRepository extends Repository<Member, Long> {
		List<Member> findByEmailAndName(String email, String name);
}
```

```sql
// 실행되는 JPQL
select m from Member where m.email = ?1 and m.name = ?2
```

쿼리 생성 규칙: [https://docs.spring.io/spring-data/jpa/docs/1.8.0.RELEASE/reference/html/#jpa.query-methods.query-creation](https://docs.spring.io/spring-data/jpa/docs/1.8.0.RELEASE/reference/html/#jpa.query-methods.query-creation)

## JPA NamedQuery

스프링 데이터 JPA는 메소드 이름으로 JPA Named 쿼리를 호출하는 기능을 제공한다. JPA Named 쿼리는 쿼리에 이름을 부여해서 사용하는 방법이다.

```java
// 쿼리 정의
@Entity
@NamedQuery(
        name = "Member.findByUseranme",
        query = "select u from Member u where u.username = :username"
)
public class Member {
    @Id
    @GeneratedValue
    private Long id;
    private String username;
}
```

```java
// 쿼리 호출
public interface MemberRepository extends Repository<Member, Long> {
		List<Member> findByUseranme(@Param("username") String username);
}
```

## @Query

레포지토리 메소드에 정적 쿼리를 직접 작성할 수 있다. 실행 시점에 문법 오류를 발견할 수 있다.

```java
public interface MemberRepository extends Repository<Member, Long> {
		@Query("select m from Member m where m.username = ?1")
		Member findByUseranme(String username);
}
```

```java
// 네이티브 SQL 작성
public interface MemberRepository extends Repository<Member, Long> {
		@Query("select * from Member where username = ?0")
		Member findByUseranme(String username);
}
```

## 파라미터 바인딩

위치 기반과 이름 기반 파라미터 바인딩을 모두 지원한다. 기본값은 위치 기반인데, 코드 가독성과 유지보수를 위해 이름 기반 파라미터 바인딩을 사용하자.

```java
public interface MemberRepository extends Repository<Member, Long> {
		@Query("select m from Member m where m.username = :name")
		List<Member> findByUseranme(@Param("name") String username);
}
```

## 벌크성 수정 쿼리

```java
@Modifying
@Query("update Product p set p.price = p.price * 1.1 where p.stockAmount < :stockAmount")
int bulkPriceUp(@Param("stockAmount") String stockAmount);
```

## 반환 타입

- 단건
    - 반환 타입 지정
    - 없으면 null 반환 (JPA 예외 무시)
    - 조회 결과가 2건 이상이면 NonUniqueResultException 예외 발생)
- 한 건 이상
    - 컬렉션 인터페이스 반환
    - 없으면 빈 컬렉션 반환

## 페이징과 정렬

- Sort: 정렬 기능 파라미터
- Pageable: 페이징 기능 파라미터, 내부에 Sort 포함

### 스프링 데이터 JPA가 페이징과 정렬 메소드 예제

```java
// count 쿼리 사용: 전체 데이터 건수 조회
Page<Member> findByName(String name, Pageable pageable);

// count 쿼리 사용 안함
List<Member> findByName(String name, Pageable pageable);

List<Member> findByName(String name, Sort sort);
```

```java
PageRequest pageRequest = new PageRequest(0, 10, new Sort(Direction.DISC, "name"));

Page<Member> result = memberRepository.findByNameStartingWith("김", pageRequest);

List<Member> members = result.getContent(); // 조회된 데이터
int totalPages = result.getTotalPages();    // 전체 페이지 수
boolean hasNextPage = result.hasNextPage(); // 다음 페이지 존재 여부
```

# 명세: 검색 조건

(헐 근데 실무에선 안쓴대..)

명세는 술어로 이루어져있고, 이는 참이나 거짓으로 평가된다. 그리고 이것은 AND, OR 같은 연산자로 조합할 수 있다. 예를 들어 데이터를 검색하기 위한 제약 조건 하나하나를 술어라 할 수 있다. 스프링 데이터 JPA는 Specification 클래스로 정의했다. 다양한 검색조건을 조립해서 새로운 검색조건을 쉽게 만들 수 있다.

```java
public interface OrderRepository extends JpaRepository<Order, Long>, 
		**JpaSpecificationExecutor<Order>** {}
```

```java
// 명세 사용 코드
// 검색조건 = 회원 이름 명세 + 주문 상태 명세
public List<Order> findOrders(String name) {
		List<Order> result = orderRepository.findAll(
				where(memberName(name)).and(isOrderStatus())
		);

		return result;
}
```

```java
// OrderSpec 명세 정의
public class OrderSpec {
		public static Specification<Order> memberName(final String memberName) {
				return new Specification<Order>() {
						public Predicate toPredicate(Root<Order> root,
								CriteriaQuery<?> query, CriteriaBuilder builder) {
	
								if(StringUtils.isEmpty(memberName)) return null;

								Join<Order, Member> m = root.join("member",
									 JoinType.INNER); //회원과 조인
								return builder.equal(m.get("name"), memberName)
						}
				}
		}

		public static Specification<Order> isOrderStatus() {
				return new Specification<Order>() {
						public Predicate toPredicate(Root<Order> root,
								CriteriaQuery<?> query, CriteriaBuilder builder) {
								
								return builder.equal(root.get("status"), OrderStatus.ORDER);
						}
				}
		}
}
```

# 사용자 정의 레포지토리 구현

다양한 이유로 데이터 계층 접근 메소드를 직접 구현해야할 때도 있다. 그렇다고 레포지토리를 직접 구현하면 공통 인터페이스가 제공하는 기능까지 모두 제공해야 한다. 스프링 데이터 JPA는 이런 문제를 우회해서 필요한 메소드만 구현할 수 있는 방법을 제공한다.

1. 사용자 정의 인터페이스 작성
    
    ```java
    public interface MemberRepositoryCustom {
    		public List<Member> findMemberCustom();
    }
    ```
    
2. 사용자 정의 구현 클래스
    
    ```java
    public class MemberRepository**Impl** implements MemberRepositoryCustom {
    		@Override
    		public List<Member> findMemberCustom(){
    				// 사용자 정의 구현
    		}
    }
    ```
    
3. 사용자 정의 인터페이스 상속
    
    ```java
    public interface MemberRepository 
    		extends JpaRepository<Member, Long>, MemberRepositoryCustom {}
    ```
    

# Web 확장

### 설정

```java
@Configuration
@EnableWebMvc
**@EnableSpringDataWebSupport**
public class WebAppConfig {}
```

### 도메인 클래스 컨버터

HTTP 파라미터로 넘어온 엔티티의 아이디로 엔티티 객체를 찾아서 바인딩해준다.

- HTTP 요청: /member/memberUpdateForm?id=1

```java
@Controller
public class MemberController {
		@RequestMapping("member/memberUpdateForm")
		public String memberUpdateForm(**@RequestParam("id") Member member**, Model model){
				model.addAttribute("member", member);
				return "member/memberSaveForm";
		}
}
```

### 페이징과 정렬

- HTTP 요청 : /members?page=0&size=20&sort=name,desc&sort=address.city
- Pageable 기본값: page=0, size=20

```java
@RequestMapping(value="/members", method = RequestMethod.GET)
public String list(@PageableDefault(size=12, sort="name", direction=Sort.Direction.DESC) Pageable pageable, Model model){
		Page<Member> page = memberService.findMembers(pageable);
		model.addAttribute("members", page.getContent());
		return "members/memberList";
}
```

### 페이징 정보가 둘 이상

@Qualifier를 이용해 접두사로 구분한다.

- HTTP 요청: /members?member_page=0&order_page=1

```java
public String list (
		@Qualifier("member") Pageable memberPageable,
		@Qualifier("order") Pageable orderPageable, ... )
```

# 스프링 데이터 JPA가 사용하는 구현체

스프링 데이터 JPA가 제공하는 공통 인터페이스는 SimpleJpaRepository 클래스가 구현한다. 코드 일부를 분석해보면 다음과 같다.

- `@Repository` : JPA 예외를 스프링이 추상화한 예외로 변환한다.
- `@Transactional` : 서비스 계층에서 트랜잭션을 시작하지 않았다면 레포지토리에서 트랜잭션을 시작한다.
- `@Transactional(readOnly=true)` : 데이터를 변경하지 않는 트랜잭션에서는 플러시를 생략하여 약간의 성능 향상을 얻을 수 있다.
- `save()` :  새로운 엔티티면 저장하고 이미 있는 엔티티면 병합한다.

# JPA 샵 예제에 적용하기

### 레포지토리

- MemberRepository
    
    ```java
    public interface MemberRepository extends JpaRepository<Member, Long> {
        List<Member> findByName(String name);
    }
    ```
    
- ItemRepository
    
    ```java
    public interface ItemRepository 
    		extends JpaRepository<Item, Long> {}
    ```
    
- OrderRepository
    
    ```java
    public interface OrderRepository 
    		extends JpaRepository<Order, Long>, JpaSpecificationExecutor<Order>, CustomOrderRepository {}
    ```
    

### 검색 조건: Criteria 대신 QueryDSL 사용

- ItemRepository
    
    ```java
    public interface ItemRepository 
    		extends JpaRepository<Item, Long>,
    						**QueryDslPredicateExecutor<Item>** {}
    ```
    
- QueryDslRepositorySupport 구현
    
    ```java
    public class OrderRepositoryImpl extends QueryDslRepositorySupport
    implements CutomOrderRepository {
    
    		public OrderRepositoryImpl(){
    				super(Order.class);
    		}
    
    		@Override
    		public List<Order> search(OrderSearch orderSearch) {
    				QOrder order = QOrder.order;
    				QMember member = QMember.member;
    				JPQLQuery query = from(order);
    	
    				if (StringUtils.hasText(orderSerach.getMemberName())){
    						query.leftJoin(order.member, member)
    								.where(member.name.contains(orderSearch.getMemberName()));
    				}
    
    				if (orderSearch.getOrderStatus() != null){
    						query.where(order.status.eq(orderSearch.getOrderStatus()));
    				}
    				
    				return query.list(order);
    		}
    ```
    
- OrderService
    
    ```java
    // 주문 검색
    QItem item = QItem.item;
    Iterable<Item> result = itemRepository.findAll(
    		item.name.contains("장난감").and(item.price.between(10000, 20000))
    /);
    ```
