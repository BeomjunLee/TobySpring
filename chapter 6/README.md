# AOP
AOP 는 서비스 추상화에 더불어 스프링의 3대 기술중 하나이다.<br><br>

## 트랜잭션 코드의 분리
트랜잭션과 비지니스 로직이 공존하는 메서드이다 (쭉 최적화해냐갈 예정이다)
```java
public void upgradeLevels() throws Exception{

    TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());

    try{
        List<User> users = userDao.getAll();
        for(User user : users){
            if(canUpgradeLevel(user))
                upgradeLevel(user);
        }
        this.transationManager.commit(status);
    }catch (Exception e){
        this.transactionManager.rollback(status);
        throw e;
    }
}
```
비지니스 로직은 트랜직션의 시작과 종료 사이에 위치하고 있다. 또한 트랜잭션과 비지니스 로직 간에 교류도 존재하지 않는다.<br>
(비지니스 로직에서 DB 를 직접 사용하지 않고 DAO 가 알아서 활용하기 때문에)<br>
따라서 트랜잭션과 비지니스로직을 분리 시킬 수 있다.<br><br>

### Service 인터페이스 사용
만약에 트랜잭션 코드를 UserService 에서 분리시킨다고 해도 UserService 라는 구현클래스를 직접 주입받아서 사용할 것이므로 트랜잭션이 빠진 UserService 가 되어버린다.<br>
DI 의 기본 아이디어는 실제 사용할 오브젝트의 클래스 정체는 감춘 채 인터페이스를 통해 간접으로 접근하는 것이다.<br>

<img src="https://user-images.githubusercontent.com/69130921/120836374-772fe980-c5a0-11eb-9877-9c0e294f8659.png">

따라서 이렇게 UserService 의 비지니스 로직이라는 책임을 가지고 있는 UserServiceImpl 이라는 구현클래스를 만들고<br>
트랜잭션의 책임을 가지고 있는 UserServiceTx 라는 구현클래스를 추가로 만든다.<br>
```java
@Service
public interface UserService{
    void add(User user);
    void upgradeLevels();
}
```
```java
@Component
@RequiredArgsConstructor
public class UserServiceImpl implements UserService
    
    private final UserDao userDao;

    public void upgradeLevels(){

        List<User> users = userDao.getAll();

        for(User user : users){
            if(canUpgradeLevel(user))
                upgradeLevel(user);
        }
    }

    ...
```
```java
@Component
public class UserServiceTx implements UserService{

    private UserService userService;

    @Autowired
    public UserServiceTx(@Qualifier("userServiceImpl") UserService userService){
        this.userService = userService;
    }

    //DI 받은 UserService 에 비지니스 로직 기능을 위임한다.
    public void upgradeLevels(){
         TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());

    try{
       
        userService.upgradeLevels();    //비지니스 로직

        this.transationManager.commit(status);
    }catch (Exception e){
        this.transactionManager.rollback(status);
        throw e;
    }
    }
}
```
```java
@Controller
public class UserController{

    private UserService userService;

    @Autowired
    public UserServiceTx(@Qualifier("userServiceTx") UserService userService){
        this.userService = userService;
    }

    ...

}
```
<br>
이렇게 클라이언트가 UserService 라는 인터페이스를 통해 사용자 관리 로직을 이용할 때 먼저 트랜잭션을 담당하는 오브젝트가 실행된 후 안에 있던 비지니스 로직이 실행되게 만든다<br>
<img src="https://user-images.githubusercontent.com/69130921/120837468-b1e65180-c5a1-11eb-8a21-e54f6e2242da.png"><br>
클라이언트(컨트롤러) 의 UserService 는 UserServiceTx 를 주입받게하고 UserServiceTx 의 UserService 는 UserServiceImpl 을 주입받게 해주면 된다.<br><br>


<br><br>


## 데코레이터 패턴
데코레이션 패턴이란 객체의 결합을 통해 기능을 동적으로 확장 확장할 수 있게 해주는 패턴이다.<br>

ex) 데코레이터 패턴 예시<br>
<img src="https://user-images.githubusercontent.com/69130921/120885870-1ea32f80-c626-11eb-85ed-b90062b2e633.png"><br>

