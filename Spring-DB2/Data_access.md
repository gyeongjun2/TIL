# 데이터 접근 기술

**DTO(data transfer object)**

- 데이터 전송 객체로 기능은 없고 데이터를 전달만 하는 용도로 사용하는 객체.
- 기능이 없어야 하는 것은 아니고 주 목적이 데이터를 전송하는 객체면 DTO임

**DTO를 어느 계층에 두는게 맞을까?**

→ 데이터(DTO)를 제공하는 마지막 부분을 확인하자. 즉 데이터를 더이상 다른 계층으로 안넘기고 그 계층에서 쓴다면 그 계층에서 사용하면 된다.

`@PostConstructor` vs `@EventListener(ApplicationReadyEvent.class)`

- `@EventListener(ApplicationReadyEvent.class)` 는 스프링 컨테이너가 완전히 초기화를 끝내고 실행 준비가 되었을 때 발생하는 이벤트. 해당 시점에 애노테이션이 붙은 메서드를 호출해준다(초기 데이터 설정할때 사용). 또한 AOP를 포함하여 스프링 컨테이너가 완전히 초기화 된 이후 호출되기 때문에 AOP가 적용되지 않는 문제점이 발생하지 않는다.
- `@PostConstructor` 도 비슷한 애노테이션이지만 `@PostConstructor` 는 AOP부분이 처리되지 않는 시점에서 호출될 가능성이 있다. 즉 `@Transactional`과 관련된 AOP가 적용되지 않고 호출될 수 있기 때문에 `@EventListener(ApplicationReadyEvent.class)` 를 사용하는 것이 좋다.

### `@Profile` ?

- 스프링 로딩 시점에 `application.properties`에 있는 `spring.profiles.active`속성을 읽어 프로필로 사용한다.
- 예를들어 로컬PC에서는 로컬에 저장된 데이터베이스에 접근해야 하고 운영 환경에서는 운영 데이터베이스에 접근해야 한다면 설정 정보가 달라야 한다. 또한 환경에 따라 다른 스프링 빈을 등록할 때 프로필을 사용한다.

```java
	@Bean
	@Profile("local")
	public TestDataInit testDataInit(ItemRepository itemRepository) {
		return new TestDataInit(itemRepository);
	}
```

**스프링 부트 프로필 공식 메뉴얼**

[Core Features](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.profiles)

### 기본 키 선택

데이터베이스 기본 키는 3가지 조건을 만족해야한다.

1. null값 허용 X
2. 유일해야 한다.
3. 변해선 안된다.

**테이블의 기본 키를 선택하는 2가지 방법**

- 자연 키(natural key)
    - 비즈니스에 의미가 있는 키
    - ex) 주민등록번호, 이메일, 전화번호
- 대리 키(surrogate key)
    - 비즈니스와 관련 없는 임의로 만들어진 키, 대체 키로도 불림
    - ex) 오라클 시퀀스, auto_increment, identity, 키생성 테이블 사용

자연 키보다는 대리 키를 **기본 키**로 권장한다.