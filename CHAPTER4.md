# 검증

## 검증 사항

컨트롤러의 중요한 역할중 하나는 HTTP 요청이 정상인지 검증하는 것이다.

- 타입 검증
- 필드 검증
- 특정 필드의 범위를 넘어서는 검증

참고: 클라이언트 검증, 서버 검증

- 클라이언트 검증은 조작할 수 있으므로 보안에 취약하다.
- 서버만으로 검증하면, 즉각적인 고객 사용성이 부족해진다.
- 둘을 적절히 섞어서 사용하되, 최종적으로 서버 검증은 필수
- API 방식을 사용하면 API 스펙을 잘 정의해서 검증 오류를 API 응답 결과에 잘 남겨주어야 함

## 직접 처리 - 개발

Map을 이용해 error 저장
타임리프에서 확인할 때, ${errors?.containsKey('key')}

**Safe Navigation Operator**
SpringEL이 제공하는 문법

>만약 여기에서 errors 가 null 일때, 등록폼에 진입한 시점에는 errors 가 없다. 따라서 errors.containsKey() 를 호출하는 순간 NullPointerException 이 발생한다. errors?. 은 errors 가 null 일때 NullPointerException 이 발생하는 대신, null 을 반환하는 문법이다. th:if 에서 null 은 실패로 처리되므로 오류 메시지가 출력되지 않는다.

### 검증 로직

```java
@PostMapping("/add") public String addItem(@ModelAttribute Item item, RedirectAttributes redirectAttributes, Model model) {
    //검증 오류 결과를 보관
    Map<String, String> errors = new HashMap<>();
    if (!StringUtils.hasText(item.getItemName())) { 
        errors.put("itemName", "상품 이름은 필수입니다.");
    }
    ...

        //검증에 실패하면 다시 입력 폼으로 
    if (!errors.isEmpty()) {
        model.addAttribute("errors", errors);
        return "validation/v1/addForm";
    }
//성공 로직
    Item savedItem = itemRepository.save(item); 
    redirectAttributes.addAttribute("itemId", savedItem.getId()); 
    redirectAttributes.addAttribute("status", true);
    return "redirect:/validation/v1/items/{itemId}";
}
```

### 필드 오류 처리 - 클래스 추가

```html
<input type="text" th:classappend="${errors?.containsKey('itemName')} ? 'field-error' : _" class="form-control">
```

### 필드 오류 처리 - 메시지 처리

```html
<div class="field-error" th:if="${errors?.containsKey('itemName')}" th:text="${errors['itemName']}">오류 메시지가 나오는 필드
</div>
```

### 정리

- 만약 검증 오류가 발생하면 입력 폼을 다시 보여준다.
- 검증 오류들을 고객에게 친절하게 안내해서 다시 입력할 수 있게 한다. 
- 검증 오류가 발생해도 고객이 입력한 데이터가 유지된다.

### Map을 이용해 에러를 넘겨주는 방법의 문제점

- 뷰 템플릿에서 중복 처리가 많다. 뭔가 비슷하다.
- 타입 오류 처리가 안된다. Item 의 price , quantity 같은 숫자 필드는 타입이 Integer 이므로 문자 타입으로 설정하는 것이 불가능하다. 숫자 타입에 문자가 들어오면 오류가 발생한다. 그런데 이러한 오류는 스프링MVC에서 컨트롤러에 진입하기도 전에 예외가 발생하기 때문에, 컨트롤러가 호출되지도 않고, 400 예외가 발생하면서 오류 페이지를 띄워준다.
- Item의 price에 문자를 입력하는 것 처럼 타입 오류가 발생해도 고객이 입력한 문자를 화면에 남겨야 한다. 만약 컨트롤러가 호출된다고 가정해도 Item 의 price 는 Integer 이므로 문자를 보관할 수가 없다. 결국 문자는 바인딩이 불가능하므로 고객이 입력한 문자가 사라지게 되고, 고객은 본인이 어떤 내용을 입력해서 오류가 발생했는지 이해하기 어렵다.
- 결국 고객이 입력한 값도 어딘가에 별도로 관리가 되어야 한다.

