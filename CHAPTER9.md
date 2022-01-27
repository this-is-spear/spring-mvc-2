# API 예외 처리

<details>
<summary></summary>
<div markdown="1">
</div>
</details>

오류 페이지는 고객에게 화면을 보여주면 되지만, API는 각 오류 상황에 맞는 오류 응답 스펙을 정하고, JSON으로 데이터를 리턴 해야한다.

## 서블릿 오류 페이지 방식

<details>
<summary>WebServerCustomizer</summary>

WAS에 예외가 전달되거나, response.sendError() 가 호출되면 위에 등록한 예외 페이지 경로가 호출된다.

```java
@Component
public class WebServerCustomizer implements
WebServerFactoryCustomizer<ConfigurableWebServerFactory> {
    @Override
    public void customize(ConfigurableWebServerFactory factory) {
        ErrorPage errorPage404 = new ErrorPage(HttpStatus.NOT_FOUND, "/error-page/404");
        ErrorPage errorPage500 = new ErrorPage(HttpStatus.INTERNAL_SERVER_ERROR, "/error-page/500");
        ErrorPage errorPageEx = new ErrorPage(RuntimeException.class, "/error-page/500");
        factory.addErrorPages(errorPage404, errorPage500, errorPageEx);
    }
}
```

<div markdown="1">
<div>
</details>

<details>
<summary>ApiExceptionController - API 예외 컨트롤러</summary>
<div markdown="1">

단순히 회원을 조회하는 기능을 만들어 URL에전달된 id의값이 ex이면 예외가 발생하도록 설정했다.

```java
@Slf4j
@RestController
public class ApiExceptionController {

    @GetMapping("/api/members/{id}")
    public MemberDto getMember(@PathVariable("id") String id) {
        if (id.equals("ex")) {
        throw new RuntimeException("잘못된 사용자");
        }
        return new MemberDto(id, "hello " + id);
    }
    
    @Data
    @AllArgsConstructor
    static class MemberDto {
        private String memberId;
        private String name;
    }
}
```

<div>
</details>

<details>
<summary>ErrorPageController - API 응답 추가</summary>
<div markdown="1">

`produces = MediaType.APPLICATION_JSON_VALUE` 코드를 이용해 `application/json`일 때 해당 메서드를  호출하도록 설정한다. 그러면 뷰가 아닌 데이터로 `result` 값과 함께 `statuc code` 값을 Map 타입으로 리턴할 수 있다.

```java
@RequestMapping(value = "/error-page/500", produces =
  MediaType.APPLICATION_JSON_VALUE)
  public ResponseEntity<Map<String, Object>> errorPage500Api(HttpServletRequest request, HttpServletResponse response) {
    log.info("API errorPage 500");
    Map<String, Object> result = new HashMap<>();
    Exception ex = (Exception) request.getAttribute(ERROR_EXCEPTION);
    result.put("status", request.getAttribute(ERROR_STATUS_CODE));
    result.put("message", ex.getMessage());
    Integer statusCode = (Integer) request.getAttribute(RequestDispatcher.ERROR_STATUS_CODE);
    return new ResponseEntity(result, HttpStatus.valueOf(statusCode));
  }
```

<div>
</details>

## 스프링 부트 기본 오류 처리

스프링 부트가 제공하는 기본 오류 방식을 사용해 API도 예외 처리할 수 있다.

BasicErrorController

```java
@RequestMapping(produces = MediaType.TEXT_HTML_VALUE)
public ModelAndView errorHtml(HttpServletRequest request, HttpServletResponse
response) {}

@RequestMapping
public ResponseEntity<Map<String, Object>> error(HttpServletRequest request) {}
```

- `errorHtml()` : `produces = MediaType.TEXT_HTML_VALUE` 코드를 이용해 요청의 Accept 값이 `text/html`인 경우에는 `errorHtml()` 메서드를 호출해 view를 제공한다.
- `error()` : `errorHtml()` 메서드 외의 모든 경우에 호출이 되고 `ResponseEntity`로 HTTP Body에 JSON 데이터를 반환한다.

