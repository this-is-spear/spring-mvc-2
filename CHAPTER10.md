# 스프링 타입 컨버터

## 스프링 타입 컨버터 소개

문자를 숫자로 변환하거나, 반대로 숫자를 문자로 변환해야 하는 것처럼 애플리케이션을 개발하다 보면 타입을 변환해야 하는 경우가 상당히 많다.

스프링이 제공하는 @RequestParam을 사용할 때, Integer 타입의 데이터를 받으면 스프링이 문자가 숫자로 자동으로 타입을 변환해준다.

스프링 타입 변환 적용 예

- 스프링 MVC 요청 파라미터
  - @RequestParam, @ModelAttribute, @PathVariable 
- @Value 등으로 YML 정보 읽을 때
- XML에 넣은 스프링 빈 정보를 변환할 때
- 뷰를 렌더링할 때

스프링과 타입 변환

스프링은 간단한 타입은 변환을 해주고, 개발자가 새로운 타입을 만들어 변환할 수 있도록 확장 가능한 컨버터 인터페이스를 제공한다.

Converter 인터페이스

```java
public interface Converter<S, T> {
      T convert(S source);
}
```

> 과거에는 `PropertyEditor` 을 이용해 타입을 변환했지만, 동시성 문제가 존재해 탕입을 변환할 때 마다 객체를 계속 생성해야하는 단점이 있다.

## 타입 컨버터 - Converter

타입 컨버터를 사용하려면 `org.springframework.core.convert.converter.Converter` 인터페이스를 구현하면 된다.

스프링은 용도에 따라 다양한 방식의 타입 컨버터를 제공한다.

컨버터 종류

- Converter : 기본 타입 컨버터
- ConverterFactory : 전체 클래스 계층 구조가 필요할 때
- GenericConverter : 정교한 구현, 대상 필드의 애너테이션 정보 사용 가능
- ConditionalGenericConverter : 특정 조건이 참인 경우에만 실행

> 스프링은 문자, 숫자, 불린, Enum등 일반적인 타입에 대한 대부분의 컨버터를 기본으로 제공한다. IDE에서 Converter, ConverterFactory, GenericConverter의 구현체를 찾아보면 수 많은 컨버터를 확인할 수 있다.

## 컨버전 서비스 _ ConversionService

컨버터를 하나하나 타입 변환에 사용하는 것은 매우 불편하기 때문에 스프링은 개별 컨버터를 모아두고 그것을 묶어서 편리하게 사용할 수 있는 기능을 제공한다.

ConversionService 인터페이스

```java
public interface ConversionService {

    boolean canConvert(@Nullable Class<?> sourceType, Class<?> targetType);

    boolean canConvert(@Nullable TypeDescriptor sourceType, TypeDescriptor
    targetType);

    <T> T convert(@Nullable Object source, Class<T> targetType);

    Object convert(@Nullable Object source, @Nullable TypeDescriptor sourceType, TypeDescriptor targetType);
}
```

등록과 사용 분리

컨버터를 등록할 때는 StringToIntegerConverter 같은 타입 컨버터를 명확하게 알아야 한다. 반면에 컨버터를 사용하는 입장에서는 타입 컨버터를 전혀 몰라도 된다. 타입 컨버터들은 모두 컨버전 서비스 내부에 숨어서 제공된다. 따라서 타입을 변환을 원하는 사용자는 컨버전 서비스 인터페이스에만 의존하면 된다. 물론 컨버전 서비스를 등록하는 부분과 사용하는 부분을 분리하고 의존관계 주입을 사용해야 한다.

> 컨버터를 추가하면 추가한 컨버터가 기본 컨버터 보다 높은 우선순위를 가진다.

인터페이스 분리 원칙 - ISP(Interface Segregation Principal)

인터페이스 분리 원칙은 클라이언트가 자신이 이용하지 않는 메서드에 의존하지 않아야 한다.

DefaultConversionService 는 다음 두 인터페이스를 구현했다.

- ConversionService : 컨버터 사용에 초점
- ConverterRegistry : 컨버터 등록에 초점

> 이렇게 인터페이스를 분리하면 컨버터를 사용하는 클라이언트와 컨버터를 등록하고 관리하는 클라이언트의 관심사를 명확하게 분리할 수 있다. 특히 컨버터를 사용하는 클라이언트는 ConversionService 만 의존하면 되므로, 컨버터를 어떻게 등록하고 관리하는지는 전혀 몰라도 된다. 결과적으로 컨버터를 사용하는 클라이언트는 꼭 필요한 메서드만 알게된다. 이렇게 인터페이스를 분리하는 것을 ISP 라 한다.

스프링은 내부에서 ConversionService 를 사용해서 타입을 변환한다. 예를 들어서 앞서 살펴본 @RequestParam 같은 곳에서 이 기능을 사용해서 타입을 변환한다. 이제 컨버전 서비스를 스프링에 적용해보자.

