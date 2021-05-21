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
### [Template Method Pattern + Factory Method Pattern] 상속을 통한 확장
Template Method Pattern : 슈퍼 클래스에 기본적인 로직의 흐름을 만들고 그 기능의 일부를 추상메서드나 오버라이딩이 가능한 protected 메서드 등으로 만든 뒤 서브 클래스에서 이런 메서드를 필요에 맞게 구현해서 사용하도록 하는 방법<br><br>
Factory Method Pattern : 서브 클래스에서 구체적인 오브젝트 생성 방법을 결정하게 하는 방법
```java
public abstract class UserDao{
    public void add(User user) ...{
        Connection c = getConnection();
        ...
    }
    public User get(String id) ...{
        Connection c = getConnection();
        ...
    }

    protected abstract Connection getConnection() throws ClassNotFoundException,SQLException;
}
```
```java
public class NaverUserDao extends UserDao throws ClassNotFoundException, SQLException{
    public Connection getConnection ...{
        //Naver DB 생성 코드
    }
}

public class KakaoUserDao extends UserDao throws ClassNotFoundException, SQLException{
    public Connectionn getConnection ...{
        //Kakao DB 생성 코드
    }
}
```
이렇게 한다면 UserDao 의 코드는 수정할 필요 없이 DB 연결 기능을 새롭게 정의하는 클래스를 만들 수 있다.<br>
-> 손쉽게 확장할 수 있다. (새로운 DB 연결시 UserDao 를 상속받아 클래스를 확장해주면 된다)<br>

### 클래스의 분리
상속을 사용했지만 자바에서는 다중 상속이 불가능하다. 그래서 클래스로 분리 해보려고 한다.

```java
public class UserDao{
    private SimpleConnectionMaker simpleConnectionMaker;

    public UserDao(){
        simpleConnectionMaker = new SimpleConnectionMaker();
    }

    public void add(User user) throws ClassNotFoundException, SQLException{
        Connection c = simpleConnectionMaker.makeNewConnection();
        ...
    }

     public User get(String id) throws ClassNotFoundException, SQLException{
        Connection c = simpleConnectionMaker.makeNewConnection();
        ...
    }
}
```
```java
public class SimpleConnectionMaker{
    public Connection makeNewConnection() throws ClassNotFoundException, SQLException{
        Class.forName("com.mysql.jdbc.Driver");
        Connection c = DriverManager.getConnection
                        ("jdbc:mysql://localhost/springbook", "spring", "book");
        return c;
    }
}
```
코드가 깔끔해 졌지만 하지만 이렇게 하면 아까 처럼 Naver 와 Kakao 의 DB 를 분리할 수 없다.<br>
Naver 와 Kakao 를 분리하려고 SimpleConnectionMaker 에 메서드를 하나 더 만든다고 해도
```java
public class SimpleConnectionMaker{
    public Connection naverConnection() throws ClassNotFoundException, SQLException{
      //Naver DB 연결
    }

     public Connection kakaoConnection() throws ClassNotFoundException, SQLException{
      //Kakao DB 연결
    }
}
```
UserDao 의 메서드가 1000개라면 DB Connection 수정시 일일히 다 바꿔줘야 할 것이다.<br>
또한 UserDao 에 SimpleConnectionMaker 라는 클래스 타입의 인스턴스 변수도 정의해 놓아서 Naver 가 다른 클래스를 구현하면<br>
UserDao 를 수정하는 작업이 필요해져 버리게 된다.<br><br>

### 인터페이스 도입
```java
public interface ConnectionMaker{
    Connection makeConnection() throws ClassNotFoundException, SQLException;
}
```
```java
public class KakaoConnectionMaker implements ConnectionMaker{
    
    public Connection makeConnection() throws ClassNotFoundException, SQLException{
        //Kakao DB 연결
    }
}
```
```java
public class UserDao{
    private ConnectionMaker connectionMaker;

    public UserDao(){
        connectionMaker = new KakaoConnectionMaker();
    }

    public void add(User user) throws ClassNotFoundException, SQLException{
        //인터페이스의 메서드를 사용하므로 클래스가 바뀐다고 해도 메서드 이름이 변경될 필요가 없다.
        Connection c = connectionMaker.makeConnection();
        ...
    }

    public User get(String id) throws ClassNotFoundException, SQLException{
        Connection c = connectionMaker.makeConnection();
        ...
    }
}
```
메서드 이름을 매번 변경할 수고는 덜었지만<br>
connectionMaker = new KakaoConnectionMaker(); 라는 클래스 이름을 만들어 오브젝트를 만드는 작업은 여전했다.

