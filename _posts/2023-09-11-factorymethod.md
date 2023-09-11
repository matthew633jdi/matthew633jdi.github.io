---
title: "[Design Pattern] Factory Method pattern"
excerpt: "Design Pattern의 Factory Method pattern"

categories:
  - Design Pattern
tags:
    - Creational patterns
    - Factory Method

toc: true
toc_sticky: true
date: 2023-09-11
---

# Factory Method

![출처 : https://refactoring.guru](https://refactoring.guru/images/patterns/content/factory-method/factory-method-en.png?id=cfa26f33dc8473e803fadae0d262100a)

## 목적
- super class에서 객체를 생성하기 위한 인터페이스 제공
- sub class가 생성될 객체의 유형을 변경할 수 있도록 허용
- 객체 생성을 도맡아 처리하는 공장 클래스와 이를 상속하는 서브 공장 클래스를 통해 여러 제품 생성을 각각 책임지는 것
- 객체 생성이 필요한 과정을 템플릿화해 객체 생성 전/후처리 가능
- 객체간 `결합도` 낮아짐

### Context
Ex
> 물류 관리 애플리케이션을 만드는 경우, 초기 버전은 트럭을 이용한 운송 수단만 처리할 수 있으므로 Truck 클래스 내에 있음.
> 
> 추후 추가 요구사항에 따라 별도의 해상 물류가 추가되는 경우, 대부분의 코드가 Truck 클래스에 결합되어 있어 선박을 추가하려면 전체 코드베이스 변경이 불가피

### Structure

![출처 : https://refactoring.guru/](https://refactoring.guru/images/patterns/diagrams/factory-method/structure.png?id=4cba0803f42517cfe8548c9bc7dc4c9b)

- Creator
  - 최상위 팩토리 클래스
- ConcreteCreator
  - 각 서브 팩토리 클래스. 각 제품 객체를 반환하도록 생성 메서드를 정의
- Product
  - 제품 추상화
- ConcreteProduct
  - 각 제품

## 구현

### Before

초기 Ship 생성하는 공장

```java
@Getter
@Setter
public class Ship {
    private String name;
    private String color;
    private String logo;
}
```
```java
public class ShipFactory {

    public static Ship orderShip(String name, String email) {
        // validate
        if (name == null || name.isBlank()) {
            throw new IllegalArgumentException("please ship name");
        }

        if (email == null || email.isBlank()) {
            throw new IllegalArgumentException("please email");
        }

        prepareFor(name);

        Ship ship = new Ship();
        ship.setName(name);

        // Customizing for specific name
        if (name.equalsIgnoreCase("whiteship")) {
            ship.setLogo("\uD83D\uDEE5️");
        } else if (name.equalsIgnoreCase("blackship")) {
            ship.setLogo("⚓");
        }

        // coloring
        if (name.equalsIgnoreCase("whiteship")) {
            ship.setColor("white");
        } else if (name.equalsIgnoreCase("blackship")) {
            ship.setColor("black");
        }

        // notify
        sendEmailTo(email, ship);

        return ship;
    }

    private static void sendEmailTo(String email, Ship ship) {
        System.out.println(ship.getName() + " was done.");
    }

    private static void prepareFor(String name) {
        System.out.println(name + " prepare!");
    }


}

```

```java
public static void main(String[] args) {
        Ship whiteship = ShipFactory.orderShip("Whiteship", "e@mail.com");
        System.out.println(whiteship);

        Ship blackship = ShipFactory.orderShip("Blackship", "e@mail.com");
        System.out.println(blackship);
    }
```
ShipFactory 클래스의 orderShip() 메서드가 너무 많은 역할을 수행.
또한, 추가로 별도의 기능이 추가되면 전체적인 코드베이스 수정이 필요

### After

각 제품에 대한 추상화. 여기서는 위 `Product` 클래스 사용

```java
public interface ShipFactory {

    default Ship orderShip(String name, String email) {
        validate(name, email);
        prepareFor(name);
        Ship ship = createShip();
        sendEmailTo(email, ship);
        return ship;
    }

    Ship createShip();

    void sendEmailTo(String email, Ship ship);

    // jdk 9
    // 이전 버전이면 추상 클래스 사용해야 함
    private void validate(String name, String email) {
        if (name == null || name.isBlank()) {
            throw new IllegalArgumentException("배 이름을 지어주세요.");
        }
        if (email == null || email.isBlank()) {
            throw new IllegalArgumentException("연락처를 남겨주세요.");
        }
    }

    private void prepareFor(String name) {
        System.out.println(name + " prepare!");
    }

    private void sendEmailTo(String email, Ship ship) {
        System.out.println(ship.getName() + " was done.");
    }
}
```

객체 인스턴스 생성에 필요한 과정을 orderShip()메서드에 설정.
각 인스턴스 생성에 필요한 것은 서브 클래스가 구현하도록 함. (`createShip() 메서드`)

아래는 각 인스턴스를 생성하는 서브 팩토리 클래스

```java
public class BlackshipFactory implements ShipFactory {
    @Override
    public Ship createShip() {
        return new Blackship();
    }
}
```

```java
public class WhiteshipFactory implements ShipFactory {
    @Override
    public Ship createShip() {
        return new Whiteship();
    }
}
```

각 Product는 아래와 같음
```java
public class Blackship extends Ship {

    public Blackship() {
        setName("blackship");
        setColor("black");
        setLogo("⚓");
    }
}
```

```java
public class Whiteship extends Ship {
    public Whiteship() {
        setName("whiteship");
        setLogo("\uD83D\uDEE5️");
        setColor("white");
    }
}
```
## 특징
- 결합도 낮추기
- 객체 유형과 종속성을 캡슐화해 정보 은닉
- SRP, OCP 준수
- 서브 클래스가 많아 질 수 있음
- 복잡도가 높아질 수 있음

## 실 사용
- Spring의 BeanFactory
- Java의 Calendar.getInstance()
- etc