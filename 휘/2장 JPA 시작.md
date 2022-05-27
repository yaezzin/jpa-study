# 2장. JPA 시작

목차
1. 객체 매핑
2. persistent.xml
3. 애플리케이션 개발
    1. 엔티티 매니저 설정
    2. 트랜잭션 관리
    3. 비즈니스 로직


## 객체 매핑

회원 클래스와 회원 테이블을 매핑한다. JPA가 제공하는 매핑 어노테이션으로 매핑 정보를 설정할 수 있다. JPA는 매핑 어노테이션을 분석해서 객체와 테이블의 매핑 관계를 확인한다.

![image](https://user-images.githubusercontent.com/53958188/169553712-1317e4f6-1490-4c85-a0ba-215f87185a7e.png)


```java
package jpabook.start;

import javax.persistence.*;  //**

/**
 * User: HolyEyE
 * Date: 13. 5. 24. Time: 오후 7:43
 */
@Entity
@Table(name="MEMBER")
public class Member {

    @Id
    @Column(name = "ID")
    private String id;

    @Column(name = "NAME")
    private String username;

    private Integer age;

    public String getId() {
        return id;
    }

    public void setId(String id) {
        this.id = id;
    }

    public String getUsername() {
        return username;
    }

    public void setUsername(String username) {
        this.username = username;
    }

    public Integer getAge() {
        return age;
    }

    public void setAge(Integer age) {
        this.age = age;
    }
}
```

## persistent.xml

- persistence : JPA 2.1을 사용하려면 xmlns, version을 명시한다
    
    ```java
    <persistence xmlns="http://xmlns.jcp.org/xml/ns/persistence" version="2.1">
    ```
    
- persistence-unit name=”jpabook” : **연결할 DB 하나당 하나의 영속성 유닛을 등록**한다. 영속성 유닛의 이름으로 jpabook을 사용한다.
- property name=””
    - JPA 표준 속성
        
        ```java
        <!-- 필수 속성 -->
        <property name="**javax.persistence**.jdbc.driver" value="org.h2.Driver"/>
        <property name="**javax.persistence**.jdbc.user" value="sa"/>
        <property name="**javax.persistence**.jdbc.password" value=""/>
        <property name="**javax.persistence**.jdbc.url" value="jdbc:h2:tcp://localhost/~/test"/>
        ```
        
    - 하이버네이트 속성
        
        ```java
        <!-- 필수 속성: 데이터베이스 방언 설정 -->
        <property name="**hibernate.dialect**" value="org.hibernate.dialect.H2Dialect" />
        ```
        
- 데이터베이스 방언
    
    JPA는 특정 DB에 종속적이지 않은 기술이다. 다른 DB로 손쉽게 교체할 수 있다. 그런데 각각의 DB들이 제공하는 SQL 문법과 함수가 조금씩 다르다. 
    
    - 데이터 타입 : MySQL은 VARCHAR, 오라클은 VARCHAR2
    - 다른 함수명 : SQL 표준은 SUBSTRING(), 오라클은 SUBSTR()
    - 페이징 처리 : MySQL은 LIMIT을 사용하지만 오라클은 ROWNUM 사용
    
    SQL 표준을 지키지 않거나 특정 DB만의 고유한 기능을 JPA에서는 방언이라고 한다. JPA 구현체들은 이런 문제를 해결하려고 다양한 DB 방언 클래스를 제공한다. 개발자는 JPA가 제공하는 표준 문법에 맞춰 JPA를 사용하면 되고, 특정 DB에 의존적인 SQL은 DB 방언이 처리해준다. 따라서 DB가 변경되어도 애플리케이션 코드를 변경할 필요 없이 데이터베이스 방언만 교체하면 된다. 
    
    - H2 : org.hibernate.dialect.H2Dialect
    - 오라클 10g : org.hibernate.dialect.Oracle10gDialect
    - MySQL : org.hibernate.dialect.MySQL5InnoDBDialect
    
- persistent.xml 전체 코드
    
    ```java
    <?xml version="1.0" encoding="UTF-8"?>
    <persistence xmlns="http://xmlns.jcp.org/xml/ns/persistence" version="2.1">
    
        <persistence-unit name="jpabook">
    
            <properties>
    
                <!-- 필수 속성 -->
                <property name="javax.persistence.jdbc.driver" value="org.h2.Driver"/>
                <property name="javax.persistence.jdbc.user" value="sa"/>
                <property name="javax.persistence.jdbc.password" value=""/>
                <property name="javax.persistence.jdbc.url" value="jdbc:h2:tcp://localhost/~/test"/>
                <property name="hibernate.dialect" value="org.hibernate.dialect.H2Dialect" />
    
                <!-- 옵션 -->
                <property name="hibernate.show_sql" value="true" />
                <property name="hibernate.format_sql" value="true" />
                <property name="hibernate.use_sql_comments" value="true" />
                <property name="hibernate.id.new_generator_mappings" value="true" />
    
                <!--<property name="hibernate.hbm2ddl.auto" value="create" />-->
            </properties>
        </persistence-unit>
    
    </persistence>
    ```
    

## 애플리케이션 개발

- 전체 코드
    
    ```java
    package jpabook.start;
    
    import javax.persistence.*;
    import java.util.List;
    
    /**
     * @author holyeye
     */
    public class JpaMain {
    
        public static void main(String[] args) {
    
            //엔티티 매니저 팩토리 생성
            EntityManagerFactory emf = Persistence.createEntityManagerFactory("jpabook");
            EntityManager em = emf.createEntityManager(); //엔티티 매니저 생성
    
            EntityTransaction tx = em.getTransaction(); //트랜잭션 기능 획득
    
            try {
    
                tx.begin(); //트랜잭션 시작
                logic(em);  //비즈니스 로직
                tx.commit();//트랜잭션 커밋
    
            } catch (Exception e) {
                e.printStackTrace();
                tx.rollback(); //트랜잭션 롤백
            } finally {
                em.close(); //엔티티 매니저 종료
            }
    
            emf.close(); //엔티티 매니저 팩토리 종료
        }
    
        public static void logic(EntityManager em) {
    
            String id = "id1";
            Member member = new Member();
            member.setId(id);
            member.setUsername("지한");
            member.setAge(2);
    
            //등록
            em.persist(member);
    
            //수정
            member.setAge(20);
    
            //한 건 조회
            Member findMember = em.find(Member.class, id);
            System.out.println("findMember=" + findMember.getUsername() + ", age=" + findMember.getAge());
    
            //목록 조회
            List<Member> members = em.createQuery("select m from Member m", Member.class).getResultList();
            System.out.println("members.size=" + members.size());
    
            //삭제
            em.remove(member);
    
        }
    }
    ```
    

### 엔티티 매니저 설정

![image](https://user-images.githubusercontent.com/53958188/169553826-8f147f85-7284-400d-849e-fd69ea4897b0.png)

1. 엔티티 매니저 팩토리 생성
    
    ```java
    EntityManagerFactory emf = Persistence.createEntityManagerFactory("jpabook");
    ```
    
    persistence.xml의 설정 정보를 사용해서 엔티티 매니저 팩토리를 생성해야 한다. 이때 Persistence 클래스를 사용하는데, 이 클래스는 엔티티 매니저 팩토리를 생성해서 JPA를 사용할 수 있게 준비한다. 위의 코드는 이름이 jpabook인 영속성 유닛을 찾아서 엔티티 매니저 팩토리를 생성한다. **엔티티 매니저 팩토리는 애플리케이션 전체에서 딱 한 번만 생성하고 공유해서 사용해야 한다.** 
    
2. 엔티티 매니저 생성
    
    ```java
    EntityManager em = emf.createEntityManager(); 
    ```
    
    엔티티 매니저 팩토리에서 엔티티 매니저는 생성한다. JPA의 기능 대부분은 이 엔티티 매니저가 제공한다. 엔티티 매니저를 사용해서 엔티티를 DB에 등록, 수정, 삭제, 조회할 수 있다. 엔티티 매니저는 내부에 데이터 소스를 유지하면서 DB와 통신한다. 애플리케이션은 엔티티 매니저를 가상의 DB로 생각할 수 있다. 엔티티 매니저는 DB 커넥션과 밀접한 관계가 있으므로 **스레드 간에 공유하거나 재사용하면 안된다.**
    
3. 종료
    
    사용이 끝난 엔티티 매니저와 엔티티 매니저 팩토리는 반드시 종료해야 한다.
    
    ```java
    em.close();
    emf.close();
    ```
    

### 트랜잭션 관리

```java
try {
		tx.begin(); //트랜잭션 시작
	  logic(em);  //비즈니스 로직
    tx.commit();//트랜잭션 커밋 : 정상 동작
} catch (Exception e) {
    e.printStackTrace();
    tx.rollback(); //트랜잭션 롤백 : 예외 발생
} finally {
    em.close(); //엔티티 매니저 종료
}
```

JPA를 사용하면 항상 트랜잭션 안에서 데이터를 변경해야 한다. 트랜잭션을 시작하려면 엔티티 매니저에서 트랜잭션 API를 받아와야 한다.

### 비즈니스 로직

회원 엔티티를 하나 생성한다. 엔티티 매니저를 통해 DB에 등록, 수정, 삭제, 조회한다. 이 작업은 엔티티 매니저 em을 통해 수행된다. 엔티티매니저는 객체를 저장하는 가상의 DB처럼 보인다.

```java
public static void logic(EntityManager em) {

        String id = "id1";
        Member member = new Member();
        member.setId(id);
        member.setUsername("지한");
        member.setAge(2);

        //등록
        em.persist(member);

        //수정
        member.setAge(20);

        //한 건 조회
        Member findMember = em.find(Member.class, id);
        System.out.println("findMember=" + findMember.getUsername() + ", age=" + findMember.getAge());

        //목록 조회
        List<Member> members = em.createQuery("select m from Member m", Member.class).getResultList();
        System.out.println("members.size=" + members.size());

        //삭제
        em.remove(member);

    }
```

**등록**

JPA는 회원 엔티티의 매핑 정보를 확인해서 다음과 같은 SQL을 만들어 DB에 전달한다.

```java
//저장할 엔티티 객체 생성
String id = "id1";
Member member = new Member();
member.setId(id);
member.setUsername("지한");
member.setAge(2);
//등록
em.persist(member); // INSERT INTO MEMBER (ID, NAME, AGE) VALUES ('id1', '지한', 2)
```

**수정**

수정 내용을 반영하려면 엔티티 값만 변경하면 된다. JPA는 어떤 엔티티가 변경되었는지 추적하는 기능을 갖추고 있다. 

```java
// 수정
member.setAge(20); // UPDATE MEMBER SET AGE=20, NAME='지한' WHERE ID='id1'
```

**삭제**

```java
//삭제
em.remove(member); // DELETE FROM MEMBER WHERE ID='id1'
```

**한건 조회**

조회할 엔티티 타입과 @Id로 DB 테이블의 기본 키와 매핑한 식별자 값으로 엔티티 하나를 조회한다.

```java
Member findMember = em.find(Member.class, id); // SELECT * FROM MEMBER WHERE ID='id1'
```

**여러건 조회**

JPA는 검색을 할 때도 테이블이 아닌 엔티티 객체를 대상으로 검색해야 한다. 이러려면 모든 데이터를 애플리케이션으로 불러와서 엔티티 객체로 변경하고 검색해야 하는데, 이는 사실상 불가능하다. 필요한 데이터만 불러오려면 검색 조건이 포함된 SQL을 사용해야 한다. JPA는 JPQL이라는 쿼리 언어로 문제를 해결한다.

```java
//목록 조회 : JPQL은 엔티티 객체를 대상으로 쿼리한다
List<Member> members = em.createQuery("select m from Member m", Member.class).getResultList();
System.out.println("members.size=" + members.size());
```

**JPQL이란?**

검색 조건이 포함된 SQL을 추상화한 객체지향 쿼리 언어이다. SQL은 데이터베이스 테이블을 대상으로 쿼리하지만, JPQL은 엔티티 객체를 대상으로 쿼리한다. JPQL은 테이블을 전혀 알지 못한다.

```java
select m from Member m
```
