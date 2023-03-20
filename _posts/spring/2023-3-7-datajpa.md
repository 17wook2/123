---
title:  "스프링 데이터 JPA"
excerpt: "jpa"

categories:
- jpa
tags:
- [jpa, dataJPA, db]

toc: true
toc_sticky: true

date: 2023-3-7
last_modified_at: 2023-3-7
---


> https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81-%EB%8D%B0%EC%9D%B4%ED%84%B0-JPA-%EC%8B%A4%EC%A0%84
해당 강의를 참고하여 작성하였습니다.

### JPA로 Repository 작성 시 코드의 중복

JPA기반 Repository를 만들다 보면 기본적인 CRUD 작성시에 엔티티마다 큰 차이점이 없다.
아래는 Member에 대한 Repository 코드 이지만, Team에 대한 코드여도 Member 부분을 Team으로 바꾸어 주면 기본 틀은 비슷하게 사용할 수 있다.

```java
@Repository
    public class MemberJpaRepository {
        @PersistenceContext
        private EntityManager em;
        public Member save(Member member) {
            em.persist(member);
            return member;
}
        public void delete(Member member) {
            em.remove(member);
}
        public List<Member> findAll() {
            return em.createQuery("select m from Member m", Member.class)
                    .getResultList();
 }
       public Optional<Member> findById(Long id) {
          Member member = em.find(Member.class, id);
          return Optional.ofNullable(member);
}
      public long count() {
          return em.createQuery("select count(m) from Member m", Long.class)
                  .getSingleResult();
}
      public Member find(Long id) {
          return em.find(Member.class, id);
	} 
}
```

### 스프링 데이터 JPA
스프링 데이터 JPA를 사용하면 기본적인 CRUD코드를 자동으로 작성하여 주기 때문에 많은 이점이 생긴다.

JpaRepository를 상속한 인터페이스만 만들어 주면, 작성이 끝난다.
제네릭 부분에는
```java
public interface MemberRepository extends JpaRepository<Member, Long> {
  }
```

인터페이스만 작성하고 실제 구현체는 구현하지 않았지만, 아래의 사진과 같은 원리로 스프링 데이터 JPA에서 구현체를 구현하여 프록시로 연결해 준다.



