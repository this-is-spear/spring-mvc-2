# Bean Validation

## Bean Validation 소개

Bean Validation 이란?
먼저 Bean Validation은 특정한 구현체가 아니라 Bean Validation 2.0(JSR-380)이라는 기술 표준이다. 쉽게 이야기해서 검증 애노테이션과 여러 인터페이스의 모음이다. Bean Validation을 구현한 기술중에 일반적으로 사용하는 구현체는 하이버네이트 Validator이다.

## Bean Validation 시작

### 의존관계 추가

build.gradle

```groovy
implementation 'org.springframework.boot:spring-boot-starter-validation'
```

spring-boot-starter-validation 의존관계를 추가하면 라이브러리가 추가 된다.

### Jakarta Bean Validation

jakarta.validation-api : Bean Validation 인터페이스 hibernate-validator : Bean Validation 구현체

## Bean Validation 적용


### 검증 애노테이션

```java
@Data
  public class Item {
    private Long id;
    
    @NotBlank
    private String itemName;
    
    @NotNull
    @Range(min = 1000, max = 1000000)
    private Integer price;
    
    @NotNull
    @Max(9999)
    private Integer quantity;
    public Item() {}
    public Item(String itemName, Integer price, Integer quantity) {
        this.itemName = itemName;
        this.price = price;
        this.quantity = quantity;
    } 
}
```

- @NotBlank : 빈값 + 공백만 있는 경우를 허용하지 않는다.
- @NotNull : null 을 허용하지 않는다.
- @Range(min = 1000, max = 1000000) : 범위 안의 값이어야 한다. 
- @Max(9999) : 최대 9999까지만 허용한다.

> javax.validation 으로 시작하면 특정 구현에 관계없이 제공되는 표준 인터페이스이고, org.hibernate.validator 로 시작하면 하이버네이트 validator 구현체를 사용할 때만 제공되는 검증 기능이다. 실무에서 대부분 하이버네이트 validator를 사용하므로 자유롭게 사용해도 된다.

## Bean Validation 테스트 코드

검증기 생성

```java
  ValidatorFactory factory = Validation.buildDefaultValidatorFactory();
  Validator validator = factory.getValidator();
```

검증 실행
검증 대상을 직접 검증기에 넣고 그 결과를 받는다. Set 에는 ConstraintViolation 이라는 검증 오류가 담긴다
```java
Set<ConstraintViolation<Item>> violations = validator.validate(item);
```
## Bean Validation 스프링

### 스프링 MVC는 어떻게 Bean Validator를 사용하는가?

스프링 부트가 spring-boot-starter-validation 라이브러리를 넣으면 자동으로 Bean Validator를 인지하고 스프링에 통합한다.

### 스프링 부트는 자동으로 글로벌 Validator로 등록한다.

LocalValidatorFactoryBean 을 글로벌 Validator로 등록한다. 이 Validator는 @NotNull 같은 애노테이션을 보고 검증을 수행한다. 이렇게 글로벌 Validator가 적용되어 있기 때문에, @Valid , @Validated 만 적용하면 된다. 검증 오류가 발생하면, FieldError , ObjectError 를 생성해서 BindingResult 에 담아준다

> 검증시 @Validated @Valid 둘다 사용가능하다. javax.validation.@Valid 를 사용하려면 build.gradle 의존관계 추가가 필요하다. @Validated 는 스프링 전용 검증 애노테이션이고, @Valid 는 자바 표준 검증 애노테이션이다. 둘중 아무거나 사용해도 동일하게 작동하지만, @Validated 는 내부에 groups 라는 기능을 포함하고 있다.

### **스프링에서 오류 검증 순서**

BeanValidator는 바인딩에 실패한 필드는 BeanValidation을 적용하지 않는다. **바인딩에 성공한 필드만 Bean Validation 적용한다.**

1. @ModelAttribute 각각의 필드에 타입 변환 시도
   1. 성공하면 다음으로
   2. 실패하면 typeMismatch 로 FieldError 추가
2. Validator 적용

> @ModelAttribute -> 각각의 필드 타입 변환시도 -> 변환에 성공한 필드만 BeanValidation 적용

## Bean Validation 에러 코드(필드 오류) - NotBlank

필드에 타입 변환을 시도하고 실패하면 typeMismatch, 변환에 성공한 BeanValidation을 적용한다.

Bean Validation이 기본으로 제공하는 오류 중 NotBlank 처럼 오류 코드를 기반으로 MessageCodesResolver 를 통해 다양한 메시지 코드가 순서대로 생성된다.

### BeanValidation에서 NotBlank 메시지 찾는 순서

1. 생성된 메시지 코드 순서대로 messageSource 에서 메시지 찾기
2. 애노테이션의 message 속성 사용
3. 라이브러리가 제공하는 기본 값 사용 -> 공백일 수 없습니다.

애너테이션의 message 사용 예

```java
@NotBlank(message = "공백은 입력할 수 없습니다.") 
private String itemName;
```

## Bean Validation 오브젝트 오류

### @ScriptAssert 추가

Bean Validation에서 특정 필드( FieldError )가 아닌 해당 오브젝트 관련 오류( ObjectError )는 다음과 같이 @ScriptAssert() 를 사용하면 된다.

```java
@Data
@ScriptAssert(lang = "javascript", script = "_this.price * _this.quantity >= 10000")
public class Item {
    //...
}
```

그런데 실제 사용해보면 제약이 많고 복잡하다. 그리고 실무에서는 검증 기능이 해당 객체의 범위를 넘어서는 경우들도 종종 등장하는데, 그런 경우 대응이 어렵다. 따라서 오브젝트 오류(글로벌 오류)의 경우 **@ScriptAssert를 억지로 사용하는 것 보다는 다음과 같이 오브젝트 오류 관련 부분만 직접 자바 코드로 작성하는 것을 권장**한다.