### 뷰 템플릿에 컨버전 적용하기

타임리프는 ${{...}} 를 사용하면 자동으로 컨버전 서비스를 사용해서 변환된 결과를 출력해준다. 물론 스프링과 통합 되어서 스프링이 제공하는 컨버전 서비스를 사용하므로, 우리가 등록한 컨버터들을 사용할 수 있다.

- 변수 표현식 : ${...}
- 컨버전 서비스 적용 : ${{...}}

> 폼 타입에 컨버전 적용할 때, 타임리프의 th:field 는 앞서 설명했듯이 id , name 를 출력하는 등 다양한 기능이 있는데, 여기에 컨버전 서비스도 함께 적용된다.

## 포맷터 - Fomatter

컨버터는 입력과 출력 타입에 제한이 없는 범용 타입 변환 기능을 제공하지만 대부분 타입 변환은 다른 타입을 문자로 변환하거나 문자를 다른 타입을 변환하는 일이 대부분이다. 그래서 객체를 특정한 포맷에 맞춰 문자를 출력하거나 그 반대의 역할을 하는 것에 특화된 기능이 바로 포맷터이다. 포맷터는 컨버터의 특별한 버전이다.

Converter vs Formatter

Converter는 범용이고, 모든 객체에서 모든 객체의 변환이 가능하다. Formatter는 문자에 특화된 기능이다. 객체에서 문자나 문자에서 객체 변환이 가능하고 거기에 현지화(Locale) 기능도 존재한다.

Locale

포맷터는 날짜 숫자의 표현 방법은 Locale인 현지화 정보가 사용될 수 있다.

Formatter 인터페이스

```java
public interface Printer<T> {
String print(T object, Locale locale);
}

public interface Parser<T> {
T parse(String text, Locale locale) throws ParseException;
}

public interface Formatter<T> extends Printer<T>, Parser<T> {
}
```

- parse() : 문자를 숫자로 변환한다.
- print() : 객체를 문자로 변환한다.

참고

스프링 용도에 따라 다양한 방식의 포맷터를 제공한다.

- Formatter : 포맷터
- AnnotationFormatterFactory : 필드의 타입이나 애너테이션 정보를 활용할 수 있는 포맷터

## 포맷터를 지원하는 컨버전 서비스

컨버전 서비스에는 컨버터만 등록할 수 있고, 포맷터를 등록할 수 는 없다. 그래서 포맷터를 지원하는 컨버전 서비스를 사용하면 컨버전 서브스에 포맷터를 추가할 수 있다. 내부에서 어댑터 패턴을 사용해 포맷터가 컨버터처럼 동작하도록 지원한다.

> 우선순위는 컨버터가 우선하므로 포맷터가 적용되지 않고, 컨버터가 적용된다.

**`FormattingConversionService`는 포맷터를 지원하는 컨버전 서비스다.** `DefaultFormattingConversionService`는 `FormattingConversionService`에 기본적인 통화, 숫자 관련 몇가지 기본 포맷터를 추가해서 제공한다.

`DefaultFormattingConversionService` 상속 관계

`FormattingConversionService`는 `ConversionService` 관련 기능을 상속받기 때문에 결과적으로 컨버터도 포맷터도 모두 등록할 수 있고, 사용할 때는 `ConversionService`가 제공하는 `convert`를 사용하면 된다.

> 스프링 부트는 `DefaultFormattingConversionService`를 상속 받은 `WebConversionService`를 내부에서 사용한다.

## 스프링이 제공하는 기본 포맷터

포맷터는 기본 형식이 지정되어 있기 때문에 객체의 각 필드마다 다른 형식으로 포맷을 지정하기 어렵다. 그래서 애너테이션을 이용해 원하는 형식을 지정해서 사용할 수 있는 포맷터 두 가지 기능을 제공한다.

- @NumberFormat : 숫자 관련 형식 지정 포맷터 사용
  - `NumberFormatAnnotationFormatterFactory`
- @DataTimeFormat : 날짜 관련 형식 지정 포맷터 사용
  - `Jsr310DateTimeFormatAnnotationFormatterFactory`

> 컨버터를 사용하든, 포맷터를 사용하든 등록 방법은 다르지만, 사용할 때는 컨버전 서비스를 통해서 일관성 있게 사용할 수 있다.

주의

**메시지 컨버터( HttpMessageConverter )에는 컨버전 서비스가 적용되지 않는다.**

HttpMessageConverter 의 역할은 HTTP 메시지 바디의 내용을 객체로 변환하거나 객체를 HTTP 메시지 바디에 입력하고, Jackson 같은 라이브러리를 사용하기 때문에 객체를 JSON으로 변환한다면 그 결과는 이 라이브러리에 따른 결과가 나온다. 그렇기 때문에 **JSON 결과로 만들어지는 숫자나 날짜 포맷을 변경하고 싶다면 해당 라이브러리가 제공하는 설정을 통해 포맷을 지정해야 한다.**
