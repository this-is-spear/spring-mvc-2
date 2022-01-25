# 예외 처리와 오류 페이지

## 서블릿 예외 처리

서블릿은 2가지 방식으로 예외 처리를 지원한다.

- Exception
- response.sendError(HTTP 상태 코드, 오류 메시지)

### `Exception`

애플리케이션에서 try ~ catch로 예외를 잡아서 처리하면 아무런 문제가 없지만 애플리케이션에서 예외를 잡지 못하고, 서블릿 밖으로 까지 예외가 전달되면 서버 내부에서 처리할 수 없는 오류가 발생한 것으로 생각해 HTTP 상태 코드 500을 반환한다.

### `response.sendError(HTTP 상태 코드, 오류 메시지)`

`response.sendError()` 를 호출하면 response 내부에는 오류가 발생했다는 상태를 저장해둔다. 서블릿 컨테이너는 고객에게 응답 전 `response`에 `sendError()` 가 호출되었는지 확인하고, 호출되면 설정한 오류 코드에 맞추어 기본 오류 페이지를 보여준다.

## 서블릿 오류 페이지 등록

서블릿은 Exception (예외)가 발생해서 서블릿 밖으로 전달되거나 또는 response.sendError() 가 호출 되었을 때 각각의 상황에 맞춘 오류 처리 기능을 제공한다. 그리고, 스프링 부트가 제공하는 기능을 사용해서 서블릿 오류 페이지를 등록하면 된다.

WebServerCustomizer

```java
@Component
  public class WebServerCustomizer implements
  WebServerFactoryCustomizer<ConfigurableWebServerFactory> {
      @Override
      public void customize(ConfigurableWebServerFactory factory) {
          ErrorPage errorPage404 = new ErrorPage(HttpStatus.NOT_FOUND, "/error-page/404");
          ErrorPage errorPage500 = new ErrorPage(HttpStatus.INTERNAL_SERVER_ERROR, "/error-page/500");
          ErrorPage errorPageEx = new ErrorPage(RuntimeException.class, "/error- page/500");
          factory.addErrorPages(errorPage404, errorPage500, errorPageEx);
      }
}
```

- `response.sendError(404)` : errorPage404 호출
- `response.sendError(500)` : errorPage500 호출
- `RuntimeException` 또는 그 자식 타입의 예외: `errorPageEx` 호출

ErrorPageController

```java
@Slf4j
  @Controller
  public class ErrorPageController {
      @RequestMapping("/error-page/404")
      public String errorPage404(HttpServletRequest request, HttpServletResponse
  response) {
          log.info("errorPage 404");
          return "error-page/404";
      }
      @RequestMapping("/error-page/500")
      public String errorPage500(HttpServletRequest request, HttpServletResponse
  response) {
          log.info("errorPage 500");
          return "error-page/500";
      }
}
```

## 오류 페이지 작동 원리

예외 발생과 오류 페이지 요청 흐름

1. WAS(여기까지 전파) <- 필터 <- 서블릿 <- 인터셉터 <- 컨트롤러(예외발생)
2. WAS `/error-page/500` 다시 요청 -> 필터 -> 서블릿 -> 인터셉터 -> 컨트롤러(/error- page/500) -> View

정리하면, 예외가 발생해서 WAS까지 전파되고 WAS는 오류 페이지 경로를 찾아서 내부에서 오류 페이지를 호출한다. 이때 오류 페이지 경로로 필터, 서블릿, 인터셉터, 컨트롤러가 모두 다시 호출된다.

> WAS는 오류 페이지를 단순히 다시 요청만 하는 것이 아니라, 오류 정보를 request 의 attribute 에 추가해서 넘겨준다. 필요하면 오류 페이지에서 이렇게 전달된 오류 정보를 사용할 수 있다

## 필터

오류가 발생하면 오류 페이지를 출력하기 위해 WAS 내부에서 다시 한번 호출이 발생한다. 이때 필터, 서블릿, 인터셉터도 모두 다시 호출된다. 서버 내부에서 오류 페이지를 호출한다고 해서 해당 필터나 인터셉트가 한번 더 호출되는 것은 매우 비효율적이가 때문에 DispatcherType 타입을 통해 비효율적인 부분을 해결할 수 있다.

### DispatcherType

> `log.info("dispatchType={}", request.getDispatcherType())`

출력해보면 오류 페이지에서는 `dispatchType=ERROR` 로 출력하고, 고객이 처음 요청하면  `dispatcherType=REQUEST` 출력한다. 이렇듯 서블릿 스펙은 실제 고객이 요청한 것인지, 서버가 내부에서 오류 페이지를 요청하는 것인지 DispatcherType 으로 구분할 수 있는 방법을 제공한다.

### DispatcherType 종류

- REQUEST : 클라이언트 요청 ERROR : 오류 요청  
- FORWARD : MVC에서 배웠던 서블릿에서 다른 서블릿이나 JSP를 호출할 때 `RequestDispatcher.forward(request, response);`
- INCLUDE : 서블릿에서 다른 서블릿이나 JSP의 결과를 포함할 때 `RequestDispatcher.include(request, response);`
- ASYNC : 서블릿 비동기 호출

### WebConfig

setDispatcherTypes() 메서드를 이용해 해당 DispatcherType 추가할지 설정하면 된다.

