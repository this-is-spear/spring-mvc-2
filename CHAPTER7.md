# 로그인 처리 2 - 필터, 인터셉터

여러 로직에서 공통으로 관심 있는 것을 공통 관심사(cross-cutting-concern)라고 하며 인증에 대해 여러 로직에서 공통으로 관심을 가지는 경우도 공통 관심사이다. 이러한 공통 관심사는 스프링의 AOP로도 해결할 수 있지만, 웹과 관련된 공통 관심사는 서블릿 필터와 스프링 인터셉터를 사용하는 것이 좋다. 웹과 관련된 공통 관심사를 처리할 때는 HTTP의 헤더나 URL 정보들이 필요한데 필터와 인터셉터는 HttpServletRequest를 제공한다.

## 서블릿 필터

### 서블릿 필터 소개

#### 필터의 흐름

HTTP 요청 -> WAS -> 필터 -> 서블릿 -> 컨트롤러

필터를 적용하면 필터가 호출 된 다음에 디스패처 서블릿이 호출 된다. 그래서 모든 고객의 요청 로그를 남기는 요구사항이 있다면 필터를 사용하면 된다. 필터는 특정 URL 패턴에 적용할 수 있고, `/*` 이라고 지정하면 모든 요청에 필터가 적용이 된다.

#### 필터 제한

HTTP 요청 -> WAS -> 필터 -> 서블릿 -> 컨트롤러 //로그인 사용자
HTTP 요청 -> WAS -> 필터(적절하지 않은 요청이라 판단, 서블릿 호출X) //비 로그인 사용자

필터에서 적절하지 않은 요청이라 판단하면 디스패처 서블릿을 호출하지 않고 필터에서 멈추게 설정할 수 있다.

#### 필터 체인

HTTP 요청 ->WAS-> 필터1-> 필터2-> 필터3-> 서블릿 -> 컨트롤러

필터는 체인으로 구성이 되고, 중간에 필터를 자유롭게 추가할 수 있다. (로그를 남기는 필터를 먼저 적용하고, 그 다음에 로그인 여부를 체크하는 필터를 만들 수 있다.)

#### 필터 인터페이스

필터 인터페이스를 구현하고 등록하면 서블릿 컨테이너가 필터를 싱글톤 객체로 생성하고 관리한다.

<details>
<summary>Filter</summary>
<div markdown="1">

```java
public interface Filter {

    public default void init(FilterConfig filterConfig) throws ServletException {}

    public void doFilter(ServletRequest request, ServletResponse response,
            FilterChain chain) throws IOException, ServletException;

    public default void destroy() {}
}
```

</div>
</details>

- init() : 필터 초기화 메서드, 서블릿 컨테이너가 생성될 때 호출된다.
- doFilter() : 고객의 요청이 올 때 마다 해당 메서드가 호출된다. 필터의 로직을 구현하면 된다.
- destroy() : 필터 종료 메서드, 서블릿 컨테이너가 종료될 때 호출이 된다.

### 서블릿 필터 요청 로그

<details>
<summary>LogFilter - doFIlter()</summary>
<div markdown="1">

필터를 사용하려면 필터 인터페이스를 구현해야 한다.

```java
@Slf4j
public class LogFilter implements Filter {

    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
        log.info("log filter init");
    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        log.info("log filter doFilter");
        HttpServletRequest httpServletRequest = (HttpServletRequest) request;
        String requestURI = httpServletRequest.getRequestURI();

        String uuid = UUID.randomUUID().toString();

        try {
            log.info("REQUEST [{}][{}]", uuid, requestURI);
            chain.doFilter(request, response);
        } catch (Exception e) {
            throw e;
        } finally {
            log.info("RESPONSE [{}][{}]", uuid, requestURI);
        }
    }

    @Override
    public void destroy() {
        log.info("log filter destroy");
    }
}
```

- `doFilter(ServletRequest request, ServletResponse response, FilterChain chain)`
  - HTTP 요청이 오면 doFilter 가 호출된다.
  - `ServletRequest request` 는 **HTTP 요청이 아닌 경우까지 고려해서 만든 인터페이스**이다. HTTP를 사용하면 `HttpServletRequest httpRequest = (HttpServletRequest) request;` 와 같이 다운 케스팅 하면 된다.
- `String uuid = UUID.randomUUID().toString();`
  - HTTP 요청을 구분하기 위해 요청당 임의의 uuid 를 생성해둔다.
- `log.info("REQUEST [{}][{}]", uuid, requestURI);`
  - uuid 와 requestURI 를 출력한다.
