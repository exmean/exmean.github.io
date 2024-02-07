---
layout: post
title: Stream API를 이용해서 레거시 코드 리팩터링 하기
category: Java
permalink: /java/1
---

**Stream API**는 Java 8 버전에서 처음 등장했습니다. Java 21 버전까지 나온 현재는 이미 Stream API에 대한 정보들이 많이 있습니다. 그래서 Stream API를 사용하는 방법에 대해 작성하기 보다는 제가 회사에 존재하는 레거시 코드들을 Stream API를 이용해 어떻게 리팩터링 했는지, 그리고 어떤 결과를 얻게 되었는지 작성해보려고 합니다.   

**Stream API**는 파이프라인을 이용한 연산입니다. 쉽게 설명하자면 연산이 끝날 때마다 새로운 Stream을 반환하고 반환된 Stream을 이용해서 또 다른 연산을 이어나가는 방식입니다. 그리고 반드시 최종연산을 거쳐야지만 모든 파이프라인 연산이 종료되어 결과를 얻을 수 있습니다. 글로 풀어 작성해보자니 어렵게 느껴지지만 막상 사용해보면 직관적으로 알 수 있을겁니다!

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
`requestClass`는 현재 유저가 접속하려고 하는 클래스입니다. `idList` 배열은 유저가 가입한 클래스 목록의 `id` 값을 저장하기 위해서 선언되었습니다. for문을 통해 `idList` 배열에 `id` 값을 모두 저장합니다. 그리고 저장된 배열에 `contains()` 메서드를 이용해서 `requestClass`와 동일한 값이 있는지 확인합니다. 현재는 기존 코드를 간단하게 재현하기 위해서 `boolean`으로 체크를 하고 있지만, 실제 서비스에 적용한다면 계획된 예외를 발생시켜 원하는 방향으로 후처리를 진행할 수 있겠습니다! 리팩터링 하기 전 코드도 적절한 주석을 달아둔다면 이해하기 쉬운 코드라고 생각합니다. 하지만 **Stream API**를 사용하면 더욱 더 간결하고 가독성이 좋게 로직을 구성할 수 있습니다.

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
`requestClass`는 이전과 동일하게 유저가 접속을 요청한 클래스입니다. 초기값으로 설정되어 있는 `joinClassList` 배열 Stream을 생성하고 `mapToLong()` 메서드를 통해 `id` 값을 기반으로한 새로운 Long Stream을 생성합니다. `mapToLong()` 연산이 종료되면 연산 결과인 Long Stream을 반환 받을 수 있습니다. 그리고 `filter()` 메서드를 통해서 유저가 요청한 `requestClass`가 Long Stream에 존재한다면 새로운 Long Stream을 반환합니다. `findFirst()`를 통해서 연산 결과를 Optional로 반환을 받고 `isPresent()`를 통해서 결과 값이 존재하는지 여부를 체크하게 됩니다. 만약 값이 존재한다면 `true`가 반환될건데, 현재 코드에서는 존재하지 않는 클래스를 요청했기 때문에 `false`를 기대값으로 설정하였습니다. 당연히 이 테스트 코드는 문제없이 통과됩니다.

## CASE. 글에 작성된 댓글과 답글
이번엔 글에 작성된 댓글과 답글을 조회해서 자료구조에 담아 응답결과를 리턴하고자 합니다. 댓글과 답글은 하나의 테이블에 저장되어 있고, 답글은 `PAR_COMMENT_ID`에 부모가 되는 댓글의 고유 값을 함께 저장하고 있습니다. 우선 댓글과 답글 구분없이 모두 조회한 다음 댓글과 답글을 구분하는 로직을 거쳐서 `Comment` 클래스의 `replies` 배열에 답글 데이터를 저장할 겁니다.