### @ScriptAssert 부분 제거후 로직 추가

```java
//특정 필드 예외가 아닌 전체 예외
if (item.getPrice() != null && item.getQuantity() != null) {
    int resultPrice = item.getPrice() * item.getQuantity();
    if (resultPrice < 10000) {
        bindingResult.reject("totalPriceMin", new Object[]{10000, resultPrice}, null);
    } 
}
```

## Bean Validation 한계

다른 로직마다 오류 검출을 다르게 하고 싶은데 다른 로직에서 같은 자바 프로퍼티에서 애너테이션을 통해 오류를 검출하게 된다면 오류를 다르게 할 수 없다.

해경하는 방법

- Bean Validation의 groups기능 (@Validated만 가능)
- 각각의 Form 전송을 위한 별도의 모델 객체 분리

### Bean Validation 한계 해결 groups

### groups 적용

저장용 groups 생성

```java
public interface SaveCheck {}
```

수정용 groups 생성

```java
public interface UpdateCheck {}
```

```java
@Data
public class Item {
    @NotNull(groups = UpdateCheck.class) //수정시에만 적용 
    private Long id;

    @NotBlank(groups = {SaveCheck.class, UpdateCheck.class})
    private String itemName;
}
```

### 저장 로직 변경 - SaveCheck Groups 적용

@Valid 에는 groups를 적용할 수 있는 기능이 없다. 따라서 groups를 사용하려면 @Validated 를 사용해야 한다.

```java
@PostMapping("/add")
public String addItemV2(@Validated(SaveCheck.class) @ModelAttribute Item item, BindingResult bindingResult, RedirectAttributes redirectAttributes) {
    //...
}
```


### 정리

groups 기능을 사용해서 등록과 수정시에 각각 다르게 검증을 할 수 있었지만, groups 기능을 사용하면서 전반적으로 복잡도가 올라갔다. 그렇기 때문에 groups 기능은 실제 잘 사용되지는 않는디.

### Bean Validation 한계 해결 Form 전송 객체 분리

실무에서는 groups 를 잘 사용하지 않는데, 바로 등록시 폼에서 전달하는 데이터가 도메인 객체와 딱 맞지 않기 때문이다.

ItemSaveForm 저장

```java
@Data
public class ItemSaveForm {
    @NotBlank
    private String itemName;

    @NotNull
    @Range(min = 1000,max = 1000000)
    private Integer price;

    @NotNull
    @Max(value = 9999) // 수정 요구사항 추가
    private Integer quantity;
}

```

상품 저장 로직 추가

```java
@PostMapping("/add")
public String addItem(@Validated() @ModelAttribute("item") ItemSaveForm form, BindingResult bindingResult, RedirectAttributes redirectAttributes, Model model) {

    //...

    //검증에 실패하면 다시 입력 폼으로
    if (bindingResult.hasErrors()) {
        log.info("errors={}", bindingResult);
        return "validation/v4/addForm";
    }

    Item itemParam = new Item();
    itemParam.setItemName(form.getItemName());
    itemParam.setPrice(form.getPrice());
    itemParam.setQuantity(form.getQuantity());

    Item savedItem = itemRepository.save(itemParam);
    redirectAttributes.addAttribute("itemId", savedItem.getId());
    redirectAttributes.addAttribute("status", true);
    return "redirect:/validation/v4/items/{itemId}";
}
```

## Bean Validation HTTP 메시지 컨버터

AAPI의 경우 3가지 경우를 나누어 생각해야 한다.

- 성공 요청: 성공(Validator 실행 후 성공)
- 실패 요청: JSON을 객체로 생성하는 것 자체가 실패함 (Validator가 실행되지 않음)
- 검증 오류 요청: JSON을 객체로 생성하는 것은 성공했고, 검증에서 실패함 (Validator 실행 후 실패)

### 실패 요청 - HttpMessageConverter 실패

HttpMessageConverter 에서 요청 JSON을 Item 객체로 생성하는데 실패한다. 이 경우는 Item 객체를 만들지 못하기 때문에 컨트롤러 자체가 호출되지 않고 그 전에 예외가 발생한다. 즉, Validator도 실행되지 않고 400 상태코드(`"Bad Request"`)를 반환한다.


> 💡 @Valid, @Validated를 적용하기 위해서는 메시지 컨버터가 정상적으로 작동해서 Item 객체를 만들 수 있어야 한다.

### 검증 오류 요청

`return bindingResult.getAllErrors();` 는 `ObjectError` 와 `FieldError` 를 반환한다. 스프링이 이 객체를 JSON으로 변환해서 클라이언트에 전달했다.검중 객체 들 중 필요한 데이터만 뽑아서 별도의 API 스펙을 정의하고 그에 맞는 객체를 만들어서 반환하면 된다.

### @ModelAttribute vs @RequestBody

@ModelAttribute는 각각의 필드 단위로 세밀하게 적용되기 때문에 특정 필드에 타입이 맞이 않는 오류가 발생해도 나머지 필드는 정삭적으로 작동한다. 하지만 HttpMessageConverter 이용해 객체를 생성하는 방법인 @RequestBody은 해당 맞는 json 객체를 받아야 메시지 컨버터가 작동을 하고, 유효성 검사를 할 수 있다.

> 💡 *HttpMessageConverter 단계에서 실패하면 예외가 발생한다. 즉, 메시지 바디를 통해 들어오는 객체는 별도의 예외처리를 해줘야 한다.*