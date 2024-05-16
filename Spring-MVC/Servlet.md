# 서블릿

## 스프링 부트에서 서블릿 환경 구성하기

`@ServletComponentScan`을 사용하여 스프링 부트에서 서블릿을 직접 등록해서 사용할 수있도록 세팅

```java
@ServletComponentScan // 서블릿 자동 등록
@SpringBootApplication
public class ServletApplication {

	public static void main(String[] args) {
		SpringApplication.run(ServletApplication.class, args);
	}

}
```

서블릿 코드를 등록하자

```java
package hello.servlet.basic;

import jakarta.servlet.ServletException;
import jakarta.servlet.annotation.WebServlet;
import jakarta.servlet.http.HttpServlet;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.io.IOException;

@WebServlet(name = "helloServlet", urlPatterns = "/hello")
public class HelloServlet extends HttpServlet {

    @Override
    protected void service(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        // 웹 브라우저가 서버에 request와 response를 만들어서 던져줌
        System.out.println("HelloServlet.service");
        System.out.println("request = " + request);
        System.out.println("response = " + response);

        //쿼리 파라메터 조회하기
        String username = request.getParameter("username");
        //Ctrl + Alt + V 값 꺼내는 단축키
        System.out.println("username = " + username);

        response.setContentType("text/plain");
        response.setCharacterEncoding("utf-8");
        response.getWriter().write("hello "+ username);
    }

}
```

여기서 `@WebServle`t은 서블릿 애노테이션이고 name은 서블릿 이름, urlPatterns는 매핑시킬 URL이다.

HTTP 요청을 통해 매핑된 URL이 호출되면 서블릿 컨테이너는`service` 메서드 실행

→HTTP 스펙을 편리하게 사용 가능

**서블릿 컨테이너 동작 방식**

1. 스프링 부트가 내장 톰캣 서버를 생성
2. 톰켓 서버에서 서블릿 코드 실행
3. HTTP 요청 메세지를 기반으로 request, response 생성
4. response 객체 정보를 통해 HTTP 응답 메시지 생성

---

## HttpServletRequest

- HTTP 요청 메시지를 직접 파싱하여 사용할 수 있지만 번거롭다. 따라서 서블릿을 통해 HTTP 요청 메시지를 편리하게 사용할 수 있게 서블릿이 HTTP 요청 메시지를 대신 파싱한다. 그리고 이 결과를 `HttpServletRequest`객체에 담아서 제공하는 것이다.

HttpServletRequest 객체를 사용하면 다음과 같은 HTTP 요청 메시지를 편리하게 조회할 수 있음

```
POST /save HTTP/1.1 //HTTP Method
Host: localhost:8080 //URL
Content-Type: application/x-www-form-urlencoded
username=kim&age=20 //쿼리 스트링
```

- Start Line
    - Http 메소드
    - URL
    - 쿼리 스트링
    - 스키마, 프로토콜
- 헤더
    - 헤더 조회
- 바디
    - form 파라미터 형식 조회
    - message body 데이터 직접 조회

### HTTP 요청 데이터

HTTP 요청 메시지를 통해 클라이언트 → 서버 데이터 전달 방법은 3가지가 있다.

1. **GET - 쿼리 파라미터**
    1. /url**?username=hello&age=20**
    2. 메시지 바디 없이, URL의 쿼리 파라미터에 데이터를 포함하여 전달
    3. 예) 검색, 필터, 페이징 등에서 많이 사용함
2. **POST - HTML Form**
    1. content-type : application/x-www-form-urlencoded → 이 형식은 html content-type의 기본형식이다.
    2. 메시지 바디에 쿼리 파라미터 형식으로 전달 username=hello&age=20
    3. 예) 회원가입, 상품 주문, HTML form 사용
3. **HTTP message body에 데이터를 직접 담아서 요청**
    1. HTTP API에서 주로 사용. JSON, XML, TEXT

데이터 형식은 주로 JSON을 사용한다.(POST, PUT, PATCH)

**HTTP 요청 데이터 - GET 쿼리 파라미터 방식**

username=kim, age=20 이라는 데이터를 클라이언트에서 서버로 전송한다고 하자

GET 쿼리 파라미터 방식에서는 메시지 바디 없이 URL의 쿼리 파라미터를 사용하여 데이터를 전송한다. 따라서 검색, 필터, 페이징에서 많이 사용됨

쿼리 파라미터는 URL에 `?`를 시작으로 보낼 수 있다. 추가 파라미터는 `&`로 구분한다.

