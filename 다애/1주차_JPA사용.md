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

