---
title:  "4/5 Issue #15 - 예외 처리를 위환 환경 개발"
excerpt: "project"

categories:
- project
tags:
- [project, exception]

toc: true
toc_sticky: true

date: 2023-4-5
last_modified_at: 2023-4-5
---

# 4/5 Issue #15 - 예외 처리를 위환 환경 개발

## 스프링 예외처리

스프링에서 예외처리를 위해 제공해주는 기능을 사용했다.

- @ControllerAdvice
  @ControllerAdvice는 스프링 MVC에서 사용되는 어노테이션으로, 일반적으로 View를 반환하는 컨트롤러에 적용된다.
  이 어노테이션을 사용하여 예외 발생 시 처리할 로직을 담은 클래스를 정의할 수 있으며, 해당 클래스는 @ExceptionHandler 어노테이션을 사용하여 예외를 처리할 수 있다. 또한, @Component 어노테이션을 포함하고 있으므로, 스프링에 의해 자동으로 스캔된다.
- @RestControllerAdvice

@RestControllerAdvice는 @RestController 어노테이션을 포함하고 있으므로, 모든 메서드가 @ResponseBody 어노테이션을 포함하고 있다.

Rest API 서버를 개발하고 있기 때문에 @RestControllerAdvice를 사용하여 예외처리를 하였다.

## 예외시 응답 객체 만들기

이 클래스는 예외 처리에서 사용되며, 예외 발생 시 반환될 데이터를 담고 있다.

- status : 요청 응답 상태 ex) 401, 404...
- code : 받은 ErrorCode에 따른 오류 코드
- description : 에러에 대한 설명
- timestamp : 에러가 발생한 시간

ErrorCode를 정의한 enum객체를 받아올 수도 있고, 혹은 필드를 하나씩 직접 받아와서 생성할 수 있다.

```java
@Data
@Builder
public class ErrorResponse {
    private Integer status;
    private String code;
    private String description;
    private LocalDateTime timestamp;

    public static ErrorResponse of(ErrorCode errorCode) {
        return new ErrorResponseBuilder()
                .status(errorCode.getStatus())
                .code(errorCode.getCode())
                .description(errorCode.getDescription())
                .timestamp(LocalDateTime.now())
                .build();
    }

    public static ErrorResponse of (Integer status, String code, String description) {
        return new ErrorResponseBuilder()
                .status(status)
                .code(code)
                .description(description)
                .timestamp(LocalDateTime.now())
                .build();
    }
}

```

## ErrorCode Enum 정의하기

이 enum은 예외 처리에서 사용될 에러 코드와 에러 메시지를 담고 있습니다.

필드로는 상태, 코드, 설명을 담고 있다.

에러코드가 많아진다면 추후 도메인별로 에러코드를 나눠야 할 지 논의를 해봐야 한다.

```java
@Getter
public enum ErrorCode {

    // 추후 도메인별로 에러코드 범위를 나눌필요가 있음.

    INVALID_INPUT_VALUE(400, "COMMON-001", "유효성 검증에 실패하였습니다."),

    INTERNAL_SERVER_ERROR(500, "COMMON-002", "서버에 문제가 있습니다."),

    DUPLICATE_LOGIN_ID(400, "ACCOUNT-001", "이미 존재하는 회원입니다."),
    INCORRECT_PASSWORD_PASSWORDCONF(404,"ACCOUNT-001","비밀번호와 비밀번호확인이 일치하지 않습니다."),
    UNAUTHORIZED(401, "ACCOUNT-002", "인증이 되지 않았습니다."),
    ACCOUNT_NOT_FOUND(404, "ACCOUNT-003", "해당 계정은 존재하지 않습니다."),
    TOKEN_NOT_EXISTS(404, "ACCOUNT-004", "해당 토큰은 존재하지 않습니다.")
    ;

    private Integer status;
    private String code;
    private String description;

    ErrorCode(Integer status, String code, String description) {
        this.status = status;
        this.code = code;
        this.description = description;
    }
}

```

## 커스텀 Exception 생성하기

자바에서 예외는 크게 두가지가 있다.

