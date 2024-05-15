# JPA

### JPA - Java Persistence API

- 자바 진영의 ORM 기술 표준이다.

ORM이란? → Object Relational mapping으로 객체는 객체로, RDB는 RDB로 설계하는 것.

ORM 프레임워크가 중간에서 매핑한다.

### JPA 적용

`build.gradle`에 의존관계 추가

`implementation 'org.springframework.boot:spring-boot-starter-data-jpa’`

**객체 - 테이블 매핑**

```java
@Data
@Entity
public class Item {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "item_name", length = 10)
    private String itemName;
    private Integer price;
    private Integer quantity;

		//기본 생성자 넣어주기!!
    public Item() {
    }

    public Item(String itemName, Integer price, Integer quantity) {
        this.itemName = itemName;
        this.price = price;
        this.quantity = quantity;
    }
}
```

`@Entity` : JPA가 사용하는 객체라는 뜻임. 이 애노테이션이 붙은 객체를 JPA 앤티티라 한다.

`@Id` : 테이블 PK값과 매핑되는 값

`@GeneratedValue(strategy = GenerationType.*IDENTITY*)` : PK 생성 값을 DB에서 생성하는 IDENTITY 방식으로 사용(자동으로 id값 생성)

`@Column` : 객체의 필드를 테이블의 컬럼과 매칭. 테이블 이름을 지정해줄 수 있음

JPA에서는 **기본 생성자**가 필수이므로 꼭 넣어줘야됨.

**Repository**

```java
@Slf4j
@Repository
@Transactional
public class JpaItemRepository implements ItemRepository {

    private final EntityManager em;

    public JpaItemRepository(EntityManager em) {
        this.em = em;
    }

    @Override
    public Item save(Item item) {
        em.persist(item);
        return item;
    }

    @Override
    public void update(Long itemId, ItemUpdateDto updateParam) {
        Item findItem = em.find(Item.class, itemId);
        findItem.setItemName(updateParam.getItemName());
        findItem.setPrice(updateParam.getPrice());
        findItem.setQuantity(updateParam.getQuantity());
    }

    @Override
    public Optional<Item> findById(Long id) {
        Item item = em.find(Item.class, id);
        return Optional.ofNullable(item);
    }

    @Override
    public List<Item> findAll(ItemSearchCond cond) {
        String jpql = "select i from Item i";
        Integer maxPrice = cond.getMaxPrice();
        String itemName = cond.getItemName();
        if (StringUtils.hasText(itemName) || maxPrice != null) {
            jpql += " where";
        }
        boolean andFlag = false;
        if (StringUtils.hasText(itemName)) {
            jpql += " i.itemName like concat('%',:itemName,'%')";
            andFlag = true;
        }
        if (maxPrice != null) {
            if (andFlag) {
                jpql += " and";
            }
            jpql += " i.price <= :maxPrice";
        }
        log.info("jpql={}", jpql);
        TypedQuery<Item> query = em.createQuery(jpql, Item.class);
        if (StringUtils.hasText(itemName)) {
            query.setParameter("itemName", itemName);
        }
        if (maxPrice != null) {
            query.setParameter("maxPrice", maxPrice);
        }
        return query.getResultList();
    }
}
```

- 위 코드에서 생성자를 통해 `EntityManager`를 주입받는다. JPA의 모든 동작은 `EntityManager`를 통해 이루어진다.
- JPA의 모든 데이터 변경은 트랜잭션 안에서 이루어져야 하므로 `@Transactional` 애노테이션을 클래스에 붙여줌.

save()에서 `persist()`은 JPA에서 **객체를 테이블에 저장**할때 사용하는 엔티티 매니저가 제공하는 메서드이다.

findAll()에서 jpql이 사용되었다.

**JPQL?**

- 객체지향 쿼리 언어로써 여러 데이터를 복잡한 조건으로 조회할 때 사용된다. JPQL은 엔티티 객체를 대상으로 SQL을 실행한다. 즉 from 다음에 `Item` 이라는 엔티티 객체 이름이 들어가게 된다. JPQL은 SQL과 문법이 거의 유사함. **but SQL은 테이블 대상, JPQL은 엔티티 객체 대상**

→ 결과적으로 JPQL을 실행하여 그 안에 포함된 엔티티 객체의 매핑 정보를 활용해 SQL을 만든다.

**Config**

```java
@Configuration
public class JpaConfig {

    private final EntityManager em;

    public JpaConfig(EntityManager em) {
        this.em = em;
    }

    @Bean
    public ItemService itemService() {
        return new ItemServiceV1(itemRepository());
    }
    @Bean
    public ItemRepository itemRepository(){
        return new JpaItemRepository(em);
    }
}
```

- JpaConfig 또한 em을 주입받아 사용