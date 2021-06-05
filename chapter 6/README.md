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
## 단위 테스트
가장 편하고 좋은 테스트는 가능한 작은 단위로 쪼개서 하는 단위 테스트이다. 작은 단위의 테스트가 좋은 이유는 테스트가 실패했을 경우 원인을 찾기 쉽기 때문이다.<br>

또한 테스트의 대상이 환경이나, 외부 서버, 다른 클래스들의 코드에 종속되고 영향을 받지 않도록 고립시킬 필요가 있다.<br>
테스트를 의존 대상으로 부터 고립 시키는 방법은 대역을 사용하는건데 Mockito 를 사용하면 편하게 할 수 있다.<br><br>

## 단위 테스트 vs 통합 테스트 가이드라인
- 항상 단위 테스트를 먼저 고려해야한다.
- 단위 테스트시 Mockito 를 사용해 외부와의 의존관계를 모두 차단한다면 빠르게 테스트를 할 수 있다.
- 외부 리소스를 사용해야만 하는 가능한 테스트는 통합 테스트로 만든다.
- 여러 개의 단위가 의존관계를 가지고 동작할 때를 위한 통합 테스트도 필요한데 단위 테스트를 잘 짜놨다면 그만큼 통합 테스트의 부담도 줄어든다.
- 단위 테스트로 만들기 너무 복잡한 코드라면 처음부터 통합 테스트를 고려해봐도 된다. 이 때 통합 테스트의 코드들 중에서 가능한 많은 부분을 단위 테스트로 짜두면 좋다.
  

## Mockito
@RunWith(MockitoJunitRunner.class)
Mockito에서 제공하는 목객체를 사용하기 하기위해 위와같은 어노테이션을 테스트클래스에 달아준다.

@InjectMocks라는 어노테이션이 존재하는데, @Mock이 붙은 목객체를 @InjectMocks이 붙은 객체에 주입시킬 수 있다.<br>
(물론 Mock 이라는 테스트용 빈 객체가 할당된다)
```java
@ExtendWith(MockitoExtension.class)
public class MemberServiceTest {

    @InjectMocks
    private MemberService memberService;

    @Mock
    private MemberRepository memberRepository;

    ...
}
```
when() 으로 memberRepository.findByUsername(...) 이 호출됐을 때 를 지정해줄 수 있고<br> 
thenReturn() 을 통해 원하는 값을 return 해줄 수 있다.
```java
    @Test
    @DisplayName("회원 정보 조회")
    public void findMember() throws Exception{
        //given
        when(memberRepository.findByUsername(any())).thenReturn(Optional.of(member));

        //when
        Member findMember = memberService.findMemberByUsername(any());

        //then
        assertThat(findMember).usingRecursiveComparison().isEqualTo(member);
    }

    @Test
    @DisplayName("회원 정보 조회 실패")
    public void findMemberFail() throws Exception{
        //given
        when(memberRepository.findByUsername(any())).thenReturn(Optional.empty());
        //when, then
        assertThatThrownBy(() -> {
            memberService.findMemberByUsername(any());
        }).isInstanceOf(UsernameNotFoundException.class).hasMessageContaining("해당되는 유저를 찾을수 없습니다");
    }
```
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















## 현재는 이렇게 @Transactional 이라는 AOP 를 지원해준다
따라서 공통로직인 트랜잭션에 집중 안하고 핵심로직인 비지니스 로직에 집중할 수 있게 해준다. 이러한 개념이 AOP 이다<br>
```java
@Service
public class UserService{

    ...

    @Transactional
    public void upgradeLevels(){

    }
}
```