## BindingResult

스프링이 제공하는 검증 오류를 보관하는 객체이다. 검증 오류가 발생하면 여기에 보관이 된다. BindingResult가 존재하면 @ModelAttribute 애너테이션이 붙은 데이터에 데이터 바인딩 시 오류가 발생해도 컨트롤러가 호출이 된다.

### @ModelAttribute 애너테이션이 붙은 데이터에 바인딩 시 타입 오류가 발생하는 상황

- BindingResult가 없으면 400 오류가 발생하면서 컨트롤러가 호출되지 않고, 오류 페이지로 이동한다.
- BindingResult가 있으면 오류 정보(FieldError)를 BindingResult에 담아 컨트롤러를 정상 호출한다.

### BindingReuslt에 검증 오류를 적용하는 3가지 방법

- 타입 오류 등으로 바인딩이 실패하는 경우 스프링이 FieldError를 생성해 BindingResult에 넣어주는 방법
- 개발자가 집접 넣어주는 방법
- Validator 사용하는 방법

### **주의**

- BindingResult 는 검증할 대상 바로 다음에 와야한다. 순서가 중요하다. 예를 들어서 @ModelAttribute Item item , 바로 다음에 BindingResult 가 와야 한다.
- BindingResult 는 Model에 자동으로 포함된다.

### BindingResult와 Errors

- org.springframework.validation.Errors
- org.springframework.validation.BindingResult

BindingResult는 인터페이스고 Errors 인터페이스를 상속받고 있다. 실제 넘어오는 구현체는 BeanPropertyBindingResult라는 것인데, 둘다 구현하고 있으므로 BindingResult 대신에 Errors를 사용해도 된다. Errors 인터페이스는 단순한 오류 저장과 조회 기능을 제공한다. BindingResult는 여기에 더해 추가적인 기능을 제공한다. addError()도 BindingResult가 제공하므로 여기서는 BindingResult를 사용하면된다.

### BindingResult 추가

BindingResult bindingResult 파라미터의 위치는 @ModelAttribute Item item 다음에 와야 한다.

```java
@PostMapping("/add")
public String addItemV1(@ModelAttribute Item item, BindingResult bindingResult, RedirectAttributes redirectAttributes) {
    if (!StringUtils.hasText(item.getItemName())) { bindingResult.addError(new FieldError("item", "itemName", "상품 이름은 필수입니다.")); }

    ...

    if (bindingResult.hasErrors()) {
        return "validation/v2/addForm";
    }
    ...
}
```

### 타임리프에서 오류 처리

```html
<div>
    <label for="itemName" th:text="#{label.item.itemName}">상품명</label>
    <input type="text" id="itemName" th:field="*{itemName}" th:errorclass="field-error" class="form-control" placeholder="이름을 입력하세요">
    <div class="field-error" th:errors="*{itemName}">
        상품명 오류
    </div>
</div>
```

### 타임리프 스프링 검증 오류 통합 기능

타임리프는 스프링의 BindingResult 를 활용해서 편리하게 검증 오류를 표현하는 기능을 제공한다.

- #fields : #fields 로 BindingResult 가 제공하는 검증 오류에 접근할 수 있다.
- th:errors : 해당 필드에 오류가 있는 경우에 태그를 출력한다. th:if 의 편의 버전이다.
- th:errorclass : th:field 에서 지정한 필드에 오류가 있으면 class 정보를 추가한다.

## FieldError, ObjectError

### FieldError 생성자 요약

필드에 오류가 있으면 FieldError 객체를 생성해서 bindingResult 에 담아두면 된다.

```java
public FieldError(String objectName, String field, String defaultMessage) {}
```

### FieldError 생성자

```java
public FieldError(String objectName, String field, String defaultMessage) 
public FieldError(String objectName, String field, @Nullable Object rejectedValue, boolean bindingFailure, @Nullable String[] codes, @Nullable Object[] arguments, @Nullable String defaultMessage)
```

