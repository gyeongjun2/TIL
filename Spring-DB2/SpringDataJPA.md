# 스프링 데이터 JPA

데이터 접근 기술로써 스프링 데이터 JPA를 사용하면 코딩량이 확연히 줄어들고 비즈니스 로직 이해가 쉬워진다.

### 스프링 데이터 JPA의 주요 기능

1. 공통 인터페이스 기능
2. 쿼리 메서드 기능

### 공통 인터페이스 기능

```java
public interface ItemRepositroy extends JpaRepository<Item, Long> {
}
```

- `JpaRepository` 인터페이스를 인터페이스 상속받고 제네릭에 관리할 엔티티, 엔티티ID를 주면 된다.
- 이렇게 되면 `JpaRepository`  가 제공하는 기본 CRUD 기능을 모두 사용할 수 있음

### 쿼리 메서드 기능

인터페이스에 메서드만 적어두면 메서드 이름을 분석해서 쿼리를 자동으로 만들어주고 실행해주는 기능 제공

```java
public interface MemberRepository extends JpaRepository<Member, Long> {
			List<Member> findByUsernameAndAgeGreaterThan(String username, int age);
}
```

물론 아무렇게 사용하는 것은 아니고 몇가지 규칙이 있다.

규칙에 대해서는 공식 문서를 참고하자.

[Spring Data JPA :: Spring Data JPA](https://docs.spring.io/spring-data/jpa/reference/#jpa.query-methods.query-creation)

또한 JPQL을 직접 작성하여 사용할 수도 있다.

- @Query 애노테이션을 통해 JPQL작성
- 이때는 메서드 이름 규칙을 따르지 않아도 됨

### 스프링 데이터 JPA적용

```java
implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
```

`build.gradle`에 추가

JpaItemRepository

```java
@Repository
@Transactional
@RequiredArgsConstructor
public class JpaItemRepositoryV2 implements ItemRepository {

    private final SpringDataJpaItemRepository repository;

    @Override
    public Item save(Item item) {
        return repository.save(item);
    }

    @Override
    public void update(Long itemId, ItemUpdateDto updateParam) {
            Item findItem = repository.findById(itemId).orElseThrow();
        findItem.setItemName(updateParam.getItemName());
        findItem.setPrice(updateParam.getPrice());
        findItem.setQuantity(updateParam.getQuantity());
    }

    @Override
    public Optional<Item> findById(Long id) {
        return repository.findById(id);
    }

    @Override
    public List<Item> findAll(ItemSearchCond cond) {
        String itemName = cond.getItemName();
        Integer maxPrice = cond.getMaxPrice();
        if (StringUtils.hasText(itemName) && maxPrice != null) {
            
            return repository.findItems("%" + itemName + "%", maxPrice);
        } else if (StringUtils.hasText(itemName)) {
            return repository.findByItemNameLike("%" + itemName + "%");
        } else if (maxPrice != null) {
            return repository.findByPriceLessThanEqual(maxPrice);
        } else {
            return repository.findAll();
        }
    }
}
```

이 레포지토리는 `ItemRepository`를 구현한다. 또한 `SpringDataJpaItemRepository`를 사용한다.

itemService → jpaItemRepositoryV2 → springDataJpaItemRepositroy(프록시 객체)

이런식으로 중간에서 `JpaRepository`가 어댑터 역할을 해서 `ItemRepository` 인터페이스를 그대로 유지할 수 있고 `ItemService`의 코드를 변경하지 않아도 된다.

김영한 선생님의 [스프링 - DB 2편] 강의를 듣고 정리한 내용입니다.