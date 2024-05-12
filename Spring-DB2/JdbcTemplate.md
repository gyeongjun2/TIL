# JdbcTemplate

JdbcTemplate는 SQL을 직접 사용하는 경우 좋은 데이터 접근 방법이다. JdbcTemplate는 JDBC를 매우 편리하게 사용할 수 있도록 도와준다.

JdbcTemplate의 장점

- 설정의 편리함
    - JdbcTemplate는 `spring-jdbc` 라이브러리에 포함되어 있는데 이 라이브러리는 JDBC를 사용할 때 기본으로 사용되는 라이브러리이다.
- 반복 문제 해결
    - JdbcTemplate는 템플릿 콜백 패턴을 사용하여 JDBC를 직접 사용할 때 발생하는 대부분의 반복 작업을 처리해준다.
    - 개발자는 SQL을 작성하고 전달할 파라미터를 정의하고 응답값을 매핑해주기만 하면 된다.
        - 커넥션 획득, statement를 준비하고 실행, 커넥션, pstst, rs종료, 커넥션 동기화 등등 모두 처리해줌

단점

- 동적 SQL을 해결하기 어려움

`build.grade`에 `implementation 'org.springframework.boot:spring-boot-starter-jdbc'` 을 추가하면 된다. 별도의 추가 설정은 없다.

### JdbcTemplate 적용

리포지토리를 메모리에서 JdbcTemplate로 변경해보자.

```java
public interface ItemRepository {

    Item save(Item item);

    void update(Long itemId, ItemUpdateDto updateParam);

    Optional<Item> findById(Long id);

    List<Item> findAll(ItemSearchCond cond);

}
```

```java
@Slf4j
public class JdbcTemplateItemRepositoryV1 implements ItemRepository {

    private final JdbcTemplate template;

    //생성자로 DataSource를 주입받아 템플릿 설정
    public JdbcTemplateItemRepositoryV1(DataSource dataSource) {
        this.template = new JdbcTemplate(dataSource);
    }

    @Override
    public Item save(Item item) {
        String sql = "insert into item(item_name, price, quantity) values(?,?,?)";
        KeyHolder keyHolder = new GeneratedKeyHolder();
        template.update(connection -> {
            //자동 증가 키
            PreparedStatement ps = connection.prepareStatement(sql, new String[]{"id"});
            ps.setString(1, item.getItemName());
            ps.setInt(2, item.getPrice());
            ps.setInt(3,item.getQuantity());
            return ps;
        }, keyHolder);

        long key = keyHolder.getKey().longValue();
        item.setId(key);
        return item;
    }

    @Override
    public void update(Long itemId, ItemUpdateDto updateParam) {
        String sql = "update item set item_name=?, price=?, quantity=? where id=?";
        template.update(sql,
                updateParam.getItemName(),
                updateParam.getPrice(),
                updateParam.getQuantity(),
                itemId);
    }

    @Override
    public Optional<Item> findById(Long id) {
        String sql = "select id, item_name, price, quantity from item where id=?";
        try {
            Item item = template.queryForObject(sql, itemRowMapper(), id);
            return Optional.of(item);
        }catch (EmptyResultDataAccessException e){
            return Optional.empty();
        }
    }

    @Override
    public List<Item> findAll(ItemSearchCond cond) {
        String itemName = cond.getItemName();
        Integer maxPrice = cond.getMaxPrice();
        String sql = "select id, item_name, price, quantity from item";
        //동적 쿼리
        if (StringUtils.hasText(itemName) || maxPrice != null) {
            sql += " where";
        }
        boolean andFlag = false;
        List<Object> param = new ArrayList<>();
        if (StringUtils.hasText(itemName)) {
            sql += " item_name like concat('%',?,'%')";
            param.add(itemName);
            andFlag = true;
        }
        if (maxPrice != null) {
            if (andFlag) {
                sql += " and";
            }
            sql += " price <= ?";
            param.add(maxPrice);
        }
        log.info("sql={}", sql);
        return template.query(sql, itemRowMapper(), param.toArray());
    }

    private RowMapper<Item> itemRowMapper() {
        return (rs, rowNum) -> {
            Item item = new Item();
            item.setId(rs.getLong("id"));
            item.setItemName(rs.getString("item_name"));
            item.setPrice(rs.getInt("price"));
            item.setQuantity(rs.getInt("quantity"));
            return item;
        };
    }
}
```

- `ItemRepository`를 구현한 JdbcTemplate리포지토리이다.
- JdbcTemplate는 데이터베이스 정보가 있는 데이터소스(DataSource)가 필요하다.
    - 생성자에서 데이터소스 주입받음

`findAll()` 메서드에서 동적 쿼리를 구성하고 있는데 이러한 부분을 MyBatis를 사용하면 동적 쿼리를 쉽게 작성할 수 있게 된다.

구성정보도 바꿔주어야 한다.

```java
@Configuration
@RequiredArgsConstructor
public class JdbcTemplateV1Config {

    private final DataSource dataSource;

    @Bean
    public ItemService itemService() {
        return new ItemServiceV1(itemRepository());
    }

    @Bean
    public ItemRepository itemRepository() {
        return new JdbcTemplateItemRepositoryV1(dataSource);
    }

}
```

ItemRepository 구현체로 JdbcTemplateItemRepositoryV1을 사용하고 dataSource를 주입한다.

그럼 dataSource의 정보는 어떻게?

`application.properties` 에서

```java
spring.datasource.url=jdbc:h2:tcp://localhost/~/test
spring.datasource.username=sa
spring.datasource.password=
```

이렇게 설정해주면 스프링 부트가 설정을 확인하고 커넥션 풀과 dataSource, 트랜잭션 매니저를 스프링 빈으로 자동 등록한다.

