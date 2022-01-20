# 메시지와 메시지 국제화

## 메시지

> 다양한 메시지를 한 곳에 관리하도록 하는 기능을 메시지 기능

여러 화면에 보이는 단어들을 변경하려면 모든 파일의 단어들을 고쳐야하지만 메시지 기능을 이용하면 한 곳에 관리해 보다 쉽게 변경하고 관리할 수 있도록 도와준다.

## 국제화

> 메시지 기능을 각 나라별로 별도로 관리할 수 있도록 국제화 지원

한국에서 접근한 것인지 영어에서 접근한 것인지는 인식하는 방법

1. HTTP accept-language 해더 값
2. 사용하거나 사용자가 직접 언어를 직접 선택
3. 쿠키 등을 사용해서 처리

> 스프링은 기본적인 메시지와 국제화 기능 모두 제공하고, 타임리프도 스프링이 제공하는 메시지와 국제화 기능을 편리하게 통합해서 제공한다.

## 스프링 메시지 소스 설정

### 직접 등록

스프링이 제공하는 MessageSource 를 스프링 빈으로 등록
MessageSource 는 인터페이고, 구현체인 ResourceBundleMessageSource 를 스프링 빈으로 등록하면 된다.

```java
@Configuration
public class WebConfig {
    @Bean
    public MessageSource messageSource() {
        ResourceBundleMessageSource messageSource = new ResourceBundleMessageSource();
        messageSource.setBasenames("register-message");
        messageSource.setDefaultEncoding("utf-8");
        return messageSource;
    }
}
```

- basenames : 설정 파일의 이름을 지정한다.
   - register-message 로 지정하면 register-message.properties 파일을 읽어서 사용한다.
   - 추가로 국제화 기능을 적용하려면 register-message_en.properties , register-message_ko.properties 와 같이 파일명 마지막에 언어 정보를 주면된다. 만약 찾을 수 있는 국제화 파일이 없으면 register-message.properties (언어정보가 없는 파일명)를 기본으로 사용한다.
   - 파일의 위치는 /resources/register-message.properties 에 두면 된다.
   - 여러 파일을 한번에 지정할 수 있다.
- defaultEncoding : 인코딩 정보를 지정한다. utf-8 을 사용하면 된다.

### 스프링 부트 메시지 소스 설정

스프링 부트를 사용하면 스프링 부트가 MessageSource를 자동으로 스프링빈으로 등록한다.

#### 스프링 부트 메시지 소스 기본 값

application.proposerties

```properties
spring.messages.basename=messages
```

### 수동으로 등록하기

- 방법 구상

직접 등록으로 MessageSource 를 스프링 빈으로 등록하지 않고, 스프링 부트와 관련된 별도의 설정을 하지 않으면 messages 라는 이름으로 기본 등록된다. 따라서 messages_en.properties , messages_ko.properties , messages.properties 파일만 등록하면 자동으로 인식된다.

## 스프링 메시지 소스 사용

**MessageSource 인터페이스**
MessgaeSource 인터페이스를 보면 코드를 포함한 일부 파라미터로 메시지를 읽어오는 기능을 제공한다.

```java
public interface MessageSource {

    String getMessage(String code, @Nullable Object[] args, @Nullable String defaultMessage, Locale locale);

    String getMessage(String code, @Nullable Object[] args, Locale locale) throws NoSuchMessageException;
```

## LocaleResolver

스프링은 Locale 선택 방식을 변경할 수 있도록 LocaleResolver 라는 인터페이스를 제공하는데, 스프링 부트는 기본으로 Accept-Language 를 활용하는 AcceptHeaderLocaleResolver 를 사용한다.

**LocaleResolver 인터페이스**
Locale 선택 방식을 변경하려면 해당 메서드의 구현체를 변경해서 쿠키나 세션 기반의 Locale 선택 기능을 사용할 수 있다.

```java
public interface LocaleResolver {
    Locale resolveLocale(HttpServletRequest request);
    void setLocale(HttpServletRequest request, @Nullable HttpServletResponse  response, @Nullable Locale locale);
}
```
