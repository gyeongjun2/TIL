# Annotation이 정말 많아

Spring mvc에서 사용하는 애노테이션은 엄청 많다. 이런 애노테이션들을 하나하나 다 알기는 쉽지 않아보인다. 그래서 강의를 들으면서 주로 쓰이는 애노테이션들을 정리해봤다.

## `@RequestMapping`이 뭘까?

- 스프링에서 제공하는 애노테이션 기반 **컨트롤러**이다.
- `@RequestMapping`은 요청 정보를 매핑하는데 예를들어 `@RequestMapping(”/test/spring”)` 이러한 요청 정보에서 해당 URI이 호출되면 이 메서드가 호출되는 방식이다.(메서드 이름은 아무거나 괜찮다.)
- 우선순위가 높은 핸들러 매핑과 핸들러 어뎁터가 `RequestMappingHandlerMapping`, `RequestMappingHandlerAdapter`인데 `RequestMapping`을 따서 만든 이름이다. 이것이 스프링에서 주로 애노테이션 기반 컨트롤러를 지원하는 핸들러 매핑과 어뎁터이다.

핸들러를 조회하는 `RequestMappingHandlerMapping`은 스프링 빈 중에 `@RequestMapping` 또는 `@Controller`가 클래스레벨에 붙어 있는 경우 매핑 정보로 인식함.

`@RequestMapping` → `@GetMapping`, `@PostMapping`가능! (다른것도 다 됨. PUT, PATCH 등등)

→ 이게 무슨 말이냐? HTTP method도 구분 할 수 있다는 것. 요청 방식에 따라 매핑을 시켜줄 수 있다. 

주의 : `@RequestMapping`에 method 속성으로 아무것도 지정하지 않으면 HTTP 메서드와 무관하게 호출됨.(모두 허용-GET,POST,PUT,PATCH,DELETE)

## `@RequestParam`과 `@ModelAttribute`

`@RequestParam`을 사용하여 HTTP 요청 파라미터를 받을 수 있다. 즉 `@RequestParam(”username”)`은 `request.getParameter(”username”)`과 거의 동일하다고 보면 된다. → 쿼리 값을 뽑아내는 것?

```java
    @PostMapping("/add")
    public String addItemV1(@RequestParam String itemName,
                       @RequestParam int price,
                       @RequestParam Integer quantity,
                       Model model){
        Item item = new Item();
        item.setItemName(itemName);
        item.setPrice(price);
        item.setQuantity(quantity);
        itemRepository.save(item);

        model.addAttribute("item", item);
        return "basic/item";
    }
```

`@ModelAttribute`는 `@RequestParam`으로 변수를 하나하나 받는 과정을 줄인 요청 파라미터를 처리하는 방식이다. `@ModelAttribute`는 파라미터의 객체를 생성하고 요청 파라미터의 값을 프로퍼티 접근법으로 설정해준다. 

```java
    @PostMapping("/add")
    public String addItemV2(@ModelAttribute("item") Item item, Model model){

        itemRepository.save(item);
        // model.addAttribute("item", item); 자동 추가가 되기때문에 생략 가능
        return "basic/item";
    }
```

@ModelAttribute를 써서 코드를 간략하게 만듬. 

또한 `@ModelAttribute`는 Model도 추가해주는 기능을 가지고 있다.`@ModelAttribute`로 지정한 객체를 자동으로 모델에 넣어준다.

## `@RestController`

@Controller는 반환값이 String이면 뷰 이름으로 인식된다. 따라서 뷰를 찾고 뷰가 랜더링 되는데 @RestController는 반환 값으로 뷰를 찾지 않고 HTTP 메시지 바디에 바로 입력한다.

## `@PathVariable`

경로변수를 사용하여 매칭되는 부분을 조회한다.

```java
    @GetMapping("/{itemId}")
    public String item(@PathVariable long itemId, Model model){ //경로변수 지정
	      Item item = itemRepository.findById(itemId); // 경로변수로 가져온 id를 통해 item 찾기
        model.addAttribute("item", item); // model에 item이란 이름으로 item 객체 저장
        return "basic/item"; // basic/item에 있는 html에 던져주기
    }
```