> `BasicErrorController`를 확장하면 JSON 메시지도 변경할 수 있지만, API 오류는 `@ExceptionHandler` 애너테이션을 사용하자

스프링 부트는 BasicErrorController 가 제공하는 기본 정보들을 활용해서 오류 API를 생성하고, 다음 옵션들을 설정하면 더 자세한 오류 정보를 추가할 수 있다.

- server.error.include-binding-errors=always
- server.error.include-exception=true
- server.error.include-message=always
- server.error.include-stacktrace=always

## HandlerExceptionResolver

### HandlerExceptionResolver 사용

스프링 MVC는 컨트롤러(핸들러) 밖으로 예외가 던져진 경우 예외를 해결하고, 동작을 새로 정의할 수 있는 방법을 제공한다. 컨트롤러 밖으로 던져진 예외를 해결하고, 동작 방식을 변경하고 싶으면 `HandlerExceptionResolver`(`ExceptionResolver`이라 불린다.)를 사용하면 된다.

<details>
<summary>HandlerExceptionResolver - 인터페이스</summary>
<div markdown="1">

`HandlerExceptionResolver`은 `ModelAndView`를 반환한다.

```java
public interface HandlerExceptionResolver {
    ModelAndView resolveException(
      HttpServletRequest request, HttpServletResponse response,
      Object handler, Exception ex);
}
```

- handler : 핸들러(컨트롤러) 정보
- Exception ex : 핸들러(컨트롤러)에서 발생한 발생한 예외

</div>
</details>

> `ExceptionResolver`가 `ModelAndView`를 반환하는 이유는 마치 `try, catch`를 하듯이, `Exception`을 처리해서 정상 흐름 처럼 변경하는 것이 목적이다. 이름 그대로 `Exception`을 해결하는 것이 목적이다.

<details>
<summary>MyHandlerExceptionResolver</summary>
<div markdown="1">

```java
@Slf4j
public class MyHandlerExceptionResolver implements HandlerExceptionResolver {
    @Override
    public ModelAndView resolveException(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) {
        try {
            if (ex instanceof IllegalArgumentException) {
                log.info("IllegalArgumentException resolver to 400");
                response.sendError(HttpServletResponse.SC_BAD_REQUEST, ex.getMessage());
                return new ModelAndView();
            }
        } catch (IOException e) {
            log.error("resolver ex", e);
        }
        return null;
    }
}
```

</div>
</details>

<details>
<summary>WebMvcConfigurer를 사용해 `ExceptionResolver` 등록</summary>
<div markdown="1">

`configureHandlerExceptionResolvers(..)` 메서드를 등록하면 스프링이 기본으로 등록하는 ExceptionResolver가 제거 되므로 `extendHandlerExceptionResolvers(...)` 메서드를 사용해야한다.

```java
@Override
public void extendHandlerExceptionResolvers(List<HandlerExceptionResolver>
resolvers) {
    resolvers.add(new MyHandlerExceptionResolver());
}
```

</div>
</details>

### `HandlerExceptionResolver`에서 반환하는 값에 따른 `DispatcherServlet` 서블릿 동작 방식

- 빈 `ModelAndView` : new ModelAndView()처럼 ModelAndView를 반환하면 뷰를 렌더링하지 않고, 서블릿이 리턴된다.(정상흐름)
- 지정한 `ModelAndView` : ModelAndView에 View, Model 등의 정보를 지정해 반환하면 뷰를 렌더링한다.
- `null` : null을 반환하면, `ExceptionResolver`를 찾아서 실행한다. 처리할 수 있는 `ExceptionResolver`가 없으면 예외 처리가 안되고, 기존에 발생한 예외를 서블릿 밖으로 던진다.