앞에서 UserServiceImpl 과 UserServiceTx 도 데코레이터 패턴을 이용한 예시이다.<br>
필요하다면 트랜잭션 외에도 다른 기능을 부여하는 데코레이터를 만들어서 UserServiceTx 와 UserServiceImpl 사이에 추가해줄 수 있다.<br>

## 프록시 패턴
프록시 패턴은 어떤 객체에 대한 접근을 제어하는 용도로 대리인이나 대변인에 해당하는 객체를 제공하는 패턴이다.<br>
프록시는 기존 코드에 영향을 주지 않고 타깃의 기능을 확장하거나 접근 방법을 제어할 수 있다.<br>
하지만 개발자들은 매번 새로운 클래스를 만들고, 인터페이스에 모든 메서드들을 일일히 구현해서 위임해야하는 작업들이 번거롭기 때문에 지향하지 않는다.<br>
```java
@Component
public class UserServiceTx implements UserService{

    private UserService userService; //타깃 오브젝트

    public void add(User user){
        this.userService.add(user); //메서드 구현과 위임
    }

    @Autowired
    public UserServiceTx(@Qualifier("userServiceImpl") UserService userService){
        this.userService = userService;
    }

    public void upgradeLevels(){
        //부가기능 수행
        TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());
    try{
       

        userService.upgradeLevels();    //위임


    //부가기능 수행
        this.transationManager.commit(status);
    }catch (Exception e){
        this.transactionManager.rollback(status);
        throw e;
    }
    }
}
```
<br>

## 다이내믹 프록시
이러한 프록시의 단점들을 보완해 몇 가지 API 를 이용해 프록시처럼 동작하는 오브젝트를 다이내믹하게 생성할 수 있다.<br>

다이내믹 프록시는 리플렉션 기능을 이용해서 프록시를 만들어준다.<br>
리플렉션이란 구체적인 클래스 타입을 몰라도 해당 클래스의 메서드, 타입, 변수 들을 알 수 있도록 해주는 Java API 이다.<br>

다이내믹 프록시로부터 요청을 전달받으려면 InvocationHandler 를 구현해야 한다. (메서드는 invoke() 하나 뿐)<br><br>

## UserServiceImpl, UserServiceTx 의 문제점
이렇게 인터페이스를 통해 트랜잭션을 분리했지만 클라이언트가 UserServiceImpl 이라는 클래스를 직접 사용해버리면 트랜잭션이 적용되지 않는다.<br>
그렇기 때문에 클라이언트는 인터페이스를 통해서만 핵심기능을 사용할 수 있게 하고 부가기능 또한 같은 인터페이스를 사용해서 끼어들어야된다.<br>

즉 사용자는 자신이 핵심기능을 가진 클래스를 사용한다고 생각하지만 사실은 부가기능을 통해 핵심기능을 이용하게된다.<br>
<img src="https://user-images.githubusercontent.com/69130921/120885363-768c6700-c623-11eb-821b-09e07f8defd7.png"><br><br>

## UserServiceTx 다이내믹 프록시 적용
다이내믹 프록시를 이용해 UserServiceTx 를 변경해 볼 것 이다.
```java
public class TransactionHandler implements InvocationHandler{
    
    private Object target; //부가기능을 제공할 타깃 오브젝트 (어떤 타입이든 적용 가능)
    private PlatformTransactionManager transactionManager;  //트랜잭션 기능을 제공
    private String pattern; //트랜잭션을 적용할 메서드 이름 패턴
    
    public void setTarget(Object target) {
        this.target = target;
    }

    public void setTransactionManager(PlatformTransactionManager transactionManager) {
        this.transactionManager = transactionManager;
    }

    public void setPattern(String pattern) {
        this.pattern = pattern;
    }

    //트랜잭션 적용 대상 메서드를 선별해서 트랜잭션 경계설정 기능을 부여해준다.
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable{
        
        if(method.getName().startWith(pattern)){
            return invokeInTransaction(method, args);
        }else{
            return method.invoke(target, args);
        }
    }
    
    //트랜잭션을 시작하고 타깃 오브젝트의 메서드를 호출한 후 트랜잭션 commit or rollback
    public Object invokeInTransaction(Method method, Object[] args) throws Throwable{

        TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());

        try{
            Object ret = method.invoke(target, args);
            this.transactionManager.commit(status);
            return yet;
        } catch (InvocationTargetException e){
            this.transactionManager.rollback(status);
            throw e.getTargetException();
        }
    }
}
```
<br>