- **`chain.doFilter(request, response);`**
  - 다음 필터가 있으면 필터를 호출하고, 필터가 없으면 서블릿을 호출한다. 만약 이 로직을 호출하지 않으면 다음 단계로 진행되지 않는다.

</div>
</details>


<details>
<summary>WebConfig - FilterRegistrationBean</summary>
<div markdown="1">

스프링 부트를 사용한다면 FilterRegistrationBean 을 사용해서 등록하면 된다.

```java
@Configuration
    public class WebConfig {
        @Bean
        public FilterRegistrationBean logFilter() {
            FilterRegistrationBean<Filter> filterRegistrationBean = new
    FilterRegistrationBean<>();
            filterRegistrationBean.setFilter(new LogFilter());
            filterRegistrationBean.setOrder(1);
            filterRegistrationBean.addUrlPatterns("/*");
            return filterRegistrationBean;
} }
```

- `setFilter(new LogFilter())` : 등록할 필터를 지정한다.
- `setOrder(1)` : 필터는 체인으로 동작한다. 따라서 순서가 필요하다. 낮을 수록 먼저 동작한다.
- `addUrlPatterns("/*")` : 필터를 적용할 URL 패턴을 지정한다. 한번에 여러 패턴을 지정할 수 있다.

</div>
</details>

`@ServletComponentScan @WebFilter(filterName = "logFilter", urlPatterns = "/*")` 로 필터 등록이 가능하지만 필터 순서 조절이 안되니 `FilterRegistrationBean`을 사용하자.

### logback mdc

### 서블릿 필터 인증 체크

<details>
<summary>LoginCheckFilter</summary>
<div markdown="1">

```java
@Slf4j
public class LoginCheckFilter implements Filter {

    private static final String[] whitelist = {"/", "/members/add", "/login", "/logout", "/css/*"/*, "/*.ico"*/};

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        HttpServletRequest httpServletRequest = (HttpServletRequest) request;
        String requestURI = httpServletRequest.getRequestURI();

        HttpServletResponse httpServletResponse = (HttpServletResponse) response;

        try {
            log.info("인증 체크 필터 시작 [{}]", requestURI);

            if (isLoginCheckPath(requestURI)) {
                log.info("인증 체크 로직 실행 {}", requestURI);
                HttpSession session = httpServletRequest.getSession(false);
                if (session == null || session.getAttribute(SessionConst.LOGIN_MEMBER) == null) {
                    log.info("미인증 사용자 요청 {}", requestURI);
                    //로그인으로 리다이렉트
                    httpServletResponse.sendRedirect("/login?redirectURL=" + requestURI);
                    return;
                }
            }
            chain.doFilter(request, response);
        } catch (Exception e) {
            throw e;
        }finally {
            log.info("인증 체크 필터 종료 [{}]", requestURI);
        }
    }

    private boolean isLoginCheckPath(String requestURI) {
        return !PatternMatchUtils.simpleMatch(whitelist, requestURI);
    }
}

```

- `whitelist = {"/", "/members/add", "/login", "/logout","/css/*"};`
  - 화이트 리스트 경로는 인증과 무관하게 항상 허용한다. 화이트 리스트를 제외한 나머지 모든 경로에는 인증 체크 로직을 적용한다.
- `httpResponse.sendRedirect("/login?redirectURL=" + requestURI);`
  - 미인증 사용자는 로그인 화면으로 리다이렉트 한다. 원하는 경로를 다시 찾아가는 기능을 위해 현재 요청한 경로인 requestURI 를 /login 에 쿼리 파라미터로 함께 전달한 다음 /login 컨트롤러에서 로그인 성공시 해당 경로로 이동하는 기능은 추가 한다.
- `return;`
  - 필터를 더는 진행하지 않는다. 필터, 서블릿, 컨트롤러가 더는 호출되지 않고, redirect 가 응답으로 적용되고 요청이 끝난다.

</div>
</details>


<details>
<summary>WebConfig - FilterRegistrationBean </summary>
<div markdown="1">

```java
@Bean
public FilterRegistrationBean loginCheckFilter() {
    FilterRegistrationBean<Filter> filterRegistrationBean = new FilterRegistrationBean<>();
    filterRegistrationBean.setFilter(new LoginCheckFilter());
    filterRegistrationBean.setOrder(2);
    filterRegistrationBean.addUrlPatterns("/*");
    return filterRegistrationBean;
}
```

- `setFilter(new LoginCheckFilter())`
  - 로그인 필터를 등록한다.
- `setOrder(2)`
  - 로그 필터 다음에 로그인 필터가 적용된다.
- `addUrlPatterns("/*")`
  - 모든 요청에 로그인 필터를 적용한다.

