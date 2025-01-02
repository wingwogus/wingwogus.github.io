---
title: "[Spring] 흘러가는대로 사는 인생 - 빈 생명 주기"
excerpt: "스프링 빈은 언제 생성되고 언제 소멸되는걸까?"

categories:
  - Spring
tags:
  - [spring, lifecycle, bean]

permalink: /spring/bean-lifecycle/

toc: true
toc_sticky: true

date: 2024-12-27
last_modified_at: 2024-12-27
---
**본 게시글은 김영한님의 인프런 [스프링 핵심 원리 - 기본편]을 바탕으로 한 공부 내용을 작성한 글입니다.**

## 빈 생명주기 콜백

- 만약 애플리케이션 시작 시점과 종료 시점에 작업을 진행해야 한다면 객체의 초기화와 종료 작업이 필요하다.
- 스프링에서는 어떻게 초기화 작업과 종료 작업을 진행하는지 살펴보자!

### 네트워크 연결과 해제

- 외부 네트워크에 연결하는 객체를 생성한다고 가정하자.
- connect()를 호출해서 연결을 맺어두어야 하고, 애플리케이션이 종료되면 disConnect()를 호출해서 연결을 끊어야 한다.

**NetworkClient 예제 코드**

```java
public class NetworkClient{

    private String url;

    public NetworkClient() {
        System.out.println("생성자 호출, url = " + url);
    }

    public void setUrl(String url) {
        this.url = url;
    }

    //서비스 시작 시 호출
    public void connect() {
        System.out.println("connect: " + url);
    }

    public void call(String message) {
        System.out.println("call: " + url + " message = " + message);
    }

    //서비스 종료 시 호출
    public void disconnect() {
        System.out.println("close " + url);
    }
}
```

**테스트 코드**

```java
class BeanLifeCycleTest {

    @Test
    public void lifeCycleTest() {
        ConfigurableApplicationContext ac = new AnnotationConfigApplicationContext(LifeCycleConfig.class);
        NetworkClient client = ac.getBean(NetworkClient.class);
        ac.close();
    }

    @Configuration
    static class LifeCycleConfig {

        @Bean
        public NetworkClient networkClient() {
            NetworkClient networkClient = new NetworkClient();
            networkClient.setUrl("http://hello-spring.dev");
            return networkClient;
        }
    }
}
```

**실행 결과**

```java
생성자 호출, url = null
connect: null
call: null message = 초기화 연결 메시지
```

- 어라? 어째서 null이 출력되는 것일까
- 어찌보면 당연하다 객체 생성 단계에서는 url이 없고, 객체를 생성한 다음에 외부에서 수정자 주입을 통해 setUrl()이 호출되어야 url이 존재하게 된다.

### 스프링 빈의 라이프 사이클

스프링 컨테이너 생성 → 스프링 빈 생성 → 의존관계 주입 → 초기화 콜백 → 사용 
→ 소멸 전 콜백 → 스프링 종료

- 초기화 콜백: 빈이 생성되고, 빈의 의존관계 주입이 완료된 후 호출
- 소멸전 콜백: 빈이 소멸되기 직전에 호출

---

## 물 흐르듯 흘러가는 생명주기

스프링은 크게 3가지 방법으로 빈 생명주기 콜백을 지원한다.

그 방법에 대해 한 번 알아보자

### 인터페이스(InitializingBean, DisposableBean)

코드를 바로 확인하자

```java
public class NetworkClient implements InitializingBean, DisposableBean{

    private String url;

    public NetworkClient() {
        System.out.println("생성자 호출, url = " + url);
    }

    public void setUrl(String url) {
        this.url = url;
    }

    //서비스 시작 시 호출
    public void connect() {
        System.out.println("connect: " + url);
    }

    public void call(String message) {
        System.out.println("call: " + url + " message = " + message);
    }

    //서비스 종료 시 호출
    public void disconnect() {
        System.out.println("close " + url);
    }
    
    @Override
    public void afterPropertiesSet() throws Exception {
	    connect();
	    call("초기화 연결 메시지");
    }
    
    @Override
    public void destroy() throws Exception {
	    disConnect();
	  }
}
```

