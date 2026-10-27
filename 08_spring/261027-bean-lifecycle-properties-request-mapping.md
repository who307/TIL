# Bean Life Cycle · 외부 설정값 주입 · Spring Boot 요청 매핑 정리 (TIL)

> 실습 환경: Spring Core (`chap04-bean`), Spring Boot 4 + Spring Web MVC (`chap00-spring-boot-introduction` ~ `chap02-handler-method`), IntelliJ HTTP Client (`api-test.http`)

## 1. Bean Life Cycle: 초기화·소멸 메소드

* 컨테이너는 bean을 **생성 → 의존성 주입 → 초기화 → 사용 → 소멸** 순서로 관리함.
* `@Bean`의 `initMethod`, `destroyMethod` 속성으로 **초기화·소멸 시점에 호출할 메소드 이름**을 지정할 수 있음.
  * `initMethod`: bean 생성과 의존성 주입이 끝난 직후 호출
  * `destroyMethod`: 컨테이너가 종료되기 직전에 호출

```java
public class Owner {
    public void openShop() {
        System.out.println("가게 문을 열었습니다.");
    }

    public void closeShop() {
        System.out.println("가게 문을 닫았습니다.");
    }
}
```

```java
@Configuration
public class ContextConfiguration {

    /* Bean 생성 후 openShop(), Container 종료 전 closeShop()을 호출하도록 지정 */
    @Bean(initMethod = "openShop", destroyMethod = "closeShop")
    public Owner owner() {
        return new Owner();
    }
}
```

```java
AnnotationConfigApplicationContext context =
        new AnnotationConfigApplicationContext(ContextConfiguration.class);

System.out.println("영업 중입니다.");

context.close();   // 컨테이너 종료 → destroyMethod 호출
```

```
가게 문을 열었습니다.     ← 컨테이너 생성 시 initMethod
영업 중입니다.
가게 문을 닫았습니다.     ← context.close() 시 destroyMethod
```

> 💡 `close()`는 `ApplicationContext` 인터페이스에는 없고 구현체(`AnnotationConfigApplicationContext`)에 있는 메소드라서, 변수 타입을 구현 클래스로 선언해야 호출할 수 있음.

> ⚠️ 소멸 메소드는 컨테이너가 끝까지 관리하는 **singleton** bean에만 호출됨. prototype bean은 생성·주입·초기화까지만 컨테이너가 관리하고 이후에는 손을 뗌.

---

## 2. 외부 설정 파일 값 주입: @PropertySource + @Value

* 상품 이름, 가격처럼 바뀔 수 있는 값을 코드에 직접 쓰지 않고 **`.properties` 파일로 분리**함.

```properties
# resources/section03/properties/product-info.properties
bread.carpbread.name=붕어빵
bread.carpbread.price=1000
beverage.milk.name=딸기우유
beverage.milk.price=1500
beverage.milk.capacity=500
beverage.water.name=지리산암반수
beverage.water.price=3000
beverage.water.capacity=500
```

* `@PropertySource`: 설정 클래스에서 사용할 외부 설정 파일을 스프링 환경(Environment)에 등록함. `classpath:`는 `resources` 폴더 기준 경로.
* `@Value("${키}")`: 등록된 설정 파일에서 **키에 해당하는 값**을 꺼내 주입함. `int` 같은 타입으로 자동 변환해줌.

```java
@Configuration
@PropertySource(value = "classpath:section03/properties/product-info.properties", encoding = "UTF-8")
public class ContextConfiguration {

    /* 1. 필드에 주입 */
    @Value("${bread.carpbread.name}")
    private String carpBreadName;

    @Value("${bread.carpbread.price}")
    private int carpBreadPrice;

    @Bean
    public Product carpBread() {
        return new Bread(carpBreadName, carpBreadPrice, new Date());
    }

    /* 2. @Bean 메소드의 매개변수에 주입 */
    @Bean
    public Product water(@Value("${beverage.water.name}") String waterName,
                         @Value("${beverage.water.price}") int waterPrice,
                         @Value("${beverage.water.capacity}") int capacity) {
        return new Beverage(waterName, waterPrice, capacity);
    }
}
```

