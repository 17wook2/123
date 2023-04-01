---
title:  "스프링 SMTP 서버를 통한 이메일 인증 및 가입 구현하기"
excerpt: "spring"

categories:
- spring
tags:
- [spring, authentication, email]

toc: true
toc_sticky: true

date: 2023-4-1
last_modified_at: 2023-4-1
---


## SMTP란?

SMTP는 Simple Mail Transfer Protocol의 약자로, 인터넷에서 이메일을 보내기 위해 사용되는 프로토콜이다. SMTP는 이메일 클라이언트나 다른 SMTP 서버로부터 이메일을 수신하는 메일 서버로 이메일을 전송하는 데 사용된다.

---

## 스프링에서 Mail 시스템 사용하기

- 구현 API : 스프링의 Mail 모듈은 JavaMailAPI을 추상화하여 편리한 메일 전송 기능을 제공한다. 이 모듈을 사용하면 간단한 설정만으로 메일을 전송할 수 있다.

- 이메일 구성 : JavaMail API에서 제공하는 MimeMessage 클래스를 사용하여 이메일을 구성
  이메일에는 수신자, 발신자, 제목, 본문 등의 정보를 포함하여 작성할 수 있고, 첨부 파일을 추가하려면 MimeMessageHelper 클래스를 사용하여 추가해야 한다.

- 서버 설정 : SMTP 서버는 스프링의 설정 파일에서 지정할 수 있으며, 인증 정보를 포함시킬 수 있다.
  Mail 모듈은 스프링의 다른 모듈과 함께 사용할 수 있기 때문에 스프링의 스케줄링 모듈과 함께 사용하면 정기적으로 이메일을 보내는 기능을 구현할 수 있다.
  스프링 부트에서는 Mail 모듈을 자동 구성해주기 때문에 별도의 설정 없이도 쉽게 메일 전송 기능을 사용할 수 있다.

---

## JAVA mail 사용 시 추가해야 하는 의존성

메일기능을 사용하기 위해선 아래의 의존성을 추가해주어야 한다.

```java
implementation 'org.springframework.boot:spring-boot-starter-mail'

```

---
## 구글 계정 SMTP 서버 설정하기

구글 2단계 설정을 한 뒤, 앱 비밀번호를 만들어 주어야 한다.

