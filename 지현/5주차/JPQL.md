### JPQL 조인

1. INNER JOIN (INNER 생략 가능)
    - JPQL 내부조인구문과 이때 생성된 SQL 구문의 차이
        
        ```sql
        -- JPQL 내부조인 구문
        SELECT m
        FROM Member m JOIN m.team t
        WHERE t.name = :teamName
        ```
        
        ```sql
        -- SQL 내부조인 구문
        SELECT 
        	M.ID AS ID,
        	M.AGE AS AGE,
        	M.TEAM AS TEAM,
        	M.NAME AS NAME
        FROM 
        	MEMBER M INNER JOIN TEAM T ON M.TEAM_ID=T.ID
        WHERE
        	T.NAME = <teamName>
        ```
        
    - JPQL은 조인할 때 연관필드를 사용한다!
        - 연관필드: 다른 엔티티와 연관관계를 가지기 위해 사용하는 필드
    - 잘못된 예시
      ```sql
      from Member m join Team t
      ```
2. OUTER JOIN (OUTER 생략 가능)
    - SQL의 외부 조인과 기능이 같음
        
        ```sql
        -- JPQL
        SELECT m
        FROM Member m LEFT JOIN m.team t
        WHERE t.name = :teamName
        ```
        
        ```sql
        -- SQL
        SELECT 
        	M.ID AS ID,
        	M.AGE AS AGE,
        	M.TEAM AS TEAM,
        	M.NAME AS NAME
        FROM 
        	MEMBER M LEFT OUTER JOIN TEAM T ON M.TEAM_ID=T.ID
        WHERE
        	T.NAME = <teamName>
        ```
        
3. 컬렉션 조인
    - 컬렉션을 사용하는 곳(일대다, 다대다)에 조인
    - 단일 값 연관필드
        - 다대일 조인
    - 컬렉션 값 연관 필드
        - 일대다 조인
        
        ```sql
        select t, m from Team t left join t.members m
        ```
        
4. THETA JOIN
    - WHERE절 사용
    - 내부 조인만 지원
    
    ```sql
    -- JPQL
    SELECT COUNT(m) FROM Member m, Team t
    WHERE m.username=t.name
    ```
    
    ```sql
    -- SQL
    SELECT COUNT(M.ID)
    FROM 
    	MEMBER M CROSS JOIN TEAM T
    WHERE
    	M.USERNAME=T.NAME
    ```
    
5. JOIN ON 절
    - WHERE절과는 필터링 시점의 차이
    ```sql
    -- JPQL
    select m, t from Member m
    left join m.team t on t.name='A'
    
    -- SQL
      SELECT m.*, t.* FROM MEMER m
      LEFT JOIN TEAM t ON m.TEAM_ID=t.ID AND t.NAME='A'
    ```

### 페치 조인

1. JPQL에서 최적화를 위해 제공하는 기능
2. 연관된 엔티티나 컬렉션을 한 번에 같이 조회
3. JPA 명세에서 페치 조인은 alias 사용 불가, 하이버네이트는 허용
4. 엔티티 페치 조인
    - 사용 예
        
        ```java
        String jpql = "select m from Member m join fetch m.team";
        
        List<Member> members = em.createQuery(jpql, Member.class)
        	.getResultList();
        
        for (Member member : members) {
        	System.out.println("username = " + member.getUsername() + ", teamname = " + member.getTeam().name());
        }
        ```
        
        ```sql
        SELECT 
        	M.*, T.*
        FROM MEMBER M
        INNER JOIN TEAM T ON M.TEAM_ID=T.ID
        ```
        
    <img width=600 src="https://user-images.githubusercontent.com/87467801/178069597-0c179845-a8ce-4b94-b3d9-bb530455f9e4.png">
    <img width=400 src="https://user-images.githubusercontent.com/87467801/178068249-5a3b4d3f-e171-4e33-a8a0-306a9c1afb76.png">
    
    - JPQL의 SELECT절에는 Member만 조회하도록 했지만 생성된 SQL은 MEMBER와  TEAM(의 컬럼들)을 모두 조회하도록 함
    - 글로벌 로딩 전략을 지연로딩으로 설정했다면 원래 팀 엔티티는 실제 사용되기 전까지 프록시 객체로 대체되지만, 위와 같은 상황에서 페치 조인은 연관된 엔티티를 함께 조회하므로 지연로딩이 발생하지 않음
    - 팀 엔티티는 프록시가 아닌 실제 엔티티이므로 회원 엔티티가 준영속 상태가 되어도 연관된 팀 조회 가능