또한 `ItemServiceApplication`에서 `@Import(JdbcTemplateV1Config.class)`로 구성정보 변경을 위해 바꿔주자

### JdbcTemplate - 이름 지정 파라미터

JdbcTemplate는 `NamedParameterJdbcTemplate`라는 이름을 지정해서 파라미터를 바인딩하는 기능을 제공한다.

`NamedParameterJdbcTemplate` 를 사용하는 이유는 기존 `JdbcTemplate`에서 SQL은 데이터의 순서대로 바인딩되는데 이때 데이터의 순서가 변경되면 다른 값이 바인딩되는 결과가 일어날 수 있다. 따라서 이러한 문제점을 제거한 기능이 `NamedParameterJdbcTemplate` 이다.

```java
@Slf4j
public class JdbcTemplateItemRepositoryV2 implements ItemRepository {

    private final NamedParameterJdbcTemplate template;

    //생성자로 DataSource를 주입받아 템플릿 설정
    public JdbcTemplateItemRepositoryV2(DataSource dataSource) {
        this.template = new NamedParameterJdbcTemplate(dataSource);
    }

    @Override
    public Item save(Item item) {
        String sql = "insert into item(item_name, price, quantity) " +
                "values(:itemName, :price, :quantity)";

        SqlParameterSource param = new BeanPropertySqlParameterSource(item);

        KeyHolder keyHolder = new GeneratedKeyHolder();
        template.update(sql, param, keyHolder);

        long key = keyHolder.getKey().longValue();
        item.setId(key);
        return item;
    }

    @Override
    public void update(Long itemId, ItemUpdateDto updateParam) {
        String sql = "update item set item_name=:itemName, price=:price, quantity=:quantity " +
                "where id=:id";

        SqlParameterSource param = new MapSqlParameterSource()
                .addValue("itemName", updateParam.getItemName())
                .addValue("price", updateParam.getPrice())
                .addValue("quantity", updateParam.getQuantity())
                .addValue("id", itemId);

        template.update(sql, param);

    }

    @Override
    public Optional<Item> findById(Long id) {
        String sql = "select id, item_name, price, quantity from item where id = :id";
        try {
            Map<String, Object> param = Map.of("id", id);
            Item item = template.queryForObject(sql, param, itemRowMapper());
                return Optional.of(item);
        } catch (EmptyResultDataAccessException e) {
                return Optional.empty();
        }
    }

    @Override
    public List<Item> findAll(ItemSearchCond cond) {
        Integer maxPrice = cond.getMaxPrice();
        String itemName = cond.getItemName();
        SqlParameterSource param = new BeanPropertySqlParameterSource(cond);
        String sql = "select id, item_name, price, quantity from item";
//동적 쿼리
        if (StringUtils.hasText(itemName) || maxPrice != null) {
            sql += " where";
        }
        boolean andFlag = false;
        if (StringUtils.hasText(itemName)) {
            sql += " item_name like concat('%',:itemName,'%')";
            andFlag = true;
        }
        if (maxPrice != null) {
            if (andFlag) {
                sql += " and";
            }
            sql += " price <= :maxPrice";
        }
        log.info("sql={}", sql);
        return template.query(sql, param, itemRowMapper());
    }
    private RowMapper<Item> itemRowMapper() {
        return BeanPropertyRowMapper.newInstance(Item.class); //camel 변환 지원
    }
}
```

- 위 코드를 살펴보면 `NamedParameterJdbcTemplate` 도 dataSource를 주입받아 생성한다.
    
    →JdbcTemplate에서는 이러한 방법을 많이 사용함
    
- save() 메서드에서는 SQL에 values에 `?`가 아닌 `:파라미터` 로 바인딩시킨다.
- 또한 `NamedParameterJdbcTemplate` 는 DB가 생성해주는 키를 쉽게 조회하는 기능도 제공한다.

파라미터를 전달하기 위해 Map처럼 key, value 한 쌍의 데이터 구조를 만들어 전달한다. 여기서 key는 `:파라미터` 이고 value는 해당 파라미터의 값이다.

→`template.update(sql, param, keyHolder)`로 파라미터를 전달한다.

이름 지정 바인딩에서 자주 사용되는 3가지 파라미터 종류

- `Map`
- `SqlParameterSource`
- `MapSqlParameterSource` (`SqlParameterSource`구현체)
    - Map과 유사하지만 SQL 타입을 지정하는 등의 기능 제공.
    - 메서드 체인을 통해 사용(update 메서드 확인)
- `BeanPropertySqlParameterSource` (`SqlParameterSource`구현체)
    - 자바빈 프로퍼티 규약을 통해 자동으로 파라미터 객체 생성(예를들어 getItemName()→itemName 이런식으로)
    - 키값은 파라미터의 이름, vlaue는 파라미터의 값

근데 그럼 가장 편한 `BeanPropertySqlParameterSource` 얘만 쓰면 되는거 아니냐? → 파라미터를 바인딩해야하는데 가져오는 Dto에서 그 값이 없다면 사용할 수 없어짐. 즉 설정해줘야되는 값(ID같은거)가 있으면 `MapSqlParameterSource`를 사용해야 된다.

### `BeanPropertyRowMapper` 사용

```java
private RowMapper<Item> itemRowMapper() {
return BeanPropertyRowMapper.newInstance(Item.class); //camel 변환 지원
}
```

`BeanPropertyRowMapper`는 resultset의 결과를 받아 자바빈 프로퍼티 규약에 맞춰 데이터를 변환한다. 

**AS**

예를 들어 sql문에 `select item_name` 으로 되어있을때 setItem_name()과 같은 메서드는 없기때문에 맞지가 않는다. 이럴때는 SQL문에 AS로 이름을 수정하자. `select item_name as itemName`