### `ExceptionResolver` 활용

- 예외 상태 코드 변환할 수 있다.
  - 예외를 `response.sendError(xxx)`호출로 변경해 서블릿에서 상태 코드에 따른 오류를 처리하도록 위임할 수 있다.
  - 이후 WAS는 서블릿 오류 페이지를 찾아서 내부 호출한다.
- 뷰 템플릿 처리
  - `ModelAndView`에 값을 채워서 예외에 따른 새로운 오류 화면 뷰 렌더링 해서 고객에게 제공
- API 응답 처리
  - `response.getWriter().println("hello");` 처럼 HTTP 응답 바디에 직접 데이터를 넣어주는 것도 가능하다. 여기에 JSON 으로 응답하면 API 응답 처리를 할 수 있지만 추천하지 않는다.

## HandlerExceptionResolver 활용

예외가 발생하면 WAS까지 예외가 던져지고, WAS에서 오류 페이지 정보를 찾아서 다시 `/error`를 호출하는 과정은 너무 복잡하기 때문에, `ExceptionResolver`를 활용해 복잡한 과정 없이 문제를 해결할 수 있다.

<details>
<summary>UserException</summary>
<div markdown="1">

```java
public class UserException extends RuntimeException {
      public UserException() {
          super();
}
      public UserException(String message) {
          super(message);
}
      public UserException(String message, Throwable cause) {
          super(message, cause);
      
 }
      public UserException(Throwable cause) {
          super(cause);
}
      protected UserException(String message, Throwable cause, boolean
  enableSuppression, boolean writableStackTrace) {
          super(message, cause, enableSuppression, writableStackTrace);
      }
}
```

</div>
</details>

<details>
<summary>UserHandlerExceptionResolver</summary>
<div markdown="1">

HTTP 요청 해더의 ACCEPT 값이 `application/json`이면 JSON으로 오류를 내려주고, 그 외 경우에는 `error/500`에 있는 HTML 오류 페이지를 보여준다.

```java
@Slf4j
  public class UserHandlerExceptionResolver implements HandlerExceptionResolver {
   
 private final ObjectMapper objectMapper = new ObjectMapper();
@Override
    public ModelAndView resolveException(HttpServletRequest request,
HttpServletResponse response, Object handler, Exception ex) {
        try {
            if (ex instanceof UserException) {
                log.info("UserException resolver to 400");
                String acceptHeader = request.getHeader("accept");
                response.setStatus(HttpServletResponse.SC_BAD_REQUEST);
                if ("application/json".equals(acceptHeader)) {
                    Map<String, Object> errorResult = new HashMap<>();
                    errorResult.put("ex", ex.getClass());
                    errorResult.put("message", ex.getMessage());
                    String result = objectMapper.writeValueAsString(errorResult);
                    response.setContentType("application/json");
                    response.setCharacterEncoding("utf-8");
                    response.getWriter().write(result);
                    return new ModelAndView();
                } else {
                    //TEXT/HTML
                    return new ModelAndView("error/500");
                }
            }
        } catch (IOException e) {
            log.error("resolver ex", e);
        }
        return null;
    }
}
```

</div>
</details>

### 정리

## 스프링이 제공하는 ExceptionResolver

스프링 부트가 기본으로 제공하는 `ExceptionResolver`는 다음과 같다. `HandlerExceptionResolverComposite`에 다음 순서로 등록이 된다.

1. `ExceptionHandlerExceptionResolver`
2. `ResponseStatusExceptionResolver`
3. `DefaultHandlerExceptionResolver`

### ExceptionHandlerExceptionResolver

@ExceptionHandler을 처리한다. API 예외 처리는 대부분 이 기능으로 해결한다.

### @ExceptionHandler

스프링은 `ExceptionHandlerExceptionResolver`를 기본으로 제공하고, 기본으로 제공하는 `ExceptionResolver` 중에 우선순위가 가장 높다.