</div>
</details>

### 정리

공통 관심사를 서블릿 필터를 사용해서 해결한 덕분에 향후 로그인 관련 정책이 변경되어도 이 부분만 변경하면 되고, 서블릿 필터를 잘 사용한 덕분에 로그인 하지 않은 사용자는 나머지 경로에 들어갈 수 없다.

### 참고

필터에는 다음에 설명할 스프링 인터셉터는 제공하지 않는 아주 강력한 기능이 있는데 `chain.doFilter(request, response);` 를 호출해서 다음 필터 또는 서블릿을 호출할 때 `request` , `response` 를 다른 객체로 바꿀 수 있다. `ServletRequest` , `ServletResponse` 를 구현한 다른 객체를 만들어서 넘기면 해당 객체가 다음 필터 또는 서블릿에서 사용된다.

## 서블릿 인터셉터

### 서블릿 인터셉터 소개

스프링 인터셉터도 서블릿 필터와 같이 웹과 관련된 공통 관심 사항을 효과적으로 해결할 수 있는 기술이다. 서블릿 필터가 서블릿이 제공하는 기술이라면, 스프링 인터셉터는 스프링 MVC가 제공하는 기술이다. 둘다 웹과 관련된 공통 관심 사항을 처리하지만, 적용되는 순서와 범위, 그리고 사용방법이 다르다.

### 스프링 인터셉터 흐름

HTTP 요청 -> WAS -> 필터 -> 서블릿 -> 스프링 인터셉터 -> 컨트롤러

- 스프링 인터셉터는 디스 패터 서블릿과 컨트롤러 사이엥서 컨트롤러 호출 직접에 호출이 된다.
- 스프링 인터셉터에도 URL 패턴을 적용할 수 있는데, 서블릿 URL 패턴과는 다르고 매우 정밀하게 설정할 수 있다.

### 스프링 인터셉터 제한

HTTP 요청 -> WAS -> 필터 -> 서블릿 -> 스프링 인터셉터 -> 컨트롤러 //로그인 사용자
HTTP 요청 -> WAS -> 필터 -> 서블릿 -> 스프링 인터셉터(적절하지 않은 요청이라 판단하면 컨트롤러 호출 X)

인터세버에서 적절하지 않은 요청이라 판단하면 인터셉터에서 끝을 낼 수 있다.

### 스프링 인터셉터 체인

HTTP 요청 -> WAS -> 필터 -> 서블릿 -> 인터셉터1 -> 인터셉터2 -> 컨트롤러

스프링 인터셉터는 체인으로 구성되는데, 중간에 인터셉터를 자유롭게 추가할 수 있다.

### 스프링 인터셉터 인터페이스

스프링 인터셉터를 사용하려면 HandlerInterceptor 인터페이스를 구현하면 된다.

```java
public interface HandlerInterceptor {

    default boolean preHandle(HttpServletRequest request, 
                                HttpServletResponse response,
                                Object handler) throws Exception {}
    
    default void postHandle(HttpServletRequest request, 
                        HttpServletResponse response,           
                        Object handler, 
                        @Nullable ModelAndView modelAndView) throws Exception {}
    
    default void afterCompletion(HttpServletRequest request, 
                                    HttpServletResponse response, 
                                    Object handler, 
                                    @Nullable Exception ex) throws Exception {}
}
```

- 서블릿 필터의 경우 단순하게 `doFilter()` 하나만 제공된다. 인터셉터는 컨트롤러 호출 전(`preHandle`), 호출 후(`postHandle`), 요청 완료 이후(`afterCompletion`)와 같이 단계적으로 잘 세분화 되어 있다.
- 서블릿 필터의 경우 단순히 `request`, `response`만 제공했지만, 인터셉터는 어떤 컨트롤러(`handler`)가 호출되는지 호출 정보도 받을 수 있다. 그리고 어떤 `modelAndView`가 반환되는지 응답 정보도 받을 수 있다.

- preHandle : 핸들러 어댑터 호출 전에 호출된다.
  - preHandle 의 응답값이 true 이면 다음으로 진행하고, false 이면 더는 진행하지 않는다. false 인 경우 나머지 인터셉터는 물론이고, 핸들러 어댑터도 호출되지 않는다.
- postHandle : 핸들러 어댑터 호출 후에 호출된다.
- afterCompletion : 뷰가 렌더링 된 이후에 호출된다.

### 스프링 인터셉터 예외 상황

- preHandle : 컨트롤러 호출 전에 호출된다.
- postHandle : 컨트롤러에서 예외가 발생하면 postHandle 은 호출되지 않는다.
- afterCompletion : afterCompletion 은 항상 호출된다. 이 경우 예외를 파라미터로 받아서 어떤 예외가 발생했는지 로그로 출력할 수 있다.

