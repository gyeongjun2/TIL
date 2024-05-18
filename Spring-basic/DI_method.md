# 의존관계 주입 방법

## 의존관계 자동 주입

의존관계 주입은 크게 4가지 방법이 있다.

- 생성자 주입
- 수정자 주입(setter 주입)
- 필드 주입
- 일반 메서드 주입

### 생성자 주입

- 생성자를 통해서 의존 관계를 주입 받는 방법
- 생성자 호출 시점에 딱 1번만 호출되는 것이 보장됨
- **불변, 필수** 의존관계에 사용

→ 불변은 한번 설정된 값을 임의로 수정할 수 없도록 막아두는 것.

### 수정자 주입

- setter라 불리는 필드의 값을 변경하는 수정자 메서드를 통해서 의존관계를 주입하는 방법
- **선택, 변경** 가능성이 있는 의존관계에 사용
- 자바빈 프로퍼티 규약의 수정자 메서드 방식을 사용하는 방법이다.

프로퍼티 규약?

- 필드의 값을 직접 변경하지 않고 setXxx, getXxx 라는 메서드를 통해 값을 읽거나 수정하는 규칙

```java
class Example{
private int age;
public void setAge(int age){
this.age = age;
	}
public int getAge(){
return age;
	}
}
```

### 옵션 처리

주입할 스프링 빈이 없어도 동작해야 할 때가 있다. 이때 `@Autowired`만 사용하면 `required`옵션의 기본값이`true`로 되어 있어서 자동 주입 대상이 없으면 오류가 발생한다.

자동 주입 대상을 옵션으로 처리하는 방법은 다음과 같다

- `@Autowired(required=false)` : 자동 주입할 대상이 없으면 수정자 메서드 자체가 호출 안됨
- `org.springframework.lang.@Nullable` : 자동 주입할 대상이 없으면 null이 입력됨
- `Optional<>` : 자동 주입할 대상이 없으면 Optional.empty가 입력된다.

```java
//호출 안됨
@Autowired(required = false)
public void steNoBean1(Member member){
System.out.println("setNoBean1 = " + member);
}
//null 호출
@Autowired
public void steNoBean2(@Nullable Member member){
System.out.println("setNoBean2 = " + member);
}
//Optional.empty 호출
@Autowired(required = false)
public void steNoBean3(Optional<Member> member){
System.out.println("setNoBean3 = " + member);
}
```

Member는 스프링 빈이 아님

`setNoBean1()`은 `@Autowired(required=false)`이므로 호출 자체가 안된다.

**생성자 주입을 선택하자.**

- 프레임워크에 의존하지 않고 순수한 자바 언어의 특징을 잘 살리는 방법임.
- 기본으로 생성자 주입을 사용하고, 필수 값이 아닌 경우에는 수정자 주입 방식을 옵션으로 부여하면 된다. 생성자 주입과 수정자 주입을 동시에 사용할 수 있다.
- 항상 생성자 주입을 선택하자.

final 키워드

- 생성자 주입을 사용하면 final 키워드를 사용할 수 있다. 따라서 혹시라도 생성자에서 값이 설정되지 않는 오류를 컴파일 시점에 막아준다.

결론 : 기본으로 생성자 주입을 선택하고 필수 값이 아닐 경우에는 수정자 주입 방식을 옵션으로 부여하자. 생성자 주입과 수정자 주입을 동시에 사용할 수 있다. 필드 주입은 되도록 사용하지 말자

### 롬복과 최신 트랜드

 롬복 라이브러리가 제공하는 `@RequiredArgsConstructor` 기능을 사용하면 final이 붙은 필드를 모아 생성자를 자동으로 만들어준다.`@Getter`, `@Settter`, `@RequiredArgsConstructor`

롬복을 사용하지 않은 코드