5. 컬렉션 페치 조인
    - 사용예
        
        ```java
        String jpql = "select t from Team t join fetch t.members where t.name='팀A'";
        List<Team> teams = em.createQuery(jpql, Team.class).getResultList();
        
        for (Team team : teams) {
        	System.out.println("teamname = " + team.getName() + ", team = " + team);
        
        	for (Member member : team) {
        			System.out.println("-> username = " + member.getUsername() + ", member = "+ member);
       		}
        }
        ```
        
        ```sql
        SELECT
        	T.*, M.*
        FROM TEAM T
        	JOIN MEMBER M ON T.ID=M.TEAM_ID
        WHERE T.NAME='팀A'
        ```
        
    <img width=600 src="https://user-images.githubusercontent.com/87467801/178068564-1fbb29d7-53a5-4719-842b-ac7139d8eeb6.png">
    <img width=800 src="https://user-images.githubusercontent.com/87467801/178068615-0bbed43c-20e2-44d8-976d-4c5934f87812.png">
    
    - 팀을 조회하면서 연관된 회원 컬렉션도 함께 조회
    - MEMBER 테이블과 조인하면서 팀A가 2건으로 증가
    - 출력 결과
        
        ```sql
        teamname = 팀A, team = Team@0x100
        → username = 회원1, member = Member@0x200
        → username = 회원2, member = Member@0x300
        teamname = 팀A, team = Team@0x100
        → username = 회원1, member = Member@0x200
        → username = 회원2, member = Member@0x300
        ```
         
6. DISTINCT
    - SQL의 중복된 결과를 제거해 주는 명령어
    - **실행되는 SQL에 distinct 명령어가 추가됨**
        
        ```sql
        -- JPQL
        select distinct t
        from Team t join fetch t.members
        where t.name = '팀A'
        ```
        
        ```sql
        -- SQL
        SELECT DISTINCT T.*, M.*
        FROM TEAM T 
        INNER JOIN MEMBER M ON T.ID = M.TEAM_ID
        WHERE T.NAME = '팀A'
        ```
        
    - 위의 SQL 실행 결과
        
        <img width=600 src="https://user-images.githubusercontent.com/87467801/178069808-901bf875-2ded-4d21-919d-a66a0c805362.png">
        
    - **애플리케이션에서 중복된 데이터를 한 번 더 걸러냄**
        
        <img width=600 src="https://user-images.githubusercontent.com/87467801/178069882-9f00ceab-25b7-41f5-97c1-b8ee632cfe88.png">
        
    - 출력결과
        
        ```sql
        teamname = 팀A, team = Team@0x100
        → username = 회원1, member = Member@0x200
        → username = 회원2, member = Member@0x300
        ```
         
7. 일반 조인과의 차이
    - JPQL은 SELECT절에서 지정한 엔티티만 조회
    - 회원 컬렉션을 지연 로딩으로 설정하면 사용 전까지는 프록시나 초기화하지 않은 컬렉션 래퍼를 반환
    → 이후에 연관 엔티티를 사용할 때 연관 엔티티를 조회하기 위한 SELECT문을 한 번 더 날림
     : **N+1 문제**
    - 일반 조인 예시(일대다 컬렉션 조인)
        
        ```sql
        -- JPQL
        select t
        from Team t join t.members m
        where t.name = '팀A'
        ```
        
        ```sql
        -- SQL
        SELECT
        	T.*
        FROM TEAM T
        INNER JOIN MEMBER M ON T.ID=M.TEAM_ID
        WHERE T.NAME = '팀A'
        ```
        
