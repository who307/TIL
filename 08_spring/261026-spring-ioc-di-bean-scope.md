# Spring IoC Container · DI · @Autowired · Bean Scope 정리 (TIL)

> 실습 환경: Spring Core (Gradle), 챕터별 독립 프로젝트 (`chap01-ioc-container` ~ `chap04-bean`)

## 1. IoC와 Spring Container

* **한 줄 비유**: 지금까지는 요리할 때 재료를 **내가 직접 장 봐서 손질**했음(`new`로 객체 만들기). 스프링을 쓰면 **밀키트 회사(컨테이너)** 가 재료를 손질해서 필요한 곳에 넣어줌. 요리사는 요리에만 집중하면 됨.
* **IoC (Inversion of Control, 제어의 역전)**: 객체를 **만들고 서로 연결하는 일**을 개발자가 `new`로 직접 하지 않고, **프레임워크(컨테이너)에게 맡기는 것**. 주도권(제어)이 개발자 → 스프링으로 **뒤집혔다(역전)** 고 해서 붙은 이름임.
* **Spring IoC Container**: 설정 정보를 읽고 객체(**Bean**)를 만들어 **보관·관리**하는 창고. 대표 인터페이스가 `ApplicationContext`임.
* **Bean**: 스프링 컨테이너가 만들고 관리하는 객체.

### 스프링 없이 직접 생성 (plain java)

```java
MemberDAO memberDAO = new MemberDAO();          // 개발자가 직접 생성
MemberDTO member = memberDAO.selectMember(1);
```

> 객체가 늘어날수록 생성 순서, 연결 관계를 개발자가 모두 관리해야 함 → 이걸 컨테이너에 맡기는 것이 IoC

* **따라가기**: 내가 `new MemberDAO()`로 DAO를 만들고 → 1번 회원을 조회. 객체가 2개면 괜찮지만, 50개가 서로 얽혀 있으면 "누구를 먼저 만들고 누구에게 넘겨줄지"를 전부 내가 챙겨야 함.

### Bean을 등록하는 3가지 방법

| 방법 | 설정 | 컨테이너 생성 |
| :--- | :--- | :--- |
| XML | `<bean>` 태그 | `new GenericXmlApplicationContext("xml 경로")` |
| Java Config | `@Configuration` + `@Bean` | `new AnnotationConfigApplicationContext(설정클래스.class)` |
| Annotation | `@Component` + `@ComponentScan` | `new AnnotationConfigApplicationContext(설정클래스.class 또는 "패키지")` |

* 어떤 방법이든 결국 "**어떤 객체를, 어떤 이름으로, 어떻게 만들지**" 컨테이너에게 알려주는 **주문서**임. 주문서를 XML로 쓰느냐, 자바 코드로 쓰느냐, 클래스에 스티커(어노테이션)를 붙이느냐의 차이.

---

## 2. Bean 등록 방법

### XML 설정

* `<bean id="이름" class="클래스 전체 경로">` → "이 클래스로 객체를 만들어서 이 이름으로 보관해 줘."
* `<constructor-arg>` → 객체를 만들 때 **생성자에 넘길 값**. `index`(몇 번째 매개변수인지, 0부터)나 `name`(매개변수 이름)으로 자리를 지정함.

```xml
<bean id="member" class="com.ohgiraffers.common.MemberDTO">
    <constructor-arg index="0" value="1"/>
    <constructor-arg name="id" value="user01"/>
    <constructor-arg index="2"><value>pass01</value></constructor-arg>
    <constructor-arg name="name"><value>홍길동</value></constructor-arg>
</bean>
```

```java
ApplicationContext context =
        new GenericXmlApplicationContext("section01/xmlconfig/spring-context.xml");

// bean의 id와 기대하는 반환 타입을 함께 전달
MemberDTO member = context.getBean("member", MemberDTO.class);
```