- objectName : 오류가 발생한 객체 이름
- field : 오류 필드
- rejectedValue : 사용자가 입력한 값(거절된 값)
- bindingFailure : 타입 오류 같은 바인딩 실패인지, 검증 실패인지 구분 값
- codes : 메시지 코드
- arguments : 메시지에서 사용하는 인자
- defaultMessage : 기본 오류 메시지

### ObjectError 생성자 요약

특정 필드를 넘어서는 오류가 있으면 ObjectError 객체를 생성해서 bindingResult 에 담아두면 된다.

```java
public ObjectError(String objectName, String defaultMessage) {}
```

- objectName : @ModelAttribute 의 이름
- defaultMessage : 오류 기본 메시지

```java
public ObjectError(String objectName, String defaultMessage)
public ObjectError(String objectName, @Nullable String[] codes, @Nullable Object[] arguments, @Nullable String defaultMessage)
```

- objectName : 오류가 발생한 객체 이름
- codes : 메시지 코드
- arguments : 메시지에서 사용하는 인자
- defaultMessage : 기본 오류 메시지

### 오류로 입력된 값 유지 - rejectedValue

사용자의 입력 데이터가 컨트롤러의 @ModelAttribute 에 바인딩되는 시점에 오류가 발생하면 모델 객체에 사용자 입력 값을 유지하기 어렵다. 예를 들어서 가격에 숫자가 아닌 문자가 입력된다면 가격은 Integer 타입이므로 문자를 보관할 수 있는 방법이 없다.

그래서 rejectedValue 값을 이용해 오류가 발생한 경우 사용자 입력 값을 보관하고, 보관한 사용자 입력 값을 검증 오류 발생시 화면에 다시 출력하면 된다.

### 타임리프의 사용자 입력 값 유지

타임리프의 th:field 는 매우 똑똑하게 동작하는데, 정상 상황에는 모델 객체의 값을 사용하지만, 오류가
발생하면 FieldError 에서 보관한 rejectedValue 값을 사용해서 값을 출력한다.(따로 설정할 필요 없이 th:field="*{objectName}" 코드를 사용하면 타임리프가 자동으로 다 설정해준다.)

## 오류 코드와 메시지 처리

FieldError , ObjectError 의 생성자는 errorCode , arguments 를 제공한다. 이것은 오류 발생시 오류 코드로 메시지를 찾기 위해 사용된다.

errors 메시지 파일 생성

```properties
required.item.itemName=상품 이름은 필수입니다. 
range.item.price=가격은 {0} ~ {1} 까지 허용합니다. 
max.item.quantity=수량은 최대 {0} 까지 허용합니다. 
totalPriceMin=가격 * 수량의 합은 {0}원 이상이어야 합니다. 현재 값 = {1}
```

스프링 부트 메시지 설정 추가 - application.properties

```properties
spring.messages.basename=messages,errors
```

```java
//required.item.itemName=상품 이름은 필수입니다. 
if (!StringUtils.hasText(item.getItemName())) {  
    bindingResult.addError(new FieldError("item", "itemName",
    item.getItemName(), false, new String[]{"required.item.itemName"}, null, null));
}

//range.item.price=가격은 {0} ~ {1} 까지 허용합니다. 
if (item.getPrice() == null || item.getPrice() < 1000 || item.getPrice() > 1000000) {
    bindingResult.addError(new FieldError("item", "price", item.getPrice(), false, new String[]{"range.item.price"}, new Object[]{1000, 1000000}, null));
}
```

- codes : required.item.itemName 를 사용해서 메시지 코드를 지정한다. 메시지 코드는 하나가 아니라 배열로 여러 값을 전달할 수 있는데, 순서대로 매칭해서 처음 매칭되는 메시지가 사용된다.
- arguments : Object[]{1000, 1000000} 를 사용해서 코드의 {0} , {1} 로 치환할 값을 전달한다.

## rejectValue() , reject()를 이용한 코드 간소화

