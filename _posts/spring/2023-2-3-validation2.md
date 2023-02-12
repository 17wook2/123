---
title:  "스프링 Bean Validation"
excerpt: "spring"

categories:
- spring
tags:
- [spring, validation]

toc: true
toc_sticky: true

date: 2023-2-3
last_modified_at: 2023-2-3


---


검증로직을 매번 코드로 작성하는 일은 처리해주어야 할 일이 너무 많다.
검증의 대부분은 필드의 값에 대한 검증이나 빈 값인지, 초과인지, 미만인지등의 일반적인 로직이다.
이 검증들을 스프링을 통해 어노테이션 기반으로 쉽게 검증할 수 있다.

---

## Bean Validation

Bean Validation은 특정한 구현체가 아니라 Bean Validation 2.0(JSR-380)이라는 기술 표준이다.
구현체로는 Hibernate의 Validator를 사용한다.

Validation은 모든 application layer에서 적용된다. 아래의 그림과 같은 방식은 중복되는 validation이 있을 수 있다.
> ![](https://velog.velcdn.com/images/wook2pp/post/8e0842cd-c0eb-4c51-8a8f-d0164d7e6fa4/image.png)
출처 : https://docs.jboss.org/hibernate/validator/8.0/reference/en-US/html_single/#preface

그렇기 때문에 validation 로직을 도메인 영역에 묶어서 클래스의 메타데이터를 표현할 수 있다.

> ![](https://velog.velcdn.com/images/wook2pp/post/5c577f5b-8c30-42b0-a924-c51f2a3c68b3/image.png)

## 검증 애노테이션


애노테이션 기반의 방식을 통해 바로 직관적으로 이해되게 검증 로직을 만들 수 있다.
- @NotBlank: 빈값 + 공백인 경우 허용하지 않는다.
- @NotNull : null 을 허용하지 않는다.
- @Range(min = 1000, max = 1000000) : 범위 안의 값이어야 한다.
- @Max(9999) : 최대 9999까지만 허용한다.


```java

@NotBlank
private String itemName;

@NotNull
@Range(min = 1000, max = 1000000)
private Integer price;

@NotNull
@Max(9999)
private Integer quantity;
```

---

## 검증기 꺼내기

Validation 구현체에서 validator를 꺼내서 사용할 수 있다.
스프링을 사용하면 빈을 통해 등록됨으로 아래의 코드를 안써도 된다.

```java
ValidatorFactory factory = Validation.buildDefaultValidatorFactory();
Validator validator = factory.getValidator();

Set<ConstraintViolation<Item>> violations = validator.validate(item); // 검증에 걸린 로직들이 반환된다.
```

```java
violation={interpolatedMessage='공백일 수 없습니다', propertyPath=itemName, rootBeanClass=class hello.itemservice.domain.item.Item, messageTemplate='{javax.validation.constraints.NotBlank.message}'} violation.message=공백일 수 없습니다

```

## 스프링 MVC 검증

스프링 부트를 사용하는 경우, spring-boot-starter-validation 라이브러리를 넣으면 자동으로 Bean Validator를 스프링에 통합한다.

스프링 부트는 LocalValidatorFactoryBean를 글로벌 Validator로 등록하기 때문에, 검증을 하려는 곳에 @Validated만 작성하면 된다. 검증 오류가 발생하면 FieldError, ObjectError를 생성해서 BindingResult에 담아준다.


## Bean Validation 에러 코드

스프링은 오류 코드를 기반으로 MessageCodesResolver 를 통해 다양한 메시지 코드가 순서대로
생성된다.

```java
@NotBlank

NotBlank.item.itemName NotBlank.itemName NotBlank.java.lang.String NotBlank
```

BeanValidation는 다음과 같은 순서로 메시지를 찾는다.
1. 생성된 메시지 코드 순서대로 messageSource 에서 메시지를 찾는다.
2. 애노테이션의 message 속성 사용 @NotBlank(message = "공백! {0}")
3. 라이브러리가 제공하는 기본 값 사용

## 상황별 도메인 제약조건이 다른 경우

데이터를 등록할 때와 수정할 때 제약조건이 다른 경우는 별도의 모델 객체를 만들어 상황별로 validation을 다르게 만들어 주어야 한다.

폼 데이터 전달을 위한 별도의 객체 사용은 다음과 같은 방법으로 진행된다.

** HTML Form -> 특정 폼 -> Controller -> 도메인 생성 및 데이터 추가 -> Repository **

특정 폼을 만들어야하고, 객체 생성하는 과정이 있다.
전송하는 폼 데이터가 복잡해도 거기에 맞춘 별도의 폼 객체를 사용해서 데이터를 전달 받을 수
있다. 또한, 별도의 폼 객체를 만들기 때문에 검증이 중복되지 않는다.

## HttpMessageConverter

@Validated는 @RequestBody에도 적용할 수 있다.

- @ModelAttribute : HTTP요청 파라미터를 다룰 때 사용한다. => URL query string, POST form
- @RequestBody : HTTP Body 의 데이터를 객체로 변환할 때 사용


API 요청의 경우 3가지의 경우가 있다.

1. 성공 : 데이터를 받는데도 문제가 없었고, 검증에도 문제가 없는 경우
2. 실패 : JSON 데이터를 객체로 만드는 과정에서 문제가 생기는 경우
3. 검증 실패 : 객체는 만들었지만, 검증에 실패한 경우

@ModelAttribute는 특정 필드가 바인딩 되지 않아도 나머지 필드는 정상 바인딩 된다.
@RequestBody는 JSON 데이터에서 객체를 만들지 못하면 검증 단계에 도달하지 못한다.