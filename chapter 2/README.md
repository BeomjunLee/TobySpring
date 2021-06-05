## Chapter 2 테스트
### 스프링이 개발자에게 제공하는 가장 중요한 가치는?
객체지향과 테스트이다.<br>
스프링은 IOC, DI 를 이용해 객체지향 프로그래밍을 도와주며<br>
만들어진 코드를 확신할 수 있게 해주고 변화에 유연하게 대처할 수 있게 도와주는 테스트 기능을 제공한다.
<br><br>

## 테스트 전략
- <strong>@SpringBootTest</strong> : 통합 테스트, 전체 테스트<br>
부모 클래스 : IntegrationTest<br>
주입되는 빈 : Bean 전체

- <strong>@WebMvcTest</strong> : 단위 테스트, Mvc 테스트<br>
부모 클래스 : MockApiTest<br>
주입되는 빈 : MVC 관련된 Bean

- <strong>@DataJpaTest</strong> : 단위 테스트, Jpa 테스트<br>
부모 클래스 : RepositoryTest<br>
주입되는 빈 : JPA 관련 Bean

## 통합 테스트
- SpringBootTest는 단위테스트와 같이 기능 검증이 아닌, 스프링에서 실제 운영 환경과 같이 전체 플로우가 제대로 동작하는지 보기 위한 통합테스트이다.

- @SpringBootTest가 동작하면 @SpringBootApplication을 찾아가서 모든 빈을 스캔한다. 즉, 모든 빈을 로드하는 통합 테스트이기 때문에 무겁다.

- spring-boot-starter-test 의존성을 추가하면 테스트에 필요한 대부분의 라이브러리가 포함되어 있다. (JUnit, assertJ, mockito 등)

## 단위 테스트
- 한꺼번에 많은 것을 몰아서 테스트하면 테스트 수행과정도 복잡해지고 오류가 발생했을 때 정확한 원인을 찾기가 힘들어진다.
  
- 따라서 테스트는 가능하면 작은 단위로 쪼개서 집중할 수 있어야 한다.<br>
-> 이렇게 작은 단위의 코드에 대해 테스트를 수행할 것을 단위 테스트 라고 한다.

- 단위 테스트를 하는 이유는 개발자가 설계하고 만든 코드가 원래 의도한 대로 동작하는지 개발자 스스로 빨리 확인받기 위해서다.
  
- 이때 확인 대상과 조건이 간단하고 명확할 수록 좋다. 그래서 작은 단위로 테스트하는게 편리하다.
  
- 통합 테스트보다 빠르다.

## 테스트 주도 개발 (TDD)
만들고자 하는 기능의 내용을 담고 있으면서 만들어진 코드를 검증도 해줄 수 있도록 테스트 코드를 먼저 만들고<br>
테스트를 성공하게 해주는 코드를 작성하는 방식의 개발 방법.<br>
<br>

## Junit Test 어노테이션
```java
public class Test {
    @Test
    void create1() {
        System.out.println("create1");
    }

    @Test
    void create2() {
        System.out.println("create2");
    }

    @BeforeAll  //해당 메서드는 static 여야한다, 모든 테스트 메서드 보다 먼저 한번 실행된다.
    static void beforeAll() {
        System.out.println("@BeforeAll");
    }

    @AfterAll   //해당 메서드는 static 여야한다, 모든 테스트 메서드가 끝나고 한번 실행된다.
    static void afterAll() {
        System.out.println("@AfterAll");
    }

    @BeforeEach //각 테스트 메서드 전에 한번씩 실행된다.
    void beforeEach() {
        System.out.println("@BeforeEach");
    }

    @AfterEach  //각 테스트 메서드 실행 후 한번씩 실행된다.
    void afterEach() {
        System.out.println("@AfterEach");
    }

    @DisplayName("테스트 메서드 이름")  //테스트 클래스나 메서드의 이름을 정의할 수 있다.
    @Disable    //테스트 클래스 또는 메서드를 비활성화 시킬수 있다.

}
```
```
@BeforeAll

@BeforeEach
create1
@AfterEach


@BeforeEach
create2
@AfterEach

@AfterAll
```
<br>

## Assert J
메소드 채이닝을 지원해서 Junit5 보다 가독성이 좋다.
```java
//비교 검사
@Test
public void equal(){
    String username = "이범준"
    assertThat(username).isEqualTo("이범준");
}

//Object 속성 값 검사
@Test
public void extracting(){
    User user1 = new User("A");
    User user2 = new User("B");
    User user3 = new User("C");

    List<User> userList = new ArrayList<>();
    userList.add(user1);
    userList.add(user2);
    userList.add(user3);

    assertThat(user).extracting("name").isEqualTo("A"); //객체 프로퍼티 추출해서 검사

    assertThat(userList).extracting("name")
                        .containsExactly("A", "B", "C");  //리스트의 프로퍼티를 추출해서 검사

    assertThat(list).filteredOn(user -> user.getName()  //필터링 가능
                    .contains("A"))
                    .containsOnly("A");
}

@Test
public void string(){
    String str = "Test String";

    //간단하게 문자열테스트
    assertThat(str).startsWith("Test").endsWith("String").contains("e"); 
}

@Test
public void exception(){
    assertThatThrownBy(() -> {
        User user = userService.findByUsername("이범준");
    }).isInstanceOf(UsernameNotFoundException.class)
        .hasMessageContaining("회원을 찾을수 없습니다."); //exception 의 메세지 검사
}
})
 
```

### @Transactional
테스트 완료후 자동으로 Rollback 처리한다.

### @ActiveProfiles
원하는 프로파일 환경 값으로 설정할 수 있다.
<br><br>

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