> afterCompletion은 예외가 발생해도 호출된다.

### 서블릿 인터셉터 요청 로그

<details>
<summary>LogInterceptor</summary>
<div markdown="1">

```java
@Slf4j
public class LogInterceptor implements HandlerInterceptor {

    public static final String LOG_ID = "logId";

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        String requestURI = request.getRequestURI();
        String uuid = UUID.randomUUID().toString();
        //넘길 때 싱글톤으로 하면 절대 안된다.
        request.setAttribute(LOG_ID, uuid);

        //@RequestMapping : HandlerMethod
        //정적 리소스 : ResourceHttpRequestHandler
        if (handler instanceof HandlerMethod) {
            HandlerMethod handlerMethod = (HandlerMethod) handler;//호출한 컨트롤러 메서드의 모든 정보가 포함되어 있다.
        }

        log.info("REQUEST [{}][{}][{}]", uuid, requestURI, handler);
        return true;
    }

    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {
        log.info("postHandle [{}]", modelAndView);
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        String requestURI = request.getRequestURI();
        String uuid = (String)request.getAttribute(LOG_ID);
        log.info("REQUEST [{}][{}][{}]", uuid, requestURI, handler);
        if (ex != null) {
            log.error("afterCompletion error!!", ex);
            throw ex;
        }
    }
}
```

</div>
</details>

<details>
<summary>WebConfig</summary>
<div markdown="1">

```java
@Configuration
public class WebConfig  implements WebMvcConfigurer {
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new LogInterceptor())
                .order(1)
                .addPathPatterns("/**")
                .excludePathPatterns("/css/**", "/*.ico", "/error");
}
```

</div>
</details>

### 서블릿 인터셉터 인증 체크

<details>
<summary>LoginCheckInterceptor</summary>
<div markdown="1">

```java
@Slf4j
public class LoginCheckInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {

        String requestURI = request.getRequestURI();
        log.info("인증 체크 인터셉터 실행 {}", requestURI);

        HttpSession session = request.getSession();

        if (session == null || session.getAttribute(SessionConst.LOGIN_MEMBER) == null) {
            log.info("미인증 사용자 요청");
            response.sendRedirect("/login?redirectURL=" + requestURI);
            return false;
        }
        return true;
    }
}
```

</div>
</details>

<details>
<summary>WebConfig</summary>
<div markdown="1">

```java
@Configuration
public class WebConfig  implements WebMvcConfigurer {
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new LoginCheckInterceptor())
                .order(2)
                .addPathPatterns("/**")
                .excludePathPatterns("/", "members/add", "/login", "/logout",
                        "/css/**", "/*.ico", "/error");
    }
}
```

</div>
</details>

## ArgumentResolver 활용


<details>
<summary>@Login</summary>
<div markdown="1">

```java
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
public @interface Login {
}
```

</div>
</details>

<details>
<summary>LoginMemberArgumentResolver</summary>
<div markdown="1">


```java
@Slf4j
public class LoginMemberArgumentResolver implements HandlerMethodArgumentResolver {

    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        log.info("supportsParameter 실행");

        boolean hasParameterAnnotation = parameter.hasParameterAnnotation(Login.class);
        boolean hasMemberType = Member.class.isAssignableFrom(parameter.getParameterType());

        return hasParameterAnnotation && hasMemberType;
    }

    @Override
    public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer, NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {

        log.info("resolveArgument 실행");
        HttpServletRequest request = (HttpServletRequest) webRequest.getNativeRequest();
        HttpSession session = request.getSession(false);
        if (session == null) {
            return null;
        }
        return session.getAttribute(SessionConst.LOGIN_MEMBER);
    }
}
```

- `supportsParameter()` : @Login 애노테이션이 있으면서 Member 타입이면 해당 ArgumentResolver 가 사용된다.
- `resolveArgument()` : 컨트롤러 호출 직전에 호출 되어서 필요한 파라미터 정보를 생성해준다. 여기서는 세션에 있는 로그인 회원 정보인 member 객체를 찾아서 반환하고 스프링MVC는 컨트롤러의 메서드를 호출하면서 여기에서 반환된 member 객체를 파라미터에 전달해준다.


</div>
</details>

<details>
<summary>WebConfig - WebMvcConfigurer</summary>
<div markdown="1">

```java
@Configuration
public class WebConfig  implements WebMvcConfigurer {

    @Override
    public void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers) {
        resolvers.add(new LoginMemberArgumentResolver());
    }
}
```

</div>
</details>
