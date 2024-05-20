# Querydsl

Querydsl이 무엇인가?

- 쿼리에 특화된 프로그래밍 언어
- 쿼리 조회를 추상화 하겠다! 라고 하면서 나온 것
- type-safe하게 만들어주는 프레임워크
- APT(Annotation Processor Tool)
- Q코드 생성을 위한 APT를 성정해줘야 한다.

**Querydsl-JPA**

Querydsl → JPQL → SQL 로 변환해줌

Querydsl 추가

```java
//build.gradle
//..
//dependencies
implementation 'com.querydsl:querydsl-jpa'
annotationProcessor "com.querydsl:querydsl-apt:${dependencyManagement.importedProperties['querydsl.version']}:jpa"
annotationProcessor "jakarta.annotation:jakarta.annotation-api"
annotationProcessor "jakarta.persistence:jakarta.persistence-api"
...
//Querydsl 추가, 자동 생성된 Q클래스 gradle clean으로 제거
clean {
delete file('src/main/generated')
}
```

build tool : gradle일때 Q타입 생성

Gradle → Tasks → build → clean 실행

Gradle → Tasks → other → compileJava 실행

build → generated → sources → annotationProcessor 하위에 QItem 확인

참고로 git에 포함하지 않는것이 좋기때문에 ignore 설정

build tool : IntelliJ IDEA일때 Q타입 생성

build → build project 하면 main에 generated 생성. Qitem 확인

### Querydsl적용하기

```java

@Repository
@Transactional
public class JpaItemRepositoryV3 implements ItemRepository {

    private final EntityManager em;
    private final JPAQueryFactory query;

    public JpaItemRepositoryV3(EntityManager em) {
        this.em = em;
        this.query = new JPAQueryFactory(em);
    }
		...
			
		...
}
```

- Querydsl을 사용하기 위해 `JPAQueryFactory`가 필요함.`JPAQueryFactory` 는 JPA 쿼리인 JPQL을 만들기 때문에 `EntityManager`도 필요하다.
- `JPAQueryFactory` 를 스프링 빈으로 등록하여 사용 가능

복잡한 쿼리 부분인 findAll에서 querydsl 사용

```java
public List<Item> findAllOld(ItemSearchCond cond) {

        String itemName = cond.getItemName();
        Integer maxPrice = cond.getMaxPrice();

        QItem item = new QItem("i");

        BooleanBuilder builder = new BooleanBuilder();
        if(StringUtils.hasText(itemName)){
            builder.and(item.itemName.like("%"+ itemName + "%"));
        }
        if(maxPrice!=null){
            builder.and(item.price.loe(maxPrice));
        }
        List<Item> result = query
                .select(item)
                .from(item)
                .where(builder)
                .fetch();
        return result;
    }
```

`BooleanBuilder`를 통해 where절의 조건들을 나열해주고 설정한다.

이 코드를 리팩토링해보면

```java
public List<Item> findAll(ItemSearchCond cond) {

        String itemName = cond.getItemName();
        Integer maxPrice = cond.getMaxPrice();

        List<Item> result = query
                .select(item)
                .from(item)
                .where(likeItemName(itemName), maxPrice(maxPrice))
                .fetch();

        return result;
    }

    private BooleanExpression likeItemName(String itemName){
        if(StringUtils.hasText(itemName)){
            return item.itemName.like("%" + itemName + "%");
        }
        return null;
    }
    private BooleanExpression maxPrice(Integer maxPrice){
        if(maxPrice != null){
            return item.price.loe(maxPrice);
        }
        return null;
    }
```

위와 같이 querydsl에서 where(a,b…)에 조건들을 차례대로 입력하여 AND 조건으로 처리한다. 또한 where절에 null이 들어간다면 그 조건은 무시된다.

**config 수정**

```java
@Configuration
@RequiredArgsConstructor
public class QuerydslConfig {

    private final EntityManager em;

    @Bean
    public ItemService itemService() {
        return new ItemServiceV1(itemRepository());
    }
    @Bean
    public ItemRepository itemRepository() {
        return new JpaItemRepositoryV3(em);
    }
}
```

`JpaItemRepositoryV3` 에 em을 주입

`Application`에 `@Import(QuerydslConfig.class)`로 수정

**결론**

- querydsl을 사용하여 동적 쿼리를 간단하게 작성 가능해진다.
- 또한 오타가 있다면 컴파일 시점에 알아낼 수 있다.

김영한 선생님의 [스프링 - DB 2편] 강의를 듣고 정리한 내용입니다.