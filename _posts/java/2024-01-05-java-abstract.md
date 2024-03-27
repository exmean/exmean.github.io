---
layout: post
title: 추상 클래스를 이용해서 모델 설계하기
category: Java
permalink: /java/2
---

최근 회사에서 운영중인 서비스 하나가 리뉴얼을 시작하면서 새롭게 기초 설계를 할 수 있는 기회가 있었습니다. 저는 일해오면서 줄곧 OOP에 대한 아쉬움이 있었기 때문에 제가 맡은 부분에 한해서라도 최대한 OOP의 원칙을 지킨 코드들을 작성하고 싶었습니다. 리뉴얼 전 기존 서비스는 **유사한 논리적인 구조를 가진 모델들이 많았고** 그로 인해 중복코드 또한 많이 존재했습니다. 그러다보니 추가 요구사항이 오면 번거로운 작업들이 굉장히 많았는데요, 이를 개선하기 위해 추상 클래스를 사용해서 공통으로 가지는 필드들은 추상 클래스에서 관리하고 추가적인 요구사항에 유연하게 대처할 수 있도록 설계해 보고자 합니다!

# 추상클래스를 이용한 설계
### 기존 구조
기존 모델 계층 클래스들의 구조는 아래와 같습니다.
``` java
public class A { // A 개발자가 구현
    ...
    private Date createAt;
    private Date modifyAt;
    private Date deleteAt;
}

public class B { // B 개발자가 구현
    ...
    private Date createAt;
    private Date modifyAt;
    private Date deleteAt;
}

...

public class J { // C 개발자가 구현
    ...
    private Date createAt;
    private Date modifyAt;
    private Date deleteAt;
}
```
사실 이 외에도 공통되는 필드들이 더 존재하지만 상세히 적을 순 없어서 감안하고 봐주시면 감사하겠습니다. 그렇다면 위와 같은 구조가 안좋은 이유가 무엇일까요? 저의 경우엔, SI 프로젝트 성격 상 클라이언트의 요구에 기획이 매일 바뀌기도 하고, 심지어는 정해놓은 정책마저도 쉽게 바뀌는 경우가 허다 했습니다. 그런데 **코드가 분산되어 있으면 이런 변동사항들이나 유지보수에 엄청난 리소스가 소요됩니다.** 간혹 같은 속성의 필드를 각 모델마다 필드명을 다르게 선언하는 경우도 종종 있었습니다... 이런 케이스들을 만나면서 설계를 해놓지 않으면 코드가 진짜 산으로 갈 수도 있겠다는 걸 체감했습니다. 추상 클래스를 이용하는 것은 간단합니다.

### 개선
공통된 필드들을 분리하고 추상 클래스로 선언합니다.
``` java
public abstract class CommonField {

    protected Date createAt;
    protected Date modifyAt;
    protected Date deleteAt;
}

public class A extends CommonField {

    private String title;
    private String content;
}

... 

public class J extends CommonField {

    private String id;
    private String description;
}
```
자식클래스(상속받는 클래스)에서만 사용할 것이기 떄문에 접근제어자를 `protected`로 선언했습니다. 필요에 따라서 Getter와 Setter를 선언하면 됩니다. 이제 공통 필드들은 `CommonField` 클래스에서 관리하게 됩니다. 만약 추가 요구사항으로 각 모델마다 스크랩 수가 추가되었다고 가정해 보겠습니다. 기존의 구조대로라면 해당하는 모든 클래스에 필드를 추가해줘야 하지만, 추상 클래스를 이용하면 `CommonField` 클래스에만 추가하면 됩니다.

``` java
public abstract class CommonField {

    protected Date createAt;
    protected Date modifyAt;
    protected Date deleteAt;
    protected int scrapCount; // 스크랩수
}
```
### 주의사항
* 이런 설계는 당연히 프로젝트 초기에 시행되어야 합니다!
* 추상 클래스는 단독으로 인스턴스를 생성할 수 없습니다. 따라서 자식 클래스로 인스턴스를 생성하거나 업캐스팅하여 사용하여야 합니다.
* 부모 타입으로 선언된 경우 자식 클래스 자원에 접근할 수 없습니다.
* 자식 클래스에서 오버라이딩된 메서드가 있다면, **자식 클래스에서 오버라이딩 된 메서드가 호출됩니다.**
* 추상 클래스는 **단일 상속입니다.** 즉, **한 클래스에서 하나의 추상 클래스만 상속할 수 있습니다.**

