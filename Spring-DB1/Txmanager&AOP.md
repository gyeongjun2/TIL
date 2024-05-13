# 스프링과 문제 해결

### 애플리케이션 구조

- 가장 단순하고 많이 사용되는 방법은 역할에 따라 3가지 계층으로 나누는 것이다.

![Untitled](%E1%84%89%E1%85%B3%E1%84%91%E1%85%B3%E1%84%85%E1%85%B5%E1%86%BC%E1%84%80%E1%85%AA%20%E1%84%86%E1%85%AE%E1%86%AB%E1%84%8C%E1%85%A6%20%E1%84%92%E1%85%A2%E1%84%80%E1%85%A7%E1%86%AF%207392e81c68f64379837ebac14cdb6e97/Untitled.png)

이런식으로 3계층으로 나누는데 하나씩 살펴보면

1. **프레젠테이션 계층**
    - UI 관련 처리 담당
    - 웹 요청과 응답
    - 사용자 요청 검증
    - 주 사용 기술로 서블릿, 스프링 MVC 등
2. **서비스 계층**
    - 비즈니스 로직을 담당
    - 서비스 계층에서는 가급적 순수 자바 코드로 작성
3. **데이터 접근 계층**
    - 실제 DB에 접근하는 코드
    - JDBC, JPA, File, …

→ 3가지 계층 중 가장 중요한 곳은 **서비스 계층**이다. 따라서 서비스 계층은 최대한 순수하게 가져가야한다. 즉 특정 기술에 종속적이지 않도록 개발해야 한다. (앞서 공부했던 멤버서비스V1 생각하면됨)

스프링을 사용하면 서비스 계층을 순수하게 유지하면서 다양한 문제들을 해결할 수 있는 방법을 제시하고 있다. → 스프링을 사용하자.

### 트랜잭션 추상화

- 서비스 계층에서 JDBC 기술에 의존하고 있다고 가정하면 나중에 JPA같은 데이터로 접근 기술을 변경할때 서비스 계층의 트랜잭션 관련 코드를 모두 수정해야 할것이다.

이 문제를 해결하려면? → 트랜잭션을 추상화

```java
public interface TxManager {
	begin();
	commit();
	rollback();
}
```

- 즉 서비스 계층은 인터페이스에 의존하고 의존성 주입(DI)를 사용하여 단일 책임 원칙을 지킬 수 있게 되는 것임.

스프링 트랜잭션 추상화의 핵심은 `PlatformTransactionManager` 인터페이스이다.

### 트랜잭션 동기화

스프링에서 트랜잭션 매니저는 2가지 역할을 한다.

1. 트랜잭션 추상화
2. 리소스 동기화

**리소스 동기화?**

- 트랜잭션을 유지하려면 트랜잭션의 시작부터 끝까지 같은 DB 커넥션을 유지해야 한다.
- 스프링은 **트랜잭션 동기화 매니저**를 제공한다. 얘가 쓰레드 로컬을 사용해서 커넥션을 동기화해준다.
- 트랜잭션 동기화 매니저를 통해 커넥션을 획득할 수 있고 파라미터로 커넥션을 전달할 필요가 없다.

**동작 방식**

1. 트랜잭션 매니저가 DataSource를 통해 커넥션을 생성하고 트랜잭션 시작
2. 트랜잭션 매니저는 시작된 커넥션을 트랜잭션 동기화 매니저에 저장
3. 리포지토리는 트랜잭션 동기화 매니저에 보관된 커넥션을 꺼내 사용한다.
4. 트랜잭션이 종료되면 트랜잭션 매니저는 트랜잭션 동기화 매니저에 보관된 커넥션을 통해 트랜잭션 종료 후 커넥션을 닫는다.

### 트랜잭션 템플릿

트랜잭션 로직을 살펴보면 같은 패턴이 반복되는 것이 보인다.

```java

TransactionStatus status = transactionManager.getTransaction(new
DefaultTransactionDefinition());
try {
	
		bizLogic(fromId, toId, money);	//비즈니스 로직
		transactionManager.commit(status); 
} catch (Exception e) {
		transactionManager.rollback(status); 
throw new IllegalStateException(e);
}
```

- 위 코드를 살펴보면 변하는것은 비즈니스 로직을 제외한 나머지만 반복됨. 따라서 템플릿 콜백 패턴을 활용하여 반복 문제를 해결한다.

```java
public class TransactionTemplate {
		private PlatformTransactionManager transactionManager;
		public <T> T execute(TransactionCallback<T> action){}
		void executeWithoutResult(Consumer<TransactionStatus> action){}
}
```

- 위와 같은 트랜잭션 템플릿 클래스로 콜백 패턴을 적용한다.
- execute() : 응답 값이 있을 때 사용
- executeWithoutResult() : 응답 값이 없을 때 사용

```java
public class MemberServiceV3_2 {
    private final TransactionTemplate txTemplate; //트랜잭션 템플릿 가져오기
    private final MemberRepositoryV3 memberRepository;

    //생성자로 PlatformTransactionManager, MemberRepositoryV3 주입
    public MemberServiceV3_2(PlatformTransactionManager transactionManager,
                             MemberRepositoryV3 memberRepository) {
        this.txTemplate = new TransactionTemplate(transactionManager);  //외부에서 매니저 주입 받아와서 트랜잭션 템플릿에 적용
        this.memberRepository = memberRepository;
    }
    public void accountTransfer(String fromId, String toId, int money) throws
            SQLException {
            //트랜잭션 템플릿 사용
        txTemplate.executeWithoutResult((status) -> {
            try {
                //비즈니스 로직
                bizLogic(fromId, toId, money);
            } catch (SQLException e) {
                throw new IllegalStateException(e);
            }
        });
    }
```

- 트랜잭션 템플릿을 사용하여 커밋, 롤백 없이 깔끔하게 코드가 유지된다.

### 트랜잭션 AOP

- 만약 서비스 계층에서 순수한 비즈니스 로직만 남기려면 어떻게 해야될까? 트랜잭션 템플릿을 사용해도 트랜잭션 로직이 남아있는 것을 확인할 수 있다. 이때 해결할 수 있는 방법이 **스프링 AOP**를 이용해 프록시를 도입하는 것이다.

프록시??

- 프록시는 쉽게 생각해서 중매하는 역할이라고 생각하면 된다.

```java
public class TransactionProxy {
		private MemberService target;
		public void logic() {

		TransactionStatus status = transactionManager.getTransaction();
		try {

		target.logic();
		transactionManager.commit(status); 
		} catch (Exception e) {
		transactionManager.rollback(status); 
		throw new IllegalStateException(e);
		}
	}
}
```

```java
public class Service {
		public void logic() {
		bizLogic(fromId, toId, money);
		}
}
```

프록시를 사용한다면 트랜잭션을 처리하는 객체와 비즈니스 로직을 처리하는 객체를 명확히 분리할 수 있다.

→트랜잭션 프록시가 트랜잭션 처리 로직을 처리하고 서비스 계층 로직을 대신 호출하여 서비스 계층에는 순수한 로직만 남길 수 있게되는 것임.

이러한 프록시를 어떻게 적용할까?

- 스프링에서는 스프링이 제공하는 AOP 기능을 사용하여 편리하게 적용할 수 있음. 트랜잭션 처리가 필요한 곳에`@Transactional` 애노테이션만 붙여주면 된다. 이제 스프링 트랜잭션 AOP는 이 애노테이션을 인식하여 트랜잭션 프록시를 적용시켜준다.
..