> 💡 한글 값이 깨지지 않도록 `encoding = "UTF-8"`을 지정함.

> 💡 JDBC 때 접속 정보를 `Properties.load()`로 직접 읽었던 것을, 스프링에서는 어노테이션만으로 주입받을 수 있음.

---

## 3. Spring Boot 시작하기

* **Spring Boot**: 스프링 설정과 서버 실행을 자동화해서 바로 실행 가능한 애플리케이션을 만들게 해주는 도구.
  * 내장 톰캣이 있어서 `main` 메소드 실행만으로 웹 서버가 뜸(기본 포트 `8080`).
  * `spring-boot-starter-webmvc` 같은 **starter** 의존성 하나로 필요한 라이브러리를 한 번에 가져옴.

```java
@SpringBootApplication   // 설정 + Component Scan + 자동 설정을 한 번에 켜는 어노테이션
public class Chap00SpringBootIntroductionApplication {

    public static void main(String[] args) {
        SpringApplication.run(Chap00SpringBootIntroductionApplication.class, args);
    }
}
```

```java
@RestController
/* 이 클래스가 웹 요청을 받는 Controller이며,
 * 메소드의 반환값을 HTTP 응답 본문으로 전달한다는 표시 */
public class HelloController {

    // GET 방식의 /hello 요청이 들어오면 hello 메소드를 실행
    @GetMapping("/hello")
    public String hello() {
        return "Spring Boot Hello";
    }
}
```

> `@SpringBootApplication`이 붙은 클래스의 패키지 하위를 Component Scan 하기 때문에, Controller는 그 하위 패키지에 있어야 bean으로 등록됨.

---

## 4. 요청 매핑 (Request Mapping)

* **핸들러 메소드**: HTTP 요청과 연결되어 실제로 실행되는 Controller 메소드.
* 클래스에 `@RequestMapping("/공통 경로")`를 붙이면 내부 메소드들의 **경로 앞부분을 공통으로** 지정할 수 있음.

| 어노테이션 | HTTP 메소드 | 용도 (REST) |
| :--- | :--- | :--- |
| `@GetMapping` | GET | 조회 |
| `@PostMapping` | POST | 등록 |
| `@PutMapping` | PUT | 수정 |
| `@DeleteMapping` | DELETE | 삭제 |

```java
@RestController
@RequestMapping("/api/v1/menus")
public class MethodMappingTestController {

    @GetMapping                              // GET /api/v1/menus
    public ResponseEntity<Map<String, String>> findMenus() {
        Map<String, String> response = Map.of("message", "메뉴 목록 조회용 핸들러 메소드 호출");
        return ResponseEntity.ok(response);
    }

    @GetMapping("/{menuCode}")               // GET /api/v1/menus/3
    public ResponseEntity<Map<String, Object>> findMenu(@PathVariable int menuCode) {
        Map<String, Object> response = Map.of("message", menuCode + "번 메뉴 조회용 핸들러 메소드 호출");
        return ResponseEntity.ok(response);
    }

    @PostMapping                             // POST /api/v1/menus
    public ResponseEntity<Map<String, Object>> registMenu() {
        Map<String, Object> response = Map.of("message", "신규 메뉴 등록용 핸들러 메소드 호출");
        return ResponseEntity.status(HttpStatus.CREATED).body(response);
    }

    @PutMapping("/{menuCode}")               // PUT /api/v1/menus/1
    public ResponseEntity<Map<String, String>> modifyMenu(@PathVariable int menuCode) { ... }

    @DeleteMapping("/{menuCode}")            // DELETE /api/v1/menus/1
    public ResponseEntity<Map<String, String>> deleteMenu(@PathVariable int menuCode) { ... }
}
```

* 같은 URL(`/api/v1/menus/1`)이라도 **HTTP 메소드가 다르면 다른 핸들러 메소드**가 실행됨.

### @PathVariable

* URL 경로의 `{변수명}` 부분 값을 매개변수로 받음. 매개변수 이름이 `{}` 안의 이름과 같아야 함.