![](https://velog.velcdn.com/images/wook2pp/post/b9ddc682-0dc1-4199-9be1-a201c1652ce0/image.png)

생성 후 제공되는 비밀키로 smtp 서버에 접속할 수 있다.


---

## 설정파일 작성하기

메일 설정
- 프로토콜 : smtp
- smtp 서버 : gmail
- port : smtp 서버 포트
- smtp 서버에서 사용할 계정
- starttls : STARTTLS는 기존의 안전하지 않은 연결의 위험을 줄이고 SSL/TLS이 쓰이는 안전한 연결로 업그레이드하도록 도와준다.

```java
spring:
  mail:
    protocol: smtp
    host: smtp.gmail.com
    port: 587
    username: ${username}
    password: ${password}
    properties:
      smtp.auth : true
      mail:
        smtp:
          starttls:
            enable: true
            required : true
    test-connection: true
  data:
    redis:
      host: localhost
      port: 6379

```

---

## 메일 메시지 작성 및 전송

Spring에서 MimeMessage는 JavaMail API를 사용하여 이메일을 작성하고 전송하는 데 사용된다. MimeMessage는 이메일의 제목, 본문, 수신자, 발신자 및 첨부 파일과 같은 다양한 속성을 지정할 수 있다.

받아온 이메일로 메시지 폼을 만들어 메시지를 반환하는 함수이다. 랜덤한 값을 만들어 인증코드를 생성하고 메일에 같이 전송하였다.
중간에 있는 redisUtil은 이메일과 인증코드 검증을 위한 키/값 검증용 redis 서버를 위해 작성되었다.

부트에서는 자동주입으로 받아온 JavaMailSender의 구현체를 사용하면 된다.

JavaMailSender의 구현체는 다음과 같이 있다.
- JavaMailSenderImpl: JavaMail을 직접 사용하여 이메일을 전송하는 기본 구현체
- JavaMailSenderImplWithSession: 미리 구성된 JavaMail 세션을 사용하여 이메일을 전송하는 구현체
- JavaMailSenderImplWithTransport: JavaMail의 Transport 클래스를 사용하여 이메일을 전송하는 구현체
- JavaMailSenderImplWithSSL: SSL 암호화를 사용하여 이메일을 전송하는 구현체
- JavaMailSenderImplWithTLS: TLS 암호화를 사용하여 이메일을 전송하는 구현체

```java
private final JavaMailSender mailSender;
```

```java
private MimeMessage createEmailForm(String email) throws MessagingException {
        MimeMessage message = mailSender.createMimeMessage();
        message.addRecipients(MimeMessage.RecipientType.TO, email);
        message.setFrom(이메일 입력);
        message.setSubject("[이메일 인증 요청]");
        String authCode = createCode();
        redisUtil.setDataWithExpire(email,authCode,60*30L);
        message.setText(authCode);
        return message;
    }
```

---

## 이메일과 인증코드 검증

이메일과 인증코드 쌍을 검증하기 위해 redis 서버를 사용했다.

스프링에서 redis를 사용하기 위해 의존성을 추가하였다.
```java
implementation 'org.springframework.boot:spring-boot-starter-data-redis'
```

redis 사용을 편의하게 하기 위해, redisUtil이라는 클래스를 만들어 사용하였다.

데이터 가져오기, 존재여부 확인, 데이터 저장과 만료기간 설정, 데이터 삭제함수를 만들어 편리하게 reis를 사용할 수 있도록 하였다.

```java
private final StringRedisTemplate redisTemplate;

    public String getData(String key){
        ValueOperations<String, String> valueOperation = redisTemplate.opsForValue();
        return valueOperation.get(key);
    }

    public boolean existData(String key) {
        return Boolean.TRUE.equals(redisTemplate.hasKey(key));
    }

    public void setDataWithExpire(String key, String value, long duration) {
        ValueOperations<String, String> valueOperations = redisTemplate.opsForValue();
        Duration expireDuration = Duration.ofSeconds(duration);
        valueOperations.set(key, value, expireDuration);
    }

    public void deleteData(String key) {
        redisTemplate.delete(key);
    }
```


validateEmailCode에서 이메일과 code 쌍이 redis에 올바르게 저장되어 있는지 검증한 후, null인 경우 false를 반환하고 그렇지 않으면 true를 반환하도록 하였다.

```java
public boolean validateEmailCode(String email, String code) {
        return Optional.ofNullable(redisUtil.getData(email))
                .map(s -> s.equals(code))
                .orElse(false);
    }
```

## 이메일로 회원가입시 요청객체 검증하기

요청받아온 객체를 검증하기 위해 @Validated 애노테이션을 사용하여 직접 검증 로직을 구현하였다.

```java
@PostMapping("/signup")
    public ResponseEntity<String> signup(@Validated SignupRequest signupRequest, BindingResult bindingResult) {
        if (bindingResult.hasErrors()) return ResponseEntity.status(404).body(bindingResult.toString());
        return ResponseEntity.ok().body("ok");
    }
```

Validator 인터페이스를 구현하고 supports메서드에는 검증에 사용할 객체를 넣어주고,
validate 메서드에는 실제 검증로직을 작성하였다.

회원가입시 이메일 중복 검증, 비밀번호와 비밀번호 확인 일치여부, 이메일 인증코드 일치여부를 확인해야 하는데, 이메일 인증코드에 집중하여 validate 메서드를 작성하였다.


```java
public class SignupRequestValidator implements Validator {

    private final EmailService emailService;

    @Override
    public boolean supports(Class<?> clazz) {
        return SignupRequest.class.isAssignableFrom(clazz);
    }

    @Override
    public void validate(Object target, Errors errors) {

        //todo
        // 1.이메일 중복 검증
        // 2. 비밀번호 일치여부 확인

        SignupRequest request = (SignupRequest) target;
        if (!emailService.validateEmailCode(request.email(), request.code())) {
            errors.rejectValue("email", "이메일 코드가 일치하지 않습니다.");
        }
    }
}
```

검증에 성공했다면, 컨트롤러에서 정상적으로 다음 로직으로 넘어가고,
그렇지 않다면 bindingResult에 error가 바인딩되어 에러 분기처리를 할 수 있도록 하였다.