```java
@Component
public class OrderServiceImpl implements OrderService {
private final MemberRepository memberRepository;
private final DiscountPolicy discountPolicy;
public OrderServiceImpl(MemberRepository memberRepository, DiscountPolicy
discountPolicy) {
this.memberRepository = memberRepository;
this.discountPolicy = discountPolicy;
}
}
```

롬복 사용 코드

```java
@Component
@RequiredArgsConstructor
public class OrderServiceImpl implements OrderService {
private final MemberRepository memberRepository;
private final DiscountPolicy discountPolicy;
}
```

위 두 코드는 완전히 동일함. 롬복이 컴파일 시점에 생성자 코드를 자동으로 생성해준다.

→생성자를 딱 1개 두고 `@Autowired`를 생략하고 여기에 Lombok 라이브러리의 `@RequiredArgsConstructor` 를 함께 사용하면 코드를 깔끔하게 가져갈 수 있다.

### 롬복 적용법

`build.gradle`에 라이브러리 및 환경 추가

```java
plugins {
id 'org.springframework.boot' version '2.3.2.RELEASE'
id 'io.spring.dependency-management' version '1.0.9.RELEASE'
id 'java'
}
group = 'hello'
version = '0.0.1-SNAPSHOT'
sourceCompatibility = '11'
//lombok 설정 추가 시작
configurations {
compileOnly {
extendsFrom annotationProcessor
}
}
//lombok 설정 추가 끝

repositories {
mavenCentral()
}
dependencies {
implementation 'org.springframework.boot:spring-boot-starter'
//lombok 라이브러리 추가 시작
compileOnly 'org.projectlombok:lombok'
annotationProcessor 'org.projectlombok:lombok'
testCompileOnly 'org.projectlombok:lombok'
testAnnotationProcessor 'org.projectlombok:lombok'
//lombok 라이브러리 추가 끝
testImplementation('org.springframework.boot:spring-boot-starter-test') {
exclude group: 'org.junit.vintage', module: 'junit-vintage-engine'
}
}
test {
useJUnitPlatform()
}
```

1. Settings → plugin → lombok 검색 설치 실행
2. Settings → Annotation Processors 검색 → Enable annotation processing 체크
3. 임의 클래스 만들고 `@Getter`, `@Settter` 확인

### 조회 빈이 2개 이상 - 문제

`@Autowired` 는 타입으로 조회된다. 따라서 `ac.getBean(DiscountPolicy.class)` 이러한 코드와 유사하게 동작을 하는데 타입으로 조회하면 선택된 빈이 2개 이상일 때 문제가 발생한다. `DiscountPolicy` 하위 타입인 `FixDiscountPolicy`, `RateDiscountPolicy` 둘다 스프링 빈으로 선언한다면 이러한 문제가 발생한다.

### 해결 방법

- `@Autowired` 필드 명 매칭
- `@Qualifier` → `@Qualifier`끼리 매칭 → 빈 이름 매칭
- `@Primary` 사용

```java
@Component
@Qualifier("mainDiscountPolicy")
public class RateDiscountPolicy implements DiscountPolicy{}

@Component
@Qualifier("fixDiscountPolicy")
public class FixDiscountPolicy implements DiscountPolicy{}
```

→ 주입시 `@Qulifier`를 붙여주고 등록한 이름을 적어준다.

```java
@Autowired
public OrderServiceImpl(MemberRepository memberRepository,
 @Qualifier("mainDiscountPolicy") DiscountPolicy discountPolicy) {
 this.memberRepository = memberRepository;
 this.discountPolicy = discountPolicy;
}
```

`@Qulifier` 정리

1. `@Qulifier` 끼리 매칭
2. 빈 이름 매칭
3. 예외 발생

`@Primary` 사용

`@Primary` 는 우선순위를 정하는 방법이다. @Autowired시 여러 빈이 매칭되면 `@Primary` 가 우선권을 가진다.

```java
@Component
@Primary
public class RateDiscountPolicy implements DiscountPolicy{}

@Component
public class FixDiscountPolicy implements DiscountPolicy{}
```