![](https://velog.velcdn.com/images/wook2pp/post/df3f8eaa-c003-43f2-8e3f-266f4ce23ed4/image.png)

### 공통 인터페이스 분석

JpaRepository 공통 기능 인터페이스는 공통적인 CRUD를 제공한다.

PagingAndSortingRepository 인터페이스나 CrudRepository 인터페이스는 DB에 공통적인 데이터 처리 인터페이스 이고, JPARepository는 JPA에 특화된 기술을 제공하는 인터페이스라고 생각할 수 있다.

```java
public interface JpaRepository<T, ID extends Serializable> extends PagingAndSortingRepository<T, ID>
{
...
}

```
- T : 엔티티
- ID : 엔티티의 식별자 타입
- S : 엔티티와 그 자식 타입

---

- save(S) : 새로운 엔티티는 저장하고 이미 있는 엔티티는 병합한다. 객체를 저장한다
- delete(T) : 엔티티 하나를 삭제한다. 내부에서 EntityManager.remove() 호출
- findById(ID) : 엔티티 하나를 조회한다. 내부에서 EntityManager.find() 호출
- getOne(ID) : 엔티티를 프록시로 조회한다. 내부에서 EntityManager.getReference() 호출, 프록시로 조회하기 때문에 원하는 시점에 프록시를 초기화 할 수 있다.
- findAll(...) : 모든 엔티티를 조회한다. 정렬( Sort )이나 페이징( Pageable ) 조건을 파라미터로 제공할 수 있다.

Spring Data JPA는 공통 메서드를 거의 모두 제공한다고 생각하면 된다.


### 메소드 이름으로 쿼리 생성
JPA에서 공통 메서드를 제공한다고 하더라도, 특정 도메인에 연결되어 있는 메서드는 직접 구현해야 하나라는 생각이 든다.
하지만 데이터 JPA에서는 쿼리 메소드 기능으로 이를 해결할 수 있다.


- JPA 리포지토리로 이름과 나이를 기준으로 검색하는 경우
  createQuery를 통해 JPQL을 작성하고 파라미터를 넣어주어 결과를 찾아 온다.

```java
 public List<Member> findByUsernameAndAgeGreaterThan(String username, int age) {
      return em.createQuery("select m from Member m where m.username = :username
  and m.age > :age")
              .setParameter("username", username)
              .setParameter("age", age)
              .getResultList();
}
```

- 스프링 데이터 JPA를 통한 이름과 나이 기준으로 검색하는 경우
  메서드 이름을 분석하여 JPQL을 생성하고 실행한다. JPQL을 직접 작성할 필요가 없으며 메서드 이름으로 어떤 역할을 하는지 충분히 유추가 가능하다.

```java
List<Member> findByUsernameAndAgeGreaterThan(String username, int age);
```

### 쿼리 메서드 필터 조건

쿼리 메서드에 필요한 필터 조건을 넣어주어 쿼리 메서드를 만들 수 있다.

![](https://velog.velcdn.com/images/wook2pp/post/daded1ee-ff11-4be0-a5f2-6986de90150f/image.png)

> https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#jpa.query-methods.query-creation
레퍼런스에서 상세하게 볼 수 있다.

위의 기능은 엔티티 필드명이 변경되면 인터페이스에서 정의한 메서드 이름도 변경되어야 한다.
예를 들어 findByUsername에서 Username이 name으로 바꾼다면 애플리케이션 로딩 시점에 오류가 발생한다.


### 리포지토리 메소드에 쿼리 정의하기

@Query 애노테이션을 통해 JPQL 쿼리를 직접 작성할 수 있다.
@org.springframework.data.jpa.repository.Query에 등록되어 있다.

```java
 @Query("select m from Member m where m.username= :username and m.age = :age")
    List<Member> findUser(@Param("username") String username, @Param("age") int
    age);
```

실행할 메서드에 직접 쿼리를 작성하기 때문에 이름 없는 NamedQuery이다.
@Query 애노테이션에 있는 JPQL가 문제가 있는 경우에, 애플리케이션 로딩 시점에 오류를 찾아낼 수 있다.

쿼리 메서드 필터조건이 길어진다면 함수 길이가 매우 길어지기 때문에, 이런 경우에는 직접 JPQL 쿼리를 작성하는 것이 좋다.

### @Query로 값, DTO 조회하기

**- 값 하나 조회**

```java
@Query("select m.username from Member m")
    List<String> findUsernameList();
```

**- DTO로 직접 조회**
dto로 조회하기 위해서는 new 명령어를 사용해야 한다.
객체를 찾아 온 뒤, MemberDto객체에 넣어준다고 생각하면 된다.

```java
 @Query("select new study.datajpa.dto.MemberDto(m.id, m.username, t.name) " +
          "from Member m join m.team t")
  List<MemberDto> findMemberDto();
```

### 파라미터 바인딩

:name에 들어갈 파라미터를 @Param을 통해 바인딩 할 수 있다.

```java
@Query("select m from Member m where m.username = :name")
        Member findMembers(@Param("name") String username);
```

**- 컬렉션 파라미터 바인딩**

in절에 들어갈 names들에 컬렉션을 넘겨 파라미터를 바인딩 할 수도 있다.
```java
@Query("select m from Member m where m.username in :names")
    List<Member> findByNames(@Param("names") List<String> names);
```

### 반환 타입

스프링 데이터 JPA는 유연한 반환 타입을 지원한다.
컬렉션, 단건, Optinal등 다양한 반환타입을 사용할 수 있다.

반환 타입별로 다음과 같은 상황이 있다.
- 컬렉션 사용 시 결과가 없다면 : 빈 컬렉션 반환

- 단건 조회시 결과가 없다면 : null 반환
- 결과가 2건 이상이라면 : javax.persistence.NonUniqueResultException 예외 발생

---


## 페이징과 정렬


### JPA를 사용한 페이징과 정렬

나이가 age이고, 이름으로 내림차순, offset 페이지에 보여줄 데이터는 limit 만큼 가져온다.

```java
public List<Member> findByPage(int age, int offset, int limit) {
        return em.createQuery("select m from Member m where m.age = :age order by
    m.username desc")
                .setParameter("age", age)
                .setFirstResult(offset)
				.setMaxResults(limit)
				.getResultList();
       
 }
```

- setFirstResult() : 어디서부터 가져올지를 지정 - offset
- setMaxResults() : 몇개를 가져올지를 지정 - limit



내 페이지가 몇번째인지 알기 위해 totalCount를 반환하는 함수 작성

```java

public long totalCount(int age) {
    return em.createQuery("select count(m) from Member m where m.age = :age",
Long.class)
	.setParameter("age", age)
	.getSingleResult();
}
```

### 스프링 데이터 JPA를 통한 페이징과 정렬

페이징과 정렬 파라미터
- org.springframework.data.domain.Sort : 정렬 기능
- org.springframework.data.domain.Pageable : 페이징 기능 (내부에 Sort 포함)


페이징과 정렬 처리를 위한 반환타입
- org.springframework.data.domain.Page : 추가 count 쿼리 결과를 포함하는 페이징
- org.springframework.data.domain.Slice : 추가 count 쿼리 없이 다음 페이지만 확인 가능 (내부적으로 limit + 1조회)
- List (자바 컬렉션): 추가 count 쿼리 없이 결과만 반환

메서드의 매개변수에 Pageable 인터페이스를 넘겨주면, Pageable 구현체에 페이징에 관한 정보를 넣고 반환타입으로 Page, Slice, List를 받아온다.
```java
Page<Member> findByUsername(String name, Pageable pageable); //count 쿼리 사용 
Slice<Member> findByUsername(String name, Pageable pageable); //count 쿼리 사용 안함
List<Member> findByUsername(String name, Pageable pageable); //count 쿼리 사용 안함
  List<Member> findByUsername(String name, Sort sort);

```

Pageable 인터페이스를 구현한 PageRequest 객체를 사용하면 된다.
PageRequest에는 현재 페이지, 조회할 데이터 수, 정렬방식을 사용할 수 있다.

```java
 PageRequest pageRequest = PageRequest.of(0, 3, Sort.by(Sort.Direction.DESC,"username"));
 Page<Member> page = memberRepository.findByAge(10, pageRequest);
```


**- Page 인터페이스**

```java
public interface Page<T> extends Slice<T> {
int getTotalPages(); //전체 페이지 수
long getTotalElements(); //전체 데이터 수
<U> Page<U> map(Function<? super T, ? extends U> converter); //변환기
}
```

**- Slice 인터페이스**
slice 인터페이스는 전체 페이지수를 가져오지 않고, 가져올 데이터 + 1개만큼 limit을 설정하고
더 있으면 다음 내용이 있다는 것이고, 없으면 마지막인 것을 알 수 있다.
```java
public interface Slice<T> extends Streamable<T> {
int getNumber();
int getSize();
int getNumberOfElements();
List<T> getContent();
boolean hasContent();
Sort getSort();
boolean isFirst();
boolean isLast();
boolean hasNext();
boolean hasPrevious();
Pageable getPageable();
Pageable nextPageable();
Pageable previousPageable();//이전 페이지 객체
<U> Slice<U> map(Function<? super T, ? extends U> converter); //변환기
}
```

**- count 쿼리 분리하기**

Member를 가져오는 상황을 가정해 본다.
전체의 개수를 가져온다면 연관된 테이블과 left join해서 가져오는 경우에, count 쿼리는 멤버의 수와 같기 때문에 countQuery를 별도로 분리 해 멤버의 수만 count해서 가져오면 빠르다.
```java
@Query(value = “select m from Member m”,
         countQuery = “select count(m.username) from Member m”)
  Page<Member> findMemberAllCountBy(Pageable pageable);
```

**- 페이지를 유지하면서 DTO로 변환하기 **

page의 map을 사용하여 페이징을 유지하면서 Member 엔티티를 MemberDto로 변환할 수 있다.

```java
Page<Member> page = memberRepository.findByAge(10, pageRequest);
Page<MemberDto> dtoPage = page.map(m -> new MemberDto());
```


### JPA에서 벌크 수정 쿼리

벌크 연산이란 여러 건(대량의 데이터)을 한 번에 수정하거나 삭제하는 방법이다.
JPA 에서는 executeUpdate() 메소드 사용하여 벌크 연산을 수행할 수 있다.

```java
 public int bulkAgePlus(int age) {
        int resultCount = em.createQuery(
                "update Member m set m.age = m.age + 1" +
                        "where m.age >= :age")
                .setParameter("age", age)
                .executeUpdate();
        return resultCount;
}
```

**벌크 연산시 주의점**

- 벌크 연산은 영속성 컨텍스트와 2차 캐시를 무시하고 데이터베이스에 직접 쿼리를 날린다.
  => 영속성 컨텍스트의 내용과 데이터베이스에 있는 내용이 다를 수 있다.

- 해결방법 : 벌크 연산 후에는 영속성 컨텍스트를 초기화 해주어야 한다.


### 스프링 데이터를 사용한 벌크성 수정 쿼리

@Query를 통해 update문을 작성하면 된다. @Modifying 애노테이션이 있어야 업데이트가 된다.

```java
  @Modifying
  @Query("update Member m set m.age = m.age + 1 where m.age >= :age")
  int bulkAgePlus(@Param("age") int age);
```

또한 벌크성 쿼리는 실행 후 영속성 컨텍스트를 초기화 해주어야 하기 때문에,

```
@Modifying(clearAutomatically = true)
```
clearAutomatically 옵션에 true를 주어 영속성 컨텍스트를 초기화 해준다.



### @EntityGraph

연관된 엔티티들을 SQL 한번에 조회하는 방법이다.
Member와 Team은 ManyToOne관계로 연결되어 있는데, 멤버 조회시 멤버에 관련된 팀이 연관되어 같이 조회된다. (N+1문제 발생)

- JPQL 사용 시 fetch join 을 통해 N+1 문제 해결
  fetch join을 사용 시 멤버를 조회할 때 join쿼리를 같이 날려서 SQL 한번에 데이터를 가져온다.

```java
@Query("select m from Member m left join fetch m.team")
List<Member> findMemberFetchJoin();
```

- EntityGraph 사용

공통메서드를 오버라이드 하여 EntityGraph를 통해 Member 조회 시, Team을 같이 조회할 수 있다.

```java
@Override
@EntityGraph(attributePaths = {"team"}) List<Member> findAll();
```

JPQL과 같이 사용할 수도 있고, 메서드 이름의 쿼리 방식에서도 사용 가능하다.
```java
@EntityGraph(attributePaths = {"team"}) 
@Query("select m from Member m") 
List<Member> findMemberEntityGraph();


@EntityGraph(attributePaths = {"team"}) 
List<Member> findByUsername(String username)
```

### JPA Hint


JPA 쿼리 힌트를 나타낸다. readOnly 힌트를 통해 영속성 컨텍스트에 있는 객체가 아닌 읽기 전용 객체를 가져올 수 있다.
```java
@QueryHints(value = @QueryHint(name = "org.hibernate.readOnly", value ="true"))
Member findReadOnlyByUsername(String username);

```

### 사용자 정의 리포지토리 구현

스프링 데이터 JPA 리포지토리는 인터페이스만 정의하고 구현체는 스프링이 자동 생성
스프링 데이터 JPA가 제공하는 인터페이스를 직접 구현하면 구현해야 하는 기능이 너무 많음
다양한 이유로 인터페이스의 메서드를 직접 구현하고 싶다면?
- JPA 직접 사용( EntityManager )
- 스프링 JDBC Template 사용
- MyBatis 사용
- 데이터베이스 커넥션 직접 사용 등등...

MemberRepositoryCustom 이라는 인터페이스 생성 뒤
이를 구현 한 구현체 MemberRepositoryImpl을 생성한다.

MemberRepositoryImpl에서는 순수한 JPA를 사용한다고 생각하고 코드를 작성하는 상황이다.

```java
@RequiredArgsConstructor
  public class MemberRepositoryImpl implements MemberRepositoryCustom {
      private final EntityManager em;
      @Override
      public List<Member> findMemberCustom() {
          return em.createQuery("select m from Member m")
                  .getResultList();
} }

```

구현체와 인터페이스를 만들었다면, 데이터 JPA 인터페이스가 커스텀 인터페이스를 상속받도록 해준다.

```java
 public interface MemberRepository
           extends JpaRepository<Member, Long>, MemberRepositoryCustom {
}
```

클라이언트에서는 Data JPA의 memberRepository에서 커스텀 메서드를 실행할 수 있다.

```java
 List<Member> result = memberRepository.findMemberCustom();
```

사용자 정의 구현 클래스는 이름을 데이터 JPA 인터페이스 이름 + "Impl" 으로 작성해야 한다.

물론 위의 방식대로 커스텀 클래스들을 MemberRepository에 모두 붙여야 하는 것은 아니다.
스프링 데이터 JPA와 무관하게 @Repository 애노테이션과 함께 만들고 싶은 커스텀 클래스를 스프링 빈에 등록하여 클라이언트에서 구현체를 주입받아서 사용하여도 된다.

### JPA Auditing

엔티티를 만들 때 시간과 작성자에 대한 필드가 필요한 경우가 있다.
등록일, 수정일, 등록자, 수정자 의 정보를 추적하기 위한 기능이다.

그렇지만 모든 엔티티에 생성자 시간등에 대한 정보를 넣어서 생성하면 비효율적일 수 있다.

- **스프링 JPA에서 Auditing을 사용하는 경우**

@MappedSuperclass는 객체의 입장에서 공통된 매핑 정보가 필요한 경우 사용한다.
공통된 매핑 정보가 필요한 경우 부모 클래스에 선언하고 속성만 상속받아서 사용하면 된다.
테이블과는 상관이 없고 단순히 엔티티가 공통으로 사용하는 매핑정보를 모으는 역할이다.
JPA에서 @Entity는 @MappedSuperclass로 지정한 클래스만 상속받을 수 있다.

```java
@MappedSuperclass
  @Getter
  public class JpaBaseEntity {
      @Column(updatable = false)
      private LocalDateTime createdDate;
      private LocalDateTime updatedDate;
      @PrePersist
      public void prePersist() {
          LocalDateTime now = LocalDateTime.now();
          createdDate = now;
          updatedDate = now;
}
      @PreUpdate
      public void preUpdate() {
          updatedDate = LocalDateTime.now();
      }
}
```

- **스프링 데이터 JPA 에서 Auditing을 사용하는 경우**

@EnableJpaAuditing -> 스프링 부트 설정 클래스에 적용해야함
@EntityListeners(AuditingEntityListener.class) -> 엔티티에 적용

@EntityListeners는 엔티티를 DB에 적용하기 전과 후에 커스텀 콜백을 요청할 수 있는 애노테이션이다.
Auditing을 수행할 때에는 JPA에서 제공하는 AuditingEntityListener를 인자로 넘겨주면 된다.
AuditingEntityListener에서 엔티티의 생성 또는 수정이 일어나면 콜백이 실행되고, 그 때 시간을 넣어준다.


@CreatedDate : Entity가 생성될 때 자동으로 생성 시간이 저장된다.
@LastModifiedDate: 최근에 수정된 시간이 저장된다.

```java
  @EntityListeners(AuditingEntityListener.class)
  @MappedSuperclass
  public class BaseEntity {
      @CreatedDate
      @Column(updatable = false)
      private LocalDateTime createdDate;
      
      @LastModifiedDate 
      private LocalDateTime lastModifiedDate;
      
      @CreatedBy
      @Column(updatable = false)
      private String createdBy;
      @LastModifiedBy
      private String lastModifiedBy;
}
```

등록자 수정자를 처리하기 위해서는 AuditorAware를 스프링 빈으로 등록해야 한다.

빈으로 등록한 AuditorAware에서 등록자 수정자를 처리하면 된다.
등록자를 가져오는 경우 세션이나 스프링 시큐리티를 통해 받아온 유저 정보를 등록하면 된다.


```java
 @Bean
    public AuditorAware<String> auditorProvider() {
        return () -> Optional.of(UUID.randomUUID().toString()); // 일단은 랜덤한 UUID로 등록
    }
```


### Web에서의 페이징과 정렬

파라미터로 Pageable을 받을 수 있다.
요청시에는 아래와 같이 page와 size , sort를 쿼리스트링으로 넘길 수 있다.
```
/members?page=0&size=3&sort=id,desc&sort=username,desc
```

```java
 @GetMapping("/members")
    public Page<Member> list(Pageable pageable) {
        Page<Member> page = memberRepository.findAll(pageable);
        return page;
    }

```

엔티티 내용을 그대로 반환하는 것은 문제가 많기 때문에, page.map()을 이용하여 DTO로 변환하여 보내주어야 한다.

```java
 @GetMapping("/members")
  public Page<MemberDto> list(Pageable pageable) {
      Page<Member> page = memberRepository.findAll(pageable);
      Page<MemberDto> pageDto = page.map(MemberDto::new);
      return pageDto;
}

```


### 스프링 데이터 JPA 분석

스프링 데이터 JPA가 제공하는 공통 인터페이스의 구현체는 SimpleJpaRepository이다.

```java
org.springframework.data.jpa.repository.support.SimpleJpaRepository
```
SimpleJpaRepository에는 @Repository가 걸려있기 때문에,
서비스 하부의 JDBC이든 DB에러와 같은 에러들을 모두 스프링이 추상화한 에러로 바꾸어 준다.

@Transactional 트랜잭션 적용
- JPA의 모든 변경은 트랜잭션 안에서 동작
- 스프링 데이터 JPA는 변경(등록, 수정, 삭제) 메서드를 트랜잭션 처리
- 서비스 계층에서 트랜잭션을 시작하지 않으면 리파지토리에서 트랜잭션 시작
- 서비스 계층에서 트랜잭션을 시작하면 리파지토리는 해당 트랜잭션을 전파 받아서 사용
- 그래서 스프링 데이터 JPA를 사용할 때 트랜잭션이 없어도 데이터 등록, 변경이 가능했음(사실은 트랜잭션이 리포지토리 계층에 걸려있는 것임)

![](https://velog.velcdn.com/images/wook2pp/post/5bd5b122-39d7-46c3-98ee-054c55b7c0e3/image.png)


**- 스프링 데이터 JPA save() 메서드 분석**

save메서드
- 새로운 엔티티인 경우: persist로 저장
- 새로운 엔티티가 아닌 경우: merge로 병합
  ![](https://velog.velcdn.com/images/wook2pp/post/fc096da9-27df-44c4-8a3f-b81370ce3e88/image.png)

새로운 엔티티를 판단하는 경우는 다음과 같다.
- 식별자가 객체인 경우 : null로 새로운 엔티티인지 판단
- 식별자가 원시타입인 경우 : 0으로 새로운 엔티티인지 판단
- Persistable 인터페이스를 구현해서 판단 로직 변경 가능

엔티티 생성 시 @GenerateValue로 기본키 값을 설정하면 save() 호출 시점에 엔티티 식별자가 없으므로 새로운 엔티티로 판단한다.
만약 @Id만 선언하고 직접 식별자를 할당하는 경우, 새로운 엔티티로 판단하지 않기 때문에 merge()를 호출한다.
merge()시 DB에 호출에 값이 있는지 확인 후, 없으면 새로운 엔티티로 판단하기 때문에 새로운 엔티티 검증로직이 겹치는 것을 알 수 있다.

이 경우, Persistable 인터페이스를 구현하여 새로운 엔티티 판단 로직을 변경할 수 있다.
판단 로직은 auditing으로 등록된 createdDate를 통해 판단할 수 있다.

```java
@Entity
@EntityListeners(AuditingEntityListener.class)
  @NoArgsConstructor(access = AccessLevel.PROTECTED)
  public class Item implements Persistable<String> {
      @Id
      private String id;
      @CreatedDate
      private LocalDateTime createdDate;
      public Item(String id) {
          this.id = id;
}
      @Override
      public String getId() {
return id; }
      @Override
      public boolean isNew() {
          return createdDate == null;
      }
}
```




