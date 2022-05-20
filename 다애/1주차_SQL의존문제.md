# Why JPA?

## 1. SQL을 직접 다룰 때의 문제

### 1. 반복

객체를 디비에 CRUD 하려면 수많은 sql문을 작성하여 JDBC API를 사용해서 전달해야 한다.

만약, 회원 조회를 하려면 Member 객체를 만들고 디비에 접근하는 MemberDAO객체를 만들어야 한다. 

그리고 MemberDAO 클래스의 find메소드를 통해 회원을 조회하는데, 이 과정은 지루함과 반복의 연속이다.

```sql
select member_id, name from member m where member_id = ? // sql 작성
ResultSet rs = stmt.executeAuery(sql); // JDBC API 사용하여 sql 실행
String memberId = rs.getString("MEMBER_ID");
String name = rs.getString("NAME"):
Member member = new Member(); // Member객체 생성
member.setMemberId(memberId); // member객체로 매핑
member.setName(name);
```

### 2. 코드 수정의 불편함

갑자기 Member객체에 회원의 전화번호를 추가해달라는 요청을 받게된다면 어떻게 될까?

먼저, Member 클래스에 tel 필드를 추가한다. 

```sql
String sql = "insert into member(member_id, name, tel) values(?,?,?,); // 등록 sql 수정
pstmt.setString(3, member.getTel()); / member 객체의 tel 값 꺼내서 등록 sql에 전달

select member_id, name, tel from member where member_id = ? // 조회 sql 수정
String tel = rs.getString("TEL");
member.setTel(tel);
```
그리고 나서 조회, 등록 sql을 각각 수정해준다.

조회와 등록은 정상적으로 처리되었는데, 회원의 연락처를 수정하려고 하니 버그가 발생하였다. 

다시 MemberDAO를 열어서 update sql, update()메소드를 변경해야 한다.

### 3. 연관 객체 처리

회원은 어떤 한 팀에 필수적으로 소속되어야 한다는 요구사항을 받았다. 그래서 Member객체의 코드에 Team필드를 추가해주었다.
```java
private Team team; // in Member.java

class Team{
  private String teamName;
} // in Team.java
```

그 다음 member.getTeam().getTeamName()으로 팀의 이름을 출력하였다.

그런데 member.getTeam()의 값이 항상 null이다. 회원과 팀은 정상적으로 연관되어 있다. 

문제는 MemberDAO의 find()메소드가 회원만 조회했다는 것이다. 그래서 findWithTeam()메소드를 추가해주었다.
```sql
select m.member_id, m.name, m.tel, t.team_id, t.team_name
from Member m
join Team t on m.team_id = t.team_id;
```
findWithTeam()메소드는 회원과 팀을 조인해주어서 회원뿐만이 아닌 팀의 정보까지 출력해준다.

결국, MemberDAO의 메소드를 열어서 sql을 확인해주어야만 버그를 수정할 수 있었다.


### 정리하자면

sql에 모든 것을 의존하면..
1. 엔티티(비즈니스 요구사항을 모델링한 객체)를 신뢰하는 개발이 불가하다.

2. DAO를 직접 열어서 어떤 sql이 실행되는지 확인해야하는 진정한 의미의 계층 분할이 힘들다.

3. sql의존적 개발이 불가피하다.

  
  