### 다이내믹 프록시 적용후 테스트
```java
@Test
public void upgradeDynamicProxyTest() throws Exception{
    ...
    TransactionHandler txHandler = new TransactionHandler();
    txHandler.setTarget(UserService);
    txHandler.setTransactionManager(transactionManager);
    txHandler.setPattern("upgradeLevels");
    
    UserService txUserService = (UserService)Proxy.newProxyInstance(
        getClass(), getClassLoader(), new Class[], {UserService.class }, txHandler);
    )
}
```

다이내믹 프록시가 잘 작동하지만 실제로 사용하려면 스프링 빈에 등록해야하는데 스프링 빈은 지정된 클래스의 이름을 가지고 리플렉션을 이용해서 해당 클래스의 오브젝트를 만들게되는데<br>
다이내믹 프록시의 오브젝트 클래스가 어떤 건지 알 수가 없기 때문에(클래스에서 지정해주기 때문) 사전에 프록시 오브젝트를 알아내서 스프링 빈에 등록할 수 가 없게된다.<br>
따라서 다이내믹 프록시는 Proxy 클래스의 newProxyInstance() 라는 스태틱 팩토리 메서드를 통해서만 만들 수 있다.<br><br>

## 팩토리 빈
스프링은 클래스 정보를 가지고 디폴트 생성자를 통해 오브젝트를 만드는 방법 말고도 여러가지 방법들을 제공하는데 대표적으로 팩토리 빈을 이용한 방법이 있다.<br>
이렇게 FactoryBean 인터페이스를 구현한 클래스를 스프링 빈으로 등록하면 팩토리빈으로 동작한다.<br>
(private 생성자를 가져서 static 으로 생성이 가능한 객체를 팩토리빈으로 스프링 빈 생성할 수 있다)
```java
public interface FactoryBean<T>{

    T getObject() throws Exception; //빈 오브젝트를 생성해서 돌려준다
    Class<? extends T> getObjectType(); //생성되는 오브젝트 타입을 알려준다
    boolean isSingleton(); //getObject() 가 돌려주는 오브젝트가 항상 같은 싱글톤 타입인지 판별
}
```
<br>

## 트랜잭션 프록시 팩토리 빈

<img src="https://user-images.githubusercontent.com/69130921/120888915-c70cc000-c635-11eb-92c5-7492b751094a.png"><br>

```java
public class TransactionHandler implements FactoryBean<Object>{
    
    private Object target; //부가기능을 제공할 타깃 오브젝트 (어떤 타입이든 적용 가능)
    private PlatformTransactionManager transactionManager;  //트랜잭션 기능을 제공
    private String pattern; //트랜잭션을 적용할 메서드 이름 패턴
    private Class<?> serviceInterface
    
    public void setTarget(Object target) {
        this.target = target;
    }

    public void setTransactionManager(PlatformTransactionManager transactionManager) {
        this.transactionManager = transactionManager;
    }

    public void setPattern(String pattern) {
        this.pattern = pattern;
    }

    public void setServiceInterface(Class<?> serviceInterface){
        this.serviceInterface = serviceInterface;
    }

    //FactoryBean 구현 메서드
    public Object getObject() throws Exception{
        TransactionHandler txHandler = new TransactionHandler();
        txHandler.setTarget(UserService);
        txHandler.setTransactionManager(transactionManager);
        txHandler.setPattern("upgradeLevels");
        return Proxy.newProxyInstance(
        getClass(), getClassLoader(), new Class[], {serviceInterface }, txHandler);
    }

    public Class<?> getObjectType(){
        return serviceInterface;
    }

    public boolean isSingleton(){
        return false;
    }
}
```
<br>

## 프록시 팩토리 빈 방식 장점과 단점
### 장점
- 인터페이스를 구현하는 클래스를 일일히 만드는 번거러움을 제거할 수 있다
- 하나의 핸들러 메서드를 구현하는 것으로 수많은 메서드에 부가기능을 부여할 수 있다 (코드 중복 해결)
- 팩토리 빈의 DI 로 번거러운 다이내믹 프록시 생성 코드도 제거할 수 있다
  