* **따라가기**
  1. `GenericXmlApplicationContext("...xml")` → 컨테이너가 XML 주문서를 읽음.
  2. `<bean id="member">`를 보고 → `MemberDTO` 생성자에 `(1, "user01", "pass01", "홍길동")`을 넣어 객체를 만들고 → `"member"`라는 이름으로 보관.
  3. `context.getBean("member", MemberDTO.class)` → "`member`라는 이름의 bean을 `MemberDTO` 타입으로 줘" → 보관해 둔 그 객체를 받음. 내가 `new`를 쓴 적이 없음.

### Java Config

* `@Configuration`: "이 클래스는 **bean 주문서(설정 클래스)** 야"라는 표시.
* `@Bean`: 이 메소드가 **돌려주는 객체**를 bean으로 등록함. 이름을 따로 안 정하면 **메소드 이름**이 bean 이름이 됨.

```java
@Configuration("configurationSection02")
public class ContextConfiguration {

    @Bean(name = "member")
    public MemberDTO getMember() {
        return new MemberDTO(1, "user01", "pass01", "홍길동");
    }
}
```

```java
ApplicationContext context =
        new AnnotationConfigApplicationContext(ContextConfiguration.class);
MemberDTO member = context.getBean("member", MemberDTO.class);
```

* **따라가기**: 컨테이너가 `ContextConfiguration`을 읽음 → `@Bean` 메소드 `getMember()`를 **스프링이 대신 호출** → 돌아온 객체를 `"member"`로 보관 → `getBean`으로 꺼냄. `name = "member"`가 없었다면 이름은 `getMember`가 됨.

### Component Scan

* **한 줄 비유**: 일일이 주문서를 쓰는 대신, 클래스에 **"나 bean으로 써 주세요" 스티커(`@Component`)** 를 붙여두면 스프링이 지정한 폴더를 **훑어보며(scan)** 스티커 붙은 클래스를 찾아 자동으로 등록함.
* `@ComponentScan(basePackages = "...")`: 이 패키지와 그 아래를 훑어보라는 지시.
* 이름을 따로 안 정하면 **클래스 이름의 첫 글자를 소문자로** 바꾼 이름이 됨 (`MemberDAO` → `memberDAO`).

```java
@Component
public class MemberDAO { ... }
```

```java
@Configuration
@ComponentScan(basePackages = "com.ohgiraffers.common")
public class ContextConfiguration {
}
```

```java
// 등록된 모든 bean 이름 확인
String[] beanNames = context.getBeanDefinitionNames();
for (String beanName : beanNames) {
    System.out.println("beanName : " + beanName);
}

MemberDAO memberDAO = context.getBean("memberDAO", MemberDAO.class);
```

* **따라가기**
  1. 컨테이너가 `ContextConfiguration`을 읽음 → `@ComponentScan`을 보고 `com.ohgiraffers.common` 아래를 훑음.
  2. `@Component`가 붙은 `MemberDAO`를 발견 → 객체를 만들어 `"memberDAO"`로 보관.
  3. `getBeanDefinitionNames()` → 등록된 bean 이름을 모두 출력 → 스프링 내부용 bean들과 함께 `contextConfiguration`, `memberDAO`가 보임.
  4. `getBean("memberDAO", ...)`로 꺼냄.

| 어노테이션 | 용도 |
| :--- | :--- |
| `@Component` | 일반 컴포넌트 |
| `@Controller` | 요청을 받는 Controller 계층 |
| `@Service` | 비즈니스 로직을 담당하는 Service 계층 |
| `@Repository` | DB 접근을 담당하는 DAO 계층 |
| `@Configuration` | 설정 클래스 (내부적으로 `@Component` 포함) |

> 모두 `@Component`를 세분화한 것이라 Component Scan 대상이 됨. 역할을 코드로 드러내기 위해 계층에 맞게 골라 씀.

* 식당으로 비유하면 `@Controller`는 **주문 받는 직원**, `@Service`는 **요리사**, `@Repository`는 **창고 관리인**. 모두 "직원(`@Component`)"이지만 이름표로 역할을 드러냄.

