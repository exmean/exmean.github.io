---
layout: post
title: Spring Security 시작하기
category: Spring Security
permalink: /spring-security/1
---

오늘은 간단한 예제 코드와 함께 `Spring Security`에 대해 제가 학습한 내용을 다뤄보고자 합니다. 학습하게 된 계기를 이야기해 보자면, 제가 다니고 있는 회사에서는 **Spring Security**를 사용하지 않고 회원 시스템을 구축했고 이에 따라 겪었던 많은 불편함이 있었습니다. 로그인 로직에 의도를 알 수 없는 코드들이 엉켜 있다든지, 문서화나 도식화가 되어 있지 않아서 로직을 이해하는데에 많은 시간이 걸린다든지, 그리고 유저의 권한이나 세션을 검증하는 중복 로직들이 각 서비스 레이어에 존재했습니다. 저는 **이런 문제들을 조금이라도 개선하기 위해서 인터셉터를 이용한 공통 모듈을 개발**했었는데요, 그럼에도 **Spring Security**를 이용하는 것보다는 **개발하는데 소요되는 리소스가 크고 XML로 설정하다 보니 가독성 측면에서도 이점을 가져가기에 어려웠습니다.** 이런저런 다른 이유도 많았지만 다음에 이직하게 될 회사에서 사용할 수도 있고, 개인 프로젝트에서라도 적용해 볼 수 있도록 학습을 시작하게 되었습니다!

우선 **Spring Security**를 이해하기 위해서 숙지해야 할 개념이 있습니다!

---

# 인증과 인가
개발자들에겐 **보안**이 중요한 과제 중 하나입니다. 보안하면 어떤 게 제일 먼저 떠오르시나요? 저는 각종 보안 프로그램으로 막혀 있는 정부 서비스가 생각나네요. 그렇다면 정부24 서비스를 이용한다고 가정해 봅시다. 정부24의 서비스를 이용하기 위해선 금융인증서나 공동인증서 등을 통해 본인임을 확인하는 절차를 거쳐야 합니다. 이것을 **인증**이라고 하고, 인증에 성공해야지만 정부24가 제공하는 서비스들을 이용할 수 있는 권한을 얻게 되는데 이것을 **인가**라고 합니다.

* **인증(Authentication)**: 접근할 수 있는 사람인지 확인하는 절차
* **인가(Authorization)**: 인증된 사람에게 권한을 부여하는 절차

그렇다면 **Spring Security**는 어떤 구조로 되어 있고 어떻게 인증과 인가 절차를 수행하는지 알아보도록 하겠습니다.

---

# Spring Security Architecture
Spring Security 구조에 대해서 먼저 살펴보도록 하겠습니다!

* DelegatingFilterProxy
* FilterChainProxy
* SecurityFilterChain

## DelegatingFilterProxy
**DelegatingFilterProxy**는 서블릿 컨테이너와 Spring ApplicationContext를 연결해 주는 역할을 수행합니다. 그렇다면 왜 서블릿 컨테이너와 ApplicationContext를 연결해야 하는 걸까요? **서블릿 컨테이너는 실행해야 하는 필터들을 자체적으로 등록하고 실행**하지만, **Spring ApplicationContext는 실행해야 할 필터들을 Bean으로 등록해 관리하기 때문**입니다!

![](/assets/images/screenshot/spring-security/1-1.png){: width="300" height="1200"}   
정리하자면 **DelegatingFilterProxy**는 **서블릿 컨테이너에 등록된 필터들과 Spring의 Bean으로 등록된 필터들을 통합적으로 실행시킬 수 있도록 도와주는 역할**을 한다고 볼 수 있겠습니다!

## FilterChainProxy
**FilterChainProxy**는 **Spring Security에서 제공하는 필터** 중 하나입니다.   
**SecurityFilterChain에 등록된 필터 인스턴스들을 실행하는 역할**을 수행합니다!

## SecurityFilterChain
**SecurityFilterChain**는 실행될 Security 필터들의 그룹입니다. 보통 **FilterChain**은 등록된 필터 인스턴스들을 연달아 실행하는 역할을 수행합니다. 위에서 살펴본 **FilterChainProxy**가 요청에 따라 어떤 필터 인스턴스를 실행할 건지 결정하기 위해 이 **SecurityFilterChain**을 사용합니다. 

