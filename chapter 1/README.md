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
## [Template Method Pattern + Factory Method Pattern] 상속을 통한 확장
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

## 클래스의 분리
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

## 인터페이스 도입
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

## 관계설정 책임의 분리
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
<br>

## 객체지향 설계 원칙 SOLID
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
-> 구현 클래스에 의존하지말고 인터페이스에 의존하라는 뜻<br><br>


## 제어의 역전 (IOC)
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

## 오브젝트 펙토리를 이용한 IOC (스프링 컨테이너, Bean)
스프링에선 빈의 생성과 관계설정 같은 것을 빈 펙토리 라고 부른다.
```java
@Configuration  //스프링 컨테이너가 해당 클래스를 스프링의 설정 정보로 사용한다.
public class AppConfig{

    @Bean   //이 어노테이션이 붙은 메서드를 호출해서 반환된 객체를 스프링 컨테이너에 빈으로 등록
    public ConnectionMaker connectionMaker(){
        return new NaverConnectionMaker();
        //return new KakaoConnectionMaker();
    }
    
    @Bean
    public UserDao userDao(){
        return new UserDao(connectionMaker());
    }
}
```
ApplicationContext 를 스프링 컨테이너 (IOC 컨테이너) 라고 한다.<br>
스프링 컨테이너는 @Configuration 이 붙은 클래스들을 설정 정보로 사용하며<br>
@Bean 이라고 적힌 메서드들을 모두 호출하여 반환된 객체를 스프링 컨테이너에 등록한다.<br>
(스프링 컨테이너에 넣는 스프링 빈은 모두 싱글톤을 보장해준다)<br>
(@Configuration 을 붙이지 않고 @Bean 을 설정한다면 싱글톤 보장이 되지않고 새로운 인스턴스가 생성되게 된다)
```java
@Test
@DisplayName("네이버 DB 테스트")
public void naverBeanTest throws Exception{
    ApplicationContext ac = new AnnotationConfigApplicationContext(AppConfig.class);
    //ac.getBean -> 스프링 컨테이너에 있는 userDao 라는 이름을 가진 스프링 빈을 반환
    UserDao dao = ac.getBean("userDao", UserDao.class);
    ...
}
```
<br>

## BeanFactory, ApplicationContext 차이
<img src="https://user-images.githubusercontent.com/69130921/119123778-e5d25a80-ba6a-11eb-8220-4a552806a397.png"><br>

BeanFactory

- 스프링 컨테이너의 최상위 인터페이스
- 스프링 빈을 관리하고 조회하는 역할
- getBean() 을 사용
- 지금까지 예제 코드로 살펴본 대부분의 기능은 BeanFactory가 제공

ApplicationContext

- BeanFactory 의 기능을 모두 상속받아서 제공

BeanFactory, ApplicationContext 의 차이

- ApplicationContext는 BeanFactory 뿐만 아니라 여러가지 인터페이스를 가지고 있다.
- 메세지 소스 : 한국에서 들어오면 한국어, 미국에서 들어오면 영어로 출력
- 환경변수 : 로컬, 개발, 운영등을 구분해서 처리
- 어플리케이션 이벤트 : 이벤트를 발행하고 구독하는 모델을 편리하게 지원
- 편리한 리소스 조회 : 파일, 클래스패스, 외부 url 같은 곳에서 리소스를 편리하게 조회

그래서 이러한 부가기능이 포함된 ApplicationContext를 사용한다.

​
## 싱글톤 패턴의 문제점
- 싱글톤 패턴을 구현하는 코드 자체가 많이 들어간다.
- 클라이언트가 구현 클래스에 의존한다 -> DIP 위반, OCP 위반
- 테스트하기 어렵다.
- 내부 속성을 변경하거나 초기화하기 어렵다.
- private 생성자로 자식 클래스를 만들기 어렵다 -> 유연성이 떨어짐(안티패턴으로 불린다)

<strong>스프링 컨테이너는 싱글톤 패턴의 문제점들을 해결하면서 싱글톤도 관리된다 (싱글톤 컨테이너 역할)</strong><br><br>

## 싱글톤 사용시 주의점
싱글톤 객체를 유지(stateful)하게 설계하면 안된다.

무상태로(stateless)하게 설계해야 한다.

- 특정 클라이언트에 의존적인 필드가 있으면 안된다.
- 특정 클라이언트가 값을 변경할 수 있는 필드가 있으면 안된다.
- 가급적 읽기만 가능해야 한다(수정 x).
- 필드 대신에 자바에서 공유되지 않는 지역변수, 파라미터, ThreadLocal 등을 사용해야 한다.
  
```java
//수정 전
public class StatefulService {

    private int price; //상태를 유지하는 필드

    public void order(String name, int price) {
        System.out.println("name = " + name + " price = " + price);
        this.price = price;     // -> 문제 코드
    }

    public int getPrice() {
        return price;
    }
}
```
```java
//수정 후
public class StatefulService {

//    private int price; //상태를 유지하는 필드

    public int order(String name, int price) {
        System.out.println("name = " + name + " price = " + price);
//        this.price = price;     // -> 문제 코드
        return price;
    }

//    public int getPrice() {
//        return price;
//    }
}
```
<br>

## 의존관계 주입 (DI)
애플리케이션 실행 시점에 외부에서 실제 구현 객체를 생성하고 클라이언트에 전달해서<br>
클라이언트와 서버의 실제 의존관계가 연결 되는 것을 의존관계 주입이라고 한다.

DI를 사용하면 클라이언트 코드를 변경하지 않고, 클라이언트가 호출하는 대상의 타입 인스턴스를 변경할 수 있다.<br>
DI를 사용하면 정적인 클래스 의존 관계를 변경하지 않고, 동적인 객체 인스턴스 의존관계를 쉽게 변경할 수 있다.

ex) UserDao 를 손대지 않고 AppConfig 를 이용해 객체를 조립했다 뺐다 하는 것

### 생성자 주입
```java
public class UserDao{
 
private final ConnectionMaker connectionMaker;

    @Autowired
    public UserDao(ConnectionMaker connectionMaker) {
        this.connectionMaker = connectionMaker;
    }
}

 //Lombok 사용시 바로 생성자 주입 가능 (필수 값인 final 이 붙은 값들을 인자로 하여 생성자를 만들어준다)
@RequiredArgsConstructor
public class UserDao{
 
private final ConnectionMaker connectionMaker;

}

```
### 수정자(setter) 주입
```java
public class UserDao{
 
private ConnectionMaker connectionMaker;

    @Autowired
    public void setConnectionMaker(ConnectionMaker connectionMaker) {
        this.connectionMaker = connectionMaker;
    }
}
```
### 필드 주입 (외부에서 변경이 불가능해서 테스트하기 힘들다는 단점이 있기 때문에 사용 x)
테스트코드 작성시 사용해도 된다.
```java
public class UserDao{

@Autowired    
private ConnectionMaker connectionMaker;

}
```