---

## 3. DI (Dependency Injection, 의존성 주입)

* **의존 관계**: `MemberDTO`가 `Account`(계좌)를 필드로 가지고 쓰면 "`MemberDTO`가 `Account`에 **의존한다**"고 함. 계좌가 없으면 회원이 제대로 동작하지 못하는 관계.
* **DI**: 필요한 객체를 내 안에서 `new`로 **직접 만들지 않고**, 바깥(컨테이너)에서 **넣어주는(주입하는)** 것. 휴대폰에 배터리를 **공장에서 끼워서** 출고하는 것과 같음.
* **인터페이스 타입으로 받기**: 필드 타입을 `PersonalAccount`(구현 클래스)가 아니라 **`Account`(인터페이스)** 로 두면, 나중에 다른 계좌 종류로 바꿔 끼워도 `MemberDTO` 코드는 **고칠 필요가 없음**. 충전기 단자가 표준(USB-C)이면 어느 회사 충전기든 꽂히는 것과 같음.

### XML: setter 주입과 ref

* `<property name="필드">` → 그 필드의 **setter**(`setName` 등)를 불러서 값을 넣음.
* 다른 **bean**을 넣을 때는 `value` 대신 **`ref="bean id"`** 를 씀. `value`는 "이 글자/숫자를 넣어", `ref`는 "보관함에 있는 **저 객체**를 넣어".
* `class` 속성에는 인터페이스가 아니라 **구현 클래스**를 써야 함(인터페이스는 객체로 만들 수 없음).

```xml
<bean id="account" class="com.ohgiraffers.common.PersonalAccount">
    <constructor-arg index="0" value="20"/>
    <constructor-arg index="1" value="110-234-567890"/>
</bean>

<!-- setter 주입 -->
<bean id="member" class="com.ohgiraffers.common.MemberDTO">
    <property name="sequence" value="1"/>
    <property name="name" value="홍길동"/>
    <property name="phone" value="010-1234-5678"/>
    <property name="email" value="hong123@gmail.com"/>
    <property name="personalAccount" ref="account"/>
</bean>
```

* **따라가기**
  1. `account` bean → `PersonalAccount(20, "110-234-567890")`으로 계좌 객체 생성.
  2. `member` bean → 기본 생성자로 빈 `MemberDTO`를 만든 뒤 setter를 차례로 호출: `setSequence(1)`, `setName("홍길동")`, `setPhone(...)`, `setEmail(...)`
  3. `setPersonalAccount(account bean)` → 1번에서 만든 **계좌 객체**를 회원에게 꽂아줌.
  4. 결과: 계좌를 가진 홍길동 회원 bean 완성. `MemberDTO` 안에는 `new PersonalAccount(...)`가 한 줄도 없음.

### Java Config: @Bean 메소드 호출로 연결

```java
@Configuration
public class ContextConfiguration {

    @Bean
    public Account accountGenerator() {
        return new PersonalAccount(20, "110-234-567890");
    }

    @Bean
    public MemberDTO memberGenerator() {
        MemberDTO member = new MemberDTO();
        member.setSequence(1);
        member.setName("홍길동");
        member.setPersonalAccount(accountGenerator());   // 다른 bean 메소드 호출
        return member;
    }
}
```

* **따라가기**: `accountGenerator` bean(계좌) 생성 → `memberGenerator`에서 회원을 만들고 `accountGenerator()`를 불러 계좌를 꽂음 → 회원 bean 등록.
* `getBean(MemberDTO.class)`처럼 이름 없이 **타입만으로** 가져올 수도 있음(해당 타입 bean이 하나일 때).

> 💡 `@Configuration` 클래스 안에서 `accountGenerator()`를 여러 번 호출해도 새 객체가 생기지 않고 **컨테이너에 등록된 같은 bean**이 반환됨. 스프링이 설정 클래스를 프록시로 감싸서 처리해주기 때문임.

