# 빈은 어떻게 만들어질까?

[목차](README.md) · [이전: 응답과 예외는 어떤 길로 나갈까?](response-and-exception.md) · [다음: 리플렉션은 어떻게 어노테이션을 읽고 메서드를 부를까?](reflection.md)

## 다루는 범위

Part 1에서 요청을 처리한 부품들(`DispatcherServlet`, 컨트롤러, `OrderService`, HikariCP)은 요청이 오기 전에 이미 만들어져 있었다. 이 노트는 그 부품들, 즉 **빈**이 서버가 켜질 때 어떻게 만들어지는지 정리한다.

- 빈이란 무엇이고, 언제 만들어지나 (콘솔 로그로 보기)
- 네 칸: 찾기(컴포넌트 스캔) → 적기(빈 정의) → 만들기(생성자 주입) → 마무리(후처리)
- 리플렉션을 쓰는 곳과 안 쓰는 곳, 클래스 로딩(Metaspace)과 객체 생성(힙)
- 순환 참조가 잡히는 곳
- 주입 방식에 따른 타이밍 차이
- 후처리기: 넣어 주기, 불러 주기, 감싸 주기(프록시)
- Nest의 DI 컨테이너와 비교

리플렉션 자체는 7번, 프록시를 만드는 방식(JDK 동적 프록시와 CGLIB)과 자기 호출은 8번에서 자세히 다룬다([공부 계획](README.md#공부-계획)). 코드와 로그는 설명용이다.

## 시작한 질문

1. 우리는 `new OrderService()`를 한 적이 없는데, 컨트롤러는 어떻게 그걸 갖고 있었을까?
2. 그럼 Spring은 그걸 언제 넣어 줄까?
3. Spring은 우리 코드를 어떻게 손에 넣고, 어떻게 만들 수 있었을까?

## 내가 이해한 내용

### 1. 빈: 우리가 `new` 하는 대신 컨테이너가 만들어 들고 있는 객체

컨트롤러는 생성자로 `OrderService`를 받는데, 프로젝트 어디에도 `new OrderService()`가 없다. `main()`에는 `SpringApplication.run(...)` 한 줄뿐이다. Spring이 없었다면 누군가 이렇게 직접 조립해야 했다.

```java
OrderRepository repository = new OrderRepository();
OrderService service = new OrderService(repository);       // 필요한 걸 먼저 만들어서
OrderController controller = new OrderController(service); // 넣어 준다
```

Spring은 이 조립을 대신 한다. 객체를 만들고, 필요한 걸 넣어 주고, 들고 있다가 필요한 곳에 준다.

| 이름 | 뜻 |
| --- | --- |
| Spring 컨테이너 (`ApplicationContext`) | 조립을 맡아서 객체들을 들고 있는 곳 |
| **빈** | 컨테이너가 만들어서 들고 있는 객체 |
| 의존성 주입 (DI) | 필요한 빈을 넣어 주는 것 |
| 제어의 역전 (IoC) | 객체를 만들고 잇는 주도권이 내 코드에서 Spring으로 넘어간 것 |

### 2. 언제: 서버가 켜질 때, 요청을 받기 전에 전부

Spring Boot 3.x + JPA 앱을 켜면 보통 이런 로그가 나온다(앞부분 생략).

```
Starting OrderApplication using Java 21 ...                          ← main() 시작, 컨테이너 생성
Tomcat initialized with port 8080 (http)                              ┐
Root WebApplicationContext: initialization completed in 1290 ms       │
HikariPool-1 - Starting...                                            │  이 구간에서 빈을
HikariPool-1 - Start completed.                                       │  찾고 · 적고 · 만든다
Initialized JPA EntityManagerFactory for persistence unit 'default'   ┘
Tomcat started on port 8080 (http) with context path '/'              ← 빈을 다 만든 다음에 문을 연다
Started OrderApplication in 2.8 seconds (process running for 3.1)
```

- 톰캣이 두 번 나온다. `initialized`는 준비만 한 것이고, 요청을 받기 시작하는 건 `started`다. 빈은 그 사이에 만들어진다. 그래서 첫 요청이 올 때는 빈이 이미 다 준비돼 있다.
- `HikariPool`이나 JPA 빈은 만들어질 때 INFO 로그를 남기지만, 우리가 만든 `OrderController`, `OrderService`는 조용히 만들어진다.
- "initialization completed"라는 이름과 달리 이때는 아직 빈을 만드는 중이다.

조용히 만들어지는 빈까지 보려면 그 로그를 남기는 클래스의 레벨만 DEBUG로 낮춘다. Spring은 빈 생성 로그를 DEBUG로 남기는데, Spring Boot 기본 레벨이 INFO라서 걸러지고 있었던 것이다(`TRACE < DEBUG < INFO < WARN < ERROR`).

```properties
logging.level.org.springframework.beans.factory.support.DefaultListableBeanFactory=debug
```

```
Creating shared instance of singleton bean 'orderController'
Creating shared instance of singleton bean 'orderService'
Autowiring by type from bean name 'orderController' via constructor to bean named 'orderService'
```

### 3. 큰 흐름: 네 칸

| 칸 | 하는 일 | 누가 | 리플렉션 |
| --- | --- | --- | --- |
| 1. 찾기 | `.class` 파일을 읽어 `@Component` 붙은 클래스를 찾는다 | 컴포넌트 스캐너 | ✕ |
| 2. 적기 | 찾은 클래스마다 설계도(빈 정의)를 표에 쌓는다 | 컴포넌트 스캐너 등 세 길 | ✕ |
| 3. 만들기 | 설계도대로 생성자를 불러 객체를 만든다. 생성자에 필요한 빈은 먼저 만든다 | 빈 팩토리 | ○ |
| 4. 마무리 (후처리) | 필드 주입, 초기화, 필요하면 프록시로 감싸기 → 보관함에 등록 | 후처리기 | ○ |

이 순서는 `SpringApplication.run()` 안에서 부르는 `AbstractApplicationContext.refresh()`에 그대로 보인다. 아래는 Spring Framework 6.2.x 소스의 실제 순서이고, ★가 이 노트와 관련된 단계다.

```java
prepareRefresh();
obtainFreshBeanFactory();                      // 빈 팩토리(시공사) 꺼내기
prepareBeanFactory(beanFactory);
postProcessBeanFactory(beanFactory);
invokeBeanFactoryPostProcessors(beanFactory);  // ★ 세 길로 설계도 모으기          → 1·2칸
registerBeanPostProcessors(beanFactory);       // ★ 후처리기부터 먼저 만들기
initMessageSource();
initApplicationEventMulticaster();
onRefresh();                                   // ★ 톰캣 준비 → "Tomcat initialized"
registerListeners();
finishBeanFactoryInitialization(beanFactory);  // ★ 남은 싱글톤 전부 시공          → 3·4칸
finishRefresh();                               // ★ 톰캣 문 열기 → "Tomcat started"
```

설계도를 먼저 다 모으고, 후처리기를 세워 두고, 나머지 빈을 전부 만든 다음, 톰캣 문을 연다.

### 4. 찾기: 컴포넌트 스캔

`@SpringBootApplication` 안에 `@ComponentScan`이 들어 있다. 따로 정하지 않으면 **이 클래스가 있는 패키지와 그 아래 전부**가 대상이다.

```
com.example.order
 ├─ OrderApplication        @SpringBootApplication  ← 여기서부터 아래로 훑는다
 ├─ controller/OrderController   @RestController  → 찾음
 ├─ service/OrderService         @Service         → 찾음
 └─ util/PriceCalculator         (어노테이션 없음)  → 안 찾음
com.example.common
 └─ SlackNotifier                @Component       → 못 찾음 (출발점 밖)
```

`@Service`, `@RestController`, `@Repository`, `@Configuration`도 걸리는 건 안에 `@Component`가 들어 있어서다.

**찾을 때는 리플렉션을 쓰지 않는다.** `com/example/order/**/*.class` 파일을 하나씩 **바이트로 읽어서**(ASM이라는 바이트코드 도구) `@Component`가 있는지만 본다. 리플렉션은 클래스가 메모리에 로딩돼 있어야 쓸 수 있는데, 후보를 전부 로딩하면 시작이 느려지고, 로딩한 클래스 정보가 자리를 계속 차지하고, 로딩하는 방식에 따라 `static` 코드까지 돌 수 있다.

여기서 "메모리에 올린다"는 객체를 만드는 것과 다르다.

| | 무엇이 | 어디에 | 언제 |
| --- | --- | --- | --- |
| 클래스 로딩 | 클래스 정보 (메서드 코드, 필드 목록, 상수) | **Metaspace** (힙 바깥, 네이티브 메모리) | 클래스가 처음 필요해질 때 |
| 객체 생성 | `new`로 만든 객체, 빈 | **힙** | 생성자를 부를 때 |

리플렉션에서 쓰는 `Class` 객체 자체는 힙에 작게 생기고, 클래스 정보 본체를 가리키는 손잡이 역할을 한다.

JVM은 클래스를 **처음 필요해질 때 하나씩**(lazy) 로딩한다. 그래도 Spring Boot 앱에서는 우리 클래스 대부분이 켤 때 올라간다.

| 종류 | 빈인가 | 클래스가 올라가는 때 | 객체가 생기는 때 |
| --- | --- | --- | --- |
| Controller, Service, Config | ○ | 서버 켤 때 | 서버 켤 때. 아무도 안 써도 만든다(`@Lazy` 제외) |
| Repository 인터페이스 | ○ (Spring Data가 따로 찾아 구현체를 빈으로) | 서버 켤 때 | 서버 켤 때 |
| Entity | ✕ (`@Entity`는 `@Component`가 아님) | 서버 켤 때. Hibernate가 매핑 정보를 만들려고 올린다 | 조회하거나 `new` 할 때 |
| DTO, enum, 예외, util | ✕ | 처음 쓰일 때 | `new` 할 때 |

Spring이 빈을 켤 때 다 만드는 건 문제를 시작할 때 잡으려는 것이다. 필요한 빈이 없으면 서버가 아예 안 뜬다. 덤으로 첫 요청도 빨라진다.

스캔이 읽는 `.class`는 `.java`를 `javac`로 컴파일한 결과다. `.jar`는 `.class` 여러 개와 리소스를 묶은 zip이다. IDE에서 실행하면 `build/classes/...`를, `java -jar`로 실행하면 jar 안의 `BOOT-INF/classes/...`를 읽는다.

### 5. 적기: 설계도(빈 정의)

찾은 클래스를 바로 만들지 않고, 먼저 **설계도(`BeanDefinition`)**를 적어 "빈 이름 → 설계도" 표에 등록한다. 이 칸이 끝날 때까지 빈 객체는 0개다.

```
빈 이름     orderService                             ← 클래스 이름 첫 글자를 소문자로
클래스      com.example.order.service.OrderService   ← 이름만 (아직 로딩 안 함)
스코프      singleton
지연 생성   false
만드는 법   이 클래스의 생성자로
```

양식(칸)은 `BeanDefinition`이 정해 두었고, 칸마다 **어노테이션이 있으면 그 값, 없으면 기본값**을 채운다. 어노테이션 값은 `.class` 바이트에서 읽어 둔 정보라 이 칸도 리플렉션을 쓰지 않는다.

| 칸 | 기본값 | 바꾸는 어노테이션 |
| --- | --- | --- |
| 빈 이름 | 클래스 이름 첫 글자 소문자 | `@Service("orders")` |
| 스코프 | `singleton` (하나만 만들어 계속 씀) | `@Scope("prototype")` |
| 지연 생성 | `false` (켤 때 만든다) | `@Lazy` |
| 우선 빈 | `false` | `@Primary` (같은 타입이 여럿일 때 이걸 먼저) |
| 만드는 법 | 생성자로 | `@Bean`이면 "이 메서드를 불러서" |

**설계도는 세 길로 들어온다.** 셋 다 `@SpringBootApplication`에서 시작한다.

| `@SpringBootApplication` 안의 | 길 | 만드는 법 |
| --- | --- | --- |
| `@ComponentScan` | ① 컴포넌트 스캔 | 클래스의 생성자로 |
| `@SpringBootConfiguration` (= `@Configuration`) | ② `@Bean` 메서드 | 메서드를 불러서 나온 걸 쓴다 |
| `@EnableAutoConfiguration` | ③ 자동 구성 (`DispatcherServlet`, `DataSource` 등) | Boot가 미리 써 둔 `@Bean` 메서드. 조건이 맞을 때만 |

세 길은 **설계도를 만드는 쪽**이다. 컴포넌트 스캐너의 이름도 `ClassPathBeanDefinitionScanner`, "클래스패스를 훑어서 BeanDefinition을 만드는 스캐너"다. 찾기와 적기는 실제로는 스캐너가 한 번에 한다. 설계도로 객체를 만드는 건 따로, 빈 팩토리가 한다.

**설계도에 적히지 않는 것**은 "무엇이 필요한가(의존성)"다. 그건 만들 때 생성자를 열어 봐야 안다. 예외로 `@DependsOn`처럼 직접 적은 건 설계도에 들어간다.

**왜 설계도부터 다 모을까.** 메모리 때문이 아니다. 싱글톤은 어차피 다 만든다.

1. 세 길에서 온 걸 한 양식으로 맞춰야 출처를 따지지 않고 똑같이 만들 수 있다.
2. 다 모아 봐야 정할 수 있는 게 있다. 자동 구성의 "`DataSource`가 **이미 있으면** 안 만든다"는 우리 설계도가 다 모여 있어야 판단할 수 있다. 그래서 우리 설계도(①②)를 먼저 모으고 자동 구성(③)을 나중에 본다.

### 6. 만들기: 빈 팩토리가 설계도대로 시공한다

순서 지휘는 `ApplicationContext`의 `refresh()`가, 실제 시공은 그 안의 빈 팩토리(`DefaultListableBeanFactory`)가 한다. 빈 팩토리는 설계도 표를 돌며 지연 생성이 아닌 싱글톤을 하나씩 만든다. 만들려는 빈이 다른 빈을 필요로 하면 **그것부터 먼저 만든다.**

```
orderController 만들자
 └ 생성자를 보니 OrderService가 필요 → 보관함에 없음 → 먼저 만들러 감
     orderService 만들자
      └ 생성자를 보니 OrderRepository가 필요 → 없음 → 먼저 만들러 감
          orderRepository: 필요한 게 없음 → 생성 → 보관함   ①
      ← orderService 생성 → 보관함                         ②
 ← orderController 생성 → 보관함                           ③
```

시작은 컨트롤러여도 완성은 안쪽부터(Repository → Service → Controller)다. 하나를 여러 빈이 같이 쓸 수 있어서 모양은 트리보다 그물(그래프)에 가깝다. 이미 보관함에 있으면 새로 만들지 않는다.

**여기서 처음 리플렉션을 쓴다.** 설계도에는 클래스 이름만 있었으므로, 이 칸에서 클래스를 로딩하고 생성자를 다룬다.

```java
// 설명용으로 단순화
Class<?> c = Class.forName("com.example.order.service.OrderService"); // 클래스 로딩 (Metaspace)
Constructor<?> ctor = c.getDeclaredConstructors()[0];                  // 생성자 찾기
Object orderService = ctor.newInstance(orderRepository);              // 파라미터 타입에 맞는 빈을 넣어 생성 (힙)
```

- 생성자가 하나면 `@Autowired` 없이 그 생성자를 쓴다(Spring 4.3부터).
- 넣을 빈은 **파라미터 타입**으로 찾는다. 안 맞으면 서버가 안 뜬다.

| 상황 | 시작할 때 나오는 메시지 |
| --- | --- |
| 그 타입 빈이 없음 | `Parameter 0 of constructor in ...OrderController required a bean of type '...OrderService' that could not be found.` |
| 그 타입 빈이 두 개 | `Parameter 0 of constructor in ...OrderController required a single bean, but 2 were found:` → `@Primary`나 `@Qualifier`로 고른다 |

만든 빈은 **보관함**(싱글톤 캐시)에 들어가고, 필요한 곳마다 같은 객체를 꺼내 준다. DEBUG 로그의 "Creating **shared** instance"가 이 뜻이다.

**순환 참조도 이 칸에서 잡힌다.** 빈 팩토리는 만들기 시작한 빈 이름을 "만드는 중" 목록에 적어 둔다. 서로를 생성자로 받으면 이렇게 된다.

```
orderService 만들자          (만드는 중: orderService)
 └ PaymentService 필요 → 만들러 감
     paymentService 만들자   (만드는 중: orderService, paymentService)
      └ OrderService 필요 → 보관함엔 없고 "만드는 중"에 있다 → 순환!
```

```
***************************
APPLICATION FAILED TO START
***************************

Description:

The dependencies of some of the beans in the application context form a cycle:

┌─────┐
|  orderService defined in file [.../OrderService.class]
↑     ↓
|  paymentService defined in file [.../PaymentService.class]
└─────┘
```

생성자 주입이면 풀 방법이 없다. 필드 주입은 옛날에는 반쯤 만든 객체를 넘겨 풀렸지만, Spring Boot 2.6부터는 기본으로 막는다(`spring.main.allow-circular-references`는 최후의 수단). 보통 공통 로직을 제3의 클래스로 빼서 고친다.

### 7. 마무리: 후처리

빈 팩토리는 원래 **설계도대로 객체를 만드는 것**까지만 할 줄 안다. `@Autowired` 필드, `@PostConstruct`, `@Transactional` 같은 어노테이션 기능은 **객체를 만든 후**에 후처리기(`BeanPostProcessor`)들이 덧붙인다. "후처리"의 "후"는 생명주기 끝이 아니라 "만든 후"다.

```java
// 설명용: 설계도 한 장으로 빈 하나를 만드는 흐름
Object createBean(설계도 d) {
    Object bean = 생성자로_만들기(d);              // ① 빈 팩토리가 직접 (생성자 주입 포함)

    // ↓ 여기부터는 전부 "만든 후" 후처리기들이 한다
    bean = @Autowired_담당.손보기(bean);            // ② 넣어 주기: 필드·세터 주입
    bean = @PostConstruct_담당.손보기(bean);        // ③ 불러 주기: 초기화 메서드
    bean = 프록시_담당.손보기(bean);                 // ④ 감싸 주기: 필요하면 프록시로 바꿔치기

    보관함.put(d.이름, bean);                      // ⑤ 손에 든 것(원본 또는 프록시)을 보관함에
    return bean;
}
```

모든 빈이 이 흐름을 지나고, 후처리기는 **자기 담당 어노테이션이 있을 때만** 일한다. 없으면 받은 걸 그대로 돌려준다. 후처리기가 다른 객체를 돌려주면 그게 빈으로 바뀌어 들어간다. 후처리기는 다른 빈들이 지나가기 전에 서 있어야 하므로 `refresh()`에서 먼저 만든다.

| 하는 일 | 어노테이션 | 후처리기 |
| --- | --- | --- |
| 넣어 주기 | `@Autowired` (필드·세터) | `AutowiredAnnotationBeanPostProcessor` |
| 불러 주기 | `@PostConstruct`, `@PreDestroy` | `CommonAnnotationBeanPostProcessor` |
| 감싸 주기 | `@Transactional`, `@Cacheable`, `@Async` | 자동 프록시 생성기 등 |

**넣어 주기: 주입 방식에 따라 타이밍이 다르다**

| | 생성자 주입 (`@RequiredArgsConstructor`, 생성자 `@Autowired`) | 필드 주입 (필드 `@Autowired`) |
| --- | --- | --- |
| 넣는 시점 | 객체를 만들면서 같이 (①) | 객체를 다 만든 다음 (②) |
| 넣는 방법 | 생성자 파라미터로 | 리플렉션으로 `private` 필드를 열고 넣는다 |
| 생성자가 도는 동안 | 이미 들어와 있다 | 아직 `null` |
| `final` | 가능 | 불가능 |
| 테스트에서 | `new OrderService(가짜)`로 바로 | Spring 없이는 넣기 어렵다 |

`@RequiredArgsConstructor`는 Lombok이 **컴파일할 때** `final` 필드를 받는 생성자를 써 주는 것이라, `.class`에는 평범한 생성자만 남는다. 필드 주입을 덜 쓰는 이유는 결국 이 타이밍 차이에서 나온다. 의존성 없이도 객체가 잠깐 존재할 수 있고, 바뀔 수 있는 필드가 되고, 넣는 길이 Spring의 리플렉션뿐이다.

**불러 주기: 초기화와 소멸** (가볍게)

Spring 빈 생명주기에서 "초기화"는 생성자가 아니라, **생성과 주입이 끝난 뒤 따로 부르는 준비 메서드**다(`@PostConstruct`, `afterPropertiesSet()`). 소멸은 서버를 끌 때 부르는 정리 메서드다(`@PreDestroy`, `close()`).

| | 예 |
| --- | --- |
| 초기화 | HandlerMapping이 `afterPropertiesSet()`에서 빈들을 훑어 주소 표를 채운다. 캐시 미리 채우기 |
| 소멸 | `DataSource`가 커넥션을 닫는다. 서버를 끄면 `HikariPool-1 - Shutdown initiated...`가 찍힌다 |

우리 코드에서는 생성자 주입이면 쓸 일이 많지 않다. 기억할 건 순서다. **초기화(③)가 프록시(④)보다 먼저**라서, `@PostConstruct` 안에서 부른 `@Transactional` 메서드에는 트랜잭션이 걸리지 않는다. 그 시점에는 프록시가 아직 없고, 안에서 부르는 건 `this` 호출이라 프록시를 거치지도 않는다. 시작할 때 트랜잭션이 필요한 일은 모든 빈이 준비된 뒤 도는 `ApplicationRunner`나 `ApplicationReadyEvent`에서 다른 빈을 통해 부른다.

**감싸 주기: 프록시**

프록시 담당 후처리기는 빈마다 "감쌀 메서드(`@Transactional` 등)가 **하나라도** 있나?"를 본다. 있으면 프록시를 만들어 원본 대신 돌려주고, 없으면 원본을 그대로 돌려준다. 그래서 보관함에도, 컨트롤러에 주입되는 것도 프록시다.

```
컨트롤러 ──▶ 프록시 ──▶ 원본 OrderService
              ├ order()  @Transactional → 트랜잭션 열기 → 원본.order() → 커밋
              └ find()                  → 그냥 원본.find()
```

- 프록시는 **객체당 하나**이고, 트랜잭션은 **붙은 메서드에만** 건다.
- Spring Boot는 기본으로 CGLIB를 써서 서버를 켜는 중에 원본을 **상속한 클래스**를 새로 만든다. `orderService.getClass().getName()`을 찍으면 `com.example.order.service.OrderService$$SpringCGLIB$$0`처럼 나온다. 감쌀 게 없는 빈은 그냥 `OrderService`다.
- 왜 상속인지, `final`이면 왜 안 되는지, JDK 동적 프록시와 무엇이 다른지는 8번에서 다룬다.

### 8. Nest와 비교

- Nest는 클래스패스를 스캔하지 않는다. `@Module({ providers: [...] })`에 직접 적는다. Spring의 "찾기·적기"를 사람이 손으로 하는 셈이다.
- 생성자 파라미터 타입은 TypeScript가 `emitDecoratorMetadata`로 남긴 메타데이터(`reflect-metadata`)로 읽는다. Spring에서 리플렉션으로 생성자 파라미터 타입을 보는 자리다.
- provider 기본 스코프도 싱글톤이다. 초기화·소멸은 `onModuleInit`, `onModuleDestroy` 같은 lifecycle hook으로 한다.
- 순환 의존은 `forwardRef()`로 명시해야 풀린다.
- provider를 프록시로 바꿔치기하는 기능은 기본으로 없다. 앞뒤를 감싸는 건 주로 Interceptor가 핸들러 단위로 한다.

## 헷갈렸던 점

| 처음의 생각 | 정리한 내용 |
| --- | --- |
| 우리가 쓰는 파일이 `.jar`이고 컴파일하면 `.class` | `.java` → (`javac`) `.class` → (빌드) `.jar`. jar는 `.class`를 묶은 zip이다 |
| 프로젝트를 켜면 모든 클래스 정보가 메모리에 올라간다 | JVM은 처음 필요할 때 하나씩 올린다. 다만 빈은 Spring이, 엔티티는 Hibernate가 켤 때 올린다 |
| 메모리에 올린다 = 힙에 올린다 | 클래스 정보는 Metaspace, 객체는 힙이다 |
| 빈으로 등록돼도 안 쓰면 메모리에 안 올라간다 | 싱글톤 빈은 안 써도 켤 때 만든다(`@Lazy` 제외) |
| JPA 프로젝트면 거의 모든 파일이 빈이다 | 엔티티, DTO, enum, 예외는 빈이 아니다 |
| 컴포넌트 스캔이 설계도로 빈을 만든다 | 스캔은 설계도까지다. 만드는 건 빈 팩토리다 |
| 설계도에 "이걸 만들려면 저게 필요" 같은 트리가 적힌다 | 설계도는 빈마다 독립된 낱장이다. 의존성은 만들 때 생성자를 열어 보고 알고, 모양은 그물에 가깝다 |
| 세 길은 설계도로 조립하는 작업이다 | 세 길은 설계도를 **만드는** 쪽이다 |
| 프록시는 객체 전체에 트랜잭션을 건다 | 프록시는 객체당 하나지만 트랜잭션은 붙은 메서드에만 건다 |
| "후처리"는 생명주기 맨 끝의 처리다 | "객체를 만든 후"의 처리다. 필드 주입, 초기화, 프록시가 다 후처리다 |
| 생성자 위 `@Autowired`와 필드 위 `@Autowired`는 같다 | 넣는 시점이 다르다. 이 차이가 필드 주입을 덜 쓰는 이유다 |
| 초기화 = 생성자에서 값 넣기 | Spring 생명주기의 초기화는 주입이 끝난 뒤 부르는 준비 메서드(`@PostConstruct`)다 |

### 확인 문제

**문제 1.** `OrderController`가 `@RequiredArgsConstructor` + `final OrderService`로 받는 상황. 서버를 켰을 때 `OrderService`가 `OrderController` 안에 들어가기까지 Spring이 하는 일을 네 칸 순서대로, 칸마다 리플렉션을 쓰는지와 함께 설명하라.

내 답:

> 먼저 컴포넌트스캔을 하겠지? 그래서 controller, service 등등의 컴포넌트 어노테이션이 달린 객체를 탐색해서 설계도를 먼저 표로 구성을 하겠지? 그리고 그 설계도를 기반으로 빈으로 제작을 하겠지?
>
> 그때 셋팅된 생성자에 의해 객체를 만들거야 지금 보면 RequiredArgs 로 시작하는게 달려있기 땜누에 필드에 있는 것도 생성자 시점으로 확인해서 현재 빈으로 등록되어 있지 않으면 빈으로 등록할꺼야 그리고 먼저 객체를 만들면 이제 후처리를 하겠지
>
> 필드에 따로 Autowired 가 있으면 그걸 추가적으로 처리할꺼야
> 그리고 postConstruct 가 붙은곳을 확인해서 검사 해야할거나 무거운 작업을 하도록 했지만 크게 다른 점은 없어
> 마지막으로 트랜잭션이 붙은걸 확인해서 Spring 의 기본 AOP 기술인 CGLIB 을 이용해서 프록시 객체를 만들어주겠지 그리고 끝
> 아! 그리고 리플렉션은 스캔하고 표를 만들때는 안쓰지만 그 이후에는 쓴다~

채점: 통과. 마지막 연결이 빠졌다. 후처리까지 끝난 `OrderService`를 보관함에 넣고, 그 완성품(프록시면 프록시)이 컨트롤러 생성자에 들어간다. 탐색하는 건 객체가 아니라 클래스(`.class`)이고, 여기서는 빈을 "등록"하는 게 아니라 "먼저 만든다"(설계도는 이미 등록돼 있다). Spring은 필드가 아니라 생성자 파라미터 타입을 본다. CGLIB는 AOP 기술이 아니라 프록시 클래스를 만드는 도구다.

**문제 2.** 컴포넌트 스캔은 왜 리플렉션 없이 `.class`를 바이트로 읽을까? ① 리플렉션을 쓰려면 무엇이 먼저 돼 있어야 하나 ② "클래스를 메모리에 올린다"는 무엇이 어디에 올라가는 것이고 객체 생성과 뭐가 다른가 ③ 빈이 된 클래스는 아무 데서도 안 쓰이면 메모리에 안 올라가나

내 답 (첫 시도):

> 리플렉션을 하려면 기본적으로 객체가 메모리에 셋팅이 되어야 하는데 단순 스캔을 위해 모든 객체를 올리면 메모리에 과부하가 오게 된다.. 특히 lazy 의 성격에 안맞기 때문에 그럴필요 없다.

채점: 부분 정답. 올라가 있어야 하는 건 객체가 아니라 클래스 정보다. ②와 ③이 빠졌고, 이유도 "과부하"보다는 느려짐, Metaspace 자리 차지, `static` 코드 실행이다.

내 답 (재시도, ②③):

> 2. 클래스가 메모리에 올라가는건 metaspace 에 클래스 정보가 올라간다는것, 객체를 만들면 힙메모리에 객체가 올라간다
> 3. 올라간다. 원래 사용을 안하면 메모리에 올리지 않지만 spring 은 빈으로써 싱글톤으로 등록되어 사용이 되어야 하기 때문에 만들어진다

채점: 통과. 이유는 "써야 해서"보다 Spring이 일부러 켤 때 다 만들어서다(문제를 시작할 때 잡으려고). 이때 클래스 정보(Metaspace)와 빈 객체(힙)가 둘 다 생긴다. 예외는 `@Lazy`.

**문제 3.** 설계도(빈 정의): ① 어떤 세 길로 만들어지나 ② 적히는 것과 적히지 않는 것, 적히지 않는 건 언제 아나 ③ 왜 바로 만들지 않고 설계도부터 다 모으나

내 답 (첫 시도):

> 1. 설계도는 컴포넌트스캔, spring 의 기본 설정, 기억 안남
> 2. 적히는것은 뭐였더라...적히지 않은건 후처리때 처리되는데 그때 전부 만들어준다
> 3. 객체를 바로 만들면 메모리에 과부하 오니까?

채점: 미통과. `@Bean` 메서드가 빠졌고, 적히는 것을 기억하지 못했다. 의존성은 후처리 때가 아니라 만들 때 생성자를 열어 보고 안다. 설계도부터 모으는 건 메모리 때문이 아니다(싱글톤은 어차피 다 만든다).

내 답 (재시도):

> 1. 컴포넌트 스캔, spring 의 기본 설정, configuration 으로 등록된 @Bean
> 2. 적히는 것은 클래스 이름, 빈의 이름, 싱글톤으로 할건지, 지연 생성인지, 빈의 순서가 어떻게 되는지, 어떻게 만들지 그리고 적히지 않는 것들은 나중에 실제 객체를 만들게 되면서 후처리로 알게 됩니다
> 3. 설계도를 한 눈에 파악하고 모양도 맞춰서 일관성있게 가져갈 수 있고 다 적어봐야 이미 만든것 추가로 만들어야 할것 과 같은게 파악하기 쉬워서

채점: 통과. "빈의 순서"는 순서가 아니라 우선 빈(`@Primary`)이다. 의존성은 생성자 주입이면 후처리 전, 만들 때 생성자를 보고 안다(필드 주입이면 후처리 때).

**문제 4.** `OrderService`에 (A) `@RequiredArgsConstructor` + `final OrderRepository`, (B) 필드 `@Autowired PaymentClient`, `@Transactional order()`, 그냥 `find()`가 있다. ① A와 B는 각각 언제 들어오고, 무엇을 권장하며 왜인가 ② 컨트롤러가 받는 `orderService`는 원본인가 프록시인가, 누가 언제 바꿔치기하나 ③ `find()`에는 트랜잭션이 걸리나

내 답:

> 1. A 는 최초에 객체를 생성하는 과정에서 바로 확인해서 만든다. B는 후처리의 첫번째 과정에서 이어진다. 그래서 좀 더 뒤에서 작업이 진행되기 때문에 문제를 뒤늦게 파악할 수 있고 단점이 좀 생길 수 있다 그래서 왠만하면 객체 생성자 타이밍에 만들어주는것이 좋다
> 2. 프록시이다. 객체를 만들고 마지막에 트랜잭션과 같은 어노테이션이 있는지 확인하는데 이때 있으면 해당 객체를 CGLIB 을 기반으로 프록시 객체를 만들어서 실제 controller 는 CGLIB 객체를 바라보게 한다
> 3. 걸리지 않는다. 트랜잭션을 기반으로 객체 전체가 프록시로 만들어지지만 그중 트랜잭션이 걸림 메서드는 확실히 구분하기 때문

채점: 통과. 권장 이유는 구체적으로, 생성자 안에서 B는 아직 `null`이고, 의존성 없이 객체가 생길 수 있고, `final`을 못 쓰고, 테스트에서 `new`로 넣기 어렵다. 바꿔치기는 프록시 담당 후처리기가 초기화 다음, 보관함에 넣기 전에 한다. `find()`도 프록시를 거치지만 프록시가 그냥 원본에게 넘긴다.

## 예제와 확인 결과

- 코드는 모두 설명용이고 실행하지 않았다.
- 시작·종료 로그와 DEBUG 로그는 Spring Boot 3.x + JPA 앱에서 보통 나오는 모양을 옮긴 것이다. 버전과 설정에 따라 줄이 다를 수 있다.
- `refresh()`의 단계 순서는 2026-11-28에 Spring Framework 6.2.x 소스(`AbstractApplicationContext`)에서 직접 확인했다.
- 직접 확인해 볼 것: 위 DEBUG 설정으로 빈 생성 로그 보기, `getClass().getName()`으로 프록시 여부 보기, `java -Xlog:class+load -jar ...`로 클래스가 로딩되는 시점 보기.

## 남은 질문

- 리플렉션이 어노테이션을 읽고 메서드를 부르는 비용은 얼마나 되고, Spring은 그걸 어떻게 줄일까? (7번)
- CGLIB는 왜 상속으로 만들고, `final`이면 왜 안 될까? JDK 동적 프록시와는 무엇이 다를까? (8번)

## 참고 자료

이번 공부는 Claude와 대화로 진행했다(2026-11-28).

- 김영한, 인프런 "스프링 핵심 원리 - 기본편"
  - [스프링 컨테이너 생성](https://www.inflearn.com/courses/lecture?courseId=325969&unitId=55352)
  - [스프링 빈 설정 메타 정보 - BeanDefinition](https://www.inflearn.com/courses/lecture?courseId=325969&unitId=55359)
  - [탐색 위치와 기본 스캔 대상](https://www.inflearn.com/courses/lecture?courseId=325969&unitId=55369)
- [Spring Framework - The IoC Container](https://docs.spring.io/spring-framework/reference/core/beans.html)
- [Spring Framework - Classpath Scanning and Managed Components](https://docs.spring.io/spring-framework/reference/core/beans/classpath-scanning.html)
- [Spring Framework - Container Extension Points (BeanPostProcessor)](https://docs.spring.io/spring-framework/reference/core/beans/factory-extension.html)
- [Spring Framework - Lifecycle Callbacks](https://docs.spring.io/spring-framework/reference/core/beans/factory-nature.html)
- [AbstractApplicationContext 소스 (6.2.x)](https://github.com/spring-projects/spring-framework/blob/6.2.x/spring-context/src/main/java/org/springframework/context/support/AbstractApplicationContext.java)

## 정리 이력

- 2026-11-28: 6번 빈 생성을 공부하고 확인 문제 4개를 통과해 정리했다.