### 관계설정 책임의 분리
UserDao 와 ConnectionMaker 를 인터페이스를 써가면서 분리를 하였는데<br>
여전히 UserDao 가 인터페이스 뿐만 아니라 구체적인 클래스를 알아야 했다.
(KakaoConnectionMaker 를 의존해야 했다)<br><br>
따라서 의존관계 주입이라는 방법을 사용해야 한다.<br>
그러면 사용자는 UserDao 를 사용하면서 세부 전략으로 볼 수 있는 ConnectionMaker 의 구현 클래스를 선택하고 오브젝트를 생성해서<br> 
UserDao 와 연결해줄 수 있다.
```java
public class UserDao{
    private final ConnectionMaker connectionMaker;

    public UserDao(ConnectionMaker connectionMaker){
        this.connectionMaker = connectionMaker;
    }
}
```
그러면 UserDao 의 코드를 수정하지 않고 UserDao 를 사용하는 사람이 직접 사용할 클래스를 만들고 주입시켜주면 된다.<br>
```java
@Test
@DisplayName("네이버 DB 테스트")
public void naverTest throws Exception{
    ConnectionMaker naver = new NaverConnectionMaker();

    UserDao dao = new UserDao(naver);
    ...
}

@Test
@DisplayName("카카오 DB 테스트")
public void kakaoTest throws Exception{
    ConnectionMaker kakao = new KakaoConnectionMaker();

    UserDao dao = new UserDao(kakao);
    ...
}
```

### 객체지향 설계 원칙 SOLID
- <strong>SRP</strong> : 단일 책임 원칙<br>
하나의 클래스는 하나의 책임만 가져야 한다.<br>
-> 하나의 책임은 애매해서 변경이 있을 때 파급 효과가 적으면 단일 책임 원칙을 잘 따른 것이다.

- <strong>OCP</strong> : 개방-폐쇄 원칙<br>
소프트웨어 요소는 확장에는 열려 있으나 변경에는 닫혀 있어야한다.<br>
-> 다형성을 이용 -> 인터페이스를 구현한 새로운 클래스를 만들어 새로운 기능을 구현

- <strong>LSP</strong> : 리스코프 치환 원칙<br>
프로그램의 객체는 프로그램의 정확성을 깨뜨리지 않으면서 하위 타입의 인스턴스로 바꿀수 있어야 한다.<br>
-> 자동차의 엑셀은 무조건 앞으로 가야됨 아니면 규칙 위반

- <strong>ISP</strong> : 인터페이스 분리 원칙<br>
특정 클라이언트를 위한 인터페이스 여러 개가 범용 인터페이스 하나보다 낫다.<br>
-> 자동차 인터페이스를 운전, 정비로 나눠서 만들고<br>
사용자 클라이언트를 운전자, 정비사로 나눈다<br>
-정비 인터페이스가 변해도 운전자 클라이언트에 영향 x<br>
-> 인터페이스가 명확해지고 대체 가능성이 높아짐

- <strong>DIP</strong> : 의존관계 역전 원칙<br>
프로그래머는 추상화에 의존해야지 구체화에 의존하면 안된다.<br>
-> 구현 클래스에 의존하지말고 인터페이스에 의존하라는 뜻


### 제어의 역전 (IOC)
프로그램의 제어 흐름을 직접 제어하는게 아니라 외부에서 관리하는 것을 제어의 역전 (IoC)라고 한다.

이렇게 외부에서 관리할 수 있는 AppConfig 클래스를 만들어 사용할 ConnectionMaker 를 등록하고 주입시켜 사용한다.
```java
public class AppConfig{

    public ConnectionMaker connectionMaker(){
        return new NaverConnectionMaker();
        //return new KakaoConnectionMaker();
    }

    public UserDao userDao(){
        return new UserDao(connectionMaker())
    }
}
```
```java
@Test
@DisplayName("네이버 DB 테스트")
public void naverTest throws Exception{
    AppConfig appConfig = new AppConfig();

    UserDao dao = appConfig.userDao();
    ...
}
```
​이제 Naver 와 Kakao 에 UserDao 를 공급할 때 UserDao, ConnectionMaker, AppConfig 를 공급하되 AppConfig 는 소스코드를 제공해서 새로운 ConnectionMaker 구현 클래스로 변경이 필요하면 AppConfig 를 수정해서 변경된 클래스를 수정하여 설정 해주면 된다.<br>
