---
title:  "스프링 게시판 서비스 개발해보기 2"
excerpt: "spring"

categories:
- spring
tags:
- [spring, service, project]

toc: true
toc_sticky: true

date: 2023-3-20
last_modified_at: 2023-3-20
---


게시판 서비스에서 구현하는 기능은 다음과 같다.
- **spring security를 통한 자원 인증처리**
- **인증된 사용자만 게시글, 댓글 작성 및 수정**
- **도메인에 auditing 추가하기**


## 게시판 도메인 구성하기
게시글과 댓글만 있는 간단한 게시판 서비스를 만들기 때문에 도메인을 심플하게 구성했다

도메인에는 게시글, 댓글, 사용자, 해시태그로 도메인을 구성하였다.

도메인을 만들기 전 ERD를 구성하면 아래의 그림과 같다.
게시글과 해시태그는 N:M 관계
게시글과 댓글은 1:N 관계
유저와 게시글은 1:N 관계

또한, 각 테이블에는 작성정보를 포함하고 있다.

![](https://velog.velcdn.com/images/wook2pp/post/9f8ac74b-7195-4ae0-81c7-b3db064790fb/image.png)

## 도메인의 엔티티 만들기



JPA를 통해 ddl 작성구문을 위임하여, 바로 엔티티를 구성하였다.

- **Article 엔티티**
  해시태그와의 N:M관계는 중간 테이블에 들어갈 내용이 없다고 판단해 @ManyToMany를 사용하여 중간 테이블을 코드로 만들지 않았다.

```java
@Entity
@Getter
@ToString
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Article extends BaseEntity{

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Setter @ManyToOne(optional = false) private UserAccount userAccount;

    @Setter @Column(nullable = false) private String title;
    @Setter @Column(nullable = false, length = 10000) private String content;


    @ToString.Exclude
    @JoinTable(
            name = "article_hashtag",
            joinColumns = @JoinColumn(name = "articleId"),
            inverseJoinColumns = @JoinColumn(name = "hashtagId")
    )
    @ManyToMany(cascade = {CascadeType.PERSIST, CascadeType.MERGE})
    private Set<Hashtag> hashtags = new LinkedHashSet<>();

    @ToString.Exclude
    @OrderBy("createdDate DESC")
    @OneToMany(mappedBy = "article", cascade = CascadeType.ALL)
    private final Set<ArticleComment> articleComments = new LinkedHashSet<>();

    private Article(UserAccount userAccount, String title, String content) {
        this.userAccount = userAccount;
        this.title = title;
        this.content = content;
    }

    public static Article of(UserAccount userAccount ,String title, String content){
        return new Article(userAccount,title,content);
    }

   
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Article that)) return false;
        return id != null && id.equals(that.getId());
    }

    @Override
    public int hashCode() {
        return Objects.hash(id);
    }
}

```

- **ArticleComment 엔티티**

```java
@Getter
@Entity
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class ArticleComment extends BaseEntity{

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Setter
    @ManyToOne(optional = false)
    private UserAccount userAccount;

    @Setter
    @ManyToOne(fetch = FetchType.LAZY)
    private Article article;

    @Column(length = 10000) private String content;

    private ArticleComment(UserAccount userAccount,Article article, String content) {
        this.userAccount = userAccount;
        this.article = article;
        this.content = content;
    }

    public static ArticleComment of(UserAccount userAccount, Article article, String content) {
        return new ArticleComment(userAccount,article,content);
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        ArticleComment that = (ArticleComment) o;
        return Objects.equals(id, that.id);
    }

    @Override
    public int hashCode() {
        return Objects.hash(id);
    }
}
```

- **UserAccount 엔티티**

```java
@Getter
@ToString
@Entity
public class UserAccount extends BaseEntity{

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Setter @Column(nullable = false, length = 50) private String userId;
    @Setter @Column(nullable = false) private String userPassword;

    @Setter @Column(length = 100) private String email;
    @Setter @Column(length = 100) private String nickname;

    @Setter private String memo;

    protected UserAccount(){}

    private UserAccount(String userId, String userPassword, String email, String nickname, String memo) {
        this.userId = userId;
        this.userPassword = userPassword;
        this.email = email;
        this.nickname = nickname;
        this.memo = memo;
    }

    public static UserAccount of (String userId, String userPassword, String email, String nickname, String memo) {
        return new UserAccount(userId, userPassword, email, nickname, memo);
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof UserAccount userAccount)) return false;
        return id != null && id.equals(userAccount.getId());
    }

    @Override
    public int hashCode() {
        return Objects.hash(id);
    }

}

```

- **hashtag 엔티티**

```java
@Getter
@ToString
@Entity
public class Hashtag extends BaseEntity{

    @Id
    @GeneratedValue
    private Long id;

    @ToString.Exclude
    @ManyToMany(mappedBy = "hashtags")
    private Set<Article> articles = new LinkedHashSet<>();

    @Setter @Column(nullable = false) private String hashtagName;

    protected Hashtag() {}

    private Hashtag (String hashtagName) {
        this.hashtagName = hashtagName;
    }

    public static Hashtag of(String hashtagName) {
        return new Hashtag(hashtagName);
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Hashtag hashtag = (Hashtag) o;
        return id.equals(hashtag.getId());
    }

    @Override
    public int hashCode() {
        return Objects.hash(id);
    }
}

```

## 컨트롤러와 서비스 로직 구현

리포지토리는 스프링 데이터 JPA를 사용하여 entityManager를 통해 DB 접근을 관리하였다.

- **검색 타입과 검색어로 게시글 가져오기**
  enum으로 만든 타입인 SearchType을 통해 검색타입을 받아왔고,
  searchValue로 검색어를 받아왔다.
  페이징 처리를 위해 스프링에서 제공하는 페이징 인터페이스를 사용하였다.
  또한, 가져온 타입을 메소드 참조 표현식으로 ArticleResponse타입으로 바꿔서 가져왔다.

```java
@GetMapping
    public String articles(
            @RequestParam(required = false) SearchType searchType,
            @RequestParam(required = false) String searchValue,
            @PageableDefault(size = 10, sort = "createdDate", direction = Sort.Direction.DESC) Pageable pageable,
            ModelMap map) {
        Page<ArticleResponse> articles = articleService.searchArticles(searchType, searchValue, pageable).map(ArticleResponse::from);
        map.addAttribute("articles", articles);
        map.addAttribute("searchTypes", SearchType.values());
        return "articles/index";
    }
```

읽기전용이니 Transactional에 readOnly를 적용해 주었다.
검색타입과 검색어의 null과 빈값 처리를 해주고, switch-case문을 이용해 검색타입에 따른 검색처리를 다르게 해주었다.

```java
@Transactional(readOnly = true)
    public Page<ArticleDto> searchArticles(SearchType searchType, String searchKeyword, Pageable pageable) {
        if (searchType == null || searchKeyword.isBlank()) {
            return articleRepository.findAll(pageable).map(ArticleDto::from);
        }
        return switch (searchType) {
            case TITLE -> articleRepository.findByTitleContaining(searchKeyword,pageable).map(ArticleDto::from);
            case CONTENT -> articleRepository.findByContentContaining(searchKeyword,pageable).map(ArticleDto::from);
            case ID -> articleRepository.findByUserAccount_UserIdContaining(searchKeyword, pageable).map(ArticleDto::from);
            case NICKNAME -> articleRepository.findByUserAccount_NicknameContaining(searchKeyword, pageable).map(ArticleDto::from);
        };
    }
```