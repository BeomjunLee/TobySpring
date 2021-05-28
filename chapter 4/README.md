## Chapter 4 예외
### Exception 종류
Exception 클래스는 체크 예외와 언체크 예외로 나뉜다.<br>
체크 예외는 Exception 클래스의 서브 클래스 이면서 RuntimeException 을 상속하지 않은 클래스들을 말하고<br>
언체크 예외는 RuntimeException 을 상속한 클래스들을 말한다.<br><br>
<img src="https://i.ibb.co/jTyMFg9/2021-05-29-12-44-55.png" width="500"><br>

체크예외가 발생할 수 있는 메서드를 사용할 경우 반드시 catch 로 예외를 잡든지 throws 를 정의해서 메서드 밖으로 던져야 한다.<br>
하지만 언체크 예외는 catch 나 throws 를 선언하지 않아도 되며 예외발생시 체크예외와 달리 트랜잭션을 roll-back 해준다.<br>

체크 예외가 예외처리를 강제하는 것 때문에 catch, throws 가 남발됐고 그렇게 때문에 최근에 새로 등장하는 자바 표준 스팩 API 들은 예외처리를 언체크 예외로 만드는 경향이 있다.<br>

### 예외처리 방법
간단하게 아이디가 중복됐을 때를 예로 들어보자면 예외처리를 할때 언체크예외로 만들어서 메세지를 전달할수 있도록 해서 예외클래스를 만들고<br>
```java
public class DuplicateUsernameException extends RuntimeException{
       public DuplicateUsernameException(String message) {
        super(message);
    }
}
```
```java
public void validateDuplicateMember(String username) {
    int findMembers = memberRepository.countByUsername(username);
    if (findMembers > 0) {
        throw new DuplicateUsernameException("아이디가 중복되었습니다");
    }
}
```
Controller 에서 발생한 예외를 한 곳에서 관리하고 처리할 수 있게 도와주는 어노테이션인 @ControllerAdvice 와 발생한 예외를 헨들링해서 처리할 수 있게 해주는 @ExceptionHandler 를 사용해 예외처리를 한 곳에서 관리할 수 있다.
```java
@ControllerAdvice
public class ErrorController {

    @ExceptionHandler(DuplicateUsernameException.class)
    public ResponseEntity duplicatedUsername(DuplicateUsernameException e) {
        log.error(e.getMessage());
        Response response = Response.builder()
                .result(ResultStatus.FAIL)
                .status(BAD_REQUEST.value())
                .message(e.getMessage())
                .build();
        return ResponseEntity.status(BAD_REQUEST).body(response);
    }

    @ExceptionHandler(...)
    ...
}
```