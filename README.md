
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
<details>
<summary>접기/펼치기 버튼</summary>
<div markdown="1">

### 서블릿 오류 페이지 등록 
```java
@Component
public class WebServerCustomizer implements WebServerFactoryCustomizer<ConfigurableWebServerFactory> {

    @Override
    public void customize(ConfigurableWebServerFactory factory) {
        ErrorPage errorPage404 = new ErrorPage(HttpStatus.NOT_FOUND, "/error-page/404");
        ErrorPage errorPage500 = new ErrorPage(HttpStatus.INTERNAL_SERVER_ERROR, "/error-page/500");
        ErrorPage errorPageRunTimeEx = new ErrorPage(RuntimeException.class, "/error-page/500");

        factory.addErrorPages(errorPage404, errorPage500, errorPageRunTimeEx);
    }
}
```
- 특정 상태코드의 예외페이지 등록
- 특정 예외 및 그 하위 타입의 예외페이지 등록

### 오류를 처리할 컨트롤러 등록
```java
@Slf4j
@Controller
public class ErrorPageController {

    @RequestMapping("/error-page/404")
    public String errorPage404(HttpServletRequest request, HttpServletResponse response) {
        log.info("errorPage 404");
        return "error-page/404";
    }

    @RequestMapping("/error-page/500")
    public String errorPage500(HttpServletRequest request, HttpServletResponse response) {
        log.info("errorPage 500");
        return "error-page/500";
    }
}
```
- 오류가 발생했을 때 처리할 컨트롤러 및 화면이 필요함

</div>
</details>

## 8.3 서블릿 예외처리 - 오류 페이지 작동 원리

<details>
<summary>접기/펼치기 버튼</summary>
<div markdown="1">

### 8.3.1 예외 발생 흐름

서블릿은 다음 상황일 때 설정된 오류 페이지를 찾는다.
   - 발생된 Exception이 서블릿 밖으로 전파 될 때
   - 또는 sendError가 호출되었을 때

#### 예외 전파
```
WAS(여기까지 전파) <- 필터 <- 서블릿 <- 인터셉터 <- 컨트롤러(예외 발생)
```
#### sendError 감지
```
WAS(sendError 호출 기록 확인) <- 필터 <- 서블릿 <- 인터셉터 <- 컨트롤러(sendError)
```

### 8.3.2 오류페이지 확인 및 내부 재요청
- 서블릿은 예외를 감지하면 해당 예외를 처리하는 오류 페이지 정보를 확인한다. 
- 오류페이지를 출력하기 위해 지정된 페이지를 다시 요청한다.
  - 오류 페이지 경로로 요청하기까지, 필터, 서블릿, 인터셉터, 컨트롤러를 다시 호출됨

#### 오류 페이지 요청 흐름
```
WAS("/error-page/500" 내부 재요청) -> 필터 -> 서블릿 -> 인터셉터 -> 컨트롤러("/error-page/500/") -> View
```

### 8.3.3 오류 정보 추가
- WAS는 오류 페이지를 다시 요청하는 것만 하는 것이 아니라, 오류 정보를 request의 attribute에 추가해서 넘겨줌
- 오류페이지에서 전달된 오류 정보를 사용할 수 있다.

#### request.attribute에 서버가 담아준 정보
- `javax.servlet.error.exception` : 예외
- `javax.servlet.error.exception_type` : 예외 타입
- `javax.servlet.error.message` : 오류 메시지
- `javax.servlet.error.request_uri` : 클라이언트 요청 URI
- `javax.servlet.error.servlet_name` : 오류가 발생한 서블릿 이름
- `javax.servlet.error.status_code` : HTTP 상태 코드

</div>
</details>

## 8.4 서블릿 예외처리 - 필터
<details>
<summary>접기/펼치기 버튼</summary>
<div markdown="1">

### 8.4.1 DispatcherType
```java
public enum DispatcherType {
    FORWARD,
    INCLUDE,
    REQUEST,
    ASYNC,
    ERROR
}
```
- 예외가 발생하거나 sendError되면 다시 예외페이지로 필터-서블릿-인터셉터-컨트롤러로 재요청 발생
- 근데 로그인 같은 로직을 다시 필터를 적용하긴 배우 불필요함
- 이런 것들을 구분하기 위해서 서블릿에서는 DispatcherType을 정의함
  - REQUEST : 클라이언트 요청
  - ERROR : 오류 요청
  - FORWARD : 서블릿에서 다른 서블릿이나 JSP를 호출할 때
    - `requestDispatcher.forward(request, response)`
  - INCLUDE : 서블릿에서 다른 서블릿이나 JSP 결과 포함
    - `requestDispatcher.include(request, response)`
  - ASYNC : 서블릿 비동기 호출