* **프록시(Proxy)**: 스프링이 설정 클래스를 **대신 서 있는 대리인**으로 한 겹 감싸둠. `accountGenerator()`를 부르면 대리인이 먼저 받아서 "이미 만든 계좌가 보관함에 있네" 하고 **그걸 돌려줌**. 그래서 계좌가 두 개 생기지 않음.

### 계층 구조(Controller → Service → Repository) 조립

* 각 계층은 바로 아래 계층을 **생성자로 받음(생성자 주입)**. 필드에 `final`을 붙일 수 있어서, 한 번 받은 의존 객체가 **나중에 바뀌지 않음**이 보장됨.

```java
public class MemoService {

    private final MemoRepository memoRepository;

    /* 생성자 주입 */
    public MemoService(MemoRepository memoRepository) {
        this.memoRepository = memoRepository;
    }
    // ...
}
```

```java
@Configuration
public class ContextConfiguration {

    @Bean
    public MemoRepository memoRepository() {
        return new MemoryMemoRepository();             // 인터페이스 타입으로 등록
    }

    @Bean
    public MemoService memoService() {
        return new MemoService(memoRepository());
    }

    @Bean
    public MemoController memoController() {
        return new MemoController(memoService());
    }
}
```

```java
MemoController memoController = context.getBean(MemoController.class);
System.out.println(memoController.findMemos());
System.out.println(memoController.createMemo("의존 객체를 생성자로 전달"));
```

* **따라가기 (조립 순서)**
  1. `memoRepository` → 메모를 메모리에 저장하는 저장소(`MemoryMemoRepository`) 생성.
  2. `memoService` → 1번 저장소를 생성자로 받은 서비스 생성.
  3. `memoController` → 2번 서비스를 생성자로 받은 컨트롤러 생성.
  4. `getBean(MemoController.class)` → 완성된 컨트롤러를 꺼냄 → `findMemos()`(메모 목록), `createMemo(...)`(메모 추가)를 부르면 컨트롤러 → 서비스 → 저장소 순서로 일이 전달됨.

> 저장소를 DB 버전으로 바꾸고 싶으면 `memoRepository()`의 반환 객체만 바꾸면 됨. Service, Controller 코드는 그대로.

---

## 4. @Autowired: 어노테이션 기반 DI

* **`@Autowired`**: "이 자리에 맞는 **타입의 bean**을 보관함에서 찾아 **자동으로** 꽂아줘." 설정 클래스에서 `new MemoService(memoRepository())`처럼 직접 조립하던 것을 스프링이 알아서 함.
* `new AnnotationConfigApplicationContext("com.ohgiraffers.section01")`처럼 **패키지 문자열**을 넘기면 설정 클래스 없이 바로 Component Scan을 수행함.

```java
@Repository("bookDAO")
public class BookDAOImpl implements BookDAO { ... }   // 인터페이스 구현 클래스는 보통 Impl을 붙임
```

* `BookDAOImpl`에 `@Repository("bookDAO")` → 스캔될 때 `"bookDAO"`라는 이름으로 등록됨. 타입은 `BookDAOImpl`이면서 `BookDAO`이기도 함(인터페이스 구현).

### 주입 방식 3가지

```java
/* 1. 필드 주입 */
@Service("bookServiceField")
public class BookService {
    @Autowired
    private BookDAO bookDAO;
}

/* 2. setter 주입 */
@Service("bookServiceSetter")
public class BookService {
    private BookDAO bookDAO;

    @Autowired
    public void setBookDAO(BookDAO bookDAO) {
        this.bookDAO = bookDAO;
    }
}

/* 3. 생성자 주입 (권장) */
@Service("bookServiceConstructor")
public class BookService {
    private final BookDAO bookDAO;

    @Autowired   // 생성자가 하나뿐이면 생략 가능
    public BookService(BookDAO bookDAO) {
        this.bookDAO = bookDAO;
    }
}
```