### 단점
- 한 번에 여러개의 클래스에 공통적인 부가기능을 제공할 수 없다
- 하나의 타깃에 여러 개의 부가기능을 적용하기 힘들다
- 핸들러 오브젝트가 프록시 팩토리 빈 개수만큼 만들어진다

<br>

## 스프링의 프록시 팩토리 빈
스프링은 프록시 오브젝트를 생성해주는 기술을 추상화한 팩토리 빈을 제공해준다.<br>
- 스프링의 ProxyFactoryBean 은 프록시를 생성해서 빈 오브젝트로 등록하게 해주는 팩토리 빈이다.
- ProxyFactoryBean이 생성하는 프록시에서 시용할 부가기능은 Methodlnterceptor 인터페이스를 구현해서 만든다.
- InvocationHandler의 invoke() 메소드는 타깃 오브 젝트에 대한 정보를 제공하지 않지만, Methodlnterceptor의 invoke() 메소드는 ProxyFactoryBean으로부터 타깃 오브젝트에 대한 정보까지도 함께 제공받기 때문에 Methodlnterceptor는 타깃 오브젝트에 상관없이 독립적으로 만들어질 수 있다.
- 따라서 Methodlnterceptor 오브젝트는 타깃이 다른 여러 프록시에서 함께 사용할 수 있고, 싱글톤 빈으로 등록 기능하다.

<br>

## 어드바이스: 타깃이 필요없는 순수한 부가기능
- InvocationHandler를 구현했을 때와 달리 Methodlnterceptor를 구현한 UppercaseAdvice에는 타깃 오브젝트가 등장하지 않는다.
- MethodInvocation은 일종의 콜백 오브젝트로, proceed() 메소드를 실행하면 타깃 오브젝트의 메소드를 내부적으로 실행해주는 기능이 있다.
- 그래서 Methodlnvocation 구현 클래스는 일종의 공유 가능한 템플릿처럼 동작히는 것이다. -> ProxyFactoryBean의 장점

<br>

## 포인트컷: 부가기능 적용 대상 메소드 선정 방법
<img src="https://user-images.githubusercontent.com/69130921/120890313-d3e0e200-c63c-11eb-9b3a-061331fe6c07.png"><br>
확장이 펼요하면 팩토리 빈 내의 프록시 생성코드를 직접 변경해야 한다. 결국 확장 에는 유연하게 열려 있지 못하고 관련 없는 코드의 변경이 필요할 수 있는.OCP 원칙을 깔끔하게 잘 지키지 못하는 어정쩡한 구조라고 볼 수 었다.<br><br>
<img src="https://user-images.githubusercontent.com/69130921/120890315-d5aaa580-c63c-11eb-89dc-046b556a7431.png"><br>
스프링의 ProxyFactoryBean 방식은 두 가지 확장 기능 인 부가기능과 메소드 선정 알고리즘을 활용히는 유연한 구조를 제공한다.<br><br>

스프링은 부가기능을 제공하는 오브젝트를 어드바이스라고 부르고, 메소드 선정 알고리즘을 담은 오브젝트를 포인트컷이라고 부른다.(모두 DI 로 주입, 스프링 빈 싱글톤 등록 가능)<br><br>

## 스프링 AOP
빈 후처리기는 이름 그대로 스프링 빈 오브젝트로 만 들어지고 난 후에, 빈 오브젝트를 다시 가공할 수 있게 해준다.<br>
<img src="https://user-images.githubusercontent.com/69130921/120890459-bceebf80-c63d-11eb-99a4-32a036d09eba.png"><br>

### DefaultAdvisorAutoProxyCreator 빈 후처리가 등록되어 있다면, 스프링은 빈 오브젝트를 만들 때마다 후처리기에게 빈을 보낸다.
- 후처리기는 빈으로 등록된 모든 어드바이저 내의 포인트컷을 이용해 전달받은 빈이 프록시 적용 대상인지 확인한다.
- 프록시 적용 대상이면 내장된 프록시 생성기를 통해 현재 빈에 대한 프록시를 생성하고 어드바이저를 연결한다.
- 프록시가 생성되면 전달받은 Target Bean 오브젝트 대신에 Proxy 오브젝트를 스프링 컨테이너에게 돌려준다.
- 컨테이너는 빈 후처리가 돌려준 Proxy 오브젝트를 빈으로 등록한다.
<br>