### 조금 더 자세히
```java
@Getter
@ToString
@NoArgsConstructor
@AllArgsConstructor
@SuperBuilder
public abstract class LearningBoard {

    private Long id;
    private String title;
    private String content;
    private String writer;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
    private LocalDateTime deletedAt;

    public String insert(LearningBoard board) {
        return String.format("title: %s\ncontent: %s\nwriter: %s", board.getTitle(), board.getContent(), board.getWriter());
    }
}
```
`LearningBoard`는 공통 필드를 모아둔 추상 클래스입니다.   
`insert` 메서드는 학습 관련 정보를 저장할 때 저장된 정보를 리턴합니다.
```java
@Getter
@SuperBuilder
public class Study extends LearningBoard {

    private LocalDate learnedAt;

    public String insert(LearningBoard board) {
        return String.format("title: %s\ncontent: %s\nwriter: %s\ntype: Study", board.getTitle(), board.getContent(), board.getWriter());
    }
}
```
`Study`는 `LearningBoard`를 상속 받았고, 추가로 학습한 날짜를 필드로 선언했습니다. 그리고 `insert` 메서드를 오버라이딩 해서 저장 시에 타입을 리턴하도록 재정의 했습니다. 이제 테스트 코드를 작성해서 차이점을 한번 보겠습니다.
```java
@Test
void 추상클래스_타입선언에_따른_차이점을_비교한다() {

    Study study = Study.builder()
            .id(3L)
            .title("스터디에 의해 작성된 제목")
            .content("스터디에 의해 작성된 내용")
            .writer("스터디")
            .learnedAt(LocalDate.now())
            .build();

    LearningBoard parentOfStudy = Study.builder()
            .id(4L)
            .title("스터디에 의해 작성된 제목")
            .content("스터디에 의해 작성된 내용")
            .writer("스터디")
            .build();

    System.out.println(study.insert(study));
    System.out.println("========================");
    System.out.println(parentOfStudy.insert(parentOfStudy));
    System.out.println("========================");
    System.out.printf("Learnt time: %s\n", study.getLearnedAt());
    System.out.printf("Learnt time: %s\n", parentOfStudy.getLearnedAt()); // 컴파일 에러

    // 출력결과
    // title: 스터디에 의해 작성된 제목
    // content: 스터디에 의해 작성된 내용
    // writer: 스터디
    // type: Study
    // ========================
    // title: 스터디에 의해 작성된 제목
    // content: 스터디에 의해 작성된 내용
    // writer: 스터디
    // type: Study
    // ========================
    // Learnt time: 2024-01-05
}
```
여기서 유의 깊게 볼 점은 **자식 클래스의 타입으로 인스턴스를 생성하면 본인 및 부모 클래스의 자원까지도 사용할 수 있게 되지만,** 부모 클래스의 타입으로 선언됐을 경우 자식 클래스의 자원인 `learnedAt`에 접근하지 못하고 컴파일 에러가 발생한다는 것, 그리고 출력 결과를 보면 **부모 타입으로 선언한 인스턴스라도 `insert` 메서드를 호출하게 되면 자식 클래스에서 오버라이딩 된 `insert` 메서드를 호출한다는 점** 입니다. 이런 점을 이해하지 못하고 넘어간다면 예상치 못한 결과가 나올 수 있습니다.

---
어쩌면 이번 포스팅은 기록으로 남기기엔 부끄러운 주제일 수 있을 것 같습니다. 프로젝트 초기에 잘 잡아놓고 시작했으면 됐을 문제인데, 개발이 어느정도 된 상태에서 바로 잡자니 생각보다 공들이게 됐습니다. 그래도 나름 시퀀스 다이어그램을 그려가면서 상속 구조를 잡고 분석하는 과정을 겪으니 도메인에 대한 이해가 한층 더 깊어지는 계기가 되었습니다.