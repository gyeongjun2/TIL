# JDBC 이해

애플리케이션을 개발할 때 중요한 데이터는 데이터베이스에 보관한다.



애플리케이션 서버와 DB의 문제점은 DB가 변경될 때 발생한다. 예전에는 데이터베이스마다 커넥션을 연결하는 방법, SQL을 전달하는 방법, 그리고 결과를 응답받는 방법 등이 모두 달랐다.

→이러한 문제를 해결하기 위해 나온것이 **JDBC**라는 자바 표준이다.

### JDBC 표준 인터페이스

- JDBC(Java Database Connectivity)는 자바에서 데이터베이스에 접속할 수 있도록 하는 자바 API이다. JDBC는 데이터베이스에서 자료를 쿼리하거나 업데이트하는 방법을 제공한다.



대표적으로 3가지 기능을 표준 인터페이스로 정의한다.

- `java.sql.Connection` - 연결
- `java.sql.Statement`- SQL을 담은 내용
- `java.sql.ResultSet`- SQL 요청 응답

이러한 JDBC 표준 인터페이스를 통해 각각의 DB는 자신의 DB에 맞도록 구현체를 라이브러리로 제공한다. 이를 JDBC 드라이버라고 한다.

따라서 JDBC로 2가지 문제점을 해결할 수 있다.

1. DB 종류를 다른 것으로 변경하면 애플리케이션 서버의 데이터베이스 사용 코드도 변경해야 하는 문제 → 애플리케이션 로직은 JDBC 표준 인터페이스에만 의존하기 때문에 다른 DB를 사용할때는 JDBC 구현체만 변경하면 된다.(MySQL 드라이버에서 Oracle 드라이버로 갈아끼우는 느낌)
2. 각각의 DB마다 연결, 전달, 결과 응답 받는 방법을 새로 학습해야 하는 문제 → 개발자는 JDBC 표준 인터페이스 사용법만 학습하면 된다.

### JDBC와 데이터 접근 기술

- JDBC는 출시된지 오래된 기술이고 사용하는 법도 복잡하기 때문에 JDBC를 직접 사용하기 보다는 JDBC를 편하게 사용하는 다양한 기술을 이용한다. 대표적으로 SQL Mapper와 ORM이 있다.

**SQL Mapper**

장점

- JDBC를 편리하게 사용하도록 도와준다.
- SQL 응답 결과를 객체로 편리하게 변환해준다.
- JDBC의 반복 코드를 제거해준다.

단점

- 개발자가 직접 SQL을 작성해야한다.

대표 기술 - 스프링 JdbcTemplate, MyBatis …

ORM 기술

- ORM은 객체를 관계형 데이터베이스 테이블과 매핑해주는 기술이다. 이 기술 덕분에 개발자는 반복적인 SQL을 직접 작성하지 않고 ORM 기술이 개발자 대신 SQL을 동적으로 만들어 실행해준다.

대표 기술 - JPA, 하이버네이트, 이클립스링크 …

→ ORM은 쉬운 기술이 아니여서 깊이있는 학습이 필요함.

### 데이터베이스 연결

H2 데이터베이스를 이용해 연결시키기

```java
package hello.jdbc.connection;

public class ConnectionConst {
    public static final String URL = "jdbc:h2:tcp://localhost/~/test";
    public static final String USERNAME = "sa";
    public static final String PASSWORD = "";
}
```

```java
@Slf4j
public class DBConnectionUtil {
    public static Connection getConnection(){
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

1. 데이터베이스에 연결하기 위해 JDBC가 제공하는 `DriverManager.getConnection()`을 사용한다.
2. DriverManager가 라이브러리에 있는 데이터베이스 드라이버를 찾아 해당하는 드라이버의 커넥션을 반환해준다.

드라이버 커넥션은 JDBC 표준 커넥션 인터페이스인 `java.sql.Connection` 인터페이스를 구현하고 있다.



인터페이스를 jdbc가 정의하고 각각의 데이터베이스 드라이버는 JDBC Connection 인터페이스를 구현한 구현체들을 제공하는것임

정리하자면,

1. 애플리케이션 로직에서 커넥션이 필요하면 DriverManager.getConnection() 호출
2. DriverManger는 라이브러리에 등록된 드라이버 목록을 자동으로 스캔, 드라이버들에게 파라미터 정보를 넘겨서 커넥션을 획득할 수 있는지 확인 → URL, 이름, 비밀번호 등 추가정보…
3. 찾은 커넥션 구현체 클라이언트로 반환


### JDBC 개발 - 등록

먼저 데이터베이스에 Member 테이블 생성

```sql
create table member(
		member_id varchar(10),
		money integer not null default 0,
		primary key(member_id);
);
```

회원 클래스 생성

```java
@Data
public class Member {