BindingResult 가 제공하는 rejectValue(), reject() 를 사용하면 FieldError, ObjectError 를 직접 생성하지 않고, 깔끔하게 검증 오류를 다룰 수 있다.

### rejectValue() - 필드가 있는 경우

```java
  void rejectValue(@Nullable String field, String errorCode, @Nullable Object[] errorArgs, @Nullable String defaultMessage);
```

- field : 오류 필드명
- errorCode : 오류 코드
- errorArgs : 오류 메시지에서 {0} 을 치환하기 위한 값
- defaultMessage : 오류 메시지를 찾을 수 없을 때 사용하는 기본 메시지

### reject() - 필드가 없는 경우 (전체, 공용)

```java
 void reject(String errorCode, @Nullable Object[] errorArgs, @Nullable String
  defaultMessage);
```

- errorCode : 오류 코드
- errorArgs : 오류 메시지에서 {0} 을 치환하기 위한 값
- defaultMessage : 오류 메시지를 찾을 수 없을 때 사용하는 기본 메시지

정리

1. rejectValue() 호출
2. MessageCodesResolver 를 사용해서 검증 오류 코드로 메시지 코드들을 생성
3. new FieldError() 를 생성하면서 메시지 코드들을 보관
4. th:erros 에서 메시지 코드들로 메시지를 순서대로 메시지에서 찾고, 노출

### 축약된 오류 코드

FieldError() 를 직접 다룰 때는 오류 코드(errorCode)를 모두 입력해야 하지만, rejectValue()는 간단하게 입력할 수 있다.

## 메시지 축약과 레벨 나누기

단순하게 만들면 범용성이 좋아서 여러곳에서 사용할 수 있지만, 메시지를 세밀하게 작성하기 어렵다. 반대로 너무 자세하게 만들면 범용성이 떨어진다. 가장 좋은 방법은 범용성으로 사용하다가, 세밀하게 작성해야 하는 경우에는 세밀한 내용이 적용되도록 메시지에 단계를 두는 방법이다.

## MessageCodesResolver

- 검증 오류 코드로 메시지 코드들을 생성한다.
- MessageCodesResolver 인터페이스이고 DefaultMessageCodesResolver 는 기본 구현체이다.
- 주로 다음과 함께 사용 ObjectError , FieldError

### DefaultMessageCodesResolver의 기본 메시지 생성 규칙

#### 객체 오류

객체 오류의 경우 다음 순서로 2가지 생성

1. code + "." + object name 2.: code
    예) 오류 코드: required, object name: item 1.: required.item
2. required

#### 필드 오류

필드 오류의 경우 다음 순서로4가지 메시지 코드 생성

1. code + "." + object name + "." + field
2. code + "." + field
3. code + "." + field type
4. code

#### 타입 오류

오류 코드: typeMismatch, object name "user", field "age", field type: int

1. "typeMismatch.user.age"
2. "typeMismatch.age"
3. "typeMismatch.int"
4. "typeMismatch"

### 동작 방식

- rejectValue() , reject() 는 내부에서 MessageCodesResolver 를 사용한다. 여기에서 메시지 코드들을 생성한다.
- FieldError , ObjectError 의 생성자를 보면, 오류 코드를 하나가 아니라 여러 오류 코드를 가질 수 있다.
- MessageCodesResolver 를 통해서 생성된 순서대로 오류 코드를 보관한다.

### FieldError rejectValue("itemName", "required")

필드 오류의 경우 다음 순서로 4가지 메시지 코드 생성

- required.item.itemName
- required.itemName
- required.java.lang.String
- required

### ObjectError reject("totalPriceMin")

다음 2가지 오류 코드를 자동으로 생성

- totalPriceMin.item
- totalPriceMin

### 오류 메시지 출력

타임리프 화면을 렌더링 할 때 th:errors 가 실행된다. 만약 이때 오류가 있다면 생성된 오류 메시지 코드를 순서대로 돌아가면서 메시지를 찾는다. 그리고 없으면 디폴트 메시지를 출력한다.