![](/assets/images/screenshot/spring-security/1-2.png){: width="500" height="1200"}   
RequestMather을 이용하면 위 사진처럼 **요청에 따라 실행할 Security 필터를 별도로 설정**할 수 있는데 이건 추후 포스팅을 작성하면서 다루도록 하겠습니다.

그러면 이제 코드를 통해 알아보도록 하겠습니다!

---

# 코드 살펴보기
개발 환경은 아래와 같습니다.

* Java 17
* SpringBoot 3.2.4
* Spring Security 6.2.3

## 의존성 추가
먼저 의존성을 추가합니다.

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-security'
    implementation 'org.springframework.boot:spring-boot-starter-web'
    developmentOnly 'org.springframework.boot:spring-boot-devtools'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'org.springframework.security:spring-security-test'
}
```
최근 Spring Security 릴리즈 버전에서는 **Lambda DSL을 지원**하므로 구현 방식이 기존과 약간 달라졌습니다.   
이 부분도 포스팅을 작성하면서 다뤄보도록 하겠습니다.

## 간단한 컨트롤러 작성
처음엔 Security에 대한 별도의 설정을 하지 않고 **DelegatingFilterProxy**가 어떻게 작동하는지 살펴보도록 하겠습니다.   
우선 인덱스 페이지로 이동하는 간단한 컨트롤러 하나를 작성합니다.

```java
@Controller
public class MainController {
    
    @GetMapping
    public String indexPage() {
        return "/index";
    }
}
```

![](/assets/images/screenshot/spring-security/2-1.png){: width="900"}   
위 사진은 Spring 애플리케이션 실행했을 때 ApplicationContext를 초기화하면서 **DelegatingFilterProxy**가 호출되는 과정입니다. 등록된 Security 필터 인스턴스들을 확인하기 위해 `targetBeanName` 인자 값으로 **`springSecurityFilterChain`**을 넘겨주고 있는 것을 확인할 수 있습니다. 만약 현재처럼 Security에 대한 아무런 설정을 하지 않은 상태라면 **DefaultSecurityFilterChain**이 사용될 겁니다. 정말 그런지 한번 확인해 보도록 할까요? 인덱스 페이지에 접근 해보겠습니다.

## DefaultSecurityFilterChain
![](/assets/images/screenshot/spring-security/2-2.png){: width="900"}   
인덱스 페이지에 접근하게 되면 어떠한 구현을 하지 않았는데도 자동으로 `/login`으로 리다이렉트 되는 것을 확인할 수 있습니다. 바로 **DefaultSecurityFilterChain**의 기본 설정 때문입니다. 

![](/assets/images/screenshot/spring-security/3-2.png){: width="900"}
위 사진에서 보는 것처럼 **`WebSecurityConfiguration` 클래스**에서 **커스텀 된 SecurityFilterChain이 없다면 기본 설정을 진행**하게 됩니다. 바로 저 설정 때문에 우리가 구현하진 않았지만, Spring Security에서 기본적으로 제공하는 페이지를 사용할 수 있게 되는 것입니다. 

* authorizeHttpRequests((authorize) -> authorize.anyRequest().authenticated()): 모든 요청에 대한 권한 검증
* formLogin(Customizer.withDefaults()): 기본 로그인 폼 설정
* httpBasic(Customizer.withDefaults()): 기본 HTTP 설정

![](/assets/images/screenshot/spring-security/3-3.png){: width="900"}   
이제 기본 로그인 폼에서 **Username**은 `user`, **Password**에는 콘솔 창에 출력되는 `Using generated security password`의 값을 입력하면 됩니다.

![](/assets/images/screenshot/spring-security/3-4.png){: width="900"}   
다음 포스팅에서는 **SecurityFilterChain**을 직접 구현하여 설정해 보도록 하겠습니다.

---

> Reference   
> [Spring Security Architecture](https://docs.spring.io/spring-security/reference/servlet/architecture.html)