# 트랜잭션 이해

### 트랜잭션의 개념

데이터를 데이터베이스에 저장하는 이유가 무엇일까?

→ 데이터베이스는 트랜잭션을 지원하기 때문이다.

**트랜잭션이란?**

- 하나의 거래를 완전하게 처리하도록 보장해주는 것

**트랜잭션의 ACID 개념**

**Atomicity(원자성)** : 트랜잭션 내에서 실행한 작업은 하나의 작업인 것처럼 모두 성공하거나 모두 실패해야 한다.

**Consistency(일관성)** : 모든 트랜잭션은 일관성 있는 데이터베이스 상태를 유지해야 한다. - 무결성 제약 조건을 만족해야 한다.

**Isolation(격리성)** : 동시에 실행되는 트랜잭션들은 서로에게 영향을 미치지 않아야 한다. → 동시에 같은 데이터를 수정하지 못하도록 해야한다. 트랜잭션 격리 수준을 고려해야함.

**Durability(지속성)** : 트랜잭션의 성공적인 결과는 항상 기록되어야 한다. 문제가 중간에 발생해도 트랜잭션 내용을 복구해야 한다.

### 데이터베이스 연결 구조와 DB 세션

- 클라이언트가 WAS(Spring boot, …)나 DB 접근 툴(H2 Console, …)을 이용하여 DB 서버에 접근 가능하다. 이때 클라이언트는 DB 서버에 연결을 요청하고 커넥션을 맺는다. 그리고 DB 서버는 내부에서 세션을 생성한다. → 이 세션으로 커넥션을 통한 요청을 실행하게 되는 것이다.
- 세션은 트랜잭션을 시작하고 커밋, 롤백을 통해 트랜잭션을 종료한다. 후에 새로운 트랜잭션을 다시 시작할 수 있다.
- 사용자가 커넥션을 닫거나 DBA가 세션을 강제종료하면 세션은 종료된다.
- 커넥션 풀이 10개의 커넥션을 생성하면 세션도 10개가 만들어짐

### 트랜잭션 적용

**DB 트랜잭션을 이용하여 비즈니스 로직 설계**

- 트랜잭션은 비즈니스 로직이 있는 서비스 계층에서 실시해야 한다. → 비즈니스 로직에서 문제가 발생하면 해당 로직을 롤백해야되기 때문
- 트랜잭션을 시작하기 위해 **커넥션 필요**. 따라서 서비스 계층에서 커넥션을 생성하고 트랜잭션 커밋 이후 커넥션을 종료한다.
- DB 트랜잭션을 사용하는 동안은 **같은 커넥션을 유지**해야한다.

→ 클라이언트가 같은 커넥션을 유지하기 위한 가장 단순한 방법은 커넥션을 파라미터로 전달하여 같은 커넥션이 사용되도록 유지하는 방법이 있다.

```java
public class MemberRepository{
	private final DataSource dataSource;
	
	public MemberRepository(DataSource dataSource){
	this.dataSource = dataSource;
	}
	...	
  ...
 
 //Connection을 파라미터로 받음
    public Member findById(Connection con, String memberId) throws SQLException {
        String sql = "select * from member where member_id = ?";

        PreparedStatement pstmt = null;
        ResultSet rs = null;

        try{
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
            JdbcUtils.closeResultSet(rs);
            JdbcUtils.closeStatement(pstmt);
        }
    }
    
        //Connection을 파라미터로 받음
    public void update(Connection con, String memberId, int money) throws SQLException {
        String sql = "update member set money=? where member_id=?";

        PreparedStatement pstmt = null;
        try {

            pstmt = con.prepareStatement(sql);
            pstmt.setInt(1, money);
            pstmt.setString(2, memberId);
            int resultSize = pstmt.executeUpdate();
            log.info("resultSize={}", resultSize);
        } catch (SQLException e) {
            log.error("db error", e);
            throw e;
        } finally {

            JdbcUtils.closeStatement(pstmt);
        }
    }
    ...
    ...
 }
```

- 위 코드를 살펴보면 파라미터로 Connection을 받아 커넥션 유지를 시켜준다.
- 이때 repository에서 커넥션을 닫아버리면 안된다. 왜냐면 이후에도 계속 사용하기 때문에 서비스 로직이 끝날 때 트랜잭션을 종료하고 커넥션을 닫아야 한다.

```java

/**
 * 트랜잭션 - 파라미터 연동, 풀을 고려한 종료
 * **/
@RequiredArgsConstructor
@Slf4j
public class MemberServiceV2 {

    private final DataSource dataSource;
    private final MemberRepositoryV2 memberRepository;

    public void accountTransfer(String fromId, String toId, int money) throws SQLException {

        Connection con = dataSource.getConnection();
        try{
            con.setAutoCommit(false);//트랜잭션 시작
            //비즈니스 로직
            bizLogic(con, fromId, toId, money);
            con.commit(); //성공시 커밋 / 트랜잭션 종료
        }catch (Exception e){
            con.rollback(); //실패시 롤백 / 트랜잭션 종료
            throw new IllegalStateException(e);
        }finally {
            if(con!=null){
                try{
                    con.setAutoCommit(true);    //커넥션 풀 고려해서 true로 바꿔주고 close
                    con.close();    // 커넥션 풀 반환
                }catch(Exception e){
                    log.info("error", e);

                }
            }
        }

    }

    private void bizLogic(Connection con, String fromId, String toId, int money) throws SQLException {
        Member fromMember = memberRepository.findById(con, fromId);
        Member toMember = memberRepository.findById(con, toId);

        memberRepository.update(con, fromId, fromMember.getMoney()- money);
        if (toMember.getMemberId().equals("ex")){
            throw new IllegalStateException("이체중 예외 발생");
        }
        memberRepository.update(con, toId, toMember.getMoney()+ money);
    }

}
```

`Connection con = dataSource.getConnection();` 를 통해 커넥션 생성

`con.setAutoCommit(false);` 를 통해 오토커밋을 끄고 트랜잭션을 시작한다.(기본값이 true임)

후에 `bizLogic`을 통해 이체를 진행하고 정상적으로 진행되었다면 `commit`을 진행한다.

위와 같이 DB 트랜잭션을 적용하려면 서비스 로직이 복잡해진다. 이러한 문제를 해결하기 위해 스프링을 사용하자

**김영한 선생님의 [스프링 - DB 1편] 강의를 듣고 정리한 내용입니다.**