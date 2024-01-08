---
layout: post
title: Java Abstract Class
category: java
---

추상 클래스는 언제 사용해야 할까?   
아래에서 캐릭터를 생성하고 공격을 시도하는 간단한 게임 시나리오를 구현하고 추상 클래스를 활용해 보고자 한다.

```java
public abstract class Character {

    private String id;

    private String gender;

    private String armor;

    private String weapon;

    private String job;

    public Character(String id, String gender, String armor, String weapon, String job) {
        this.id = id;
        this.gender = gender;
        this.armor = armor;
        this.weapon = weapon;
        this.job = job;
    }

    public void walk() {
        System.out.printf("%s(%s)님이 걷기를 시작합니다.\n", id, job);
    }

    public void run() {
        System.out.printf("%s(%s)님이 뛰기를 시작합니다.\n", id, job);
    }

    public void jump() {
        System.out.printf("%s(%s)님이 점프를 시작합니다.\n", id, job);
    }

    abstract void attack();

    public String getId() {
        return id;
    }

    public String getGender() {
        return gender;
    }

    public String getArmor() {
        return armor;
    }

    public String getWeapon() {
        return weapon;
    }

    public String getJob() {
        return job;
    }
}
```

- Character Class는 유저가 캐릭터를 생성할 때 공통으로 가지는 요소들 입니다. 만약 이 추상 클래스를 구현하지 않고 각 직업에 대한 Class를 구현한다면 Character Class가 가지는 요소들을 모두 작성해야 하고, 만약 캐릭터에 대한 추가 요구사항이 들어온다면 각 직업의 Class들을 모두 수정해야 하는 리소스가 발생합니다.
- attack() **<u>추상 메서드</u>**는 Character Class를 상속받는 **<u>자식 클래스에서 반드시 구현</u>**해야 합니다.
- 필요에 따라 일반 메서드도 자식 클래스에서 구현하여 오버라이딩 할 수 있습니다.

---

```java
public class Job {
    public static class Warriors extends Character {

        public Warriors(String id, String gender, String armor, String weapon) {
            super(id, gender, armor, weapon, "전사");
        }

        @Override
        void attack() {
            System.out.printf("%s(으)로 내려치는 전사의 공격!\n", super.getWeapon());
        }
    }

    public static class Magician extends Character {

        public Magician(String id, String gender, String armor, String weapon) {
            super(id, gender, armor, weapon, "마법사");
        }

        @Override
        void attack() {
            System.out.printf("%s(으)로 파이어볼을 던지는 마법사의 공격!\n", super.getWeapon());
        }
    }
}
```

- super()를 통해 부모 클래스 생성자를 호출하여 초기값을 설정합니다.
- 서브 클래스 구현 시 정적 클래스로 선언하지 않는다면 외부참조[^1]에 의해 OutOfMemoryError가 발생할 수 있습니다.

---

```java
public class Game {

    public void start() {
        Character character1 = new Job.Warriors("나는짱짱", "female", "허름한 갑옷", "초보자 두손무기");
        Character character2 = new Job.Magician("나는최강", "male", "허름한 갑옷", "초보자 마법 지팡이");

        character1.walk();
        character1.run();
        character1.jump();

        character1.attack();
        character2.attack();
    }
}

/* 
* Output
* 나는짱짱(전사)님이 걷기를 시작합니다.
* 나는짱짱(전사)님이 뛰기를 시작합니다.
* 나는짱짱(전사)님이 점프를 시작합니다.
* 초보자 두손무기(으)로 내려치는 전사의 공격!
* 초보자 마법 지팡이(으)로 파이어볼을 던지는 마법사의 공격!
*/
```

---

> ⚠️ 다음과 같은 상황에서 추상 클래스 사용을 고려해볼 수 있을 것 같다.
> - **<u>여러 클래스에 중복되고 있는 요소</u>**들이 있는가
> - 중복되는 요소들은 유동적인 값을 가지는가
> - 다형성을 이용한 이익**<u>(재사용성)을 기대할 수 있는가</u>**
> - **<u>유지보수 및 기능 확장에 대한 불필요한 리소스를 방지</u>**할 수 있는가

---
{: data-content="footnotes"}

[^1]: 외부 클래스를 통해 서브 클래스의 인스턴스를 생성하는 것을 **외부참조**라고 하는데, 외부참조 시 외부 클래스를 사용하지 않아도 서브 클래스에 의해 GC의 회수 대상이 되지 않아 메모리 부족 현상이 발생한다. 해당 현상은 서브 클래스를 정적 클래스로 생성한다면 해결된다.