### 리팩터링 전
``` java
    @Test
    void 글에_등록된_댓글과_답글을_조회한다_리팩토링전() {

        long start = System.currentTimeMillis();

        List<Comment> list = commentRepository.findAll();
        List<Comment> comments = new ArrayList<>();
        List<Comment> replies = new ArrayList<>();

        for (Comment comment : list) {
            if (Objects.nonNull(comment.getParCommentId()) && comment.getParCommentId() != 0L) {
                replies.add(comment);
            }

            if (Objects.isNull(comment.getParCommentId()) || comment.getParCommentId() == 0L) {
                comments.add(comment);
            }
        }

        for (Comment comment : comments) {
            for (Comment reply : replies) {
                if (comment.getCommentId().equals(reply.getParCommentId())) {
                    comment.getReplies().add(reply);
                }
            }
        }

        long end = System.currentTimeMillis();
        System.out.println("Time : " + (end - start) + "ms");
    }
```
`list`는 댓글과 답글이 구분되지 않은 데이터 리스트입니다. `comments`는 `list`의 데이터 중 댓글에 해당하는 데이터를 저장하는 배열이고, `replies`는 `list`의 데이터 중 답글에 해당하는 데이터를 저장하는 배열입니다. 우선 `parCommentId`의 값이 있는 경우에는 답글로, 값이 없는 경우에는 댓글로 분류합니다. 그리고 한번 더 `comments` 배열에서 `commentId`와 `parCommentId`의 값을 비교해서 만약 같은 값을 가지고 있다면 해당 `comment`의 답글 배열에 저장합니다.   

### 리팩터링 후
``` java
    @Test
    void 글에_등록된_댓글과_답글을_조회한다_리팩토링후() {

        long start = System.currentTimeMillis();

        List<Comment> list = commentRepository.findAll();
        List<Comment> comments = list.stream()
                .filter(comment -> Objects.isNull(comment.getParCommentId()) || comment.getParCommentId() == 0L)
                .peek(comment -> {
                    comment.setReplies(list.stream()
                            .filter(reply -> comment.getCommentId().equals(reply.getParCommentId()))
                            .toList());
                }).toList();

        long end = System.currentTimeMillis();
        System.out.println("Time : " + (end - start) + "ms");
    }
```
`list`의 스트림을 생성하고 **filter()**를 통해서 `parCommentId`가 `null`이거나 `0`인 값을 `true`로 설정하여 댓글 데이터로 분류합니다. 그리고 **peek()** 연산에서 각 댓글에 달린 답글 데이터를 추가합니다. 확실히 리팩터링 전보다 코드가 훨씬 간결해졌습니다. 그렇다면 성능앤 이점이 있었을까요? 메서드 실행 시간을 비교해보면 각 약 200ms로 큰 차이는 없는 것으로 확인 됩니다.

# 리팩터링 후 결과
리팩터링 후에 **코드라인 수를 대폭 감축**시킬 수 있었고, 주석이 달려있지 않으면 의도를 알 수 없었던 코드에서 **어떤 기능을 하는 로직인지 명확한 코드로 개선**되었습니다(믈론 적절한 주석은 계속 지향해야 합니다). 그리고 무엇보다 제가 스트림을 사용한 후 팀원들이 함수형 프로그래밍에 관심을 가지기 시작했고 사용하기 시작했습니다. 기존엔 새로운 시도를 하지 않는 팀내 분위기가 있어서 이런 관심이 굉장히 반가웠습니다.

# 내가 느낀 단점
물론 스트림을 사용하면서 장점만 있었던 것은 아닙니다. 생각보다 고려해야 할 사항들이 있었고(병렬 스트림 사용시 발생할 수 있는 상황들), **함수형 프로그래밍에 익숙하지 않은 사람들과 협업하는 환경이라면 오히려 가독성 측면에서 이점을 가져가기가 어렵습니다.** 또한 스트림의 단점으로 자주 언급되는 **디버깅이 어렵다는 점**을 사용하면서 체감할 수 있었습니다.

---
오늘은 제가 실제로 스트림을 사용해서 리팩터링을 한 몇가지 사례에 대해서 기록해 보았습니다. 복잡한 코드없이 배열에 대한 자유로운 연산이 가능하고 메서드 체이닝을 통한 가독성 증가가 스트림이 꾸준히 사랑받는 이유가 아닐까 싶습니다. 개인적으로 아직 병렬 스트림에 대한 이해가 부족해서 활용을 못하고 있지만 적용해볼 수 있는 사례가 있다면 추후에 병렬 스트림에 대한 포스팅을 작성해보도록 하겠습니다!