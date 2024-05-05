# 싱글톤 컨테이너

## 싱글톤 컨테이너

지금까지 만들었던 스프링 없는 순수한 DI 컨테이너인 AppConfig는 요청을 할 때마다 객체를 생성했다. 이렇게 되면 만약 고객 트래픽이 초당 100개가 나오면 그에 따른 메모리 낭비가 엄청 심할 것.

→해결방안은 해당 객체가 딱 1개만 생성되고 공유하도록 설계하면 되는데 이게 싱글톤 패턴이다.

**싱글톤 패턴**

- 클래스의 인스턴스가 **딱 1개**만 생성되는 것을 보장하는 디자인 패턴
- 그래서 객체 인스턴스를 2개 이상 생성하지 못하도록 막아야 한다.
    - `private`생성자를 사용하여 외부에서 임의로 `new` 키워드를 사용하지 못하도록 막아야 한다

```java
package hello.core.singleton;

public class SingletonService {

        //1. static 영역에 객체를 딱 1개만 생성
        private static final SingletonService instance = new SingletonService();

        //2. public으로 열어서 객체 인스턴스가 필요하면 이 static 메서드를 통해서만 조회하도록 허용한다.
        public static SingletonService getInstance(){
        return instance;
    }

    //3. 생성자를 private으로 선언해서 외부에서 new 키워드를 사용한 객체 생성을 못하게 막는다.
    private SingletonService(){}

    public void logic(){
        System.out.println("싱글톤 객체 로직 호출");
    }

}
```

```java
@Test
    @DisplayName("싱글톤 패턴을 적용한 객체 사용")
    void singletonServiceTest(){
        SingletonService singletonService1 = SingletonService.getInstance();
        SingletonService singletonService2 = SingletonService.getInstance();

        System.out.println("singletonService1 = " + singletonService1);
        System.out.println("singletonService2 = " + singletonService2);

        assertThat(singletonService1).isSameAs(singletonService2);

    }
```

즉 싱글톤 패턴을 적용하면 고객의 요청이 올 때마다객체를 생성하는 것이 아니라, 이미 만들어진 객체를 공유해서 효율적으로 사용이 가능하다. 하지만 싱글톤 패턴은 몇가지 문제점이 있음

**싱글톤 패턴 문제점**

- 싱글톤 패턴을 구현하는 코드 자체가 많이 들어간다.
- 의존관계상 클라이언트가 구체 클래스에 의존한다. → DIP 위반!
- 클라이언트가 구체 클래스에 의존해서 OCP 원칙을 위반할 가능성이 높다.

### 싱글톤 컨테이너

근데 스프링 컨테이너는 싱글톤 패턴의 문제점을 해결하면서, 객체 인스턴스를 싱글톤(1개만 생성)으로 관리한다. 지금까지 학습한 스프링 빈이 바로 싱글톤으로 관리되는 빈이다.

- 스프링 컨테이너는 싱글톤 패턴을 적용하지 않아도, 객체 인스턴스를 싱글톤으로 관리한다.
- 스프링 컨테이너는 싱글톤 컨테이너 역할을 한다. 이렇게 싱글톤 객체를 생성하고 관리하는 기능을 **싱글톤 레지스트리**라 한다.
- 스프링 컨테이너의 이런 기능 덕분에 싱글톤 패턴의 모든 단점을 해결하면서 객체를 싱글톤으로 유지가 가능하다.
    - DIP, OCP, 테스트, private 생성자로 부터 자유롭게 싱글톤을 사용할 수 있다.

```java
@Test
    @DisplayName("스프링 컨테이너와 싱글톤")
    void springContainer(){
        //스프링 컨테이너 생성
        ApplicationContext ac = new AnnotationConfigApplicationContext(Appconfig.class);
        //호출한 고객들에게 동일한 memberService(인스턴스) 반환
        MemberService memberService1 = ac.getBean("memberService", MemberService.class);
        MemberService memberService2 = ac.getBean("memberService", MemberService.class);

        //참조값이 같은 것을 확인
        System.out.println("memberService1 = " + memberService1);
        System.out.println("memberService2 = " + memberService2);

        //memberService1 == memberService2
        assertThat(memberService1).isSameAs(memberService2);

    }
```

- 스프링 컨테이너 덕분에 고객 요청이 올 때마다 객체를 생성하는 것이 아니라 이미 만들어진 객체를 공유해서 효율적으로 재사용할 수 있다.
    

    

**싱글톤 방식의 주의점**

- 싱글톤 패턴이든, 스프링 같은 싱글톤 컨테이너를 사용하든, 객체를 인스턴스 하나만 생성해서 공유하는 싱글톤 방식은 여러 클라이언트가 하나의 같은 객체 인스턴스를 공유하기 때문에 싱글톤 객체는 상태를 유지(stateful)하게 설계하면 안됨
- **무상태(stateless)**로 설계해야 한다.
    - 특정 클라이언트에 의존적인 필드가 있으면 안됨
    - 특정 클라이언트가 값을 변경할 수 잇는 필드가 있으면 안됨
    - 가급적 읽기만 가능해야 한다.
    - 필드 대신에 자바에서공유되지 않는 지역변수, 파라메터, ThreadLocal 등을 사용해야 함

