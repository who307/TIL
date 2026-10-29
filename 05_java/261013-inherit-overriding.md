# Java 상속 · 오버라이딩 · 다형성 정리 (TIL)

## 1. 상속 (Inheritance)과 `super` 키워드

### 상속

* **한 줄 비유**: 상속은 **부모님께 물려받는 것**임. 부모 클래스가 가진 필드와 메소드를 자식 클래스가 **그대로 받아** 쓰고, 자기만의 것을 더 붙일 수 있음.
* **쓰는 법**: `class 자식 extends 부모`
* **왜 쓰나**: `Car`(자동차)와 `FireCar`(소방차)는 "달리기, 멈추기"가 똑같음. 소방차에 그 코드를 또 쓰지 않고 `Car`에게 물려받은 뒤, 소방차만의 "물 뿌리기"만 추가하면 됨.

```java
public class Car {
    public void run() {
        System.out.println("자동차가 달립니다.");
    }
}

public class FireCar extends Car {
    public void sprayWater() {
        System.out.println("물을 뿌립니다.");
    }
}
```

* **따라가기**: `FireCar fc = new FireCar();`
  * `fc.run()` → `FireCar`에는 없지만 부모 `Car`에서 물려받음 → `자동차가 달립니다.`
  * `fc.sprayWater()` → 자기 메소드 → `물을 뿌립니다.`

### `super`

* **`super`**: 자식 안에서 **부모**를 가리키는 말. (`this`가 "나"라면 `super`는 "우리 부모님")
  * `super(값들)`: **부모의 생성자**를 부름. 자식 객체를 만들 때 부모 부분을 먼저 채워야 하므로, 자식 생성자의 **첫 줄**에 씀. 안 쓰면 자바가 몰래 `super()`(부모 기본 생성자)를 넣어줌.
  * `super.메소드()`: 자식이 같은 이름으로 재정의했더라도 **부모 버전**을 부름.
* `Product`(상품) → `Computer`, `SmartPhone` 예시: 상품 공통 정보(코드, 이름, 가격)는 부모 생성자가 채우고, 자식은 자기 정보(CPU, 통신사 등)만 채움.

```java
public class Product {
    private String name;
    private int price;

    public Product(String name, int price) {
        this.name = name;
        this.price = price;
    }
}

public class SmartPhone extends Product {
    private String carrier;

    public SmartPhone(String name, int price, String carrier) {
        super(name, price);      // 부모 생성자로 공통 정보 먼저 채움
        this.carrier = carrier;  // 그다음 내 정보
    }
}
```

* **따라가기**: `new SmartPhone("갤럭시", 1000000, "SKT")`
  1. `SmartPhone` 생성자 시작 → 첫 줄 `super("갤럭시", 1000000)` → `Product` 생성자가 `name`, `price`를 채움.
  2. 돌아와서 `carrier = "SKT"`를 채움.
  3. 부모 부분과 자식 부분이 모두 채워진 스마트폰 객체 완성.

---

## 2. 메소드 오버라이딩 (Method Overriding)

* **오버라이딩**: 부모에게 물려받은 메소드를 자식이 **같은 이름·같은 매개변수로 다시 만들어 덮어쓰는 것**. 할머니 김치찌개 레시피를 물려받아 **내 입맛대로 고쳐** 끓이는 것과 같음.
* **규칙**: 이름, 매개변수, 반환형이 부모와 같아야 함. 접근 범위는 부모보다 좁힐 수 없음. `@Override`를 붙이면 실수로 이름을 틀렸을 때 컴파일러가 알려줌.
* **오버로딩과 헷갈리지 않기**: 오버**로딩**은 같은 클래스에서 이름은 같고 **매개변수가 다른** 메소드를 여러 개 만드는 것, 오버**라이딩**은 상속받은 메소드를 **똑같은 모양으로 덮어쓰는** 것.

```java
public class VendingMachine {
    public void sell() {
        System.out.println("음료를 판매합니다.");
    }
}

public class CoffeeVendingMachine extends VendingMachine {
    @Override
    public void sell() {
        System.out.println("커피를 판매합니다.");
    }
}
```

* **따라가기**
  * `new VendingMachine().sell()` → `음료를 판매합니다.`
  * `new CoffeeVendingMachine().sell()` → 자식이 덮어쓴 버전이 실행 → `커피를 판매합니다.`

---

## 3. 다형성 (Polymorphism)

* **다형성**: 하나의 타입(부모)으로 **여러 모양(자식 객체들)** 을 다룰 수 있는 성질. "동물"이라는 이름으로 토끼도, 호랑이도 부를 수 있는 것과 같음.
* **업캐스팅**: 자식 객체를 부모 타입 변수에 담는 것 → `Animal a = new Rabbit();` (자동으로 됨)
* **동적 바인딩**: 변수 타입이 `Animal`이어도, 실제로 실행되는 메소드는 **진짜 객체(Rabbit)가 오버라이딩한 버전**임. 무엇이 실행될지가 실행할 때 정해짐.
* **장점**: `Zookeeper`(사육사)는 토끼용, 호랑이용 메소드를 따로 만들 필요 없이 `Animal` 하나만 받아서 먹이를 주면 됨. 새로운 동물이 늘어도 사육사 코드는 그대로임.

```java
public class Animal {
    public void eat() { System.out.println("동물이 먹습니다."); }
}

public class Rabbit extends Animal {
    @Override
    public void eat() { System.out.println("토끼가 풀을 먹습니다."); }
}

public class Tiger extends Animal {
    @Override
    public void eat() { System.out.println("호랑이가 고기를 먹습니다."); }
}

public class Zookeeper {
    public void feed(Animal animal) {   // 어떤 동물이든 받음
        animal.eat();
    }
}
```

* **따라가기**: `Zookeeper keeper = new Zookeeper();`
  * `keeper.feed(new Rabbit())` → `animal` 변수 타입은 `Animal`이지만 진짜 객체는 토끼 → `토끼가 풀을 먹습니다.`
  * `keeper.feed(new Tiger())` → `호랑이가 고기를 먹습니다.`

---

## 4. 추상 클래스 (Abstract Class)

* **추상 클래스**: `abstract`를 붙인 **미완성 설계도**. 그 자체로는 `new`로 객체를 만들 수 없음.
* **추상 메소드**: 몸통(`{ }`) 없이 이름만 적어둔 메소드 → `public abstract void sound();`
* **역할**: "동물이면 **반드시** 울음소리를 낼 수 있어야 해. 어떻게 우는지는 각자 정해"라고 **자식에게 구현을 강제**함. 자식이 추상 메소드를 구현하지 않으면 컴파일 에러.
* 위 `Animal`을 `public abstract class Animal`로 바꾸면, "그냥 동물"이라는 애매한 객체를 만드는 실수를 막을 수 있음.

---

## 느낀 점

* 상속과 `super` 키워드를 통해 코드의 중복을 줄이고 클래스 간의 계층적 구조를 명확히 구축하는 방식을 체득함
* 메소드 오버라이딩을 활용하여 자식 클래스에서 상황에 맞게 기능을 유연하게 변경하는 방법을 파악함
* 다형성과 추상 클래스를 다루며 객체지향 설계의 확장성과 유연성을 높이는 핵심 패턴을 이해함
