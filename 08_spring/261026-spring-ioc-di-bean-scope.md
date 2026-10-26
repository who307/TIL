# Spring IoC Container · DI · @Autowired · Bean Scope 정리 (TIL)

> 실습 환경: Spring Core (Gradle), 챕터별 독립 프로젝트 (`chap01-ioc-container` ~ `chap04-bean`)

## 1. IoC와 Spring Container

* **IoC (Inversion of Control, 제어의 역전)**: 객체의 생성과 연결을 개발자가 `new`로 직접 하지 않고, **프레임워크(컨테이너)에 맡기는 것**.
* **Spring IoC Container**: 설정 정보를 읽어서 객체(**Bean**)를 생성·관리하는 주체. 대표 인터페이스가 `ApplicationContext`임.
* **Bean**: 스프링 컨테이너가 생성하고 관리하는 객체.

### 스프링 없이 직접 생성 (plain java)

```java
MemberDAO memberDAO = new MemberDAO();          // 개발자가 직접 생성
MemberDTO member = memberDAO.selectMember(1);
```

> 객체가 늘어날수록 생성 순서, 연결 관계를 개발자가 모두 관리해야 함 → 이걸 컨테이너에 맡기는 것이 IoC

### Bean을 등록하는 3가지 방법

| 방법 | 설정 | 컨테이너 생성 |
| :--- | :--- | :--- |
| XML | `<bean>` 태그 | `new GenericXmlApplicationContext("xml 경로")` |
| Java Config | `@Configuration` + `@Bean` | `new AnnotationConfigApplicationContext(설정클래스.class)` |
| Annotation | `@Component` + `@ComponentScan` | `new AnnotationConfigApplicationContext(설정클래스.class 또는 "패키지")` |

---

## 2. Bean 등록 방법

### XML 설정

* `id`는 bean 이름, `class`는 생성할 클래스의 전체 경로.
* `<constructor-arg>`로 생성자에 값을 전달함. `index`(순서)나 `name`(매개변수명)으로 지정할 수 있음.

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

### Java Config

* `@Configuration`: 이 클래스가 bean 설정 클래스임을 표시함.
* `@Bean`: 메소드의 **반환값**을 bean으로 등록함. 이름을 지정하지 않으면 **메소드 이름**이 bean 이름이 됨.

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

### Component Scan

* `@ComponentScan(basePackages = "...")`: 지정한 패키지 하위에서 `@Component`가 붙은 클래스를 찾아 bean으로 등록함.
* bean 이름을 지정하지 않으면 **클래스 이름의 첫 글자를 소문자로** 바꾼 이름으로 등록됨. (`MemberDAO` → `memberDAO`)

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

| 어노테이션 | 용도 |
| :--- | :--- |
| `@Component` | 일반 컴포넌트 |
| `@Controller` | 요청을 받는 Controller 계층 |
| `@Service` | 비즈니스 로직을 담당하는 Service 계층 |
| `@Repository` | DB 접근을 담당하는 DAO 계층 |
| `@Configuration` | 설정 클래스 (내부적으로 `@Component` 포함) |

> 모두 `@Component`를 세분화한 것이라 Component Scan 대상이 됨. 역할을 코드로 드러내기 위해 계층에 맞게 골라 씀.

---

## 3. DI (Dependency Injection, 의존성 주입)

* **의존 관계**: `MemberDTO`가 `Account`를 필드로 가지고 사용하면 "`MemberDTO`가 `Account`에 의존한다"고 함.
* **DI**: 필요한 객체(의존 객체)를 내부에서 `new`로 만들지 않고 **외부(컨테이너)에서 넣어주는 것**.
* 필드 타입을 구현 클래스(`PersonalAccount`)가 아닌 **인터페이스(`Account`)** 로 두면, 구현체를 바꿔도 `MemberDTO` 코드는 수정하지 않아도 됨.

### XML: setter 주입과 ref

* `<property name="...">`는 해당 필드의 **setter**를 호출해서 값을 넣음.
* 다른 bean을 넣을 때는 `value` 대신 **`ref="bean id"`** 를 사용함.
* `class` 속성에는 인터페이스가 아닌 **구현 클래스**를 작성해야 함(인터페이스는 객체 생성이 불가능).

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

* `getBean(MemberDTO.class)`처럼 이름 없이 **타입만으로** 가져올 수도 있음(해당 타입 bean이 하나일 때).

> 💡 `@Configuration` 클래스 안에서 `accountGenerator()`를 여러 번 호출해도 새 객체가 생기지 않고 **컨테이너에 등록된 같은 bean**이 반환됨. 스프링이 설정 클래스를 프록시로 감싸서 처리해주기 때문임.

### 계층 구조(Controller → Service → Repository) 조립

* 각 계층은 아래 계층을 **생성자로 전달받음(생성자 주입)**. 필드에 `final`을 붙일 수 있어서 한 번 주입된 의존 객체가 바뀌지 않음.

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

