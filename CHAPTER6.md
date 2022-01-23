# 로그인 처리 - 쿠키, 세션

## 로그인 서비스

<details>
<summary>접기/펼치기 버튼</summary>
<div markdown="1">

**LoginService**
`lonin()` 메서드를 이용해 저장소에서 해당 아이디의 데이터를 찾은 다음, 패스워드가 일치하는지 확인했다.

```java
@Service
@RequiredArgsConstructor
public class LoginService {

    private final MemberRepository memberRepository;
    
    /**
    * @return null이면 로그인 실패
    */
    public Member login(String loginId, String password) {
        return memberRepository.findByLoginId(loginId)
            .filter(m -> m.getPassword().equals(password))
            .orElse(null);
    } 
}
```

**LoginController**
로그인 컨트롤러는 로그인 서비스를 호출해서 로그인에 성공하면 홈 화면으로 이동하고, 로그인에 실패하면 `bindingResult.reject()` 메서드를 이용해 글로벌 오류(ObjectError)를 생성한다.

```java
@Slf4j
@Controller
@RequiredArgsConstructor
public class LoginController {
    
    private final LoginService loginService;
    
    @GetMapping("/login")
    public String loginForm(@ModelAttribute("loginForm") LoginForm form) {
        return "login/loginForm";
    }
    
    @PostMapping("/login")
    public String login(@Valid @ModelAttribute LoginForm form, BindingResult bindingResult) {
        if (bindingResult.hasErrors()) {
            return "login/loginForm";
        }
        Member loginMember = loginService.login(form.getLoginId(), form.getPassword());
        log.info("login? {}", loginMember);
        if (loginMember == null) {
            bindingResult.reject("loginFail", "아이디 또는 비밀번호가 맞지 않습니다.");
            return "login/loginForm";
        }
    
        //로그인 성공 처리 TODO 
        
        return "redirect:/";
    } 
}
```

**loginForm**
로그인 뷰 템플릿에는 loginId와 password가 틀리면 글로벌 오류가 나타난다.

```html
<div class="container">
    <div class="py-5 text-center">
        <h2>로그인</h2> 
    </div>
    <form action="item.html" th:action th:object="${loginForm}" method="post">
        <div th:if="${#fields.hasGlobalErrors()}">
            <p class="field-error" th:each="err : ${#fields.globalErrors()}"
            th:text="${err}">전체 오류 메시지</p> 
        </div>
        <div>
            <label for="loginId">로그인 ID</label>
            <input type="text" id="loginId" th:field="*{loginId}" class="form-control" th:errorclass="field-error">
            <div class="field-error" th:errors="*{loginId}" />
        </div> 
        <div>
            <label for="password">비밀번호</label>
            <input type="password" id="password" th:field="*{password}" class="form-control" th:errorclass="field-error">
            <div class="field-error" th:errors="*{password}" />
        </div>
        <hr class="my-4">
        <div class="row">
                <div class="col">
                    <button class="w-100 btn btn-primary btn-lg" type="submit"> 
                    로그인
                    </button>
                </div>
                <div class="col">
                    <button class="w-100 btn btn-secondary btn-lg" onclick="location.href='items.html'" th:onclick="|location.href='@{/}'|" type="button">
                        취소
                    </button>
                </div>
        </div>
    </form>
</div> 
```

</div>
</details>

## 쿠키를 사용한 로그인 처리

로그인 상태 유지

서버에서 로그인에 성공하면 HTTP 응답에 쿠키를 담아 브라우저에 전달하면 브라우저는 앞으로 해당 쿠키를 지속해서 보내준다.

쿠키 종류

- 영속 쿠키 : 만료 날짜를 입력하면 해당 날짜가 유지
- 세션 쿠키 : 만료 날짜를 생략하면 브라우저 종료시 까지만 유지

쿠키 생성 로직

로그인에 성공하면 쿠키를 생성하게 하고 `HttpServletResponse에` 담는다. 쿠키 이름은 `cookieName`이고, 값은 회원의 id를 담아둔다. 해당 쿠키는 만료 날짜를 생략했기 때문에, 웹 브라우저 종료 전까지만 회원의 id를 서버에 계속 보내준다.