### AOP 용어
<img src="https://user-images.githubusercontent.com/69130921/120890698-700be880-c63f-11eb-9d75-2dbce434d362.png"><br>
- Aspect<br>
  AOP의 기본 모듈이다. 한개 또는 그 이상의 포인트컷과 어드바이스의 조합으로 만들어진다. (트랜잭션 기능, 보안 기능, 로그 기능, 인증 기능 등)

- Target<br>
  부가기능을 부여할 대상 (메서드, 클래스, 프록시 객체 등)

- Advice<br>
  실질적인 부가기능을 담은 구현체 Aspect 가 언제 적용될지 정의
  - @Before: Target 을 실행하기 전에 부가 기능 실행
  - @After: Target 실행 후 (해당 Target Exception 또는 정상리턴 여부 상관없이) 실행
  - @Around: Before + AfterReturning
  - @AfterReturning: Target 실행 후 성공적인 리턴할 때
  - @AfterThrowing: Target 실행하다, 예외가 생길 때

- Joint Point<br>
  Advice가 적용될 위치를 표시한다(ex:메소드 실행 단계), 타깃의 코드가 실행할 때 나타날 수 있는 여러 시점

- Point Cut<br>
  Advice를 적용할 Target를 선별하는 역할을 한다.

    ```java
    @Pointcut(
          "execution("
        + "public "
        + "User "
        + "com.Beomjun.practice.UserService"
        + ".findUserId(String) "
        + "throws NotFoundUsernameException"
        + ")"
          )
    
    @Pointcut(
            "execution("    
        + "[접근제한자 패턴] "  
        + "리턴타입 패턴"       
        + "[패키지명, 클래스 경로 패턴]"         
        + "메소드명 패턴(파라미터 타입 패턴|..)"  
        + "[throws 예외 타입 패턴]"            
        +")"   
            )
    ```

- Weaving<br>
  AOP에서 Joinpoint들을 Advice로 감싸는 과정을 Weaving이라고 한다. Weaving 하는 작업을 도와주는 것이 AOP 툴이 하는 역할이다.

<br><br>

## 트랜잭션 격리수준, 전파옵션 (isolation, propagation)
[포스팅 확인하기](https://blog.naver.com/qjawnswkd/222385156627)

<br>

## AOP 가 @Transactional 이라는 기능을 지원해준다
<img src="https://user-images.githubusercontent.com/69130921/120891340-0a216000-c643-11eb-8efe-c79b0fa24f73.png"><br>
공통로직인 트랜잭션에 집중 안하고 핵심로직인 비지니스 로직에 집중할 수 있게 해준다. 이러한 개념이 AOP 이다<br>
```java
//어노테이션을 사용할 대상을 지정(메서드, 클래스, 인터페이스)
@Target({ElementType.TYPE, ElementType.METHOD}) 

//어노테이션 정보가 언제까지 유지되는지 설정 (이렇게 하면 런타임 때도 어노테이션 정보를 리플렉션으로 얻을 수 있다)
@Retention(RetentionPolicy.RUNTIME) 

@Inherited //상속을 통해서도 어노테이션 정보를 받을 수 있다
@Documented
public @interface Transactional {
```
이런 설정들이 default 값으로 되어있다.
```java
@AliasFor("transactionManager")
	String value() default "";
	@AliasFor("value")
	String transactionManager() default "";
	String[] label() default {};
	Propagation propagation() default Propagation.REQUIRED;
	Isolation isolation() default Isolation.DEFAULT;
	int timeout() default TransactionDefinition.TIMEOUT_DEFAULT;
	String timeoutString() default "";
	boolean readOnly() default false;
	Class<? extends Throwable>[] rollbackFor() default {};
	String[] rollbackForClassName() default {};
	Class<? extends Throwable>[] noRollbackFor() default {};
	String[] noRollbackForClassName() default {};
```