무상태를 유지하지 않을 경우 발생하는 문제점 예시

```java
package hello.core.singleton;

public class StatefulService {
 private int price; 
 public void order(String name, int price) {
 System.out.println("name = " + name + " price = " + price);
	 this.price = price;  
 }
 public int getPrice() {
 return price;
 }
}
```

```java
package hello.core.singleton;
import org.assertj.core.api.Assertions;
import org.junit.jupiter.api.Test;
import org.springframework.context.ApplicationContext;
import
org.springframework.context.annotation.AnnotationConfigApplicationContext;
import org.springframework.context.annotation.Bean;
public class StatefulServiceTest {
 @Test
 void statefulServiceSingleton() {
 ApplicationContext ac = new AnnotationConfigApplicationContext(TestConfig.class);
 
 StatefulService statefulService1 = ac.getBean("statefulService", StatefulService.class);
 StatefulService statefulService2 = ac.getBean("statefulService", StatefulService.class); //ThreadA: A사용자 10000원 주문
 
 statefulService1.order("userA", 10000);
 //ThreadB: B사용자 20000원 주문
 statefulService2.order("userB", 20000);
 //ThreadA: 사용자A 주문 금액 조회
 int price = statefulService1.getPrice();
 //ThreadA: 사용자A는 10000원을 기대했지만, 기대와 다르게 20000원 출력
 System.out.println("price = " + price);
 Assertions.assertThat(statefulService1.getPrice()).isEqualTo(20000);
 }
 static class TestConfig {
 @Bean
 public StatefulService statefulService() {
 return new StatefulService();
 }
 }
}
```

- StatefulService의 price 필드는 공유되는 필드. 근데 이걸 특정 클라이언트가 값을 변경할 수 있다
- 따라서 사용자A 주문금액은 10000원이 되야하는데 20000원이 나와버림 → 공유되는 필드가 있어서 이러한 문제 발생

이렇게 바꿔주자

```java
package hello.core.singleton;

public class StatefulService {

    public int order(String name, int price){
        System.out.println("name = " + name + " price = " + price);
        return price;
    }

}
```

- 멤버변수 제거하고 메서드가 return해주게 바꿈

```java
package hello.core.singleton;

import org.assertj.core.api.Assertions;
import org.junit.jupiter.api.Test;
import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;
import org.springframework.context.annotation.Bean;

import static org.junit.jupiter.api.Assertions.*;

class StatefulServiceTest {

    @Test
    void statefulServiceSingleton(){
        ApplicationContext ac = new AnnotationConfigApplicationContext(TestConfig.class);

        StatefulService statefulService1 = ac.getBean("statefulService",StatefulService.class);
        StatefulService statefulService2 = ac.getBean("statefulService",StatefulService.class);

        //ThreadA: A사용자 10000원 주문
        int priceA = statefulService1.order("userA", 10000);
        //ThreadB: B사용자 20000원 주문
        int priceB = statefulService2.order("userB", 20000);

        //ThreadA: 사용자A 주문 금액 조회
//        int price = statefulService1.getPrice();
        System.out.println("priceA = " + priceA);

//        Assertions.assertThat(statefulService1.getPrice()).isEqualTo(20000);
    }

    static class TestConfig{

        @Bean
        public StatefulService statefulService(){
            return new StatefulService();
        }
    }
}
```

이제 유저A가 값을 넘기면 바로 return해주기 때문에 공유되는 변수 없음

결론 : 공유필드 조심. 스프링 빈은 무상태로 설계

### @Configuration과 싱글톤

```java
@Configuration
public class AppConfig {
@Bean
public MemberService memberService() {
return new MemberServiceImpl(memberRepository());
}
@Bean
public OrderService orderService() {
return new OrderServiceImpl(memberRepository(), discountPolicy());
}
@Bean
public MemberRepository memberRepository() {
return new MemoryMemberRepository();
}
}
```

위의 AppConfig 코드를 보면 각각 다른 memberRepository()를 호출하여 결과적으로 2개의 MemoryMemberRepository를 생성하여 싱글톤이 깨진 것처럼 보인다. 이게 뭐지?

→ 스프링은 싱글톤 레지스트리기 때문에 스프링 빈이 싱글톤이 되도록 보장해줘야 한다. 따라서 스프링은 클래스의 바이트코드를 조작하는 라이브러리를 사용한다.

@Configuration을 적용한 AppConfig도 스프링 빈으로 등록된다. 더 자세히 말하자면 여기서 스프링이 CGLIB라는 바이트코드 조작 라이브러리를 사용하여 AppConfig 클래스를 상속받은 임의의 다른 클래스를 만들고 그 다른 클래스를 스프링 빈으로 등록하는 것이다.

결론 : 스프링 설정 정보는 항상 @Configuration을 사용하자. 싱글톤을 보장해준다.

김영한 선생님의 [스프링 - 기본편] 강의를 듣고 정리한 내용입니다.