    private String memberId;
    private int money;

    public Member(){
    }

    public Member(String memberId, int money) {
        this.memberId = memberId;
        this.money = money;
    }
}
```

이 클래스로 member 테이블에 저장하고 조회할 때 사용.

이제 JDBC를 통해 회원 객체를 데이터베이스에 저장

```java
@Slf4j
public class MemberRepositoryV0 {
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

    private void close(Connection con, Statement stmt, ResultSet rs) {
        if (rs != null) {
            try {
                rs.close();
            } catch (SQLException e) {
                log.info("error", e);
            }
        }
        if (stmt != null) {
            try {
                stmt.close();
            } catch (SQLException e) {
                log.info("error", e);
            }
        }
        if (con != null) {
            try {
                con.close();
            } catch (SQLException e) {
                log.info("error", e);
            }
        }
    }
    private Connection getConnection() {
        return DBConnectionUtil.getConnection();
    }
}
```

1. 커넥션 획득
- getConnection()을 만들어뒀던 DBConnectionUtil을 통해 DB 커넥션을 얻는다.
1. save()
- DB에 전달할 SQL을 정의. 데이터 등록을 위해 insert 사용.
- `con.prepareStatement(sql)` : DB에 전달할 SQL과 파라미터로 전달할 데이터를 준비
- `pstmt.setString(1, member.getMemberId())` : SQL의 첫번째 ? 값을 지정. 문자이므로 setString
- `pstmt.setInt(2, member.getMoney())` : SQL의 두번째 ? 값을 지정. 정수이므로 SetInt
- `pstmt.executeUpdate()` : Statement를 통해 준비된 SQL을 실제 데이터베이스에 전달. 이때 return 값으로 DB 행의 수를 반환한다. ex) 위 코드처럼 하면 1개의 행을 등록했으니까 1을 반환.

리소스 정리

- 쿼리 실행 후 항상 리소스를 정리해야한다. 리소스를 정리할때는 역순으로 한다. Connection → PreparedStatement를 사용했으므로 반대로 Pre 종료 → Con 종료
- finally 구문에 작성하자.

### JDBC 개발 - 조회

저장된 데이터 조회 기능

```java
public Member findById(String memberId) throws SQLException {
        String sql = "select * from member where member_id = ?";

        Connection con = null;
        PreparedStatement pstmt = null;
        ResultSet rs = null;

        try{
            con = getConnection();
            pstmt = con.prepareStatement(sql);
            pstmt.setString(1, memberId);

            rs = pstmt.executeQuery();

            if(rs.next()){
                Member member = new Member();
                member.setMemberId(rs.getString("member_id"));
                member.setMoney(rs.getInt("money"));
                return member;
            }else{
                throw new NoSuchElementException("member not found memberId= "+memberId);
            }
        } catch (SQLException e){
            log.error("db error", e);
            throw e;
        }finally {
            close(con, pstmt, rs);
        }
    }
```

- 데이터를 조회할때는 `executeQuery()`를 사용한다.
    - `executeQuery()`는 결과를 ResultSet에 담아서 반환한다.
- Resultset은 저장되있는 데이터 구조로 select 쿼리의 결과 순서대로 들어가있음
- rs은 초기에는 데이터를 가리키지 않기때문에 `rs.next()`를 통해 다음 커서(데이터가 있는곳)로 가리키게 해서 데이터를 조회해야됨

### JDBC 개발 - 수정

```java
public void update(String memberId, int money) throws SQLException {
        String sql = "update member set money=? where member_id=?";
        Connection con = null;
        PreparedStatement pstmt = null;
        try {
            con = getConnection();
            pstmt = con.prepareStatement(sql);
            pstmt.setInt(1, money);
            pstmt.setString(2, memberId);
            int resultSize = pstmt.executeUpdate();
            log.info("resultSize={}", resultSize);
        } catch (SQLException e) {
            log.error("db error", e);
            throw e;
        } finally {
            close(con, pstmt, null);
        }

    }
```

- `executeUpdate()`- 쿼리 실행 후 영향받은 행의 수 리턴

출처 : 김영한 선생님의 [스프링 -DB 1편] 강의를 듣고 정리한 내용입니다.