---
layout: post
title: Java Interface
category: java
---

- Java Interface는 어느 경우에 활용할 수 있을까?
- Abstract Class(추상 클래스)와는 어떤 차이가 있을까?

---

나는 `Java Interface`를 규격이라고 정의하고 싶다. 지금부터 그 예시로 기부 상점의 기능을 구현하려고 한다. 오프라인에 있는 기부 상점에 방문한 고객은 키오스크로 기부된 품목을 구입하거나 기부를 할 수 있다.   

```java
public interface DonationShop {

    void buy();

    void donation();
}
```

이렇게 인터페이스를 보면 설계 규격을 한 눈에 알 수 있다. 그리고 인터페이스의 구현체는 선언된 추상 메서드를 반드시 구현해야 하기 때문에 정해진 **규격대로 일관성 있게 개발**을 할 수 있다.   

```java
public class ShopKiosk implements DonationShop {

    @Override
    public void buy() {
        System.out.println("기부 품목을 구입합니다.");
    }

    @Override
    public void donation() {
        System.out.println("물건을 기부합니다.");
    }
}
```

---

온라인 기부에 대한 추가 요구사항을 들어왔다고 가정하겠다. 온라인 기부 상점에서는 물건을 기부할 순 없고 돈으로만 기부할 수 있다.   

```java
public class OnlineShop implements DonationShop {

    @Override
    public void buy() {
        System.out.println("기부 품목을 구입합니다.");
    }

    @Override
    public void donation() {
        System.out.println("돈을 기부합니다.");
    }
}
```

이렇게 온라인 기부 기능도 DonationShop Interface 규격에 따라 설계 되었다. 또한 **인터페이스는 기본적인 변수를 가질 순 없지만 상수를 필드로 선언할 수 있다.** 만약에 기부 품목을 최대 10개까지만 구매할 수 있도록 정책이 정해졌다고 가정해보자. 각 구헌 클래스에서 최대 개수인 10을 상수로 선언하고 관련된 비즈니스 로직을 각각 구현할 수 있겠지만, 그렇게 된다면 유지보수에 대한 생산성이 굉장히 떨어질 것이다.   

```java
public interface DonationShop {

    Integer POSSIBLE_DONATION_ITEM_COUNT = 10;

    void buy();

    void donation();

    default boolean isPossibleDonationItemCount(int countDonationItem) {
        return countDonationItem < POSSIBLE_DONATION_ITEM_COUNT ? true : false;
    }
}
```

이렇게 추가 요구사항에 대한 대처도 유연하게 할 수 있다. 그래서 인터페이스는 유연한 설계가 가능하다는 것이다. 만약 오프라인 기부 상점의 키오스크에서만 겨울 간식을 팔기 위해 기능을 추가한다고 해보자. 기존에 있던 DonationShop Interface에 추가적인 설계를 하게 된다면 기존에 동일한 인터페이스를 상속을 받고 있던 OnlineShop Class에서도 전혀 상관없는 기능을 구현해야 할 것이다.   

```java
public interface WinterSnack {

    void buyWinterSnack();
}

public class ShopKiosk implements DonationShop, WinterSnack {

    @Override
    public void buy() {
        System.out.println("기부 품목을 구입합니다.");
        int countBuyDonationItem = 11;

        System.out.println("구매 한정수량: ");
        System.out.println("구매하려는 수량: " + countBuyDonationItem);

        if (isPossibleDonationItemCount(countBuyDonationItem)) {
            System.out.println("구매완료!");
        } else {
            System.out.println("구매실패...");
        }
    }

    @Override
    public void donation() {
        System.out.println("물건을 기부합니다.");
    }

    @Override
    public void buyWinterSnack() {
        System.out.println("겨울 간식을 구입합니다.");
    }
}
```

하지만 위 코드처럼 새로운 기능에 대한 새로운 인터페이스를 생성하고 ShopKiosk에 상속한다면 ShopKiosk에서만 새로운 기능을 구현할 수 있도록 설계할 수 있다.   

---

그렇다면 비슷한 역할을 하는 **추상 클래스와 인터페이스는 어떤 차이점을 가지고 있는 것일까?**

- 인터페이스는 **다중 상속을 지원**한다.
- 사용할 수 있는 메서드가 한정적이다(private, default, static, abstract)
- **변수로 상수만 선언이 가능**하다.

개인적인 판단으로는 개념적으로 논리적인 연관이 있는 공통 분모들은 추상 클래스로 설계를 하고, 그 외 유연한 설계가 필요한 기능들은 인터페이스로 설계를 하는 것이 좋지 않을까 생각한다.

> - **Reference**   
> [인터페이스 vs 추상클래스 차이점 완벽 이해하기](https://inpa.tistory.com/entry/JAVA-%E2%98%95-%EC%9D%B8%ED%84%B0%ED%8E%98%EC%9D%B4%EC%8A%A4-vs-%EC%B6%94%EC%83%81%ED%81%B4%EB%9E%98%EC%8A%A4-%EC%B0%A8%EC%9D%B4%EC%A0%90-%EC%99%84%EB%B2%BD-%EC%9D%B4%ED%95%B4%ED%95%98%EA%B8%B0)   