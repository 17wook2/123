---
title:  "스프링 DB 데이터 접근"
excerpt: "spring"

categories:
- spring
tags:
- [spring, jdbc, transaction, db]

toc: true
toc_sticky: true

date: 2023-2-28
last_modified_at: 2023-2-28
---



> 해당 강의를 참고하고 만들었습니다.
> 
참고 : https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81-db-1

## JDBC 알아보기

### 데이터베이스와 애플리케이션을 연결하는 방법

연결을 수립하고 데이터를 전달하는 과정은 다음과 같다.

1) 애플리케이션과 데이터베이스의 TCP 연결을 통한 커넥션 연결

2) SQL 전달

3) 결과를 응답받기

이때, 데이터베이스가 바뀌게 되면 애플리케이션에서 해당 DB에 알맞는 연결 코드를 또 만들어야 하고, 데이터베이스가 늘어나면 그 수 만큼 연결을 늘려야 한다.
이에 JDBC표준 인터페이스를 통해 자바에서 데이터베이스에 접속할 수 있는 규약이 필요했다.

### JDBC 표준 인터페이스

Java Database Connectivity의 약자로, 데이터베이스에 접근 할 수 있는 자바 API이다.

![](https://velog.velcdn.com/images/wook2pp/post/4379c24e-0f10-4309-9778-3352585861e6/image.png)

JDBC 인터페이스의 장점
- 애플리케이션의 입장에선 어떤 드라이버가 연결이 되던, JDBC 표준 인터페이스에만 의존하면 된다.
- 다른 데이터베이스를 사용하고 싶다면, 애플리케이션 로직을 변경하지 않아도, 드라이버만 교체해주면 된다.


### JDBC, SQL Mapper, ORM

> ![](https://velog.velcdn.com/images/wook2pp/post/357728ba-bd63-4d26-8f0a-1f4fdfbddb08/image.png)
SQL Mapper의 장점
- SQL 응답 결과를 객체로 편리하게 바꿔준다.
- JDBC에서 반복되는 코드를 줄여준다.

> ![](https://velog.velcdn.com/images/wook2pp/post/35f04a98-2e83-47bd-8442-eb5fa45357a6/image.png)
ORM 장점
- SQL을 작성하지 않아도 ORM 기술이 동적으로 SQL을 생성 가능하다.
- 각각의 데이터베이스마다 다른 SQL을 사용하는 것도 해결해 준다.

SQL Mapper나 ORM의 공통점은 둘다 결국 JDBC로 바뀌어 전달된다는 점이다.


### 데이터베이스 연결

```java
@Slf4j
public class DBConnectionUtil {

    public static Connection getConnection() {
        try {
            Connection connection = DriverManager.getConnection(URL, USERNAME, PASSWORD);
            log.info("get connection={}, class={}", connection, connection.getClass());
            return connection;
        } catch (SQLException e) {
            throw new IllegalStateException(e);
        }
    }
}
```

JDBC가 제공하는 DriverManager를 이용해 해당 드라이버가 제공하는 커넥션을 받아올 수 있고,
이 커넥션은 JDBC 표준 커넥션 인터페이스인 java.sql.Connection 인터페이스를 구현하고 있다.

흐름은 다음과 같다.
![](https://velog.velcdn.com/images/wook2pp/post/94c9b0d5-c248-4cdb-a791-17885e213965/image.png)

애플리케이션은 DriverManager를 통해 커넥션을 요청하고,
DriverManager는 라이브러리에 등록된 드라이버 목록에 연결 요청 후, 애플리케이션으로 연결을 반환한다.


### 애플리케이션에서 연결을 맺는 과정

**1) 저장 로직**

```java
public Member save(Member member) throws SQLException {
        String sql = "insert into member(member_id, money) values(?, ?)";

        Connection con = null;
        PreparedStatement pstmt = null;

        try {
            con = getConnection();
            pstmt = con.prepareStatement(sql);
            pstmt.setString(1, member.getMemberId());
            pstmt.setInt(2, member.getMoney());
            pstmt.executeUpdate();

            return member;
        } catch (SQLException e) {
            log.error("db error", e);
            throw e;
        } finally {
            close(con, pstmt, null);
        }
    }
    
    private Connection getConnection() {
        return DBConnectionUtil.getConnection();
    }
```

- 1) DBConnectionUtil에 있는 DriverManager를 통해 커넥션을 받아온다.

- 2) DB에 전달할 SQL 구문과 파라미터를 넘겨준다.

- 3) 쿼리를 실행하고 나면 자원할다을 해제해 준다.