8. 페치 조인의 특징과 한계
    - 객체 그래프를 유지할 때 사용하면 효과적, SQL 호출 횟수를 줄일 수 있음
    - 여러 테이블을 조인해서 개발자의 수요에 따라 필요한 필드만 뽑아야 한다면 DTO로 반환하는 것이 효과적
    - **글로벌 로딩 전략을 되도록 지연 로딩을 사용하되, 최적화가 필요하면 페치 조인 적용**
    - 페치 조인의 한계
        1. 페치 조인 대상에는 alias 사용 불가 (하이버네이트를 포함한 몇몇 구현체들은 지원)
            - 잘못된 예시
                
                ```sql
                select t from Team t join fetch t.members m where m.age > 10
                ```
                
            - 연관데이터 수가 달라짐 → 데이터 무결성 깨짐
            - 별칭은 해당 엔티티를 더 탐색하기 위해 붙임
        2. 둘 이상의 컬렉션을 페치할 수 없음
            - 컬렉션 페치 조인의 경우 SQL이 생성되어 실행되면 하나의 row당 연관테이블의 row만큼 늘어남. 컬렉션이 늘어나는 만큼 카테시안 곱이 만들어짐
        3. 컬렉션을 페치 조인하면 페이징 API 사용 불가
            - 위와 같은 이유에서 페이지네이션을 할 때 중복으로 생긴 row를 고려해서 페이징을 할지 무시할지 결정해야 하는데 이를 모두 메모리에 올려버리고 페이징 처리함
            → 하이버네이트에서는 MultipleBagFetchException 발생

### 경로 표현식

1. 용어 정리
    - 상태 필드: 단순 값을 저장하기 위한 필드, 프로퍼티
    - 연관 필드: 연관관계를 위한 필드, 프로퍼티 (임베디드 타입 포함)
        - 단일 값 연관 필드: 연관 대상이 엔티티 (다대일, 일대일)
        - 컬렉션 값 연관 필드: 연관 대상이 컬렉션 (일대다, 다대다)
2. 경로표현식
    
    ```java
    @Entity
    public class Member {
    		
    	@Id @GeneratedValue
    	private Long id;
    
    	@Column(name = "name")
    	private String username;
    	private Integer age;
    
    	@ManyToOne(...)
    	private Team team;
    
    	@OneToMany(...)
    	private List<Order> orders;
    
    	//...
    }
    ```
    
    - 상태 필드: m.username, m.age
    - 단일 값 연관 필드: m.team
    - 컬렉션 값 연관 필드: m.orders
3. 경로표현식의 특징 
    - 상태 필드 경로: 경로 탐색의 끝
    - 단일 값 연관 경로: 묵시적 내부 조인 발생. 객체 그래프 계속 탐색 가능
        - 단일값 연관 경로 탐색 시 SQL에서 묵시적 내부 조인
            
            ```sql
            -- jpql
            select o.member.team
            from Order o
            where o.product.name = 'productA' and o.address.city = 'JINJU'
            ```
            
            ```sql
            -- sql
            SELECT T.*
            FROM ORDERS O
            INNER JOIN MEMBER M ON O.MEMBER_ID = M.ID
            INNER JOIN TEAM T ON O.TEAM_ID = T.ID
            INNER JOIN PRODUCT P ON O.PRODUCT_ID = P.ID
            WHERE P.NAME = 'productA' AND O.CITY = 'JINJU'
            ```
            
    - 컬렉션 값 연관 경로: 묵시적 내부 조인 발생. 더 탐색할 수 없으나 FROM절에서 별칭을 붙이면 별칭으로 탐색 가능
        - 컬렉션 값에서 경로 탐색 불가
            - 잘못된 예시
                
                ```sql
                select t.members.username from Team t
                ```
                
        - 별칭 사용하여 컬렉션 값에서 경로를 탐색하는 예시
            
            ```sql
            select m.username from Team t join t.members m
            ```
            
    - 경로 탐색 묵시적 조인 시 주의사항
        - 항상 내부 조인
        - 컬렉션에서 경로 탐색을 하려면 명시적 조인으로 별칭 붙여야 함
        - 경로 탐색은 주로 select절, where절에 사용하지만 조인으로 인해 SQL의 from절에 영향을 줌 → 성능에 영향이 큼