* **따라가기 (공통)**: 스프링이 `BookService`를 만들 때 `@Autowired`가 붙은 자리를 봄 → 필요한 타입은 `BookDAO` → 보관함에서 `BookDAO` 타입 bean(`bookDAO` = `BookDAOImpl`)을 찾음 → 그 자리에 꽂아줌.
  * 필드 주입: 객체를 만든 뒤 `private` 필드에 **직접** 넣음.
  * setter 주입: 객체를 만든 뒤 `setBookDAO(...)`를 **호출**해서 넣음.
  * 생성자 주입: 객체를 **만드는 순간** 생성자에 넣음.

| 방식 | 특징 |
| :--- | :--- |
| 필드 주입 | 코드가 가장 짧지만, `final` 불가, 스프링 없이 테스트할 때 값을 넣기 어려움 |
| setter 주입 | 주입 후에도 의존 객체를 바꿀 수 있음(선택적 의존성에 사용) |
| **생성자 주입** | `final`로 **불변** 보장, 객체 생성 시점에 의존성이 반드시 채워짐 → **가장 권장** |

* 생성자 주입이 좋은 이유: 배터리 없이는 휴대폰이 **아예 출고되지 않음**(생성자에 꼭 넣어야 만들어짐) + 출고 후 배터리를 **바꿔 끼울 수 없음**(`final`) → 반쯤 조립된 객체가 생길 일이 없음.

---

## 5. 같은 타입의 Bean이 여러 개일 때

* `Pokemon` 인터페이스를 구현한 `Pikachu`, `Charmander`, `Squirtle`이 모두 bean이면, `Pokemon` 하나를 주입하려 할 때 스프링이 **셋 중 누굴 넣을지 모름** → 따로 정해주지 않으면 예외가 나서 실행되지 않음. "포켓몬 한 마리 주세요"라고만 하면 가게 주인이 곤란한 것과 같음.

### @Primary

* 같은 타입 bean 중 **기본으로 넣을 것**을 정함. 같은 타입 중 **하나에만 붙여야** 함(여러 개에 붙이면 다시 누가 기본인지 모호해져서 예외가 남).

```java
@Component
@Primary
public class Charmander implements Pokemon {
    @Override
    public void attack() {
        System.out.println("파이리 불꽃 공격");
    }
}
```

```java
@Autowired
public PokemonService(Pokemon pokemon) {   // → Charmander 주입
    this.pokemon = pokemon;
}
```

* **따라가기**: `Pokemon` 타입 후보 3마리 → `@Primary`가 붙은 파이리를 고름 → `pokemon.attack()` → `파이리 불꽃 공격`

### @Qualifier

* 넣을 bean을 **이름으로 콕 집어** 정함. `@Primary`보다 **우선**함.

```java
@Autowired
public PokemonService(@Qualifier("pikachu") Pokemon pokemon) {   // → Pikachu 주입
    this.pokemon = pokemon;
}
```

* **따라가기**: `@Primary`는 파이리지만, `@Qualifier("pikachu")`로 이름을 지정함 → 이름이 `pikachu`인 bean(`Pikachu` 클래스, 첫 글자 소문자)이 들어감.

### List로 모두 주입

* 매개변수를 `List<Pokemon>`으로 두면 그 타입의 bean을 **전부** 리스트에 담아서 넣어줌.

```java
@Autowired
public PokemonService(List<Pokemon> pokemonList) {
    this.pokemonList = pokemonList;
}

public void pokemonAttack() {
    pokemonList.forEach(Pokemon::attack);   // 피카츄, 파이리, 꼬부기 모두 공격
}
```

* **따라가기**: 리스트에 피카츄, 파이리, 꼬부기가 담겨 들어옴 → `forEach(Pokemon::attack)` → 세 마리가 차례로 공격 메시지를 출력함.

---

## 6. Bean Scope

* **scope(스코프)**: bean 객체를 **언제 만들고, 얼마나 같이 쓰는지** 정하는 범위.

