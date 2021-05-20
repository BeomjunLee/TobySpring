## chapter 1

## DAO 클래스 리펙토링 해보기
회원을 저장하고 조회할 수 있는 간단한 DAO 를 하나 만들어봤다.
```java
@Getter
@Setter
public class User{
    private String id;
    private String password;
    private String name;
}
```

JDBC 를 이용한 회원 등록, 조회 DAO 라고 해보겠다.
```java
public class UserDao {
    public void add(User user) throws ClassNotFoundException, SQLException { 
        Class.forName("com.mysql.jdbc.Driver");
        Connection c = DriverManager.getConnection(
                    "jdbc:mysql://localhost/springbook","spring","book");
        PreparedStatement ps = c.prepareStatement(
            "insert into users(id, name, password) values(?, ?, ?)");
        ps.setString(1, user.getId()); 
        ps.setString(2, user.getName()); 
        ps.setString(3, user.getPassword());
        ps.executeUpdate();
        ps.close(); 
        c.close();
    
    public User get(String id) throws ClassNotFoundException, SQLException { 
        Class.forName("com.mysql.jdbc.Driver") ;
        Connection c = DriverManager.getConnection(
                    "jdbc:mysql://localhost/springbook","spring","book");
        PreparedStatement ps = c.prepareStatement("select * from users where id = ?");
        ps.setString(1 , id);
        ResultSet rs =ps.executeQuery();
        rs.next();
        User user = new User();
        user.setId(rs.getString("id"));
        user.setName(rs.getString("name"));
        user.setPassword(rs.getString("password"));
        
        rs.close();
        ps.close(); 
        c.close();
        return user;
    }
}
```

이제 리펙토링을 시작하겠다. 중복 메서드를 추출해보자<br>
UserDao 의 클래스의 메서드가 1000개가 넘는다면 DB 연결과 관련된 부분에 변경이 일어났을 경우 엄청난 코드 수정이 일어나야 하지만<br>
이렇게 수정한다면 getConnection() 관련 부분만 코드 수정해주면 된다.
```java
public void add(User user) throws ClassNotFoundException, SQLException{
    Connection c = getConnection();
    ...
}

public User get(String id) throws ClassNotFoundException, SQLException{
    Connection c = getConnection();
    ...
}

private Connection getConnection() throws ClassNotFoundException, SQLException{
    Class.forName("com.mysql.jdbc.Driver") ;
    Connection c = DriverManager.getConnection(
                    "jdbc:mysql://localhost/springbook","spring","book");
    return c;
} 
```
만약에 UserDao 가 발전하여 Naver 나 Kakao 가 구매하고 싶다고 하였고<br>
Naver 랑 Kakao 는 다른 DB Connection 을 사용하고 나는 UserDao 의 소스를 공개하고 싶지 않다.<br> 
Naver 나 Kakao 에게 소스코드를 제공해 주지 않고 스스로 원하는 DB Connection 방식을 적용하면서 사용할 수 있게 해보려 한다.<br>
### [Template Method Pattern] 상속을 통한 확장
```java
public abstract class UserDao{
    public void add(User user) ...{
        ...
    }
    public User get(String id) ...{
        ...
    }

    public abstract Connection getConnection() throws ClassNotFoundException, SQLException;
}
```
```java
public class NaverUserDao extends UserDao{
    public Connection getConnection ...{
        //Naver DB 생성 코드
    }
}

public class KakaoUserDao extends UserDao{
    public Connectionn getConnection ...{
        //Kakao DB 생성 코드
    }
}
```
이렇게 한다면 UserDao 의 코드는 수정할 필요 없이 DB 연결 기능을 새롭게 정의하는 클래스를 만들 수 있다.<br>
-> 손쉽게 확장할 수 있다.(새로운 DB 연결시 UserDao 를 상속받아 클래스를 확장해주면 된다)<br>


