---
title: "[Design Pattern] Singleton pattern"
excerpt: "Design Pattern의 Singleton pattern"

categories:
  - Design Pattern
tags:
    - Creational patterns
    - Singleton

toc: true
toc_sticky: true
---

# Singleton pattern
## 목적
- 클래스의 인스턴스화를 제한하여 JVM에 클래스의 __인스턴스가 하나만 존재하도록 보장__
  - 이유 : 일부 공유 자원(ex. DB나 file 등 리소스를 많이 차지하는 무거운 클래스)에 대한 접근 제어, <u>메모리 절약</u>
- 클래스의 인스턴스를 가져오기 위한 __글로벌 액세스 지점을 제공__
- logging, drivers 객체, caching, thread pool에 사용됨
- java.lang.Runtime, java.awt.Desktop에서도 사용됨

예를들어, 게임이나 IDE의 설정화면

## 구현 방법

총 8가지 구현 방법이 존재하며 각각이 장단점을 가진다.

- Lazy initialization
- Eager initialization
- Static block initialization
- Thread safe
- Double-Checked Locking
- Bill Pugh Solution (LazyHolder)
- Enum

### Lazy initialization
```java
public class Settings {
    private static Settings instance;

    private Settings() {}

    public static Settings getInstance() {
        if (instance == null) {
            instance = new Settings();
        }

        return instance;
    }
}
```
- 가장 naive한 방식
- 객체 생성에 대한 관리를 내부적으로 처리
- Eager initialization 방식의 고정 메모리 차지 문제점 해결
- Thread-safe하지 않음

### Eager initialization
```java
public class Settings {
    private static final Settings INSTANCE = new Settings();

    private Settings() {}

    public static Settings getInstance() {
        return INSTANCE;
    }
}
```
- 클래스 로딩 시 인스턴스 생성
- Thread-safe
- 인스턴스가 크지 않은 경우 고려할 수 있는 방식
- 해당 인스턴스를 사용하지 않는 경우에도 인스턴스가 생성되는 단점
- 예외 처리할 수 없음

### Static block initialization
```java
public class Settings {
    private static Settings instance;
    
    private Settings() {}
    
    static {
        try {
            instance = new Settings();
        } catch (Exception e) {
            throw new RuntimeException("Exception occurred in creating singleton instance");
        }
    }
    
    public static Settings getInstance() {
        return instance;
    }
}
```
- Eager initialization와 같이 클래스 로딩 시 인스턴스 생성
- 예외 처리 가능함
- 해당 인스턴스를 사용하지 않는 경우에도 인스턴스가 생성되는 단점

### Thread safe
```java
public class Settings {
    private static Settings instance;

    private Settings() {}

    public static synchronized Settings getInstance() {
        if (instance == null) {
            instance = new Settings();
        }

        return instance;
    }
}

```
- `synchronized` 키워드를 사용하여 lock을 걸어 race condition(경쟁상태) 예방
- 메서드 전체에 `synchronized`가 걸려 overhead가 발생해 성능 저하 발생

### Double-Checked Locking
```java
public class Settings {
    private static volatile Settings instance;

    private Settings() {}

    public static Settings getInstance() {
        if (instance == null) {
            synchronized (Settings.class) {
                if (instance == null) {
                    instance = new Settings();
                }
            }
        }

        return instance;
    }
}
```
- 메서드 전체에 `synchronized`가 걸려 overhead가 발생해 성능 저하 방지하기 위한 기법
- `volatile` 키워드를 멤버 변수에 붙여야 함. JDK 1.5 이상과 JVM에 대한 이해와 JVM에 따라 Thread-safe 하지 않아 지양

### Bill Pugh Solution (LazyHolder)
```java
public class Settings implements Serializable {

    private Settings() {}

    private static class SettingsHolder {
        private static final Settings INSTANCE = new Settings();
    }

    public static Settings getInstance() {
        return SettingsHolder.INSTANCE;
    }

    /**
     * 역직렬화 대응 방안
     * @return
     */
    protected Object readResolve() {
        return getInstance();
    }
}
```
- 권장되는 방법
- Thread-safe하며, lazy loading 가능
- 클래스 안에 내부 클래스(Holder)를 두어 JVM의 클래스 로더 방식과 클래스 로드되는 시점을 이용
- 다만, singleton ~~파괴 기법(?)~~ 뚫을 수 있음(Reflection, 직렬화/역직렬화 ~~- 위 대응이 있기는 하지만~~)

### Enum
```java
public enum Settings {
    INSTANCE;
}
```
- 권장되는 방법
- enum은 개념 자체가 private 생성자이며 한번만 초기화하여 Thread-safe
- Reflection, 직렬화/역직렬화 공격에도 안전
- 상속이 불가능

> 최종 정리하자면, 싱글톤 패턴 클래스를 만들기 위해서는 Bill Pugh Solution 기법을 사용하거나 Enum으로 만들어 사용하면 된다.
> 다만, 이 둘의 사용 선택은 자신의 싱글톤 클래스의 목적에 따라 갈리게 된다고 보면 된다.
> - LaszHolder : 성능이 중요시 되는 환경
> - Enum : 직렬화, 안정성 중요시 되는 환경
> <br> [<cite>inpa</cite> GOF-💠-싱글톤Singleton-패턴-꼼꼼하게-알아보자](https://inpa.tistory.com/entry/GOF-%F0%9F%92%A0-%EC%8B%B1%EA%B8%80%ED%86%A4Singleton-%ED%8C%A8%ED%84%B4-%EA%BC%BC%EA%BC%BC%ED%95%98%EA%B2%8C-%EC%95%8C%EC%95%84%EB%B3%B4%EC%9E%90)

## 단점
1. 모듈간 의존성 증가
2. SOLID 원칙 위배 가능성 높음 (SRP, OCP, DIP)
3. TDD 단위 테스트 어려움

