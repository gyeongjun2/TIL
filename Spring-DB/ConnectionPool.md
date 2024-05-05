# ConnectionPool

### 커넥션 풀

- 애플리케이션에서 DB 커넥션을 획득하기 위해서는 매번 조회 → 연결 → 커넥션 반환 이러한 과정이 이루어진다.
- 항상 이런 복잡한 로직을 거쳐 새로 커넥션을 만드는것은 시간과 리소스 모두 많이 소비되기 때문에 이러한 문제를 해결하는 게 바로 **커넥션 풀**이다.

**커넥션 풀을 사용하면 이제 DB 드라이버를 통해 커넥션을 얻는것이 아닌 커넥션 풀에서 클라이언트가 꺼내쓰는 것이다.**

→ 커넥션을 커넥션 풀에 요청하면 커넥션 하나를 반환해주고 다 사용한다면 클라이언트는 커넥션을 커넥션 풀에 반환해야한다.(커넥션 종료X)

대표적인 커넥션 풀 오픈소스 commons-dbcp2, tomcat-jdbc pool, HikariCP 등이 있는데 그냥 HikariCP가 최고다

### DataSource

- 커넥션을 얻는 방법은 JDBC DriverManager를 사용하거나 커넥션 풀(조회)을 이용하는 방법이 있다.
- 커넥션을 얻는 방법을 바꾸려면 로직을 바꿔야되는데 이러한 문제점을 어떻게 해결하지?

**커넥션 획득 방법을 추상화하자** 

- `DataSource`라는 커넥션 획득을 추상화하는 인터페이스를 사용한다.
- `DataSource`의 핵심 기능은 **커넥션 조회**

→즉 직접적인 커넥션 풀에 의존하도록 코드를 설계하지 말고 `DataSource`인터페이스에만 의존하도록 코드를 설계하자.

참고 : `DriverManager`는 `DataSource`인터페이스를 사용하지 않음. 따라서 `DriverManager`를 쓰다가 다른걸로 쉽게 바꾸기 위해서는 `DriverManagerDataSource`라는 `DataSource`를 구현한 클래스를 이용하자

### DataSource 커넥션 풀 사용

DataSource를 통해 커넥션 풀 사용하기

```java
void dataSourceConnectionPool() throws SQLException, InterruptedException
//커넥션 풀링
    HikariDataSource dataSource = new HikariDataSource();
    dataSource.setJdbcUrl(URL);
    dataSource.setUsername(USERNAME);
    dataSource.setPassword(PASSWORD);
    dataSource.setMaximumPoolSize(10);
    dataSource.setPoolName("MyPool");

    useDataSource(dataSource);
    Thread.sleep(1000);
}

//설정과 사용의 분리
//DataSource만 주입받아서 URL, USERNAME 등 속성에 의존하지 않아도 됨.
// 그냥 dataSource.getConnection()만 호출 ㄱㄱ
private void useDataSource(DataSource dataSource) throws SQLException{
//단순히 getConnection()만 호출하면 됨.
    Connection con1 = dataSource.getConnection();
    Connection con2 = dataSource.getConnection();
    log.info("connection={}, class={}", con1, con1.getClass());
    log.info("connection={}, class={}", con2, con2.getClass());
}
```

**참고** - 히카리 공식 문서 확인

[https://github.com/brettwooldridge/HikariCP](https://github.com/brettwooldridge/HikariCP)

### DataSource 적용

```java
private final DataSource dataSource;

    public MemberRepositoryV1(DataSource dataSource) {
        this.dataSource = dataSource;
    }
        private void close(Connection con, Statement stmt, ResultSet rs) {
        JdbcUtils.closeResultSet(rs);
        JdbcUtils.closeStatement(stmt);
        JdbcUtils.closeConnection(con);
    }

    private Connection getConnection() throws SQLException{
        Connection con = dataSource.getConnection();
        log.info("get connection={}, class={}", con, con.getClass());
        return con;
    }
```

- 외부에서 `DataSource`를 주입받아 사용.
- `JdbcUtils` → 이걸 사용해서 커넥션을 편리하게 닫을 수 있다.

```java
 MemberRepositoryV1 repository;

    @BeforeEach
    void beforeEach(){
    
        //커넥션 풀링 : HikariProxyConnection -> JdbcConnection
        HikariDataSource dataSource = new HikariDataSource();
        dataSource.setJdbcUrl(URL);
        dataSource.setUsername(USERNAME);
        dataSource.setPassword(PASSWORD);
        repository = new MemberRepositoryV1(dataSource);

    }
    
    
    crud()
    ...
```

- `MemberRepositoryV1`은 `DataSource`의존관계 주입 필요
- 커넥션 풀 사용시 이제 con0 커넥션이 계속 재사용 될 것.
- 웹에서 동시 요청이 들어온다면 커넥션 풀의 커넥션을 많이 사용할 것이다.

**참고** - `DriverManagerDataSource`에서 `HikariDataSource`로 변경해도 Repository의 코드는 변경하지 않아도 됨. `DataSource`인터페이스에만 의존하고 있기 떄문.

**김영한 선생님의 [스프링 - DB 1편] 강의를 듣고 정리한 내용입니다.**