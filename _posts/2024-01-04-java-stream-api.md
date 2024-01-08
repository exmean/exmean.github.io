---
layout: post
title: Java Stream API
category: java
---

- Java Stream API은 하나의 흐름으로 진행된다.
- `중간연산`을 진행하고 `최종연산`을 거쳐야만 Stream Pipeline이 종료된다.

---

```java
public class User {
    private String name;
    private Integer age;

    public User(String name, Integer age) {
        this.name = name;
        this.age = age;
    }

    public String getName() {
        return name;
    }

    public Integer getAge() {
        return age;
    }
}
```
- User Class는 사용자의 이름과 나이 정보를 가지고 있습니다.   

```java
private List<User> members;

{
    members = new ArrayList<>();
    members.add(new User("김범수", 39));
    members.add(new User("김수월래", 23));
    members.add(new User("홍길동", 58));
    members.add(new User("홍사과", 17));
    members.add(new User("박재범", 30));
    members.add(new User("박찬호", 47));
}
```
- Members는 여러 명의 유저들을 분류 없이 저장하고 있습니다.   

```java
public class Team {
    private String name;
    private List<User> members;

    public Team(String name, List<User> members) {
        this.name = name;
        this.members = members;
    }

    public String getName() {
        return name;
    }

    public List<User> getMembers() {
        return members;
    }
}
```
- 유저들은 특정한 기준에 따라 Team에 소속될 수 있습니다.   

위 시나리오를 토대로 Stream API를 활용해 보도록 하자.

## Stream API 활용

- 저는 문득 유저들의 평균 나이가 궁금해졌습니다.
    
    ```java
    Double average = members.stream()
                  .mapToInt(x -> x.getAge())
                  .average()
                  .getAsDouble();
    
    int expected = (39 + 23 + 58 + 17 + 30 + 47) / members.size(); // 35
    Assertions.assertEquals(expected, average.intValue());
    ```
    - mapToInt() 메서드를 통해 정의한 람다 함수식을 실행하고 IntStream을 반환한다.
    - average() 메서드가 데이터 총합의 평균을 반환한다.
    - getAsDouble() 메서드가 Double 타입의 데이터를 리턴하고, 결과가 없다면 예외를 발생시킨다.   

<br>

- 갑자기 요청사항으로 김씨 성을 가진 유저들의 평균 나이 통계를 요구 받았습니다.
    
    ```java
    Double average = members.stream()
                  .filter(user -> user.getName().startsWith("김"))
                  .mapToInt(x -> x.getAge())
                  .average()
                  .getAsDouble();
    
    int expected = (39 + 23) / 2;   // 31
    Assertions.assertEquals(expected, average.intValue());
    ```
    - filter() 메서드를 통해 정의한 Predicate 람다식의 조건에 맞는 데이터를 리턴합니다.   

<br>

- 다른 추가 요구사항이 들어왔습니다. 메인 페이지에서 유저의 성을 뺸 이름 데이터가 필요하게 됐습니다.
    
    ```java
    String[] names = members.stream()
                  .map(member -> member.getName().substring(1))
                  .toArray(String[]::new); // [범수, 수월래, 길동, 사과, 재범, 찬호]
    ```
    - map() 메서드는 정의한 Fucntional 람다식을 실행한 데이터를 리턴한다.   

<br>

- 이번에는 유저들의 나이 총합의 데이터가 필요합니다.
    
    ```java
    int totalSumAge = members.stream()
                    .reduce(0, (total, user) -> total += user.getAge(), Integer::sum);
    ```
    - 0으로 설정한 identity 인자는 초기값을 설정한다.
    - reduce() 메서드는 지정한 인자에 대한 연산을 진행한다.
    - 연산을 위해 선언한 첫번째 인자(total)와 두번째 인자(user)의 타입이 서로 다를 경우 Integer::sum과 같이 BinaryOperator의 `combiner`를 통해 수행할 연산과 타입을 명시적으로 선언하여야 한다.
    
    ```java
    int totalSumAge = members.stream()
                    .mapToInt(user -> user.getAge())
                    .reduce(0, Integer::sum);
    
    int expected = 214;
    Assertions.assertEquals(expected, totalSumAge);
    ```
    - 위 코드와 같이 Int 연산을 수행할 것이라면 IntStream을 반환하고 같은 유형의 연산을 처리하는 것이 개인적으론 가독성 면에서 더 좋은 것 같다.
    
    ```java
    String names = members.stream()
                .map(member -> member.getName().substring(1))
                .reduce("인원목록: ", String::concat);
    
    String expected = "인원목록: 범수수월래길동사과재범찬호";
    Assertions.assertEquals(expected, names);
    ```
    - 문자열 처리도 가능하다.
    - 메서드 참조 대신 직접 accumulator를 구현하면 원하는 문자열 양식을 구현할 수 있다.   
    
<br>

- 특정 팀들의 소속 멤버 중 30대 이상 인원 수를 구해야 합니다.
    
    ```java
    Team team1 = new Team("1팀", List.of(members.get(0), members.get(3)));
    Team team2 = new Team("2팀", List.of(members.get(2), members.get(5)));
    List<Team> teams = List.of(team1, team2);
    
    long over30s = teams.stream()
            .flatMap(x -> x.getMembers().stream())
            .filter(user -> user.getAge() >= 30)
            .count();
    
    long expected = 3l;
    Assertions.assertEquals(expected, over30s);
    ```
    - flatMap() 메서드는 2차원 배열이나 Stream<Stream<?>> 같은 구조일 때 하나의 Stream으로 평탄화 하는 기능을 수행한다.   

<br>

- 팀의 소속된 인원을 조회할 때 조회자의 정보를 로그에 출력하려고 합니다.
    
    ```java
    User client = new User("조회자", 43);
    String targetName = "김범수";
    Team team1 = new Team("1팀", List.of(members.get(0), members.get(3)));
    
    User findUserResult = team1.getMembers().stream()
            .filter(member -> member.getName().equals(targetName))
            .peek(e -> System.out.println(client.getName() + "님이 " + targetName + "님의 정보를 조회하였습니다."))
            .findFirst()
            .orElseThrow(() -> new NoSuchElementException("해당 팀에는 " + targetName + "님이 소속되어 있지 않습니다."));
    
    Assertions.assertEquals(targetName, findUserResult.getName());
    ```
    - peek() 메서드는 결과를 리턴하지 않는 Consumer accept 람다식을 구현하여 로직을 수행한다.   

---

자주 사용하는 Stream API에 대한 기능을 간단한 시나리오와 함께 정리해 보았다(정리한 내용 외에도 쇼트서킷 기반 기능, 정렬 기능 등 많은 것들이 있지만 누구나 API Document를 보면 직관적으로 이해될 법한 기능들이라고 생각한다). Stream API를 실제 서비스에 적용하면서 가독성 측면에선 많은 이득을 볼 수 있었다(성능 면에서는 유의미한 차이를 확인할 순 없었다). 하지만 함께 일하는 팀원들이 함수형 프로그래밍에 대한 이해가 없다면 오히려 이전 코드보다 가독성 효율이 떨어질 수도 있다. 팀의 컨벤션을 지키는 것이 참 중요하다는 것을 느끼고 같이 스터디 해나가는 방법도 좋을 것 같다.

---

> - **Reference**   
> [java-8-flatmap-example]([https://mkyong.com/java8/java-8-flatmap-example/](https://mkyong.com/java8/java-8-flatmap-example/))   
> [java-stream-reduce]([https://www.baeldung.com/java-stream-reduce](https://www.baeldung.com/java-stream-reduce))   