```java
Cookie idCookie = new Cookie("cookieNakme", String.valueOf(loginMember.getId()));
response.addCookie(idCookie);
```

### 로그인 기능

1. 로그인 쿠키가 없는 사용자는 기존 'home'으로 보낸다. 쿠키가 있어도 회원이 없으면 'home'으로 보낸다.
2. 로그인 쿠키가 있는 사용자는 로그인 사용자 전용 홈 화면인 loginHome으로 보낸다.
3. 추가로 홈 화면에 회원 관련 정보도 출력해야 하니 해당 사용자 데이터도 모델에 담아서 전달한다.

> `@CookieValue` 를 사용하면 편리하게 쿠키를 조회할 수 있고, 로그인 하지 않은 사용자도 홈에 접근할 수 있기 때문에 `required = false` 를 사용한다. true이면 없으면 생성, 있으면 사용지만, false는 없으면 생성하지 않는다.

```java
@GetMapping("/")
public String homeLogin( 
    @CookieValue(name = "memberId", required = false) Long memberId,
    Model model) {
    //1
    if (memberId == null) {
        return "home";
    }
    //2
    //로그인
    Member loginMember = memberRepository.findById(memberId); 
    if (loginMember == null) {
        return "home";
    }
    //3
    model.addAttribute("member", loginMember);
    return "loginHome";
}
```

### 로그아웃 기능

세션 쿠키의 경우 웹 브라우저가 종료될 때, 쿠키가 만료되고, 해당 쿠키의 종료 날짜를 0으로 지정했을 때에도 만료가 된다. 영속 쿠키의 경우 서버에서 해당 쿠키의 종료 날짜를 0으로 지정해야 만료가 된다.

```java
private void expireCookie(HttpServletResponse response, String cookieName) {
      Cookie cookie = new Cookie(cookieName, null);
      cookie.setMaxAge(0);
      response.addCookie(cookie);
}
```

## 쿠키와 보안 문제

쿠키를 사용하 로그인 ID를 전달해 로그인을 유지할 수 있지만, 임의로 변경 가능하는 등 심각한 보안 문제가 존재한다.

### 보안 문제

- 쿠카 값은 임의로 변경할 수 있다.
- 쿠키에 보관된 정보는 훔쳐갈 수 있다.
- 해커가 쿠키를 한번 훔쳐가면 평생 사용할 수 있다.

### 대안

- 쿠키에 중요한 값은 노출하지 않고, 사용자 별로 예측 불가능한 임의의 토큰(랜덤 값)을 노출하고, 서버에서 토큰과 사용사 id를 매핑해 인식한다. 그리고 서버에서 토큰을 관리한다.
- 토큰은 해커가 임의의 값을 넣어도 찾을 수 없도록 예상 불가능하도록 설정해야 한다.
- 해커가 토큰 값을 알아도 시간이 지나면 사용할 수 없도록 서버에서 해당 토큰의 만료시간을 짧게 유지해야 한다.
- 해킹이 의심되는 경우 서버에서 해당 토큰을 강제로 제거하면 된다.

## 세션 동작 방식

쿠키에 중요한 정보를 보관하는 방법은 여러가지 보안 이슈가 있다. 이 문제를 해결하려면 결국 중요한 정보를 모두 서버에 저장해야 하고, 클라이언트와 서버는 추정 불가능한 임의의 식별자 값으로 연결해야 한다.

> 이렇게 서버에 중요한 정보를 보관하고 연결을 유지하는 방법을 세션이라 한다.

### 세션 동작 방식

1. 로그인(클라이언트 -> 서버)
   1. 사용자가 ID, PW 정보를 전달하면 서버에서 해당 사용자가 맞는지 확인한다.
2. 세션 생성(서버)
   1. 사용자가 맞으면 추정 불가능한 세선 ID를 생성한다.(UUID)
   2. 생성된 세션 ID와 세션에 보관할 값(ex : 사용자 ID)을 서버의 세션 저장소에 보관한다.