```java
@Bean
public FilterRegistrationBean logFilter() {
    FilterRegistrationBean<Filter> filterRegistrationBean = new
  FilterRegistrationBean<>();
    filterRegistrationBean.setFilter(new LogFilter());
    filterRegistrationBean.setOrder(1);
    filterRegistrationBean.addUrlPatterns("/*");
    filterRegistrationBean.setDispatcherTypes(DispatcherType.REQUEST,
  DispatcherType.ERROR);
    return filterRegistrationBean;
}
```

## 인터셉터

필터의 경우에는 필터를 등록할 때 어떤 DispatcherType 인 경우에 필터를 적용할 지 선택할 수 있었다. 그런데 인터셉터는 서블릿이 제공하는 기능이 아니라 스프링이 제공하는 기능이다. 따라서 DispatcherType 과 무관하게 항상 호출된다.

### WebConfig

설정을 사용해서 오류 페이지 경로를 excludePathPatterns 를 사용해서 빼주면 된다.

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
          registry.addInterceptor(new LogInterceptor())
                  .order(1)
                   .addPathPatterns("/**")
                .excludePathPatterns( "/css/**", "/*.ico" , "/error", "/error-page/", "**" //오류 페이지 경로);
    }
}
```

## 전체 흐름 정리

### 정상 요청

> WAS(/hello, dispatchType=REQUEST) -> 필터 -> 서블릿 -> 인터셉터 -> 컨트롤러 -> View

### 오류 요청

필터는 DispatchType 으로 중복 호출 제거(`setDispatcherTypes(DispatchType.REQUEST)`), 인터셉터는 경로 정보로 중복 호출 제거(`excludePathPatterns("/error-page/**")`)

1. WAS(/error-ex, dispatchType=REQUEST) -> 필터 -> 서블릿 -> 인터셉터 -> 컨트롤러
2. WAS(여기까지 전파) <- 필터 <- 서블릿 <- 인터셉터 <- 컨트롤러(예외발생)
3. WAS 오류 페이지 확인
4. WAS(/error-page/500, dispatchType=ERROR) -> 필터(x) -> 서블릿 -> 인터셉터(x) -> 컨트롤러(/error-page/500) -> View

## 오류 페이지

스프링 부트를 이용하지 않고 예외 처리 페이지를 만들기 위해서는 `WebServerCustomizer`를 생성하고, 예외 종류에 따라 `ErrorPage`를 추가하고, 예외 처리용 컨트롤러인 `ErrorPageController를` 통해 에러 페이지를 호출했다.

스프링 부트는 위의 모든 과정을 제공한다.

- ErrorPage를 자동으로 등록하고, '/error'라는 경로로 기본 오류 페이지를 설정한다.
  - new ErrorPage("/error")와 같이 상태코드와 예외를 설정하지 않으면 기본 오류 페이지로 사용한다.
  - 서블릿 밖으로 예외가 발생하거나 response.sendError()가 호출되면 모든 오류는 `/error`를 호출하게 된다.
- BasicErrorController라는 스프링 컨트롤러를 자동으로 등록한다.
  - ErrorPage에서 등록한 `/error`를 매핑해서 처리하는 컨트롤러다.

> ErrorMvcAutoConfiguration 클래스가 오류 페이지를 자동으로 등록하는 역할을 한다.

### 개발자는 오류 페이지만 등록

BasicErrorController 는 기본적인 로직이 모두 개발되어 있기 때문에 개발자는 오류 페이지 화면만 BasicErrorController 가 제공하는 룰과 우선순위에 따라서 등록하면 된다. 정적 HTML이면 정적 리소스, 뷰 템플릿을 사용해서 동적으로 오류 화면을 만들고 싶으면 뷰 템플릿 경로에 오류 페이지 파일을 만들어서 넣어두기만 하면 된다.

### BasicErrorController가 제공하는 기본 정보들

BasicErrorController가 오류 컨트롤러에서 다음 오류 정보를 model 에 포함할지 여부 선택할 수 있다.

application.properties

```properties
server.error.include-exception=true (exception 포함 여부( true , false ))
server.error.include-message=on_param (message 포함 여부)
server.error.include-stacktrace=on_param (trace 포함 여부)
server.error.include-binding-errors=on_param (errors 포함 여부)
```

기본 값이 never인 부분은 다음 3가지 옵션을 사용할 수 있다.

> `never`, `always`, `on_param`

- never : 사용하지 않음
- always :항상 사용
- on_param : 파라미터가 있을 때 사용

> on_param 은 파라미터가 있으면 해당 정보를 노출한다. 디버그 시 문제를 확인하기 위해 사용할 수 있다. 그런데 이 부분도 개발 서버에서 사용할 수 있지만, 운영 서버에서는 권장하지 않는다.

### 스프링 부트 오류 관련 옵션

- `server.error.whitelabel.enabled=true`
  - 오류 처리 화면을 못 찾을 시, 스프링 whitelabel 오류 페이지 적용
- `server.error.path=/error`
  - 오류 페이지 경로, 스프링이 자동 등록하는 서블릿 글로벌 오류 페이지 경로와 BasicErrorController 오류 컨트롤러 경로에 함께 사용된다.

### 확장 포인트

에러 공통 처리 컨트롤러의 기능을 변경하고 싶으면 ErrorController 인터페이스를 상속 받아서 구현하거나 BasicErrorController 상속 받아서 기능을 추가하면 된다.