| scope | 동작 | 비유 |
| :--- | :--- | :--- |
| `singleton` (기본값) | 컨테이너당 **인스턴스 하나**만 만들어서 모든 `getBean()`에 같은 객체를 반환 | 교실에 하나뿐인 칠판 |
| `prototype` | `getBean()`을 호출할 때마다 **새 인스턴스**를 생성해서 반환 | 한 장씩 뽑아 쓰는 프린트물 |

```java
@Bean
@Scope("prototype")   // 생략하면 singleton
public ShoppingCart cart() {
    return new ShoppingCart();
}
```

* 같은 코드로 카트 두 개를 꺼내 상품을 담아보면 차이가 드러남.

```java
ShoppingCart cart1 = context.getBean("cart", ShoppingCart.class);
cart1.addItem(carpBread);
cart1.addItem(milk);

ShoppingCart cart2 = context.getBean("cart", ShoppingCart.class);
cart2.addItem(water);
System.out.println("cart2 : " + cart2.getItems());
```

* **따라가기**
  * **singleton일 때**: 첫 `getBean` → 카트 생성 → 붕어빵, 딸기우유 담음 → 두 번째 `getBean` → **같은 카트**를 돌려줌 → 물을 담음 → `cart2`에는 세 개가 다 보임.
  * **prototype일 때**: 두 번째 `getBean` → **새 카트**를 만들어 돌려줌 → `cart2`에는 물만 있음.

| scope | `cart2` 출력 결과 |
| :--- | :--- |
| `singleton` | 붕어빵, 딸기우유, 지리산암반수 → `cart1`과 **같은 카트**라 상품이 섞임 |
| `prototype` | 지리산암반수만 → 손님마다 **별도의 카트** |

> ⚠️ 상태(데이터)를 가지는 객체를 singleton으로 두면 여러 사용자가 같은 객체를 공유하게 됨. 이 예제처럼 `getBean()`으로 꺼낼 때마다 새 카트가 필요하면 prototype이 맞음.
> 다만 실제 웹 서비스에서 **사용자(로그인 세션)마다** 따로 있어야 하는 장바구니는 `session` scope가 알맞음. 또 prototype bean을 singleton bean에 `@Autowired`로 주입하면, singleton이 처음 만들어질 때 **딱 한 번만** 주입되므로 결국 모두가 같은 카트를 쓰게 됨.

---

## 📝 오늘의 정리

1. 객체 생성과 연결을 컨테이너에 맡기는 것이 **IoC**, 필요한 객체를 외부에서 넣어주는 것이 **DI**임
2. Bean은 **XML**, **Java Config(`@Configuration` + `@Bean`)**, **Component Scan(`@Component`)** 으로 등록할 수 있음
3. `@Autowired` 주입 방식 중 `final`로 불변을 보장하는 **생성자 주입**이 가장 권장됨
4. 같은 타입 bean이 여러 개면 `@Primary`(기본 우선)나 `@Qualifier`(이름 지정, 더 우선)로 고르고, `List`로 모두 받을 수도 있음
5. Bean의 기본 scope는 **singleton**이고, 매번 새 객체가 필요하면 `@Scope("prototype")`을 씀

---

## 느낀 점

* 지금까지 `new`로 직접 만들고 넘겨주던 객체를 설정만 해두면 스프링이 알아서 만들고 연결해준다는 점이 신기했고, JDBC 때 Controller·Service·DAO를 직접 조립하던 코드가 왜 필요 없어지는지 이해됨
* 필드를 인터페이스 타입으로 두고 생성자로 주입받으면 구현체를 바꿔도 다른 계층 코드는 그대로라는 점에서, Java에서 배운 다형성이 실제로 이렇게 쓰인다는 것을 실감함
* singleton과 prototype을 장바구니 예제로 비교하니, bean을 공유해도 되는 객체와 따로 만들어야 하는 객체를 구분하는 것이 중요하다고 느낌
