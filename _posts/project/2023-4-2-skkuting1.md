---
title:  "4/2 Issue #5 - 물리 테이블 및 도메인 생성"
excerpt: "project"

categories:
- project
tags:
- [project, erd]

toc: true
toc_sticky: true

date: 2023-4-2
last_modified_at: 2023-4-2
---

# 4/2 Issue #5 - **물리 테이블 및 도메인 생성**

[https://github.com/skku-met/skkuting/issues/5](https://github.com/skku-met/skkuting/issues/5)

작성했던 ERD를 바탕으로 물리 테이블 및 도메인을 생성하였음.

현재는 로컬환경에서 DB를 운영하고 있지만, 추후에 확장성을 고려해 로컬환경과 배포환경의 데이터베이스를 구분할 필요가 있다.

DB 테이블 생성에는 JPA ddl auto를 사용하여 자동 생성하였다.

엔티티 생성에는 JPA 기술을 사용하여 작성하였다.

---

## 개발 완료 한 엔티티

### **Meetup - 모임**

![스크린샷 2023-04-03 오후 10.31.09.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/7909ebfe-8c19-4ddf-8ec3-1cbdde1dda7d/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2023-04-03_%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE_10.31.09.png)

```java
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;

@Setter @Column(nullable =false) private String title;
@Setter @Column(nullable =false, length = 10000) private String content;
private Integer max_member;
private Integer min_member;
private LocalDateTime due_date;
private LocalDateTime start_date;
private String duration;
private String place;

@Enumerated(EnumType.STRING)
private AuthorizingPolicy authorizingPolicy;

@Enumerated(EnumType.STRING)
private MeetupStatus meetupStatus;

@ToString.Exclude
@ManyToOne(fetch = FetchType.LAZY)
private UserAccount host;
```

id : GeneratedValue 의 생성 전략중 IDENTITY를 사용하여 데이터베이스에서 ID 값을 자동으로 생성하도록 하였음. 데이터베이스는 AUTO_INCREMENT 전략을 사용함.

title : 모임의 제목,  반드시 저장되어야 함, 추후 글자수를 수정에 대해 논의가 필요하다

content : 모임의 내용, 반드시 저장되어야 함. 글자수를 10000자로 제한

max_member : 모임의 최대 인원

min_member : 모임의 최소 인원

start_date : 모임 시작시간

duration : 모임을 하는 기간

place : 모이는 장소

authorizingPolicy : Enum 타입으로 회원 자동 수락방식과, 수동 수락방식이 있다

meetupStatus : Enum 타입으로 현재 모임의 상태를 나타낸다. 모집 완료, 모임 시작, 모임 종료

host : 모임을 주최한 사용자

### **MeetupReview - 모임후기**

![스크린샷 2023-04-03 오후 10.42.17.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/2a7159dd-eb29-4337-8b2d-b39c8e7e0e75/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2023-04-03_%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE_10.42.17.png)

```java
public class MeetupReview extends AuditingFields{
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    @ManyToOne
    private Meetup meetup;
    @ManyToOne
    private UserAccount review_from;
    @ManyToOne
    private UserAccount review_to;
    @Column(length = 1000) private String content;
    private Integer rating;
}
```

모임이 끝나고 난 뒤, 모임에 대해 모임에 참가한 유저는 참가한 유저를 평가할 수 있다.

- meetup : 모임과 모임후기는 1:n의 관계를 가지고 있다. @ManyToOne으로 매핑
- review_from : 유저와 후기작성자는 1:n의 관계를 가지고 있다.
- review_to : 유저와 후기받는사람은 1:n의 관계를 가지고 있다.
- content : 리뷰 내용
- rating : 평점

### **UserAccount - 유저**

![스크린샷 2023-04-03 오후 10.52.14.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/c676fd6c-1883-4b77-b417-0bf338a970e5/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2023-04-03_%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE_10.52.14.png)

```java
public class UserAccount extends AuditingFields{

    @Id
    private String email;

    @Column(nullable = false) private String nickname;
    @Column(nullable = false) private String password;
    private Integer studentNumber;
    @Column(length = 1000) private String description;

    @ToString.Exclude
    @OneToMany(mappedBy = "host", fetch = FetchType.LAZY)
    private Set<Meetup> hostingAppointment = new LinkedHashSet<>();

    @ToString.Exclude
    @OneToMany(mappedBy = "userAccount", fetch = FetchType.LAZY)
    private Set<UserMeetupRel> joinedMeetupList = new LinkedHashSet<>();
}
```

- id : 학생인증을 받은 사용자만 회원으로 받기 때문에, email을 기본키로 설정
- nickname, password : 필수 필드
- studentNumber : 학번
- description : 소개글

@OneToMany로 호스팅중인 모임과, 참가중인 모임에 대해 양방향 매핑을 해주었다.

### UserAccountRel - 참가중인 회원

![스크린샷 2023-04-03 오후 10.58.17.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/2b0053df-09f8-4229-90f4-491c7abc152d/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2023-04-03_%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE_10.58.17.png)

```java
public class UserMeetupRel extends AuditingFields{
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    @ManyToOne
    private UserAccount userAccount;
    @ManyToOne
    private Meetup meetup;
    private boolean isAllowed;
}
```

회원은 여러개 모임에 참가할 수 있고, 모임에는 여러명의 회원이 존재할 수 있기 때문에 N:M 관계이다.

@ManyToMany 를 사용할 수도 있지만, 중간 테이블에 생기는 필드를 고려해 중간 테이블을 생성하였다.

- userAccount : 모임에 참가신청한 회원
- meetup : 모임정보
- isAllowed : 모임에 참가신청한 회원이 승인이 되었는지, 아닌지를 알려주는 필드