### ResponseStatusExceptionResolver

예외에 따라서 HTTP 상태 코드를 지정해주는 역할을 한다. 다음 두 가지 경우를 처리한다.

- @ResponseStatus 애너테이션이 추가된 예외
- ResponseStatusException 예외

`@ResponseStatus` 애노테이션을 적용하면 HTTP 상태 코드를 변경해준다. `@ResponseStatus`안의 `reason`은 `MessageSource`에서 메시지를 찾는 기능도 제공한다. `reason = "error.bad"` 아래와 같이 설정하면 `BadRequestException` 오류가 throw 될 때, 아래 설정한 Http 상태 코드와 메시지를 응답한다.

```java
//@ResponseStatus(code = HttpStatus.BAD_REQUEST, reason = "잘못된 요청 오류")
@ResponseStatus(code = HttpStatus.BAD_REQUEST, reason = "error.bad") 
public class BadRequestException extends RuntimeException {}
```

> @ResponseStatus 는 개발자가 직접 변경할 수 없는 예외에는 적용할 수 없다. 추가로 애노테이션을 사용하기 때문에 조건에 따라 동적으로 변경하는 것도 어렵다. 이때는 ResponseStatusException 예외를 사용하면 된다.

### DefaultHandlerExceptionResolver

스프링 내부에서 발생하는 스프링 예외를 해결한다. 파라미터 바인딩은 대부분 클라이언트가 HTTP 요청 정보를 잘못 호출해서 발생하는 문제이다. HTTP 에서는 이런 경우 HTTP 상태 코드 400을 사용하도록 되어 있다. `DefaultHandlerExceptionResolver`는 이것을 500 오류가 아니라 HTTP 상태 코드 400 오류로 변경한다. 스프링 내부 오류를 어떻게 처리 하는지의 내용이 정의되어 있다.

### API 예외처리의 어려운 점

금까지 살펴본 BasicErrorController 를 사용하거나 HandlerExceptionResolver 를 직접 구현하는 방식으로 API 예외를 다루기는 쉽지 않다. 스프링은 API 예외 처리 문제를 해결하기 위해 `@ExceptionHandler`라는 애노테이션을 사용하는 매우 편리한 예외 처리 기능을 제공한다.

## @ControllerAdvice

 `@ExceptionHandler`를 사용해서 예외를 깔끔하게 처리할 수 있게 되었지만, 정상 코드와 예외 처리 코드가 하나의 컨트롤러에 섞여 있다. `@ControllerAdvice` 또는 `@RestControllerAdvice`를 사용하면 둘을 분리할 수 있다.

### @ControllerAdvice란

- @ControllerAdvice 는 대상으로 지정한 여러 컨트롤러에 @ExceptionHandler , @InitBinder 기능을 부여해주는 역할을 한다.
- @ControllerAdvice 에 대상을 지정하지 않으면 모든 컨트롤러에 적용된다. (글로벌 적용)
- @RestControllerAdvice 는 @ControllerAdvice 와 같고, @ResponseBody 가 추가되어 있다. @Controller , @RestController 의 차이와 같다.

### 대상 컨트롤러를 지정하는 방법

```java
// Target all Controllers annotated with @RestController
@ControllerAdvice(annotations = RestController.class)
public class ExampleAdvice1 {}
```

> 스프링 공식 문서 예제에서 보는 것 처럼 특정 애노테이션이 있는 컨트롤러를 지정할 수 있고, 특정 패키지를 직접 지정할 수도 있다. 패키지 지정의 경우 해당 패키지와 그 하위에 있는 컨트롤러가 대상이 된다. 그리고 특정 클래스를 지정할 수도 있다. 대상 컨트롤러 지정을 생략하면 모든 컨트롤러에 적용된다.

### @ControllerAdvice 정리

@ExceptionHandler 와 @ControllerAdvice 를 조합하면 예외를 깔끔하게 해결할 수 있다.