**2)조회 로직 **

```java
public Member findById(String memberId) throws SQLException {
        String sql = "select * from member where member_id = ?";

        Connection con = null;
        PreparedStatement pstmt = null;
        ResultSet rs = null;

        try {
            con = getConnection();
            pstmt = con.prepareStatement(sql);
            pstmt.setString(1, memberId);
            rs = pstmt.executeQuery();

            if (rs.next()) {
                Member member = new Member();
                member.setMemberId(rs.getString("member_id"));
                member.setMoney(rs.getInt("money"));

                return member;
            } else {
                throw new NoSuchElementException("member not found memberId=" + memberId);
            }
        } catch (SQLException e) {
            log.error("db error", e);
            throw e;
        } finally {
            close(con, pstmt, rs);
        }
    }
```

조회시에는 데이터를 ResultSet을 통해 받아오는데, select 쿼리의 결과가 순서대로 들어간다.

### 커넥션 풀

커넥션 풀은 데이터베이스와 연결된 커넥션을 미리 만들어 놓고 이를 pool로 관리하는 것이다. 즉, 필요할 때마다 커넥션 풀의 커넥션을 이용하고 반환하는 기법이다.

![](https://velog.velcdn.com/images/wook2pp/post/63f8c1b1-8262-4473-a760-f76725b5e395/image.png)
DB 드라이버는 DB에 TCP 연결, 인증, 커넥션 받기의 과정을 거쳐야 하는데, 매번 이 과정을 거치면 매우 비효율적이기 때문에 커넥션 풀을 이용하는 것이 좋다.

![](https://velog.velcdn.com/images/wook2pp/post/f6c55f9d-b974-4e3c-b819-85a00d7d0408/image.png)
애플리케이션은 DB 드라이버를 통해 커넥션을 받아오는 것이 아닌, 커넥션 풀을 통해 커넥션을 받아오면 된다.

![](https://velog.velcdn.com/images/wook2pp/post/9ee7d8f2-e948-4f78-9cfc-a8fcd34e5e44/image.png)

커넥션을 사용하고 나면, 커넥션을 종료하는 것이 아니라, 커넥션이 살아있는 상태로 커넥션 풀에 반환해야 한다.
스프링에서는 커넥션 풀로 hikariCP를 사용하는 것이 일반적이다.

### DataSource

애플리케이션에서 DriverManager를 통해 커넥션을 받아오다가, 커넥션 풀로 커넥션을 받아오도록 하기 위해서는 의존관계를 HikariCP로 변경해야 하기 때문에 코드 수정이 일어난다.

자바에서는 이를 위해 javax.sql.DataSource 인터페이스를 제공한다.

![](https://velog.velcdn.com/images/wook2pp/post/0a6d49fb-bbf9-4fd6-8150-09c1621e6d54/image.png)


DriverManager는 DataSource 인터페이스를 구현하지 않기 때문에,
DriverManager도 DataSource를 통해서 사용할 수 있도록, DriverManagerDataSource라는 DataSource를 구현한 클래스를 사용해야 한다.

이제 애플리케이션 코드에서는 DataSource를 통해서 커넥션을 받아오면 된다.

```java
@Slf4j
public class ConnectionTest {

    @Test
    public void driverManager() throws Exception {
        Connection con1 = DriverManager.getConnection(URL, USERNAME, PASSWORD);
        Connection con2 = DriverManager.getConnection(URL, USERNAME, PASSWORD);
        log.info("connection={}, class={}", con1, con1.getClass());
        log.info("connection={}, class={}", con2, con2.getClass());
    }

    @Test
    public void dataSourceDriverManager() throws Exception {
        // 항상 새로운 커넥션 획득
        DataSource dataSource = new DriverManagerDataSource(URL, USERNAME, PASSWORD);
        userDataSource(dataSource);
    }

    private void userDataSource(DataSource dataSource) throws SQLException {
        Connection con1 = dataSource.getConnection();
        Connection con2 = dataSource.getConnection();
        log.info("connection={}, class={}", con1, con1.getClass());
        log.info("connection={}, class={}", con2, con2.getClass());
    }
}
```
DriverManagerDataSource를 통해 받아온 커넥션에는 항상 매개변수로 정보를 넣어주어야 하지만, DataSource를 사용하는 방식은 생성되는 시점에 미리 다 넣어놓기 때문에, 매개변수에 의존하지 않아도 된다는 장점이 있다.


### 트랜잭션

트랜잭션은 데이터베이스의 작업 단위로 트랜잭션 단위로 커밋 또는 롤백이 된다.


### 트랜잭션 격리수준

트랜잭션 격리수준이란 동시에 여러 트랜잭션이 처리될 때, 트랜잭션끼리 얼마나 서로 고립되어 있는지를 나타낸다.

트랜잭션 간에 변경한 데이터를 볼 수 있도록 허용할지 말지를 결정하는 것이다.

격리수준

- READ UNCOMMITTED : 커밋되지 않은 트랜잭션을 읽을 수 있다.
  ex) txA가 A값을 1로 바꾸고 커밋하지 않은 상태인데, txB가 A값 조회해 1로 값 가져왔는데,
  txA가 롤백되는 경우 데이터 정합성 문제 발생

- READ COMMITTED : 주로 채택되는 격리수준. 트랜잭션이 커밋되어야만 다른 트랜잭션에서 읽을 수 있다.

- REPEATABLE READ : 트랜잭션이 시작되기 전에 커밋된 내용에 대해서만 조회할 수 있는 수준
- SERIALIZABLE : 읽기 작업에도 공유 잠금을 설정하는 격리수준

아래로 내려갈수록 트랜잭션간 고립 정도가 높아지며, 성능이 떨어지는 것이 일반적이다.
일반적으로 READ COMMITTED나 REPEATABLE READ를 사용한다.


### 데이터베이스 연결 구조


![](https://velog.velcdn.com/images/wook2pp/post/a0b901c6-6e60-4589-a550-7e8469ec8775/image.png)

클라이언트에서 서버로 커넥션을 요청해 커넥션을 맺는다.
이 과정에서 DB 서버는 세션을 생성하고 해당 커넥션은 만들어진 세션을 통해서만 실행된다.

세션은 트랜잭션을 실행, 커밋, 롤백을 통해 종료한다.


### 애플리케이션에 트랜잭션 적용

![](https://velog.velcdn.com/images/wook2pp/post/3682eca3-8f06-43ac-aa50-181159ba60f6/image.png)

비즈니스 로직의 시작점 부터 트랜잭션을 시작해야 한다.
비즈니스 로직으로 인해 문제가 되는 시점에는 트랜잭션을 롤백해야 한다.

결국 서비스단에 커넥션을 가져와야 하고, 트랜잭션이 끝난 시점에야 커넥션을 끊을 수 있다.

### 스프링 트랜잭션 문제

- 트랜잭션을 적용하기 위해 서비스 계층에 커넥션을 가져오고 넘겨주는 작업을 해야한다.
- 트랜잭션을 유지하기 위해 커넥션을 파라미터로 넘겨야 하는데, 트랜잭션을 위해 로직을 나누어야 한다.
- 트랜잭션 코드를 적용하기 위한 반복되는 코드가 많다. try, catch..
- JDBC 에러인 SQLException이 서비스 계층까지 넘어오게 되는데, 추후 다른 데이터 접근 로직 JPA를 사용하면 JPA에 알맞는 에러도 처리해야 하고 분기가 생긴다.

### 트랜잭션 추상화

위 문제를 해결하기 위해서 트랜잭션을 추상화하여 사용하면 된다. 서비스는 TxManager에만 의존하면 구체적인 구현체 바꿔 끼워주기만 하면 된다.
![](https://velog.velcdn.com/images/wook2pp/post/e2e5fc0b-6bc2-4332-a7ea-2ef9a7ef7439/image.png)

스프링에서의 트랜잭션 추상화는 PlatformTransactionManager를 의존한다.
![](https://velog.velcdn.com/images/wook2pp/post/edf2dc95-e7b4-4beb-89b3-e9ba8ba867d7/image.png)

### 트랜잭션 동기화

![](https://velog.velcdn.com/images/wook2pp/post/bfdd4f14-274a-4445-9fc4-4496f6b5b83f/image.png)

트랜잭션 매니저는 DataSource를 통해 받아온 정보를 통해 커넥션을 생성한다.
트랜잭션 동기화 매니저에게 커넥션을 보관하고, 데이터 접근 로직에서는 커넥션을 꺼내와 사용한다.
트랜잭션이 종료되면 트랜잭션 동기화 매니저에 저장된 커넥션을 통해 트랜잭션 매니저는 트랜잭션을 종료한다.

### 트랜잭션 매니저
```java
// DataSource와 TransactionManager 주입
DriverManagerDataSource dataSource = new DriverManagerDataSource(URL, USERNAME, PASSWORD);
PlatformTransactionManager transactionManager = new DataSourceTransactionManager(dataSource);
```


```java
@RequiredArgsConstructor
@Slf4j
public class MemberServiceV3_1 {

    private final PlatformTransactionManager transactionManager;
    private final MemberRepositoryV3 memberRepository;

    public void accountTransfer(String fromId, String toId, int money) throws SQLException {
        TransactionStatus status = transactionManager.getTransaction(new DefaultTransactionDefinition());

        try {
            // 비즈니스 로직
            transactionManager.commit(status); // 트랜잭션 매니저에게 트랜잭션 위임
        } catch (Exception e) {
            transactionManager.rollback(status); // 실패시 롤백
            throw e;
        }
    }

}
```

transactionManager를 주입받는다.
JDBC이면 DataSourceTransactionManager 구현체를, JPA이면 JpaTransactionManager를 주입받는다.

transactionManager.getTransaction()을 통해 트랜잭션을 시작하고, TransactionStatus를 반환한다. 이후 commit rollback할때 status가 필요하다

### 트랜잭션 AOP

트랜잭션 매니저 코드를 줄이기 위해 트랜잭션 매니저 템플릿을 사용해도 서비스 로직에 데이터 접근 로직이 들어가는 부분은 피해갈 수 없다.
이를 해결하기 위해 스프링이 제공하는 AOP기능을 사용하여 프록시를 적용할 수 있다.

![](https://velog.velcdn.com/images/wook2pp/post/9532e340-a933-4558-bd20-61ba45edcab5/image.png)

@Transactional 애노테이션을 통해 트랜잭션을 쉽게 적용할 수 있다.
```java
@Transactional
    public void accountTransfer(String fromId, String toId, int money) throws SQLException {
        // 비즈니스 로직
    }
```

AOP 프록시를 통해 트랜잭션이 생성되고 종료되는 과정은 다음과 같다.

![](https://velog.velcdn.com/images/wook2pp/post/26566e19-3626-4c4d-b06b-72155016a830/image.png)


### 스프링 부트 데이터 소스 자동 등록

```java
spring.datasource.url=jdbc:h2:tcp://localhost/~/test
spring.datasource.username=sa
spring.datasource.password=
```

스프링 부트는 DataSource를 스프링 빈으로 자동 등록한다. 만약 개발자가 직접 빈에 DataSource를 등록하면 스프링 부트는 자동으로 등록하지 않는다.

스프링 부트는 application.properties에 있는 속성을 통해 datasource를 등록한다.

### 트랜잭션 매니저 등록

스프링 부트는 적절한 트랜잭션 매니저를 자동등록 해준다.
PlatformTransactionManager 트랜잭션 매니저의 이름은 transactionManager로 등록된다.


### 예외의 종류

예외의 종류

![](https://velog.velcdn.com/images/wook2pp/post/37ea1903-605b-471e-837d-3e1d07d5421a/image.png)

1) 체크 예외 : RuntimeException을 상속하지 않은 예외이다. 예외가 발생할 수 있는 메소드를 사용할 경우, 복구가 가능한 예외들이기 때문에 반드시 예외를 처리하는 코드를 함께 작성해야 한다. catch문으로 예외를 잡거나, throws로 자신을 호출한 클래스로 예외를 던지는 방법이 있다. 이를 해결하지 않으면 컴파일 에러 가 발생한다.
   ex) IOException, SQLException

2) 언체크 예외 : RuntimeException을 상속한 예외들은 언체크 예외라고 부르는데, 명시적으로 예외처리를 강제하지 않는다. 그렇기 때문에 catch나 throws로 잡을 필요가 없다.
   ex) NullPointerException, IllegalArgumentException

### 데이터 접근 로직 예외 처리

체크 예외 처리의 문제점 : SQLException과 같이 데이터베이스에서 발생하는 문제를 서비스 로직에서 처리 할 방법이 없다.

또한 대부분의 예외는 복구 불가능한 예외인데, throws를 통해 데이터베이스에서 발생한 예외가 컨트롤러단까지 올라가는 경우가 생길 수 있다.

![](https://velog.velcdn.com/images/wook2pp/post/4a7d4d11-7e6f-400a-ac54-1ee952dd14f9/image.png)

### 언체크 예외로 돌리기

![](https://velog.velcdn.com/images/wook2pp/post/44698d3f-3644-47fb-bb0d-59b369d15842/image.png)

Repository 코드를 보면 SQLException을 catch해서 RuntimeException을 상속한 예외를 던지는 것을 볼 수 있다. 런타임 예외의 경우에 예외처리가 강요되지 않기 때문에, 그냥 예외를 던지면 된다.

```java
public class UnCheckedAppTest {

    @Test
    public void unchecked() throws Exception {
        Controller controller = new Controller();
        assertThatThrownBy(() -> controller.request()).isInstanceOf(RuntimeSQLException.class);
    }

    static class Controller {
        Service service = new Service();

        public void request() {
            service.logic();
        }
    }

    static class Service {
        Repository repository = new Repository();
        NetworkClient networkClient = new NetworkClient();

        public void logic() {
            repository.call();
            networkClient.call();
        }
    }

    static class NetworkClient {
        public void call() {
            throw new RuntimeConnectException("연결 실패");
        }
    }

    static class Repository {
        public void call() {
            try {
                runSQL();
            } catch (SQLException e) {
                throw new RuntimeSQLException(e);
            }
        }

        public void runSQL() throws SQLException {
            throw new SQLException("ex");
        }
    }

    static class RuntimeConnectException extends RuntimeException {
        public RuntimeConnectException(String message) {
            super(message);
        }
    }

    static class RuntimeSQLException extends RuntimeException {
        public RuntimeSQLException(Throwable cause) {
            super(cause);
        }
    }
}
```

### 체크 예외에서 인터페이스

체크 예외를 던지게 되면, 아래와 같이 인터페이스가 JDBC기술에 종속적인 SQLException에 의존하고 있음을 알 수 있다. 이를 런타임 에러로 바꾸어 순수한 인터페이스 코드로 유지해야 한다.
```java
public interface MemberRepositoryEx {
    Member save(Member member) throws SQLException;
    Member findById(String memberId) throws SQLException;
    void update(String memberId, int money) throws SQLException;
    void delete(String memberId) throws SQLException;
}
```

### 런타임 예외 만들기

런타임 예외를 상속받는 예외를 만들어 체크 예외가 발생하면 아래의 언체크 예외를 던진다.
이렇게 되면 서비스 로직에서도 체크 예외를 받지 않아도 된다.
```java
public class MyDbException extends RuntimeException {
    public MyDbException() {
        super();
    }

    public MyDbException(String message) {
        super(message);
    }

    public MyDbException(String message, Throwable cause) {
        super(message, cause);
    }

    public MyDbException(Throwable cause) {
        super(cause);
    }
}
```

### 데이터 접근 예외 세분화하기

![](https://velog.velcdn.com/images/wook2pp/post/c23411d0-e40b-4c9c-839b-469f3a0f23db/image.png)

MyDbException을 상속받아 새로운 객체를 만들어, DB 에러를 더 세분화하여 던질 수 있다.
또한 아래의 코드는 JDBC나 JPA에 종속적이지 않은 코드이기 때문에, 서비스 계층의 순수성을 유지할 수 있다.


```java
public class MyDuplicateKeyException extends MyDbException {
    public MyDuplicateKeyException() {
    }

    public MyDuplicateKeyException(String message) {
        super(message);
    }

    public MyDuplicateKeyException(String message, Throwable cause) {
        super(message, cause);
    }

    public MyDuplicateKeyException(Throwable cause) {
        super(cause);
    }
}
```

### 스프링 예외 추상화

스프링은 데이터 접근 계층에 대한 예외를 일관되게 정리하여 제공한다.
아래의 사진과 같이 스프링이 제공하는 예외는 가장 상위에 RuntimeException을 상속받고 있는 런타임 예외이다.

DataAccessException은 두가지로 나뉘는데,
- transient : 쿼리 타임아웃같은 오류로 다시 시도할 때 성공할 수 있다.
- non- transient : SQL 문법에 오류가 있거나 위배가 있어 같은 요청을 해도 같은 오류가 발생한다

![](https://velog.velcdn.com/images/wook2pp/post/a767542e-5d1c-4aa5-a9a9-0cdfc5f666c9/image.png)


### 스프링 예외 변환기

translate() 메서드에 첫번째 파라미터는 읽을 수 있는 설명, 실행한 sql, SQLException 을 전달하면 스프링 데이터 접근 계층 예외로 변환해서 예외를 반환한다.

```java
            SQLErrorCodeSQLExceptionTranslator exTranslator = 
            new SQLErrorCodeSQLExceptionTranslator(dataSource);
            DataAccessException resultEx = exTranslator.translate("select", sql, e);
```