3. 세션 ID를 응답 쿠키로 전달(서버 -> 클라이언트)
   1. 서버는 클라이언트에 해당 세션 이름(ex : mySession)으로 세션 ID만 쿠키에 담아 전달한다.
   2. 클라이언트는 쿠키 저장소에 해당 세션 이름(ex : mySession)으로 쿠키를 보관한다.
4. 클라이언트의 세션 ID 쿠키 전달(클라이언트 -> 서버)
   1. 클라이언트는 요청시 항상 해당 세션 이름(ex : mySession)으로 쿠키를 선달한다.
   2. 서버에서는 클라이언트가 전달한 세션(ex : mySession)의 쿠키 정보로 세션 저장소를 조회해 로그인시 보관한 세션 정보를 사용한다.

> 클라이언트와 서버는 결국 쿠키로 연결이 되고, 쿠키의 보안을 위해서 세션을 사용한다.

### 중요

- 여기서 중요한 포인트는 회원과 관련된 정보는 전혀 클라이언트에 전달하지 않는다는 것이다.
- 오직 추정 불가능한 세션 ID만 쿠키를 통해 클라이언트에 전달한다.

### 쿠키와 보안 정리

세션을 사용해 서버에 중요한 정보를 관리하게 되면서 보안 문제를 해결할 수 있다.

- 쿠키 값을 변조 가능
  - 예상 불가능한 세션 ID 사용
- 쿠키에 보관하는 정보는 클라이언트 해킹시 털릴 가능성이 있다.
  - 세션 ID를 해킹당해도 해당 쿠키에 중요한 정보 없음
- 쿠키 탈취 후 사용
  - 탈취해도 세션의 만료 시간을 짧게 유지해 시간이 지나면 사용할 수 없도록 설정
  - 해킹이 의심되는 경우 서버에서 해당 세션을 제거

## 서블릿 HTTP 세션 1

### HttpSession

서블릿이 제공하는 `HttpSession도` 결국 우리가 직접 만든 `SessionManager와` 같은 방식으로 동작한다. 서블릿을 통해 `HttpSession을` 생성하면 다음과 같은 쿠키를 생성한다. 쿠키 이름이 JSESSIONID이고, 값은 추정 불가능한 랜덤 값이다.

### 세션 생성과 조회

- `request.getSession(true)`
  - 세션이 있으면 기존 세션을 반환한다.
  - 세션이 없으면 새로운 세션을 생성해서 반환한다.
- `request.getSession(false)`
  - 세션이 있으면 기존 세션을 반환한다.
  - 세션이 없으면 새로운 세션을 생성하지 않는다. null 을 반환한다.
- `request.getSession()`
  - 신규 세션을 생성하는 `request.getSession(true)` 와 동일하다.

### `setAttribute()` 세션에 로그인 회원 정보 보관

`setAttribute()` 메서드를 이용해 하나의 세션에 여래 값을 저장할 수 있다.

> 'session.setAttribute(SessionConst.LOGIN_MEMBER, loginMember);'

## 서블릿 HTTP 세션 2

### @SessionAttribute

스프링은 세션을 더 편리하게 사용할 수 있도록 @SessionAttribute을 지원한다. 이미 로그인된 사용자를 찾을 때는 다음과 같이 사용하면 세션을 새로 생성하지 않고 로그인 된 사용자만 찾는다. 세션을 찾고, 세션에 들어있는 데이터를 찾는 번거러운 과정을 스프링이 한번에 편리하게 처리해준다.

`@SessionAttribute(name = "loginMember", required = false) Member loginMember`

### TrackingModes

로그인을 처음 시도하면 URL에 jsessioinid를 포함하고 있다.

`http://localhost:8080/;jsessionid=F59911518B921DF62D09F0DF8F83F872`

스프링이 URL에 남기는 이유는 쿠키를 지원하지 않는 웹 브라우저가 있을까봐 URL을 통해 jsessionid를 URL을 통해 인자값을 넘기는 방법이다. 지원하는지 최초에 확인하지 못하므로, 쿠키 값도 전달하고, URL에 jsessionid도 함께 전달한다.