[http://localhost:8080/request?username=hello&age=20](http://localhost:8080/request-param?username=hello&age=20)

- 서버에서는 HttpServletRequest가 제공하는 메서드를 통해 쿼리 파라미터를 편리하게 조회할 수 있다.

```java
String username = request.getParameter("username"); //단일 파라미터 조회
Enumeration<String> parameterNames = request.getParameterNames(); //파라미터 이름들 모두 조회
Map<String, String[]> parameterMap = request.getParameterMap(); //파라미터를 Map으로 조회
String[] usernames = request.getParameterValues("username"); //복수 파라미터 조회
```

---

**POST - HTML Form 방식**

HTML에 Form을 사용해서 클라이언트에서 서버로 데이터를 전송하자. 이 방식은 주로 회원 가입, 상품 주문 등에서 사용하는 방식이다.

**특징**

- content-type : application/x-www-form-urlencoded → 이 형식은 html content-type의 기본형식이다. form으로 데이터를 전송하는 형식.
- 메시지 바디에 쿼리 파라미터 형식으로 전달 username=hello&age=20

html form

```html
```html
<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<title>Title</title>
</head>
<body>
<form action="/request-param" method="post">
username: <input type="text" name="username" />
age: <input type="text" name="age" />
<button type="submit">전송</button>
</form>
</body>
</html>
```

위 html에 들어가서 form을 전송하면 웹 브라우저는 다음 형식으로 HTTP 메시지를 만든다.

- 요청 URL : [http://localhost:8080/request-param](http://localhost:8080/request-param)
- content-type : application/x-www-form-urlencoded
- message body : username=hello&age=20 (입력한 값이 들어감)

→ content-type 형식은 GET에서 본 쿼리 파라미터 형식과 같기 때문에 조회 메서드를 그대로 사용 가능하다.

결론적으로 request.getParameter()는 GET URL 쿼리 파라미터 형식도 지원하고, POST HTML Form형식도 지원한다.

→content-type은 메시지 바디 데이터 형식을 지정하기 때문에 GET URL 쿼리 파라미터 형식으로 전달할때는 바디를 안쓰니까 content-type이 없다. POST HTML Form일때는 바디가 포함되기 때문에 꼭 지정해줘야됨

---

**HTTP 요청 데이터 - API 메시지 바디 - JSON**

JSON 형식 전송

- POST http://localhost:8080/request-body-json
- content-type : application/json
- message body : {”username” : “hello”, “age”:20}
- 결과 : messageBody =  {”username” : “hello”, “age”:20}

JSON형식 파싱 추가

```java
package hello.servlet.basic;
import lombok.Getter;
import lombok.Setter;

//lombok 기능 개꿀
@Getter @Setter
public class HelloData {

    private String username;
    private int age;

}
```

→ lombok 라이브러리 사용하여 `@Getter` `@Setter` 사용

```java
package hello.servlet.basic.request;

import com.fasterxml.jackson.databind.ObjectMapper;
import hello.servlet.basic.HelloData;
import jakarta.servlet.ServletException;
import jakarta.servlet.ServletInputStream;
import jakarta.servlet.annotation.WebServlet;
import jakarta.servlet.http.HttpServlet;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.util.StreamUtils;

import java.io.IOException;
import java.nio.charset.StandardCharsets;

@WebServlet(name="requestBodyJsonServlet", urlPatterns = "/request-body-json")
public class RequestBodyJsonServlet extends HttpServlet {

    // JSON 결과를 파싱해서 사용할 수 있는 자바 객체로 변환하기 위한 것. ObjectMapper
    private ObjectMapper objectMapper = new ObjectMapper();

    @Override
    protected void service(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        ServletInputStream inputStream = request.getInputStream();
        String messageBody = StreamUtils.copyToString(inputStream, StandardCharsets.UTF_8);

        System.out.println("messageBody = " + messageBody);

        HelloData helloData = objectMapper.readValue(messageBody, HelloData.class);

        System.out.println("helloData.username = " + helloData.getUsername());
        System.out.println("helloData.age = " + helloData.getAge());
        response.getWriter().write("ok");

    }
}
```

 `ObjectMapper` → Json 결과를 파싱하여 사용 가능한 자바 객체로 변환시켜주는 Jackson 라이브러리 기능

**Postman으로 실행**



**출력 결과**



정리

Http 요청,응답 메세지의 스펙을 편리하게 사용할 수 있도록 하는게 httpServlet req, resp이다.

클라이언트에서 서버로 http 요청을 보낼 때는 3가지 방법을 사용한다. → GET, POST, HTTP message body