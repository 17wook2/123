---
title:  "4/5 Issue #1 - 이메일 인증 코드 검증 서버 구현"
excerpt: "project"

categories:
- project
tags:
- [project, redis]

toc: true
toc_sticky: true

date: 2023-4-5
last_modified_at: 2023-4-5
---

# 4/5 Issue #1 - 이메일 인증 코드 검증 서버 구현

## Redis 도입

Redis는 인증 서버에서 이메일 인증을 구현하는 데 매우 유용한 데이터 저장 및 검색 도구이다.

단순 키/값의 조합을 저장하고 TTL 을 설정할 수 있다는 장점이 있기 때문에 이메일 인증 서버로 Redis를 사용했다.

현재는 배포환경이 아닌 로컬환경에서 도커를 통해 redis를 켜고 통신하도록 하였다.

## 1. 의존성 추가

```java
implementation 'org.springframework.boot:spring-boot-starter-data-redis'
```

## 2. Redis 설정

스프링 부트에서는 application.yaml 파일을 사용하여 Redis 설정을 구성할 수 있다.

다음과 같이 Redis 호스트와 포트를 구성하였다.

```java
spring:
	data:
	  redis:
	    port: 6379
	    host: localhost
```

## 3. RedisTemplate 구성

RedisTemplate은 Redis와 상호 작용하는 데 사용되는 주요 클래스이다.

Redis를 편리하게 사용하고 이메일과 코드를 저장하고 검색하는 데 사용할 수 있는 RedisTemplate을 구성하였다.

```java
@Configuration
@ConfigurationProperties("spring.data.redis")
@Data
public class RedisConfig {

    private String host;
    private int port;

    @Bean
    public RedisConnectionFactory redisConnectionFactory(){
        return new LettuceConnectionFactory(host,port);
    }

    @Bean
    public RedisTemplate<String, Object> redisTemplate() {
        RedisTemplate<String, Object> redisTemplate = new RedisTemplate<>();
        redisTemplate.setKeySerializer(new StringRedisSerializer());
        redisTemplate.setValueSerializer(new StringRedisSerializer());
        redisTemplate.setConnectionFactory(redisConnectionFactory());
        return redisTemplate;
    }

    @Bean
    public StringRedisTemplate stringRedisTemplate(){
        StringRedisTemplate stringRedisTemplate=new StringRedisTemplate();
        stringRedisTemplate.setKeySerializer(new StringRedisSerializer());
        stringRedisTemplate.setValueSerializer(new StringRedisSerializer());
        stringRedisTemplate.setConnectionFactory(redisConnectionFactory());
        return stringRedisTemplate;
    }
}
```

## 4. RedisUtil을 통해 값 가져오기

RedisUtil이라는 클래스를 만들어 좀더 가져오기 쉽게 구성하였다.

```java
@RequiredArgsConstructor
@Service
public class RedisUtil {

private final StringRedisTemplate redisTemplate;

public String getData(String key){
ValueOperations<String, String> valueOperation = redisTemplate.opsForValue();
return valueOperation.get(key);
    }

public boolean existData(String key) {
return Boolean.TRUE.equals(redisTemplate.hasKey(key));
    }

public void setDataWithExpire(String key, String value,longduration) {
ValueOperations<String, String> valueOperations = redisTemplate.opsForValue();
        Duration expireDuration = Duration.ofSeconds(duration);
        valueOperations.set(key, value, expireDuration);
    }

public void deleteData(String key) {
        redisTemplate.delete(key);
    }

}
```

## 5. 이메일 인증 코드 생성 및 검증

- 이메일/검증코드 저장

setDataWithExpire함수를 통해 redis에 이메일/인증코드 쌍에 TTL을 적용하여 redis에 생성하였다.

```java
private MimeMessage createEmailForm(String email)throws MessagingException {
    // 메시지 생성 로직
    redisUtil.setDataWithExpire(email,authCode,60*30L);
return message;
}
```

- 이메일/검증코드 확인

받아온 값이 null이거나 false면 false를 반환하고, 검증이 성공했다면 true를 반환하였다.