### ResponseEntity

* 응답 **본문(body)** 과 함께 **HTTP 상태 코드**, 헤더까지 직접 지정할 수 있는 응답 객체.
* `Map`을 body로 넣으면 JSON으로 변환되어 응답됨.

| 코드 | 상태 코드 |
| :--- | :--- |
| `ResponseEntity.ok(body)` | `200 OK` |
| `ResponseEntity.status(HttpStatus.CREATED).body(body)` | `201 Created` (등록 성공) |

---

## 5. 요청 파라미터 받기: @RequestParam

* `?keyword=coffee&categoryCode=2` 같은 **query string** 값을 매개변수로 받음.

| 속성 | 의미 |
| :--- | :--- |
| `name` | 받을 파라미터 이름 |
| `required = false` | 파라미터가 없어도 오류 없이 `null`로 받음 (기본값은 `true` → 없으면 `400 Bad Request`) |
| `defaultValue` | 파라미터가 없을 때 사용할 값 (지정하면 자동으로 `required = false`) |

```java
@GetMapping
public ResponseEntity<Map<String, Object>> findMenus(
        @RequestParam(name = "keyword", required = false) String keyword,
        @RequestParam(name = "categoryCode", defaultValue = "0") int categoryCode,
        @RequestParam(name = "page", defaultValue = "1") int page) {

    Map<String, Object> response = new LinkedHashMap<>();   // 넣은 순서대로 응답
    response.put("message", "메뉴 목록 조회 요청 파라미터 확인");
    response.put("keyword", keyword);
    response.put("categoryCode", categoryCode);
    response.put("page", page);

    return ResponseEntity.ok(response);
}
```

```http
### 모든 검색 조건 전달
GET http://localhost:8080/api/v1/menus?keyword=coffee&categoryCode=2&page=1

### 선택값을 생략해 기본값 확인 → keyword: null, categoryCode: 0, page: 1
GET http://localhost:8080/api/v1/menus
```

> ⚠️ `int` 같은 기본형 매개변수는 `null`을 담을 수 없어서, 생략 가능한 파라미터라면 `defaultValue`를 지정해야 함.

> 💡 `Map.of()`는 순서가 보장되지 않고 `null` 값을 넣을 수 없어서, `keyword`처럼 `null`일 수 있는 값을 응답에 담을 때는 `LinkedHashMap`을 사용함.

---

## 📝 오늘의 정리

1. `@Bean(initMethod, destroyMethod)`로 bean 생성 직후와 컨테이너 종료 직전에 실행할 메소드를 지정할 수 있다.
2. `@PropertySource`로 외부 설정 파일을 등록하고 `@Value("${키}")`로 값을 필드나 매개변수에 주입한다.
3. Spring Boot는 `@SpringBootApplication`과 `SpringApplication.run()`만으로 내장 서버까지 실행해준다.
4. `@RestController` + `@RequestMapping`과 `@GetMapping`·`@PostMapping`·`@PutMapping`·`@DeleteMapping`으로 URL과 HTTP 메소드에 맞는 핸들러 메소드를 연결한다.
5. 경로 값은 `@PathVariable`, query string은 `@RequestParam`으로 받고, 응답은 `ResponseEntity`로 상태 코드와 함께 돌려준다.

---

## 느낀 점

* `initMethod`·`destroyMethod`로 bean의 생성부터 소멸까지 흐름을 직접 출력해보니, 컨테이너가 객체를 만드는 것뿐 아니라 끝날 때까지 관리한다는 의미가 와닿음.
* JDBC 때 직접 읽던 properties 파일을 `@Value` 하나로 주입받을 수 있어서 설정값 분리가 훨씬 간단해짐.
* Spring Core에서 설정 클래스와 컨테이너를 직접 만들던 것과 달리, Boot에서는 `main` 실행만으로 서버가 뜨고 Controller가 바로 요청을 받는 점이 편리했음.
* 같은 URL이라도 GET·POST·PUT·DELETE에 따라 다른 메소드가 실행되는 구조를 보며, REST API가 HTTP 메소드로 행위를 구분한다는 것을 이해함.
