# 자바 예외

### 예외 계층

- 자바에서 Throwable이라는 최상위 예외가 있고 하위에 `Exception`과 `Error`가 있다.
- 이때 Error는 애플리케이션에서 복구가 불가능한 시스템 예외로 이 예외를 잡으려고 하면 안된다.
- 그럼 무엇을 잡아야 하나? → 애플리케이션 로직에 적용하는 예외는 `Exception`으로 크게 **체크 예외**와 **언체크 예외(런타임 예외)** 두가지로 나뉘는데 이 예외를 잡는 것임.

체크 예외라는 것은 `Exception`과 하위 예외를 모두 컴파일러가 체크하는 것인데 이때 `Exception` 하위에 있는 `RuntimeException`만 예외로 언체크 예외로 둔다.

→ `RuntimeException` 과 하위 예외들은 언체크 예외로 컴파일러가 체크하지 않는 예외임.

계층이 어떤식으로 이루어졌냐면

1. `Object`(예외도 객체이기 때문에 최상위 부모는 Object)
2. `Throwable`
3. `Exception`, `Error`
4. `Exception` → `SQLException`, `IOException`, `RuntimeException` , `Error` → `OutOfMemoryError`
5. `RuntimeException` → `NullPointerException`, `IllegalArgumentException`

### 예외  규칙

1. 예외는 잡아서 처리하거나 처리할 수 없으면 밖으로 던져야 한다.
2. 예외를 잡거나 던질때는 지정한 예외 뿐만 아니라 예외의 자식들도 함께 처리됨
    1. 예를들어 Exception을 catch로 잡으면 하위 예외들도 모두 잡음
    2. throw로 던지면 하위 예외도 모두 던짐

→ 만약 예외를 처리하지 못하고 계속 돌린다면 자바 main() 쓰레드의 경우는 시스템 종료, 웹 서비스인 경우는 WAS가 예외를 받아 클라이언트에게 개발자가 지정한 오류 페이지를 보여줌.

**체크 예외의 장단점**

체크 예외는 예외를 잡아서 처리할 수 없을때 예외를 밖으로 던지는 `throws`예외를 필수로 선언해야 한다. 안한다면 컴파일 오류 발생한다.

장점 : 실수로 예외를 누락하지 않도록 컴파일러를 통해 문제를 잡아주는 안전 장치 역할을 한다.

단점 : 너무 번거롭다. 또한 의존관계 문제가 발생한다.

언체크 예외의 장단점

언체크 예외는 예외를 잡아(`catch`)서 처리하지 않아도 `throws`를 생략할 수 있다.

장점 : 불필요한 언체크 예외를 무시할 수 있다. 체크 예외의 경우는 잡지않은 예외를 전부 throws 예외를 선언해야 하지만 언체크는 생략 가능하다. 이렇게 된다면 의존관계를 참조하지 않아도 되는 장점이 있다.

단점 : 실수로 예외를 누락할 가능성이 있다.

### 체크 예외 활용

그럼 체크 예외와 언체크 예외를 언제 어떻게 사용해야 할까?

1. 기본적으로 언체크(런타임) 예외를 사용하자
2. 체크 예외는 비즈니스 로직상 의도적으로 던지는 예외에만 사용하자. → 예외를 반드시 잡아서 처리해야 하는 문제일때(계좌 이체 실패 예외 등)만 사용한다.

**체크 예외의 문제점**

```java
public class CheckedAppTest {

    @Test
    void checked() {
        Controller controller = new Controller();
        assertThatThrownBy(() -> controller.request())
                .isInstanceOf(Exception.class);
    }

    static class Controller {
        Service service = new Service();

        public void request() throws SQLException, ConnectException {
            service.logic();
        }
    }

    static class Service {
        Repository repository = new Repository();
        NetworkClient networkClient = new NetworkClient();

        public void logic() throws SQLException, ConnectException {
            repository.call();
            networkClient.call();
        }
    }

    static class NetworkClient {
        public void call() throws ConnectException {
            throw new ConnectException("연결 실패");
        }
    }

    static class Repository {
        public void call() throws SQLException {
            throw new SQLException("ex");
        }
    }
}
```

- 위 코드를 살펴보면 리포지토리, 네트워크 로직에서 체크 예외가 발생하는데 서비스나 컨트롤러 로직에서 이를 해결할 수가 없다 → 복구 불가능한 예외
- 또한 체크예외여서 계속 메서드에 thorws를 선언해주고 있다. 이렇게 계속 선언해주면 의존관계 문제가 발생할 수 있다.(예를들어 jdbc에서 jpa로 바꾼다면? SQLException은 모두 JPAException으로 싹 바꿔줘야되는 문제 발생)  → 의존관계 문제

따라서 체크 예외를 사용하면 이러한 문제가 발생하기 때문에 런타임 예외를 사용하는 것이 대부분 좋다

**언체크(런타임) 예외로 교체**

```java
public class UnCheckedAppTest {

    @Test
    void unchecked() {
        Controller controller = new Controller();
        assertThatThrownBy(() -> controller.request())
                .isInstanceOf(Exception.class);
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

        public void logic(){
            repository.call();
            networkClient.call();
        }
    }
    static class NetworkClient {
        public void call()  {
            throw new RuntimeConnectException("연결 실패");
        }
    }
    static class Repository {
        public void call()  {
            try {
                runSQL();
            }catch (SQLException e){
                throw new RuntimeSQLException(e);
            }
        }
        private void runSQL() throws SQLException {
            throw new SQLException("ex");
        }
    }
    static class RuntimeConnectException extends RuntimeException{
        public RuntimeConnectException(String message) {
            super(message);
        }
    }

    static class RuntimeSQLException extends RuntimeException{
        public RuntimeSQLException() {
        }

        public RuntimeSQLException(Throwable cause) {
            super(cause);
        }
    }
}
```

- 여기서는 리포지토리에서 체크 예외인 SQLException이 발생하면 런타임 예외인 RuntimeSQLException으로 전환하고 예외를 던지게 설정했다.
- 언체크 예외를 사용함으로써 해당 객체가 처리할 수 없는 예외는 무시하고 강제로 의존하지 않아도 되게 되었다. → `throws`를 생략함

런타임 예외는 문서화를 잘해야 한다.

### 예외 포함과 스택 트레이스

예외를 전활할 때는 꼭 **기존 예외를 포함**하여야 한다.

```java
@Test
void printEx() {
		Controller controller = new Controller();
		try {
		controller.request();
		} catch (Exception e) {
			log.info("ex", e);
		}
}
```

- 로그 출력할때 마지막 파라미터의 예외를 넣어주면 로그에 스택 트레이스를 출력할 수 있다.
    - `log.info(”message={}”, “msg”, e)` 이런식으로 마지막에 예외(e)를 넣어준다.

```java
public void call() {
		try {
		runSQL();
		} catch (SQLException e) {
			throw new RuntimeSQLException(e);
		}
}
```

이런식으로 기존 예외를 포함해주어야 한다.

**김영한 선생님의 [스프링 - DB 1편] 강의를 듣고 정리한 내용입니다.**