> 저장소를 DB 버전으로 바꾸고 싶으면 `memoRepository()`의 반환 객체만 바꾸면 됨. Service, Controller 코드는 그대로.

---

## 4. @Autowired: 어노테이션 기반 DI

* `@Autowired`: 해당 **타입과 일치하는 bean**을 컨테이너에서 찾아 자동으로 주입함.
* `new AnnotationConfigApplicationContext("com.ohgiraffers.section01")`처럼 **패키지 문자열**을 넘기면 설정 클래스 없이 바로 Component Scan을 수행함.

```java
@Repository("bookDAO")
public class BookDAOImpl implements BookDAO { ... }   // 인터페이스 구현 클래스는 보통 Impl을 붙임
```

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

| 방식 | 특징 |
| :--- | :--- |
| 필드 주입 | 코드가 가장 짧지만, `final` 불가, 스프링 없이 테스트할 때 값을 넣기 어려움 |
| setter 주입 | 주입 후에도 의존 객체를 바꿀 수 있음(선택적 의존성에 사용) |
| **생성자 주입** | `final`로 **불변** 보장, 객체 생성 시점에 의존성이 반드시 채워짐 → **가장 권장** |

---

## 5. 같은 타입의 Bean이 여러 개일 때

* `Pokemon` 인터페이스를 구현한 `Pikachu`, `Charmander`, `Squirtle`이 모두 bean으로 등록되어 있으면, `Pokemon` 타입 하나를 주입할 때 **어떤 bean을 넣을지 모호함** → 별도 설정이 없으면 예외 발생.

### @Primary

* 같은 타입 bean 중 **우선적으로 주입할 bean**을 지정함. 같은 타입 중 하나에만 붙일 수 있음.

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

### @Qualifier

* 주입할 bean을 **이름으로 직접 지정**함. `@Primary`보다 **우선**함.

```java
@Autowired
public PokemonService(@Qualifier("pikachu") Pokemon pokemon) {   // → Pikachu 주입
    this.pokemon = pokemon;
}
```

### List로 모두 주입

* 매개변수 타입을 `List<Pokemon>`으로 두면 해당 타입의 **bean을 모두** 담아서 주입함.

```java
@Autowired
public PokemonService(List<Pokemon> pokemonList) {
    this.pokemonList = pokemonList;
}

public void pokemonAttack() {
    pokemonList.forEach(Pokemon::attack);   // 피카츄, 파이리, 꼬부기 모두 공격
}
```

---

## 6. Bean Scope

* **scope**: bean 인스턴스가 **언제 생성되고 얼마나 공유되는지** 정하는 범위.

| scope | 동작 |
| :--- | :--- |
| `singleton` (기본값) | 컨테이너당 **인스턴스 하나**만 만들어서 모든 `getBean()`에 같은 객체를 반환 |
| `prototype` | `getBean()`을 호출할 때마다 **새 인스턴스**를 생성해서 반환 |

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

| scope | `cart2` 출력 결과 |
| :--- | :--- |
| `singleton` | 붕어빵, 딸기우유, 지리산암반수 → `cart1`과 **같은 카트**라 상품이 섞임 |
| `prototype` | 지리산암반수만 → 손님마다 **별도의 카트** |

> ⚠️ 상태(데이터)를 가지는 객체를 singleton으로 두면 여러 사용자가 같은 객체를 공유하게 됨. 장바구니처럼 사용자별로 따로 있어야 하는 객체는 prototype이 맞음.

---

## 📝 오늘의 정리

1. 객체 생성과 연결을 컨테이너에 맡기는 것이 **IoC**, 필요한 객체를 외부에서 넣어주는 것이 **DI**다.
2. Bean은 **XML**, **Java Config(`@Configuration` + `@Bean`)**, **Component Scan(`@Component`)** 으로 등록할 수 있다.
3. `@Autowired` 주입 방식 중 `final`로 불변을 보장하는 **생성자 주입**이 가장 권장된다.
4. 같은 타입 bean이 여러 개면 `@Primary`(기본 우선)나 `@Qualifier`(이름 지정, 더 우선)로 고르고, `List`로 모두 받을 수도 있다.
5. Bean의 기본 scope는 **singleton**이고, 매번 새 객체가 필요하면 `@Scope("prototype")`을 쓴다.

---

## 느낀 점

* 지금까지 `new`로 직접 만들고 넘겨주던 객체를 설정만 해두면 스프링이 알아서 만들고 연결해준다는 점이 신기했고, JDBC 때 Controller·Service·DAO를 직접 조립하던 코드가 왜 필요 없어지는지 이해됨.
* 필드를 인터페이스 타입으로 두고 생성자로 주입받으면 구현체를 바꿔도 다른 계층 코드는 그대로라는 점에서, Java에서 배운 다형성이 실제로 이렇게 쓰인다는 것을 실감함.
* singleton과 prototype을 장바구니 예제로 비교하니, bean을 공유해도 되는 객체와 따로 만들어야 하는 객체를 구분하는 것이 중요하다고 느낌.