### 서브쿼리

1. EXISTS
    
    ```sql
    select m from Member m
    where exists (select t from m.team t where t.name = '팀A')
    ```
    
2. {ALL | ANY | SOME}
    
    ```sql
    select o from Order o
    where o.orderAmount > all (select p.stockAmount from Product p)
    
    select m from Member m
    where m.team = any (select t from Team t)
    ```
    
3. IN
    
    ```sql
    select t from Team t
    where t in (select t2 from Team t2 join t2.members m2 where m2.age >= 20)
    ```
    

### 조건식 (SQL과 동일)

1. 타입 표현
    
    
    | 종류 | 예제 |
    | --- | --- |
    | 문자 | ‘HELLO’, ‘She’’s’ |
    | 숫자 | 10L(Long), 10D(Double), 10F(Float) |
    | 날짜 | {d ‘2012-03-24’}(DATE), {t ‘10-11-11’}(TIME), {ts ‘2012-03-24 10-11-11.123’}(DATETIME) |
    | boolean | true, false |
    | Enum | jpaboob.MemberType.Admin(패키지명을 포함한 전체 이름) |
    | 엔티티 | TYPE(m) = Member |
2. 연산자 우선 순위
    - 경로탐색 연산: .
    - 수학 연산: +, -(단항연산자) *, /, +, -
    - 비교연산: =, >, ≥, <, ≤, <>, BETWEEN A AND B, LIKE, IN, IS NULL, IS EMPTY, MEMBER, EXISTS
    - 논리연산: NOT, AND, OR
3. 문자함수: CONCAT, SUBSTRING, TRIM, LOWER, UPPER, LENGTH, LOCATE
4. 수학함수: ABS, SQRT, MOD, SIZE, INDEX
5. 날짜함수: CURRENT_DATE, CURRENT_TIME, CURRENT_TIMESTAMP
6. CASE 식
    
    ```sql
    -- 기본 CASE
    select
    	case when m.age<=10 then '학생요금'
    	     when m.age>=60 then '경로요금'
   			  else '일반요금'
    	end
    from Member m
    
    -- 심플 CASE
    select 
    	case t.name
    	  when '팀A' then '인센티브110%'
    		when '팀B' then '인센티브120%'
    		else '인센티브105%'
    	end
    from Team t
    
    -- COALESCE
    select coalesce(m.username, '이름없는 회원') from Member m
    
    -- NULLIF
    select nullif(m.username, '관리자') from Member m
    ```
    

### 다형성 쿼리

1. JPQL로 부모 엔티티를 조회하면 부모 엔티티를 상속받는 자식 엔티티도 함께 조회
2. 상속관계 매핑 전략에 따라 생성되는 SQL 쿼리
    
    <img width=600 src="https://user-images.githubusercontent.com/87467801/178075008-db1085f1-4d90-4bb2-b73c-a0481245dec8.png">
    
    `List resultList = em.createQuery(”select i from Item i”).getResultList();`
    
    - 단일 테이블 전략
        
        ```sql
        SELECT * FROM ITEM
        ```
        
    - 조인 전략
        
        ```sql
        SELECT 
        	i.ITEM_ID, i.DTYPE, i.NAME, i.PRICE, i.STOCKQUANTITY,
        	b.AUTHOR, b.ISBN,
       		a.ARTIST, a.ETC, 
       		m.ACTOR, m.DIRECTOR
        FROM ITEM i
        LEFT OUTER JOIN 
       		BOOK b ON i.ITEM_ID = b.ITEM_ID
        LEFT OUTER JOIN
        	ALBUM a ON i.ITEM_ID = a.ITEM_ID
        LEFT OUTER JOIN
        	MOVIE m ON i.ITEM_ID = m.ITEM_ID
        ```
        