### 8.4.2 DispatcherType과 필터
```java
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        String requestURI = httpRequest.getRequestURI();

        String uuid = UUID.randomUUID().toString();

        try {
            log.info("REQUEST [{}][{}][{}]", uuid, request.getDispatcherType(), requestURI);
            chain.doFilter(request, response);
        } catch (Exception e) {
            log.info("exception! {}", e.getMessage());
            throw e;
        } finally {
            log.info("RESPONSE [{}][{}][{}]", uuid, request.getDispatcherType(), requestURI);
        }
    }
```
```java

@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Bean
    public FilterRegistrationBean logFilter() {
        FilterRegistrationBean<Filter> filterFilterRegistrationBean = new FilterRegistrationBean<>();
        filterFilterRegistrationBean.setFilter(new LogFilter());
        filterFilterRegistrationBean.setOrder(1);
        filterFilterRegistrationBean.addUrlPatterns("/*");
        filterFilterRegistrationBean.setDispatcherTypes(DispatcherType.REQUEST, DispatcherType.ERROR);
        return filterFilterRegistrationBean;
    }
}

```
- FilterRegistrationBean에 setDispatcherTypes(...)에 필터링을 적용하고 싶은 DispatcherType을 지정할 수 있음
  - 기본값 : `DispatcherType.REQUEST` 만
    - 기본값이 REQUEST로 되어있기 때문에, 재요청 시 다시 필터를 거치지 않음
  - 만약 Request, Error만 적용하고 싶으면 REQUEST, ERROR을 지정
```
2022-05-18 17:48:49.323  INFO 4912 --- [nio-8080-exec-6] hello.exception.filter.LogFilter         : REQUEST [17b39eb2-4b68-404c-98f3-05d884daee42][REQUEST][/error-ex]
2022-05-18 17:48:49.324  INFO 4912 --- [nio-8080-exec-6] hello.exception.filter.LogFilter         : exception! Request processing failed; nested exception is java.lang.RuntimeException: 예외 발생!
2022-05-18 17:48:49.324  INFO 4912 --- [nio-8080-exec-6] hello.exception.filter.LogFilter         : RESPONSE [17b39eb2-4b68-404c-98f3-05d884daee42][REQUEST][/error-ex]
2022-05-18 17:48:49.324 ERROR 4912 --- [nio-8080-exec-6] o.a.c.c.C.[.[.[/].[dispatcherServlet]    : Servlet.service() for servlet [dispatcherServlet] in context with path [] threw exception [Request processing failed; nested exception is java.lang.RuntimeException: 예외 발생!] with root cause

java.lang.RuntimeException: 예외 발생!
// 중략

// 재요청
2022-05-18 17:48:49.325  INFO 4912 --- [nio-8080-exec-6] hello.exception.filter.LogFilter         : REQUEST [407961e5-1b01-4b54-93fa-24dd336f79dc][ERROR][/error-page/500]
2022-05-18 17:48:49.326  INFO 4912 --- [nio-8080-exec-6] h.exception.servlet.ErrorPageController  : errorPage 500
2022-05-18 17:48:49.327  INFO 4912 --- [nio-8080-exec-6] h.exception.servlet.ErrorPageController  : ERROR_EXCEPTION: ex=

java.lang.RuntimeException: 예외 발생!
// 중략

2022-05-18 17:48:49.327  INFO 4912 --- [nio-8080-exec-6] h.exception.servlet.ErrorPageController  : ERROR_EXCEPTION_TYPE: class java.lang.RuntimeException
2022-05-18 17:48:49.327  INFO 4912 --- [nio-8080-exec-6] h.exception.servlet.ErrorPageController  : ERROR_MESSAGE: Request processing failed; nested exception is java.lang.RuntimeException: 예외 발생!
2022-05-18 17:48:49.327  INFO 4912 --- [nio-8080-exec-6] h.exception.servlet.ErrorPageController  : ERROR_REQUEST_URI: /error-ex
2022-05-18 17:48:49.327  INFO 4912 --- [nio-8080-exec-6] h.exception.servlet.ErrorPageController  : ERROR_SERVLET_NAME: dispatcherServlet
2022-05-18 17:48:49.327  INFO 4912 --- [nio-8080-exec-6] h.exception.servlet.ErrorPageController  : ERROR_STATUS_CODE: 500
2022-05-18 17:48:49.327  INFO 4912 --- [nio-8080-exec-6] h.exception.servlet.ErrorPageController  : dispatcherType = ERROR
2022-05-18 17:48:49.329  INFO 4912 --- [nio-8080-exec-6] hello.exception.filter.LogFilter         : RESPONSE [407961e5-1b01-4b54-93fa-24dd336f79dc][ERROR][/error-page/500]
```
- 실제로 setDispatcherType로 REQUEST, ERROR를 등록해두면 오류로 인한 재요청 시에도 다시 필터를 거치게 됨

</div>
</details>

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