- Checked Exception
  Checked Exception은 RuntimeException 클래스를 상속받지 않은 예외 클래스이다. 컴파일러가 예외 처리를 강제하기 때문에, 메서드에서 이를 처리하지 않을 경우 컴파일 에러가 발생한다.
  ex) IOException, ClassNotFoundException, SQLException
- Unchecked Exception
  Unchecked Exception은 RuntimeException 클래스를 상속받은 예외 클래스이다.
  예외 처리를 강제하지 않는다. 주로 프로그램 실행 시 발생하는 예외를 의미하며, 개발자가 적절한 예외처리를 구현하도록 해야 한다.
  ex) NullPointerException, IllegalArgumentException, IndexOutOfBoundsException, ArithmeticException

로직에 문제가 있는 경우 처리를 위해 RuntimeException을 상속받은 Exception을 만들어, 예외 클래스를 생성한다.

예외는 도메인 별로 나누어 생성하도록 하였다.
유저에서 발생하는 에러는 모두 UserException 객체를 던져서 추후 ControllerAdvice에서 처리할 수 있도록 한다.
또한, 예외를 던질 때 ErrorCode를 넘겨서 어떤 오류가 발생했는지 남긴다.

```java
public class UserException extends RuntimeException {
    @Getter
    private final ErrorCode errorCode;
    public UserSignupException(ErrorCode errorCode) {
        this.errorCode = errorCode;
    }
}

```

## @RestControllerAdvice를 통한 예외처리

현재는 예외에 대한 사항이 많이 없지만, 복잡한 로직을 구현시 예외를 발생시켜 아래의 코드처럼 처리하기로 하였다.

- 서블릿 컨테이너 기본 예외 처리기 - default exception handler
  예외가 발생하면 기본 예외 처리기를 호출하는데, 서블릿 컨테이너는 handle이라는 메서드에서 예외처리를 한다.

```java
public void handle(HttpServletRequest request, HttpServletResponse response, Throwable ex) throws IOException, ServletException {
    if (ex instanceof ServletException) {
        Throwable rootCause = ((ServletException) ex).getRootCause();
        if (rootCause != null) {
            ex = rootCause;
        }
    }
    response.sendError(HttpServletResponse.SC_INTERNAL_SERVER_ERROR);
}

```

위와 같이 코드를 작성하면 들어온 exception에 따라 if문을 통해 분기를 많이 쳐야하는데, 한 눈에 들어오지 않고 예외가 많아지면 구분하기 힘들다는 점이 있다.

- @ExceptionHandler
  일반적으로 컨트롤러에서 예외가 발생하면, 서블릿 컨테이너는 해당 예외를 기본 예외 처리기(default exception handler)에 전달하게 된다. 하지만 @ExceptionHandler를 사용하면, 예외 발생 시 해당 어노테이션이 적용된 메소드를 호출하여 예외 처리를 수행하게 된다.

```java
@RestControllerAdvice
public class GlobalControllerAdvice {

    @ExceptionHandler(UserException.class)
    public ErrorResponse handleUserDuplicatedException(UserException e) {
        return ErrorResponse.of(e.getErrorCode());
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ErrorResponse handleMethodArgumentNotValidException(MethodArgumentNotValidException e) {
        String errorDescription = e.getFieldErrors().stream()
                .map(DefaultMessageSourceResolvable::getDefaultMessage)
                .collect(Collectors.joining(", "));
        return ErrorResponse.of(
                e.getStatusCode().value(),
                ErrorCode.INVALID_INPUT_VALUE.getCode(),
                errorDescription
        );
    }

}

```

- MethodArgumentNotValidException

MethodArgumentNotValidException은 메소드에 전달되는 인자가 유효하지 않을 때 발생하는 예외이다. 보통 HTTP 요청에 대한 처리를 담당하는 컨트롤러에서 발생한다.

Spring MVC에서는 @Valid나 @Validated 어노테이션을 이용하여 메소드 인자에 대한 검증(Validation)을 수행할 수 있다.
만약 인자 값이 검증에 실패하면, MethodArgumentNotValidException이 발생한다.

현재 예외처리에는 해당 에러가 발생한 경우 필드에러 메시지들을 모두 취합해서 ErrorResponse 객체를 만들고 반환해주었다.