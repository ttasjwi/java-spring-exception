
# java-spring-exception

- 인프런 김영한님의 '스프링 MVC 2편 - 백엔드 웹개발 활용 기술' 강의
- spring의 예외처리 학습을 위한 Repository

---

# Section 8. 예외처리와 오류 페이지

## 8.1 서블릿 예외처리 - 시작

<details>
<summary>접기/펼치기 버튼</summary>
<div markdown="1">

### 준비
```properties
server.error.whitelabel.enabled=false
```
- application.properties : 스프링이 제공하는 기본 예외 페이지 끄기

### 순수 java의 예외 전파
- 어떤 메서드에서 예외가 발생했을 경우, CallStack에서 상위 StackFrame의 메서드로 예외 전파
- 스레드의 최상위 메서드에서 예외가 던져지면, 예외 정보를 남기고 스레드 종료
- 참고 : 서블릿은 요청당 스레드.

### 서블릿에서의 예외 전파
```
WAS(여기까지 전파) <- 필터 <- 서블릿 <- 인터셉터 <- 컨트롤러(예외 발생)
```
- 결국은 Tomcat과 같은 WAS까지 예외가 전파됨
- 서블릿 컨테이너가 제공하는 기본 오류 화면이 보임

### 예외 throw
```java
@Slf4j
@Controller
public class ServletExController {

    @GetMapping("/error-ex")
    public void errorEx() {
        throw new RuntimeException("예외 발생!");
    }
}
```
```
Http Status 500 - Internal Server Error
```
- 컨트롤러에서 Exception이 던져져서 WAS까지 도달하면, 서버 내부에서 처리할 수 없는 예외가 발생한 것으로 간주하고, HTTP 상태코드 500을 반환

### 등록되지 않은 페이지 접근
```
HTTP Status 404 - Not Found
```
- 톰캣이 기본적으로 제공하는 404 오류 화면 제공

### response.sendError
```
WAS(snedError 호출 기록 확인) <- 필터 <- 서블릿 <- 인터셉터 <- 컨트롤러(response.sendError)
```
- `HttpServletResponse`의 sendError 메서드를 사용
  - response.sendError(상태코드)
  - response.sendError(상태코드, 오류 메시지)
- response.sendError을 호출하면, response 내부에 예외가 발생했다는 상태를 저장
- 서블릿 컨테이너는 응답 전에 response에 sendError()가 호출되었는지 확인, 호출되었을 경우 오류 코드에 맞추어 기본 오류 페이지를 보여줌

### 정리
- 별다른 처리를 하지 않을 경우 컨트롤러에서 발생한 예외는 WAS까지 전파
- 별다른 예외 페이지를 설정하지 않을 경우 톰캣에서 제공하는 기본 예외페이지가 띄워짐
- 하지만 기본 예외페이지는 사용자가 보기에 불편하므로 별도로 의미 있는 오류 화면을 제공할 필요성이 있다.

</div>
</details>

## 8.2 서블릿 예외처리 - 오류 화면 제공

## 8.3 서블릿 예외처리 - 오류 페이지 작동 원리

## 8.4 서블릿 예외처리 - 필터

## 8.5 서블릿 예외처리 - 인터셉터

## 8.6 스프링 부트 - 오류 페이지1

## 8.7 스프링 부트 - 오류 페이지2

---

# Section 9. API 예외처리

## 9.1 스프링 부트 기본 오류 처리

## 9.2 HandlerExceptionResolver 시작

## 9.3 HandlerExceptionResolver 활용

## 9.4 스프링이 제공하는 HandlerExceptionResolver1

## 9.5 스프링이 제공하는 HandlerExceptionResolver2

## 9.6 @ExceptionHandler

## 9.7 @ControllerAdvice

---
