## 패러다임의 불일치

### 패러다임의 불일치란?
: 객체와 관계형데이터베이스가 지향하는 목적이 다름으로 인해 발생하는 기능과 표현 방법의 차이

### 1. 상속
객체는 상속기능O, 테이블은 상속기능X

만약, 이 패러다임의 불일치를 해결하려고 한다면...

```java
abstract class Item

class Album extends Item // 앨범 객체는 아이템 객체 상속

```
```sql
insert into item...
insert into album... // 앨범 객체 저장을 위해 두개의 sql필요

item, album  테이블 join 후 객체 조회 가능
```
#### JPA 사용
 자바 컬렉션에 객체를 저장하듯 JPA에게 객체 저장
```java
jpa.persist(album) // persist() 메소드는 객체를 디비에 저장함.
# persist()가 알아서 insert sql을 실행해서 두 테이블에 나누어 저장함.

String albumId = "id100";
Album album = jpa.find(Album.class, albumId) // find() 메소드는 객체 하나를 디비에서 조회함.
# find()가 알아서 두 테이블을 조인해서 select sql 생성해서 객체 반환함.
```

### 2. 연관관계
객체는 참조를 통해 연관 객체 조회, 테이블은 외래키(fk)를 사용해서 조인을 통해 연관 테이블 조회

불일치를 해결하려면 객체지향 모델링 포기해야함.

저장 : 참조를 외래키 값으로 변환해야함

조회 : 외래키 값을 참조로 변환해야함

--> 모두 개발자가 중간에서 변환 역할을 해야하는 번거로움

#### JPA 사용
```java
member.setTeam(team); // member과 team연관관계 설정
jpa.persist(member); // member와 연관관계 함께 저장
# JPA가 team의 참조를 외래키로 변환해서 insert sql 디비에 전달

Member member = jpa.find(Member.class, memberId);
Team team = member.getTeam();
# 조회시 외래키를 참조로 변환한다.
```

### 3. 객체 그래프 탐색 및 비교
객체 그래프 탐색 : 참조를 사용하여 연관된 객체를 찾는 과정. 객체는 마음껏 그래프를 탐색할 수 있어야 한다.

sql을 직접 사용하면 처음 실행하는 sql에 따라 객체 그래프 탐색 범위가 정해진다. --> 객체 그래프 신뢰 불가

#### JPA 사용
 JPA는 객체를 사용하는 시점에 적절한 select문을 날려준다. 따라서 연관된 객체를 신뢰 및 마음껏 사용이 가능하다.
 
 지연로딩 : 실제 객체를 사용하는 시점까지 데이터베이스 조회를 미루는 것
 
 ```java
 Member member = jpa.find(Member.class, memberId);
 
 Order o = member.getOrder();
 order.getOrderDate(); // order객체 사용 시점에 select문 날림
 ```

동일성 비교 vs 동등성 비교

동일성 비교는 ==. 객체 인스턴스 주소 값 비교

동등성 비교는 equals(). 객체 내부 값 비교

테이블은 pk로 각 row구분

객체는 호출 시 새로 생성되므로 디비가 같은 로우라고 해도 동일성 비교가 실패한다.

#### JPA 사용
 JPA는 같은 트랜젝션이면 같은 객체 조회를 보장해준다.
```java
String mId = "1";
Member m1 = jpa.find(Member.class, mId);
Member m2 = jpa.find(Member.class, mId);
m1 == m2 // 동일성 비교 성공!
```
