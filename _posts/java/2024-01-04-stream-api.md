---
layout: post
title: Stream API를 이용해서 레거시 코드 리팩터링 하기
category: Java
permalink: /java/1
---

**Stream API**는 Java 8 버전에서 처음 등장했습니다. Java 21 버전까지 나온 현재는 이미 Stream API에 대한 정보들이 많이 있습니다. 그래서 Stream API를 사용하는 방법에 대해 작성하기 보다는 제가 회사에 존재하는 레거시 코드들을 Stream API를 이용해 어떻게 리팩터링 했는지, 그리고 어떤 결과를 얻게 되었는지 작성해보려고 합니다. Stream API는 파이프라인을 이용한 연산입니다. 쉽게 설명하자면 연산마다 새로운 Stream을 반환하고 그 Stream을 이용하여 또 다른 연산을 이어나가는 방식입니다. 그리고 반드시 최종연산을 거쳐야지만 모든 파이프라인 연산이 종료되어 연산 결과를 얻을 수 있습니다. 글로 풀어 작성해보자니 어렵게 느껴지지만 막상 사용해보면 직관적으로 알 수 있을겁니다!

# 실제 서비스에 적용하기
## CASE. 비정상적인 접근 제한
지금 개발하고자 하는 교육 플랫폼은 학생들이 이용할 수 있는 클래스라는 개념이 있습니다.   
만약 비정상적인 루트로 가입되지 않은 클래스로 접근할 경우 가입하지 않은 클래스의 정보들까지 모두 조회할 수 있기 때문에 검증 로직을 통해 접근을 제한하고자 합니다.
### 리팩터링 전

``` java
    private List<ClassInformation> joinClassList = Arrays.asList(
            new ClassInformation(10001L, "1번 클래스"),
            new ClassInformation(10002L, "2번 클래스"),
            new ClassInformation(10003L, "3번 클래스")
    );

    @Test
    void 비정상적인_접근_가입되지_않은_클래스_리팩터링전() {

        Long requestClass = 10004L;
        List<Long> idList = new ArrayList<>();
        for (ClassInformation clazz : joinClassList) {
            idList.add(clazz.getId());
        }

        boolean expected = idList.contains(requestClass);
        assertThat(expected).isFalse();
    }
```
`requestClass`는 현재 유저가 접속하려고 하는 클래스입니다. `idList` 배열은 유저가 가입한 클래스 목록의 `id` 값을 저장하기 위해서 선언되었습니다. for문을 통해 `idList` 배열에 `id` 값을 모두 저장합니다. 그리고 저장된 배열에 `contains()` 메서드를 이용해서 `requestClass`와 동일한 값이 있는지 확인합니다. 현재는 기존 코드를 간단하게 재현하기 위해서 boolean으로 체크를 하고 있지만, 실제 서비스에 적용한다면 계획된 예외를 발생시켜 원하는 방향으로 후처리를 진행할 수 있겠습니다! 리팩터링 하기 전 코드도 적절한 주석을 달아둔다면 이해하기 쉬운 코드라고 생각합니다. 하지만 **Stream API**를 사용하면 더욱 더 간결하고 가독성이 좋게 로직을 구성할 수 있습니다.

### 리팩터링 후
``` java
    private List<ClassInformation> joinClassList = Arrays.asList(
            new ClassInformation(10001L, "1번 클래스"),
            new ClassInformation(10002L, "2번 클래스"),
            new ClassInformation(10003L, "3번 클래스")
    );

    @Test
    void 비정상적인_접근_가입되지_않은_클래스_리팩터링후() {

        Long requestClass = 10004L;
        boolean expected = joinClassList.stream()
                .mapToLong(clazz -> clazz.getId())
                .filter(id -> id == requestClass)
                .findFirst()
                .isPresent();

        assertThat(expected).isFalse();
    }
```
`requestClass`는 이전과 동일하게 유저가 접속을 요청한 클래스입니다. 초기값으로 설정되어 있는 `joinClassList` 배열 Stream을 생성하고 mapToLong() 메서드를 통해 `id` 값을 기반으로한 새로운 Long Stream을 생성합니다. mapToLong() 연산이 종료되면 연산 결과인 Long Stream을 반환 받을 수 있습니다. 그리고 filter() 메서드를 통해서 유저가 요청한 `requestClass`가 Long Stream에 존재한다면 새로운 Long Stream을 반환합니다. findFirst()를 통해서 연산 결과를 Optional로 반환을 받고 isPresent()를 통해서 결과 값이 존재하는지 여부를 체크하게 됩니다. 만약 값이 존재한다면 `true`가 반환될건데, 현재 코드에서는 존재하지 않는 클래스를 요청했기 때문에 `false`를 기대값으로 설정하였습니다. 당연히 이 테스트 코드는 문제없이 통과됩니다.

## CASE. 글에 작성된 댓글과 답글의 총 개수

---
자주 사용하는 Stream API에 대한 기능을 간단한 시나리오와 함께 정리해 보았다(정리한 내용 외에도 쇼트서킷 기반 기능, 정렬 기능 등 많은 것들이 있지만 누구나 API Document를 보면 직관적으로 이해될 법한 기능들이라고 생각한다). Stream API를 실제 서비스에 적용하면서 가독성 측면에선 많은 이득을 볼 수 있었다(성능 면에서는 유의미한 차이를 확인할 순 없었다). 하지만 함께 일하는 팀원들이 함수형 프로그래밍에 대한 이해가 없다면 오히려 이전 코드보다 가독성 효율이 떨어질 수도 있다. 팀의 컨벤션을 지키는 것이 참 중요하다는 것을 느끼고 같이 스터디 해나가는 방법도 좋을 것 같다.

# Reference
[java-8-flatmap-example]([https://mkyong.com/java8/java-8-flatmap-example/](https://mkyong.com/java8/java-8-flatmap-example/))   
[java-stream-reduce]([https://www.baeldung.com/java-stream-reduce](https://www.baeldung.com/java-stream-reduce))   