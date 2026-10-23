# Java 배열 정렬 · 객체지향 기초 정리 (TIL)

## 1. 배열 정렬 알고리즘 (Sorting Algorithms)

* **값 교환(Swap) 메커니즘**: 배열 내부 요소의 위치를 변경하기 위해 임시 변수(`temp`)를 선언하고 두 변수의 값을 안전하게 교환하는 기본 패턴을 습득함.


* **버블 정렬 (Bubble Sort)**: 인접한 두 개의 원소를 비교하며 조건에 따라 자리를 교환하여, 순차적으로 가장 큰 값을 배열의 끝으로 보내는 정렬 알고리즘 제어 방식을 파악함.


* **선택 정렬 (Selection Sort)**: 해당 자리에 들어갈 최솟값(또는 최댓값)의 인덱스를 찾아 배열 전체를 순회한 후, 한 번의 교환으로 정렬 위치를 확정하는 방식을 체득함.


* **삽입 정렬 (Insertion Sort)**: 이미 정렬된 부분 배열과 비교하여 적절한 위치에 원소를 삽입해 나가는 정렬 구조 및 이중 `for`문 인덱스 핸들링 기법을 이해함.



---

## 2. 객체지향 프로그래밍 (OOP) 기초

### 사용자 정의 타입 (User Defined Type)

* 클래스(Class)를 활용하여 서로 다른 자료형의 변수들을 하나의 단위로 묶어, 개발자가 필요한 새로운 데이터 타입(`Member`)을 직접 정의하고 인스턴스화하는 개념을 이해함.



### 캡슐화 (Encapsulation)

* 클래스 내부 필드에 `private` 접근 제어자를 적용하여 외부의 직접적인 데이터 접근 및 오염을 원천 차단함.


* 올바른 데이터 검증 로직을 거쳐 필드 값을 변경 및 조회하도록 유도하는 정보 은닉(Information Hiding) 메커니즘을 체득함.



### 추상화 및 객체 간 협력 (Abstraction & Cooperation)

* 현실 세계의 사물과 행위를 추상화하여 클래스(`Car`, `CarRacer`)로 모델링함.


* 운전자가 자동차의 시동을 걸고 Accel/Break 메소드를 호출하듯, 객체가 다른 객체에게 메시지(메소드 호출)를 전달하여 상호작용하는 협력 관계 구조를 파악함.



### DTO (Data Transfer Object) 패턴

* 계층 간 데이터 전송을 목적으로 하는 객체 구조인 DTO(`MemberDTO`) 설계 표준을 습득함.


* `private` 필드와 이를 안전하게 참조·수정할 수 있는 `public` Getter/Setter 메소드를 구성하는 정석적인 클래스 구조를 이해함.



### 생성자 (Constructor) & `this` 키워드

* `new` 연산자로 객체가 생성될 때 필드의 초기화를 담당하는 기본 생성자(Default Constructor) 및 매개변수 있는 생성자 작성법을 체득함.


* `this` 키워드를 활용해 인스턴스 자기 자신의 메모리 주소를 참조함으로써, 매개변수명과 필드명이 동일할 때 발생하는 이름 충돌(Shadowing) 문제를 해결하는 방식을 파악함.



---

## 핵심 코드 예시

```java
// 1. 선택 정렬 및 Swap 구현 예시 (section04/sort)
int[] arr = {2, 5, 4, 1, 3};

for (int i = 0; i < arr.length - 1; i++) {
    int minIndex = i;
    for (int j = i + 1; j < arr.length; j++) {
        if (arr[j] < arr[minIndex]) {
            minIndex = j; // 최솟값 인덱스 탐색[cite: 32]
        }
    }
    // Swap 메커니즘[cite: 32]
    int temp = arr[i];
    arr[i] = arr[minIndex];
    arr[minIndex] = temp;
}

```

```java
// 2. DTO 구조 및 생성자 / this 활용 예시 (section03/dto & section04/constructor)
package com.woong3e.section03.dto;

public class MemberDTO {
    private String id;   // private 캡슐화[cite: 33]
    private String name;

    // 기본 생성자[cite: 33]
    public MemberDTO() {}

    // 매개변수 생성자 및 this 활용[cite: 33]
    public MemberDTO(String id, String name) {
        this.id = id;     // 인스턴스 필드 명시[cite: 33]
        this.name = name;
    }

    // Getter & Setter[cite: 33]
    public String getId() {
        return id;
    }

    public void setId(String id) {
        this.id = id;
    }
}

```

---

## 느낀 점

* 단순 일차원 배열 조작을 넘어, 대표적인 정렬 알고리즘(버블, 선택, 삽입)의 동작 원리와 인덱스 연산 패턴을 정리하며 조건/반복문 제어 능력을 다짐.


* 단순히 변수와 메소드를 나열하는 방식에서 벗어나, `private` 필드와 `public` 메소드를 조합한 캡슐화의 필요성을 명확히 인지함.


* DTO 패턴의 구조와 생성자(`this`) 활용법을 다져봄으로써, 데이터를 안전하고 표준화된 방식으로 객체에 담아 전달하는 객체지향적 설계의 첫걸음을 내딛음.