이 방법을 사용하려면 URL에 계속 포함해서 전달해야 하는데 타임리프 같은 템플릿 엔진을 통해 링크를 걸면 jsessionid를 URL에 자동으로 포함해준다.

URL 전달 방식을 끄고 항상 쿠키를 통해서만 세션을 유지하고 싶으면 다음 옵션을 넣어주면 URL에 jsessionid이 노출되지 않는다.

application.properties

```properties
server.servlet.session.tracking-modes=cookie
```

## 세션 정보와 타임아웃 정리

### 세션 정보 확인

- `sessionId` : 세션Id, JSESSIONID 의 값이다. 예) 34B14F008AA3527C9F8ED620EFD7A4E1
- `maxInactiveInterval` : 세션의 유효 시간, 예) 1800초, (30분)
- `creationTime` : 세션 생성일시
- `lastAccessedTime` :세션과연결된사용자가최근에서버에접근한시간,클라이언트에서서버로 `sessionId( JSESSIONID )`를 요청한 경우에 갱신된다.
- `isNew` : 새로 생성된 세션인지, 아니면 이미 과거에 만들어졌고, 클라이언트에서 서버로  `sessionId( JSESSIONID )`를 요청해서 조회된 세션인지 여부

### 세션 타임 아웃 설정

세션은 사용자가 로그아웃을 직접 호출해서 `session.invalidate()`가 호출 되는 경우에 삭제대는데 대부분의 사용자는 로그아웃을 선택하지 않고, 그냥 웹 브라우저를 종료한다. 문제는 HTTP가 비 연결성(ConnectionLess)이므로 서버 입장에서는 해당 사용자가 웹 브라우저를 종료한 것인지 아닌지를 인식할 수 없어 서버에서 세션 데이터를 언제 삭제해야 하는지 판단하기가 어렵다.

이 경우 남아있는 세션을 무한정 보관하면 다음과 같은 문제가 발생한다.

- 해당 쿠키로 악의적인 요청을 할 수 있다.
- 세션에 메모리에 저장이되면서 메모리가 가득차는 상황이 발생한다.

### 세션의 종료 시점

세션의 종료 시점은 세션 생성 시점에서 유효 시간을 측정하는 것이 아니라 **사용자가 최근에 요청한 시간을 기준으로 측정하는 게 좋다.** 이렇게 하면 사용자가 서비스를 지속적으로 사용해도 계속해서 로그인을 시도할 필요가 없다. 그리고 `HttpSession`은 이 방식을 사용한다.

### 세션 타임아웃 설정

#### 스프링 부트를 이용한 글로벌 설정

application.properties

```properties
server.servlet.session.timeout=60s #-60초
#server.servlet.session.timeout=60m -60분
#server.servlet.session.timeout=60h -60시간
```

#### 특정 세션 단위로 시간 설정

```java
session.setMaxInactiveInterval(1800); //1800초
```

### 세션 타임아웃 발생

세션 타임아웃 시간은 해당 세션과 관련된 JSESSIONID을 전달하는 HTTP 요청이 있으면 현재 시간으로 다시 초기화 한다. 이렇게 초기화 하면 세션 타임아웃으로 설정한 시간동안 세션을 추가로 사용할 수 있다.

#### 최근 세션 접근 시간

`LastAccessedTime` 이후로 시간이 지나면 WAS가 내부에서 해당 세션을 제거한다.

```java
session.getLastAccessedTime()
```

### 정리

서블릿의 `HttpSession이` 제공하는 타임아웃 기능 덕분에 세션을 안전하고 편리하게 사용할 수 있지만 많은 양을 보관하게 된다면 메모리 사용량이 급격하게 늘어나 장애로 이어질 수 있다. 추가로 세션의 시간을 너무 길게 가져가면 메모리 사용이 계속 누적될 수 있으므로 적당한 시간을 선택하는 것이 필요하다.
