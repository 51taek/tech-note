
### Spring Framework

- 전통적인 방식
- 수동으로 설정을 적용해야함
- 필요한 라이브러리의 버전을 직접 관리
- 별도의 WAS를 설치하고, 결과물인 WAR 파일을 서버에 배포해야함

### Spring Boot

- Auto Configuration: 프로젝트에 추가된 라이브러리를 기반으로 알아서 공통적인 설정 구성
- 라이브러리 묶음을 제공하여 버전관리와 종속성 간소화
- 내장 WAS: Tomcat이 내부에 내장되어 있어 별도의 설치 필요 없이 내부의 jar만으로 실행 가능

## Spring Boot가 설정을 자동화 하는 방법

조건부 빈 등록(Conditional Configuration)

- **동작 원리:** `@SpringBootApplication` 내부의 `@EnableAutoConfiguration` 어노테이션이 실행되면, `org.springframework.boot.autoconfigure.AutoConfiguration.imports`파일을 읽음
    - 이 파일에 적힌 클래스들만 자동 설정 후보로 보겠다"라고 명시함으로써, 불필요한 스캔을 방지하고 설정의 확장성을 확보
    - 수많은 외부 라이브러리(jar) 전체를 스캔하는 것은 구동 시간을 엄청나게 잡아먹음 → @ComponentScan 불가능
- 핵심 기술: `@ConditionalOnClass`, `@ConditionalOnMissingBean` 를 통해 필요한 빈만 선택적으로 등록합니다.
    - `@ConditionalOnClass`  ← 클래스 패스에 특정 라이브러리가 있는지, 사용자가 이미 수동으로 등록한 빈이 없는지 체크
- **면접 포인트:** "자동 설정이 편리하지만, 의존성 충돌이나 원치 않는 빈 등록으로 인한 메모리 낭비가 발생할 수 있는데 이를 어떻게 제어하느냐"는 질문에 대비해야함 (`exclude` 속성 사용 등)

<aside>
💡

이미 수동으로 등록한 빈 여부를 어떻게 확인하는 걸까?

Spring 컨테이너(ApplicationContext)가 클래스로더를 이용해 클래스의 존재 여부를 '확인'한 뒤 내부적으로 판단한다.

`@ConditionalOnMissingBean`은 JVM 레벨이 아니라 Spring의 Bean Definition(빈 정의) 레벨에서 동작한다.

- **동작 방식:** Spring 컨테이너는 모든 설정을 읽어들일 때, 사용자가 `@Bean`이나 `@Component`로 직접 등록한 빈들을 먼저 스캔하여 `BeanDefinitionRegistry`에 저장합니다.
- 판단 기준: 자동 설정 단계에서 `BeanDefinitionRegistry`를 조회했을 때, 동일한 타입이나 이름을 가진 빈 정의가 이미 존재한다면 자동 설정을 포기(Skip)합니다.
- 우선순위: 사용자의 수동 설정이 자동 설정보다 우선순위가 높게 설정
</aside>

#### 정리

- 외부 라이브러리에서 **자동 설정 후보 명단**을 싹 다 긁어모은다.
- **개발자가 직접 만든 빈**들을 먼저 등록한다.
- 개발자 설정과 겹치지 않고 라이브러리도 갖춰진 **나머지 빈들만 골라서** 컨테이너가 대신 등록해 준다.

클로드가 만들어준 다이어그램

<img width="500" height="700" alt="image" src="https://github.com/user-attachments/assets/35d039b3-dbfc-49f8-afb7-328d299eedf1" />


### Spring Boot의 실행 과정과 내장 WAS

Spring의 핵심인 `refresh()` 과정 중에 `onRefresh()`라는 메서드가 호출된다. 바로 여기서 내장 웹 서버(Tomcat 등)를 생성하는 로직이 실행된다.

<img width="500" height="700" alt="image" src="https://github.com/user-attachments/assets/81d4fa1c-5cb0-4876-a09a-75aa53e8062a" />



