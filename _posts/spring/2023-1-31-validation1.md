---
title:  "스프링 검증 Validation"
excerpt: "spring"

categories:
- spring
tags:
- [spring, validation]

toc: true
toc_sticky: true

date: 2023-1-31
last_modified_at: 2023-1-31


---



## 스프링 검증 처리
스프링은 검증 처리를 위한 기능을 제공하는데, BindingResult라는 것을 통해 처리할 수 있다.
BindingResult는 스프링이 제공하는 검증 오류를 보관하는 객체이다. 검증 오류가 발생하면 여기에 보관하면 된다.
![](https://velog.velcdn.com/images/wook2pp/post/f53c915b-fb0e-4858-822a-41767c12b651/image.png)
만약 @ModelAttribute를 통해 받아온 값이 문제가 있는데 BindingResult가 있다면 FieldError 객체를 BindingResult에 담아서 컨트롤러를 호출한다.

BindingResult를 통해 필드에러를 처리할 수도 있고, 이름에서도 알수 있듯이 결과를 어떤것과 연결해주는 역할을 한다.
BindingResult는 FieldError와 ObjectError를 받을 수 있다.

**FieldError**

FieldError는 다음과 같은 매개변수를 생성자로 받을 수 있다
- objectName : 오류가 발생한 객체 이름
- field : 오류 필드
- rejectedValue : 사용자가 입력한 값(거절된 값)
- bindingFailure : 타입 오류 같은 바인딩 실패인지, 검증 실패인지 구분 값
- codes : 메시지 코드
- arguments : 메시지에서 사용하는 인자
- defaultMessage : 기본 오류 메시지

```java
bindingResult.addError(new FieldError("item", "itemName", "Field Error 설명"));
```
```java
bindingResult.addError(new ObjectError("item", "Global Error 설명"));
```
또한 bindingResult 파라미터는 @ModelAttribute 파라미터 뒤에 위치해 있어야 한다.

FieldError와 ObjectError를 사용자가 직접 만들어서 하나하나 붙여주면 매개변수도 많고 복잡하다.

bindingResult에서 reject()와 rejectedValue()를 통해 직접 에러객체를 생성하지 않고 검증 오류를 처리할 수 있다.


**rejectValue**

- field : 오류 필드명
- errorCode : 오류 코드. MessageResolver를 통해 찾는 오류코드이다.
- errorArgs : 오류 메시지에서 {0} (매개변수) 을 치환하기 위한 값
- defaultMessage : 오류 메시지를 찾을 수 없을 때 사용하는 기본 메시지


![](https://velog.velcdn.com/images/wook2pp/post/2e7a4b04-ee4c-4f2b-a5a9-f837b2d2061c/image.png)

bindingResult는 이미 어떤 target을 대상을 하는지 알고 있으므로, 오류 필드명만 적어주면 된다.
errorCode에는 스프링 message를 통해 작성된 오류 코드를 넣어주면 해당되는 오류를 받을 수 있다.
defaultMessage는 오류메시지가 없다면 사용할 기본 메시지이다.

```java
bindingResult.rejectValue("price", "range", new Object[]{1000, 1000000}, null)
```

MessageSource를 통해 message를 받아와 code와 arguments에 값을 넣어줄 수도 있다.
codes와 arguments는 오류 발생시 오류코드로 메시지를 찾을 수 있도록 해준다.


### 타임리프에서 검증하기

타임리프에서는 #fields를 통해 bindingResult가 제공하는 검증 오류에 접근할 수 있다.
```html
<div th:if="${#fields.hasGlobalErrors()}">
          <p class="field-error" th:each="err : ${#fields.globalErrors()}"
th:text="${err}">글로벌 오류 메시지</p> </div>
```

th:if로 error가 있는 경우 태그를 출력하거나 클래스 정보를 추가하지 않아도
th:error, th:errorclass로 if문을 안쓰고도 분기를 만들 수 있다.

- **th:errors** 해당 필드에 오류가 있는 경우 태그 출력
```html
<div class="field-error" th:errors="*{itemName}">상품명 오류</div>
```

- **th:errorclass** th:field에서 지정한 필드에 오류가 있다면 class 정보 추가

```html
<input type="text" id="price" th:field="*{price}"
                 th:errorclass="field-error" class="form-control"
placeholder="가격을 입력하세요">
```

----
## 오류 코드와 메시지 처리

**MessageCodesResolver**

검증 오류 코드로 메시지 코드들을 생성한다.
MessageCodesResolver 인터페이스이고 DefaultMessageCodesResolver 는 기본 구현체이다.
주로 ObjectError , FieldError와 같이 사용한다.

FieldError와 ObjectError는 여러 오류 코드를 가질 수 있는데,
MessageCodesResolver를 통해 생성된 순서대로 가지고 있다.

bindingResult의 rejectValue(), reject()는 내부에서 MessageCodesResolver를 호출에 메시지를 생성한다.

**구현체 DefaultMessageCodesResolver의 기본 메시지 생성 규칙**

객체 오류의 경우 다음 순서로 2가지 생성
1 : code + "." + object name
2 : code

```java
 [required.item , required]
```

필드 오류의 경우 다음 순서로 4가지 생성
1 : code + "." + object name + "." + field
2 : code + "." + field
3 : code + "." + field type
4 : code

```java
["typeMismatch.user.age", "typeMismatch.age", "typeMismatch.int", "typeMismatch"]
```

**오류코드 관리하기**

구체적인 오류먼저 만들고, 범용된 오류(구체적이지 않은 오류)는 나중에 만들어 주어야
크게 중요하지 않은 메시지는 범용 메시지로 처리하고, 중요한 메시지는 구체적으로 적어주어 사용할 수 있다.

---

## Validator 분리하기

컨트롤러에서 검증 처리를 위한 로직이 많아진다면, 컨트롤러에 관심사가 분산될 가능성이 크다.
그렇기에 별도에 클래스로 분리해서 관리하는 것이 좋다.

직접 별도의 클래스를 만들어 로직을 분리해도 되지만, 스프링에서 제공하는 Validator 인터페이스를 구현하여 사용하면 추후에 빈 관리등 여러가지 기능이 많기 때문에, Validator 인터페이스를 구현하여 사용하는것이 좋다.

![](https://velog.velcdn.com/images/wook2pp/post/2519dbf2-0c52-4340-91de-f646ef67a8ca/image.png)


Validator 인터페이스는 두개의 메서드를 구현해야 한다.
- supports : target object가 해당 검증기 구현체를 지원하는지 확인
- validate : 검증 대상 객체와 Errors를 받아 검증로직 구현

생성자 주입 방식으로 Validator의 구현체 ItemValidator를 가져오고, 아래와 같이 validate 메서드를 실행해주면, bindingResult에 오류객체가 담기게 된다.

```java
itemValidator.validate(item, bindingResult);
```

---

## WebDataBinder 사용하기

WebDataBinder 는 스프링의 파라미터 바인딩의 역할을 해주고 검증 기능도 내부에 포함한다.

WebDataBinder에 Validator를 추가해주면 컨트롤러에서 검증기를 자동으로 적용할 수 있다.
@InitBinder는 해당 컨트롤러에만 영향을 준다.

```java
@InitBinder
  public void init(WebDataBinder dataBinder) {
       dataBinder.addValidators(itemValidator);
  }

```

**Validate 적용하기**

@Validated라는 어노테이션이 있으면 WebDataBinder에 등록한 검증기를 찾아서 실행한다.
여러 검증기가 있다면 supports()를 통해 어떤 검증기를 사용할지 결정할 수 있다.

```java
public String doSomething(@Validated @ModelAttribute Item item, BindingResult
  bindingResult, RedirectAttributes redirectAttributes) {
      if (bindingResult.hasErrors()) {
          return "doSomething";
}
```

@Validated: 스프링 전용 검증 어노테이션
@Valid: 자바 표준 검증 어노테이션