## 오류 코드 관리 전략

핵심은 구체적인 것에서 덜 구체적인 것으로 설계한다. 이렇게 하면 메시지와 관련된 공통 전략을 편리하게 도입할 수 있다.

### 복잡하게 사용하는 이유

범용성있는 메시지는 requried같은 간단한 메시지로 끝내고 중요한 메시지는 requried.item.itemName처럼 구체적으로 사용하면 더 쉽게 관리할 수 있다.

```properties
#Level3 타입 오류
required.java.lang.String = 필수 문자입니다. 
required.java.lang.Integer = 필수 숫자입니다. 

#Level3
min.java.lang.String = {0} 이상의 문자를 입력해주세요. 
min.java.lang.Integer = {0} 이상의 숫자를 입력해주세요. 
range.java.lang.String = {0} ~ {1} 까지의 문자를 입력해주세요. 
range.java.lang.Integer = {0} ~ {1} 까지의 숫자를 입력해주세요. 
max.java.lang.String = {0} 까지의 문자를 허용합니다. 
max.java.lang.Integer = {0} 까지의 숫자를 허용합니다.

#Level4
required = 필수 값 입니다.
min= {0} 이상이어야 합니다.
range= {0} ~ {1} 범위를 허용합니다. max= {0} 까지 허용합니다.
```

범용성에 따라 레벨을 나눈 경우이고, 타입 오류가 나왔을 때에도 나오는 오류도 관리했다. 이렇게 메시지 코드를 기반으로 순서대로 MessageSource에서 메시지를 찾는다.

## validationUtils

### 사용 전

```java
if (!StringUtils.hasText(item.getItemName())) { 
    bindingResult.rejectValue("itemName", "required", "기본: 상품 이름은 필수입니다."); 
}
```

### 사용 후

다음과 같이 한줄로 가능, 제공하는 기능은 Empty , 공백 같은 단순한 기능만 제공

```java
ValidationUtils.rejectIfEmptyOrWhitespace(bindingResult, "itemName", "required");
```

## 스프링이 직접 만든 오류 처리

### 검증 오류는 두가지로 나눌 수 있다.

- 개발자가 직접 설정한 오류 코드 -> rejectValue() 직접 호출
- 스프링이 직접 검증 오류에 추가한 경우 **(주로 타입 정보가 맞지 않는다.)**

### typeMismatch

숫자가 들어가야하는 필드에 문자를 넣을 때 나오는 메시지는 typeMismatch.\*이다. 이 오류들을 개발자가 직접 설정하지 않으면 th:errors에서 메시지 코드들로 메시지를 순서대로 메시지에서 찾고, 노출할 때, 스프링이 생성한 기본 메시지가 출력이 된다.

```
Failed to convert property value of type java.lang.String to required type
java.lang.Integer for property price; nested exception is
java.lang.NumberFormatException: For input string: "A"
```

### errors.properties에 직접 추가

```properties
typeMismatch=타입 오류입니다.
```

프로퍼티를 이용해 설정하면 소스코드를 건들지 않고 메시지를 설정할 수 있다.

## Validator 분리

검증 로직을 분리하면 로직을 재사용할 수 있고, 관리 차원에서도 월등하다.

### 직접 분리해서 사용 - 검증과 관련된 부분 분리

스프링은 검증을 체계적으로 제공하기 위해 다음 인터페이스를 제공한다.

```java
public interface Validator {
    boolean supports(Class<?> clazz);
    void validate(Object target, Errors errors);
}
```

- supports() {} : 해당 검증기를 지원하는 여부 확인(뒤에서 설명) 
- validate(Object target, Errors errors) : 검증 대상 객체와 BindingResult

**ItemValidator**