3. TYPE
    - 조회 대상을 특정 자식 타입으로 한정
    
    ```sql
    -- jpql
    select i from Item i
    where type(i) IN (Book, Movie)
    
    -- sql
    select i from Item i 
    where i.DTYPE IN ('B', 'N')
    ```
    
4. TREAT
    - jpa 2.1에 추가. 부모 타입을 특정 자식 타입으로 다룰 때
        
        ```sql
        -- jpql
        select i.* from Item i where treat(i as Book).author = 'kim'
        
        -- sql
        select i.* from Item i
        where
        	i.DTYPE = 'B'
        	and i.author = 'kim'
        ```
        

### 사용자 정의 함수

- 하이버네이트: 방언클래스를 상속해서 구현하고, 사용할 데이터베이스 함수 persistence.xml에 미리 등록
    
    ```java
    public class MyH2Dialect extends H2Dialect {
    		
    	public MyH2Dialect() {
    			registerFunction("group_concat", new StandardSQLFunction("group_concat", StandardBasicTypes.STRING));
   		}
    }
    ```
    
    ```xml
    <property name="hibernate.dialect" value="hello.MyH2Dialect" />
    ```
    

### 엔티티 직접 사용

1. 기본 키 값
    
    ```java
    String qlString = "select m from Member m where m = :member";
    List resultList = em.createQuery(qlString)
    	.setParameter("member", member)
    	.getResultList();
    ```
    
    ```java
    String qlString = "select m from Member m where m = :memberId";
    List resultList = em.createQuery(qlString)
    	.setParameter("memberId", 4L)
    	.getResultList();
    ```
    
    - 둘 다 생성되는 SQL문은 같다
        
        ```sql
        select m.*
        from Member m
        where m.id=?
        ```
        
2. 외래 키 값
    
    ```java
    Team team = em.find(Team.class, 1L);
    
    String qlString = "select m from Member m where m.team = :team";
    List resultList = em.createQuery(qlString)
    	.setParameter("team", team)
    	.getResultList();
    ```
    
    ```java
    String qlString = "select m from Member m where m.team.id = :teamId";
    List resultList = em.createQuery(qlString)
   		.setParameter("teamId", 1L)
   		.getResultList();
    ```
    
    - 둘 다 생성되는 SQL 쿼리문은 같음
        
        ```sql
        select m.*
        from Member m
        where m.team_id=?
        -- 묵시적 조인X
        ```
        

### Named 쿼리: 정적 쿼리

1. 정적 쿼리: 애플리케이션 로딩 시점에 문법을 체크하고 미리 파싱해두는 JPQL 쿼리, 재사용 용이
2. 생성 방법
    - 애노테이션에 정의
        
        ```java
        @Entity
        @NamedQueries({
        	@NamedQuery(
       			name = "Member.findByUsername",
     				query = "select m from Member m where m.username = :username"),
       		@NamedQuery(
     				name = "Member.count",
        		query = "select count(m) from Member m")
        })
        public class Member {...}
        ```
        
    - XML에 정의
        
        ```xml
        <?xml version="1.0" encoding="UTF-8"?>
        <entity-mappings xmlns="http://xmlns.jcp.org/xml/ns/persistence/orm" version="2.1">
          <named-query name="Member.findByUsername">
       			<query><CDATA[
        		  select m
        			from Member m
   						where m.username = :username
     				]></query>
        	</named-query>
        
        	<named-query name="Member.count">
        		<query>select count(m) from Member m</query>
        	</named-query>
        </entity-mappings>
        ```
        
        ```xml
        <persistence-unit name="jpabook">
        	<mapping-file>META-INF/ormMember.xml</mapping-file>
        ```
        
    - XML은 복잡하지만 여러 줄의 로우를 다루기 좋고 한 곳에 정적 쿼리를 모아둘 수 있음
    - XML과 애노테이션에 같은 설정이 있으면 XML이 우선권 가짐
    - 운영 환경에 따른 쿼리를 실행할 때 각 환경에 대한 XML을 준비하고 XML만 변경해서 배포하는 것이 좋음
