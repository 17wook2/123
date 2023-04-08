---
title:  "4/4 Issue #1 - 이메일 인증"
excerpt: "project"

categories:
- project
tags:
- [project, authentication]

toc: true
toc_sticky: true

date: 2023-4-4
last_modified_at: 2023-4-4
---
# 4/4 Issue #1 - 이메일 인증

## SMTP란?

SMTP는 Simple Mail Transfer Protocol의 약자로, 인터넷에서 이메일을 보내기 위해 사용되는 프로토콜이다. SMTP는 이메일 클라이언트나 다른 SMTP 서버로부터 이메일을 수신하는 메일 서버로 이메일을 전송하는 데 사용된다.

## 스프링에서 Mail 시스템 사용하기

- 구현 API : 스프링의 Mail 모듈은 JavaMailAPI을 추상화하여 편리한 메일 전송 기능을 제공한다. 이 모듈을 사용하면 간단한 설정만으로 메일을 전송할 수 있다.
- 이메일 메시지 구성 : JavaMail API에서 제공하는 MimeMessage 클래스를 사용하여 이메일을 구성이메일에는 수신자, 발신자, 제목, 본문 등의 정보를 포함하여 작성할 수 있고, 첨부 파일을 추가하려면 MimeMessageHelper 클래스를 사용하여 추가해야 한다.
- SMTP 서버 설정 : SMTP 서버는 스프링의 설정 파일에서 지정할 수 있으며, 인증 정보를 포함시킬 수 있다.Mail 모듈은 스프링의 다른 모듈과 함께 사용할 수 있기 때문에 스프링의 스케줄링 모듈과 함께 사용하면 정기적으로 이메일을 보내는 기능을 구현할 수 있다.스프링 부트에서는 Mail 모듈을 자동 구성해주기 때문에 별도의 설정 없이도 쉽게 메일 전송 기능을 사용할 수 있다.

## JAVA mail 사용 시 추가해야 하는 의존성

메일기능을 사용하기 위해선 아래의 의존성을 추가해주어야 한다.

```java
implementation 'org.springframework.boot:spring-boot-starter-mail'
```

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
```

## 메일 메시지 작성 및 전송

Spring에서 MimeMessage는 JavaMail API를 사용하여 이메일을 작성하고 전송하는 데 사용된다. MimeMessage는 이메일의 제목, 본문, 수신자, 발신자 및 첨부 파일과 같은 다양한 속성을 지정할 수 있다.

부트에서는 자동주입으로 받아온 JavaMailSender의 구현체를 사용하면 된다.

JavaMailSender의 구현체는 다음과 같이 있다.

- JavaMailSenderImpl: JavaMail을 직접 사용하여 이메일을 전송하는 기본 구현체
- JavaMailSenderImplWithSession: 미리 구성된 JavaMail 세션을 사용하여 이메일을 전송하는 구현체
- JavaMailSenderImplWithTransport: JavaMail의 Transport 클래스를 사용하여 이메일을 전송하는 구현체
- JavaMailSenderImplWithSSL: SSL 암호화를 사용하여 이메일을 전송하는 구현체
- JavaMailSenderImplWithTLS: TLS 암호화를 사용하여 이메일을 전송하는 구현체