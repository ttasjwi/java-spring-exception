
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
<details>
<summary>접기/펼치기 버튼</summary>
<div markdown="1">

### 8.5.1 인터셉터에서의 중복호출 제거
```java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new LogInterceptor())
                .order(1)
                .addPathPatterns("/**")
                .excludePathPatterns(
                        "/css/**", "/*.ico",
                        "/error", "/error-page/**" // 에러 페이지 경로
                );
    }
}
```
- 인터셉터는 특정 DispatcherType에 대한 필터링 기능을 제공하지 않음
- 대신, 적용하지 않을 url 조건을 추가하여 에러페이지로의 내부 재요청에 대해서는 인터셉터를 적용하지 않는 식으로 처리 가능

### 8.5.2 정상 호출 및 오류발생 시 오류 페이지 요청 흐름
#### 정상호출 
```
WAS -> 필터 -> 서블릿 -> 인터셉터 -> 컨트롤러 -> View
```
#### 오류 발생, 내부 재요청의 흐름
```
WAS(전파) <-필터 <- 서블릿 <- 인터셉터 <- 컨트롤러
WAS -> 필터 -> 서블릿 -> 인터셉터(x) -> 컨트롤러 -> View
```
1. 요청, 컨트롤러에서 예외 발생
   - WAS -> 필터 -> 서블릿 -> 인터셉터 -> 컨트롤러(예외 발생)

2. 예외 전파
   - WAS(전파) <-필터 <- 서블릿 <- 인터셉터 <- 컨트롤러

3. 내부 재요청
   - WAS : 오류 확인, 에러페이지 내부 재요청

4. 필터/인터셉터에서 중복 호출 제거, View 반환
   - 필터 : DispatcherType으로 중복 요청 제거
   - 인터셉터 : 오류페이지 url을 제외하여 인터셉터 적용 
     - WAS -> 필터 -> 서블릿 -> 인터셉터(x) -> 컨트롤러 -> View

</div>
</details>

## 8.6 스프링 부트 - 오류 페이지1
<details>
<summary>접기/펼치기 버튼</summary>
<div markdown="1">

### 8.6.1 기존 예외 처리 페이지 등록 방식
- WebServerCustomizer 생성, 예외 종류에 따라서 ErrorPage 등록
- 예외처리용 컨트롤러 ErrorPageController를 생성

### 8.6.2 스프링 부트에서 지원하는 예외 처리 페이지 추가 기능
- ErrorPage 자동 등록 : `/error`로 기본 오류 페이지 설정
  - `new ErrorPage("/error")`,  상태코드와 예외를 설정하지 않으면 기본 오류 페이지를 설정
  - 서블릿 밖으로 예외가 발생하거나, `response.sendError`가 호출되면 모든 오류는 `/error`를 호출
  - 참고) `ErrorMvcAutoConfiguration`이라는 클래스가 오류 페이지를 자동으로 등록
- `BasicErrorController`라는 스프링 컨트롤러를 자동으로 등록
  - ErrorPage에서 등록한 `/error`를 매핑해서 처리하는 컨트롤러
- 별다른 오류 페이지를 등록하지 않았다면, 스프링은 기본적으로 오류 페이지로 `/error`을 호출한다.

### 8.6.3 개발자는 오류 페이지만 등록
- BasicErrorController는 기본적인 로직이 모두 개발되어 있다.
- 오류 페이지 화면만 `BasicErrorController`가 제공하는 룰과 우선순위에 따라 등록하면 됨.
- 정적 HTML이면 정적 리소스(`/static/error/...`)에, 동적 HTML이면 (`/templates/error/...`)에 오류 페이지 파일을 넣어두기

### 8.6.4 뷰 선택 우선순위
해당 경로 위치에 HTTP 상태 코드 이름의 뷰 파일을 넣어서 처리하면 됨. (예외는 500으로 처리된다.) 우선순위는 다음과 같으며, 5xx보다는 500과 같은 구체적인 것이 덜 구체적인 것보다 우선순위가 높다.

1. 뷰 템플릿
   - `resources/templates/error/500.html`
   - `resources/templates/error/5xx.html`
   - ...

2. 정적 리소스(static, public)
   - `resources/static/error/400.html`
   - `resources/static/error/404.html`
   - `resources/static/error/4xx.html`
   - ...

3. 적용 대상이 없을 때 뷰 이름(error)
   - `resources/templates/error.html`

</div>
</details>