- InitializingBean은 afterPropertiesSet() 메서드로 초기화를 지원한다.
- DisposableBean은 destroy() 메서드로 소멸을 지원한다.
- 단점
    - 스프링 전용 인터페이스이므로 해당 코드가 스프링 전용 인터페이스에 의존하게 된다.
    - 초기화, 소멸 메서드의 이름을 변경할 수 없다.
    - 코드를 고칠 수 없는 외부 라이브러리에는 사용하지 못한다.

### 설정 정보에 초기화 메서드, 종료 메서드 지정

@Bean(initMethod = “init”, destroyMethod = “close”)로 초기화, 소멸 메서드를 지정할 수 있다.

**설정 정보를 사용하도록 코드 변경**

```java
public class NetworkClient implements{

    private String url;

    public NetworkClient() {
        System.out.println("생성자 호출, url = " + url);
    }

    public void setUrl(String url) {
        this.url = url;
    }

    //서비스 시작 시 호출
    public void connect() {
        System.out.println("connect: " + url);
    }

    public void call(String message) {
        System.out.println("call: " + url + " message = " + message);
    }

    //서비스 종료 시 호출
    public void disconnect() {
        System.out.println("close " + url);
    }
    
    public void init(){
	    System.out.println("NetworkClient.init");
	    connect();
	    call("초기화 연결 메시지");
    }

    public void close(){
	    System.out.println("NetworkClient.close");
	    disConnect();
	  }
}
```

설정 정보에 초기화 소멸 메소드 지정

```java
@Configuration
    static class LifeCycleConfig {

        @Bean(initMethod = "init", destroyMethod = "close")
        public NetworkClient networkClient() {
            NetworkClient networkClient = new NetworkClient();
            networkClient.setUrl("http://hello-spring.dev");
            return networkClient;
        }
    }
```

- 특징
    - 메서드 이름을 자유롭게 줄 수 있다.
    - 스프링 빈이 스프링 코드에 의존하지 않는다.
    - 코드를 고칠 수 없는 외부 라이브러리에도 초기화, 종료 메서드를 적용할 수 있다.
- 종료 메서드 추론
    - @Bean의 destroyMethod 속성에는 아주 특별한 기능이 있는데 close, shutdown이라는 이름의 메서드를 자동으로 호출해준다.
    - destroyMethod의 기본값이 (inferred: 추론)으로 등록되어 있기 때문이다.
    - 따라서 직접 스프링 빈으로 등록하면 종료 메서드는 따로 적지 않아도 된다.
    - 추론 기능을 사용하지 않으려면 destroyMethod = “” 처럼 빈 공백을 지정하면 된다

### @PostConstruct, @PreDestroy 애노테이션 지원

바로 코드부터 보자

```java
public class NetworkClient{

    private String url;

    public NetworkClient() {
        System.out.println("생성자 호출, url = " + url);
    }

    public void setUrl(String url) {
        this.url = url;
    }

    //서비스 시작 시 호출
    public void connect() {
        System.out.println("connect: " + url);
    }

    public void call(String message) {
        System.out.println("call: " + url + " message = " + message);
    }

    //서비스 종료 시 호출
    public void disconnect() {
        System.out.println("close " + url);
    }

    @PostConstruct
    public void init() {
        System.out.println("NetworkClient.init");
        connect();
        call("초기화 연결 메시지");
    }

    @PreDestroy
    public void close(){
        System.out.println("NetworkClient.close");
        disconnect();
    }
}
```

---

- @PostConsturct, @PreDestroy
    - 지금까지의 방법 중 가장 편리하고 가장 권장하는 방법이다.
    - 스프링에 종속적인 기능이 아니라 JSR-250이라는 자바 표준이기 때문에 스프링이 아닌 다른 컨테이너에서도 동작한다.
    - 컴포넌트 스캔과 아주 잘 어울린다.
    - 외부 라이브러리에는 적용하지 못한다는 유일한 단점이 있다.