```java
@Slf4j
@Component
public class ItemValidator implements Validator {
    @Override
    public boolean supports(Class<?> clazz) {
        return Item.class.isAssignableFrom(clazz);
        //item == clazz
        //item == subItem (자식까지 커버가 된다.)
    }

    @Override
    public void validate(Object target, Errors errors) {
        Item item = (Item) target;

        if (!StringUtils.hasText(item.getItemName())) {
            errors.rejectValue("itemName", "required");
        }

        if (item.getPrice() == null || item.getPrice() < 1000 || item.getPrice() > 1000000) {
            errors.rejectValue("price", "range", new Object[]{1000, 1000000}, null);
        }

        if (item.getQuantity() == null || item.getQuantity() >= 9999) {
            errors.rejectValue("quantity", "max", new Object[]{9999}, null);
        }

        //특정 필드가 아닌 복합 룰 검증
        if (item.getPrice() != null && item.getQuantity() != null) {
            int resultPrice = item.getPrice() * item.getQuantity();
            if (resultPrice < 10000) {
                errors.reject("totalPriceMin", new Object[]{10000, resultPrice}, null);
            }
        }

    }
}
```

```java
private final ItemValidator itemValidator;
@PostMapping("/add")
public String addItemV5(@ModelAttribute Item item, BindingResult bindingResult, RedirectAttributes redirectAttributes) {
    itemValidator.validate(item, bindingResult);
    if (bindingResult.hasErrors()) {
        log.info("errors={}", bindingResult);
        return "validation/v2/addForm";
    }
    //성공 로직
    Item savedItem = itemRepository.save(item); redirectAttributes.addAttribute("itemId", savedItem.getId()); redirectAttributes.addAttribute("status", true);
    return "redirect:/validation/v2/items/{itemId}";
}
```

### 스프링이 제공하는 데이터 바인더

#### 해당 클래스에만

**WebDataBinder을 통한 파라미터 바인딩과 검증 기능**

WebDataBinder는 스프링의 파라미터 반인딩의 역할을 해주고 검증 기능도 내부에 포함한다.

```java
@InitBinder
public void init(WebDataBinder dataBinder) {
    log.info("init binder {}", dataBinder);
    dataBinder.addValidators(itemValidator);
}
```

이렇게 WebDataBinder 에 검증기를 추가하면 해당 컨트롤러에서는 검증기를 자동으로 적용할 수 있다. @InitBinder 해당 컨트롤러에만 영향을 준다.

**@Validated 적용해 검증 대상 추가**

```java
PostMapping("/add")
  public String addItemV6(@Validated @ModelAttribute Item item, BindingResult
  bindingResult, RedirectAttributes redirectAttributes) {
    if (bindingResult.hasErrors()) {
        log.info("errors={}", bindingResult);
        return "validation/v2/addForm";
    }
    //성공 로직
    Item savedItem = itemRepository.save(item);
    redirectAttributes.addAttribute("itemId", savedItem.getId()); 
    redirectAttributes.addAttribute("status", true);
    return "redirect:/validation/v2/items/{itemId}";
}
```

**동작 방식**

@Validated 는 검증기를 실행하라는 애노테이션이다. 이 애노테이션이 붙으면 앞서 WebDataBinder 에 등록한 검증기를 찾아서 실행한다. 그런데 여러 검증기를 등록한다면 그 중에 어떤 검증기가 실행되어야 할지 구분이 필요하다. 이때 supports() 가 사용된다.

#### 글로벌 설정

존 컨트롤러의 @InitBinder 를 제거해도 글로벌 설정으로 정상 동작한다.

```java
@SpringBootApplication
  public class ItemServiceApplication implements WebMvcConfigurer {
    public static void main(String[] args) {
        SpringApplication.run(ItemServiceApplication.class, args);
    }
    @Override
    public Validator getValidator() {
        return new ItemValidator();
    }
}
```

> 글로벌 설정을 하면 다음에 설명할 BeanValidator가 자동 등록되지 않는다.

**검증시 @Validated @Valid 둘다 사용가능하다.**

@Valid 는 자바 표준 검증 애노테이션

- javax.validation.@Valid

@Validated 는 스프링 전용 검증 애노테이션

- org.springframework.boot:spring-boot-starter-validation