## 8.7 스프링 부트 - 오류 페이지2
<details>
<summary>접기/펼치기 버튼</summary>
<div markdown="1">

### 8.7.1 BasicErrorController가 제공하는 기본 정보들
```
* timestamp: Fri Feb 05 00:00:00 KST 2021
* status: 400
* error: Bad Request
* exception: org.springframework.validation.BindException
* trace: 예외 trace
* message: Validation failed for object='data'. Error count: 1
* errors: Errors(BindingResult)
* path: 클라이언트 요청 경로 (`/hello`)
```
- BasicController는 기본적으로 위의 정보를 model에 담아서 view에 전달.
- 뷰 템플릿은 이 값을 활용해서 출력할 수 있다.
- 하지만 오류관련 내부 정보를 고객에게 노출하는 것은 보안상 문제, 고객측 혼란을 야기시킬 수 있음.
- 후술할 설정으로 어느 정도를 model에 포함할 지 여부를 선택할 수 있다.

### 8.7.2 스프링부트 오류 관련 옵션
application.properties에 다음을 등록해서 사용하면 된다.

#### 오류 컨트롤러에서 오류 정보를 model에 포함할 지 여부
```properties
# 기본값들
server.error.include-exception=false
server.error.include-message=never
server.error.include-stacktrace=never
server.error.include-binding-errors=never
```
- true/false로 조절하는 옵션
  - `server.error.include-exception=false` : exception 포함 여부
- never(사용하지 않음)/always(항상)/on_param(파라미터가 있을 때)으로 조절하는 옵션. 보통 never가 기본값
  - `server.error.include-message=never` : 메시지 포함 여부
  - `server.error.include-stacktrace=never` : trace 포함 여부
  - `server.error.include-binding-errors=never` : errors 포함 여부
- on_param 옵션은 http 요청 시 파라미터에 추가하면 적용됨
  - 예) `?messaga=&error&trace=`

#### whitelabel 오류페이지, 기본 글로벌 오류페이지 경로
```properties
# 오류처리 화면을 찾지 못 했을 경우 스프링 whitelabel 오류 페이지 적용 옵션 (기본 true)
server.error.whitelabel.enabled=false

# 오류 페이지 경로, 스프링이 자동 등록하는 서블릿 글로벌 오류 페이지 경로, BasicErrorController 오류 컨트롤러 경로에 함께 사용
server.error.path=/error
```

### 8.7.3 확장 포인트
- 예외 공통처리 컨트롤러의 기능 변경
  - ErrorController 인터페이스를 상속 받아 구현하거나
  - BasicErrorController를 상속받아서 기능 추가하기

</div>
</details>

---

# Section 9. API 예외처리

## 9.1 서블릿 오류 페이지 방식으로 API 예외처리
<details>
<summary>접기/펼치기 버튼</summary>
<div markdown="1">

```java
@RequestMapping(value = "/error-page/500", produces = MediaType.APPLICATION_JSON_VALUE)
public ResponseEntity<Map<String, Object>> errorPage500Api
        (HttpServletRequest request, HttpServletResponse response) {
    log.info("API errorPage 500");

    Map<String, Object> result = new HashMap<>();
    Exception ex = (Exception) request.getAttribute(ERROR_EXCEPTION);
    result.put("status", request.getAttribute(ERROR_STATUS_CODE));
    result.put("message", ex.getMessage());

    Integer statusCode = (Integer) request.getAttribute(ERROR_STATUS_CODE);
    return new ResponseEntity<>(result, HttpStatus.valueOf(statusCode));
}
```
```json
{
    "message": "잘못된 사용자",
    "status": 500
}
```
- 예외가 발생하면 WAS까지 전파되고, WAS는 내부적으로 예외 페이지로 재요청
- Accept가 `application/json`인 경우에 한하여 json으로 응답하도록 하기
  - Accept가 `*/*`인 경우 에러페이지로 등록한 html이 응답됨.
- ResponseEntity에 전달할 예외 api를 담아 반환.
  - 넘겨줄 Http Body 데이터
  - 넘겨줄 상태코드

</div>
</details>

## 9.2 HandlerExceptionResolver 시작

## 9.3 HandlerExceptionResolver 활용

## 9.4 스프링이 제공하는 HandlerExceptionResolver1

## 9.5 스프링이 제공하는 HandlerExceptionResolver2

## 9.6 @ExceptionHandler

## 9.7 @ControllerAdvice

---