> 
> 
> 
> `onRefresh()`에서 Tomcat은 **생성만** 됩니다. 실제 포트를 열고 요청을 받기 시작하는 `start()`는 `finishRefresh()`에서 호출됩니다. 이유는 `finishBeanFactoryInitialization()`에서 모든 싱글톤 Bean이 완전히 준비된 다음에야 트래픽을 받아야 하기 때문입니다. 만약 Tomcat이 `onRefresh()`에서 바로 start()됐다면, Bean이 아직 초기화 안 된 상태에서 HTTP 요청이 들어와서 NPE가 발생할 수 있습니다.
> 
> **`5번`과 `10번`의 차이도 단골 질문입니다** — `invokeBeanFactoryPostProcessors()`는 "Bean을 어떻게 만들지" 정의서(BeanDefinition)를 등록하는 단계고, `finishBeanFactoryInitialization()`은 그 정의서를 보고 실제로 `new`하는 단계입니다. 설계도 vs 실제 공장 가동이라고 보면 됩니다.
> 

과거에는 Tomcat을 별도로 설치하고 그 안에 우리 코드를 넣었지만, Spring Boot는 반대로 우리 코드(Spring)가 Tomcat 객체를 생성한다.

`ServletWebServerFactory`: Spring Boot는 Tomcat, Jetty, Undertow 같은 다양한 서버를 추상화한 `Factory` 인터페이스를 가지고 있다.

자동 설정 메커니즘에 의해, 클래스패스에 `tomcat-embed-core.jar`가 있으면 `TomcatServletWebServerFactory` 빈이 자동으로 등록된다. (톰캣이 아니고 Jetty.jar가 있으면 jar를 사용할 듯)

이 Factory 빈이 `getWebServer()`를 호출하면, 우리가 아는 Tomcat 객체가 내부적으로 `new` 생성되고 포트(8080)를 점유하며 실행됩니다.

<aside>
💡

**표준 JAR의 한계:** 원래 자바의 표준 JAR는 내부에 다른 JAR(Nested JAR)를 포함하고 이를 바로 읽는 기능이 없다.

**Spring Boot의 해결책 (Fat JAR):** 

1. Spring Boot는 프로젝트의 클래스들과 모든 라이브러리 JAR를 한곳에 때려 넣는다.
2. 그리고 `JarLauncher`라는 특별한 클래스를 `Main-Class`로 지정
3. 우리가 `java -jar`를 실행하면 `JarLauncher`가 먼저 실행되어, 내부에 압축된 JAR들을 읽을 수 있도록 특수한 클래스로더를 셋팅한 뒤, 비로소 우리의 `main()` 메서드를 호출

</aside>

현재는 거의 대부분 내장 톰캣을 사용하고 있는 것으로 봐도 무방하다. 그럼 외장 톰캣을 써야하는 경우는 뭐가 잇을까?

#### 외장 톰캣을 쓰는 경우

아주 오래된 인프라 구조에서 하나의 톰캣 서버에 여러개의 서비스를 올리고 싶을 때 사용할 수 있다.

## **Spring Boot Starter**와 **BOM**

> 
> 
> 
> Starter는 특정 목적(예: 웹 개발, 데이터베이스 연동)을 위해 필요한 **관련 라이브러리들을 하나로 묶은 패키지**입니다.
> 
> - **기존 방식:** 웹 개발을 하려면 `spring-web`, `spring-webmvc`, `jackson-databind`, `tomcat-embed-core` 등을 일일이 찾아 `build.gradle`에 적어야 했습니다.
> - **Starter 방식:** `implementation 'org.springframework.boot:spring-boot-starter-web'` 한 줄이면 끝납니다.
> - **핵심 가치:** "이 기능을 쓰려면 보통 이 라이브러리들이 필요해"라는 **Best Practice**를 미리 구성해둔 것입니다.

#### BOM

A 라이브러리는 `Logback 1.2`를 원하는데, B 라이브러리는 `Logback 1.4`를 원하면 런타임에 에러가 발생한다. 

Spring Boot 팀은 매 버전마다 수백 개의 오픈소스 라이브러리를 직접 테스트하여 "이 버전들끼리는 충돌 없이 잘 돌아간다"는 조합표(BOM)를 만들어 놓는다.

우리가 `build.gradle`에서 라이브러리 뒤에 **버전 번호를 적지 않아도 되는 이유**가 바로 이것 때문. 상위의 `spring-boot-dependencies`(BOM)가 최적의 버전을 강제로 지정
