# 리플렉션은 어떻게 어노테이션을 읽고 메서드를 부를까?

[목차](README.md) · [이전: 빈은 어떻게 만들어질까?](bean-creation.md) · [다음: AOP 프록시는 어떻게 @Transactional을 감쌀까?](aop-proxy.md)

## 다루는 범위

[6번](bean-creation.md)에서 빈을 만들 때 리플렉션을 쓰는 곳과 안 쓰는 곳을 나눠 봤다. 이 노트는 그 도구 자체, 리플렉션이 Spring에서 하는 일을 정리한다.

- 리플렉션이란 무엇이고, Spring에 왜 필요한가
- 리플렉션, DI 컨테이너, AOP의 관계
- 어노테이션 읽기: `@Retention`, 메타 어노테이션을 펼쳐 읽기
- 객체 만들기와 필드 넣기 (6번 복습)
- 메서드 부르기: 주소 표에 저장한 `Method`로 `invoke`
- 비용: 왜 Spring에서는 괜찮은가
- Nest와 비교

코드는 설명용이다.

## 시작한 질문

1. 리플렉션이란 무엇일까? 실행 중에 내가 모르는 코드를 보려고 만든 걸까?
2. Spring은 메서드에 붙은 어노테이션을 어떻게 읽고, 우리 메서드를 어떻게 부를까?
3. 리플렉션은 느리다는데 Spring은 괜찮을까?

## 내가 이해한 내용

### 1. 리플렉션: 코드를 짜는 시점에는 모르는 클래스를, 실행 중에 보고·만들고·부르기

```java
// 평소: 코드를 쓰는 시점에 이미 OrderService를 안다
OrderService s = new OrderService();
s.order();

// 리플렉션: 실행 중에 "이름"만 가지고 똑같은 일을 한다 (기본 생성자라고 치면)
Class<?> c = Class.forName("com.example.order.service.OrderService"); // 클래스 로딩
Object s = c.getDeclaredConstructor().newInstance();                   // 객체 만들기
Method m = c.getMethod("order");                                        // 메서드 찾기
m.invoke(s);                                                            // 메서드 부르기
boolean isService = c.isAnnotationPresent(Service.class);              // 어노테이션 확인
```

Spring 코드는 우리 `OrderService`를 모르는 채로 만들어졌다. 그래서 `new OrderService()`를 미리 적어 둘 수 없고, 스캔으로 클래스 이름을 얻은 뒤 리플렉션으로 다룬다. 남의 코드를 받아 돌려야 하는 도구들이 주로 쓴다.

| 도구 | 실행 중에 모르는 것 | 리플렉션으로 하는 일 |
| --- | --- | --- |
| Spring | 우리가 만든 `OrderService` | 객체 만들기, 어노테이션 읽기, 메서드 부르기 |
| JUnit | 우리가 짠 테스트 클래스 | `@Test` 붙은 메서드를 찾아서 부르기 |
| Jackson | 우리가 만든 DTO | 필드를 읽어서 JSON 만들기 |

리플렉션이 읽는 건 객체가 아니라 **클래스 정보**다. 그래서 클래스가 먼저 로딩돼 있어야 한다(클래스 정보는 Metaspace, 6번 참고).

### 2. 리플렉션, DI 컨테이너, AOP는 층이다

```
AOP (8번)          부를 때마다 앞뒤에 일을 끼워 넣기 (@Transactional)
DI 컨테이너 (6번)   객체를 만들고 이어 주기
리플렉션 (7번)      ← 이 둘을 받치는 바닥 도구
```

DI 컨테이너는 리플렉션으로 생성자를 찾고 필드에 값을 넣는다. AOP 프록시는 컨테이너가 빈을 만드는 중에 후처리기가 끼우고, 리플렉션으로 `@Transactional` 붙은 메서드를 찾는다. AOP인지 아닌지는 **읽고 끝나느냐, 부를 때마다 앞뒤에 일이 끼어드느냐**로 가른다.

| 어노테이션 | 읽은 다음 하는 일 | AOP? |
| --- | --- | --- |
| `@RestController`, `@Service` | 빈으로 만든다 | ✕ 읽고 끝 |
| `@GetMapping` | 주소 표에 적는다 | ✕ 읽고 끝 |
| `@Autowired` | 필드에 값을 넣는다 | ✕ 읽고 끝 |
| `@Transactional`, `@Cacheable`, `@Async` | 메서드를 감싸서 부를 때마다 일을 더한다 | ○ |

AOP도 우리 클래스 코드를 고치지는 않는다. 원본은 그대로 두고 앞에 프록시를 세운다.

### 3. 어노테이션 읽기

어노테이션은 그 자체로는 아무 일도 안 하는 **메모**다. 누군가 읽어야 의미가 생기고, 실행 중에 읽으려면 그때까지 살아 있어야 한다.

| `@Retention` | 어디까지 남나 | 예 |
| --- | --- | --- |
| `SOURCE` | `.java`에만. 컴파일하면 사라진다 | Lombok `@RequiredArgsConstructor`, `@Override` |
| `CLASS` (기본값) | `.class`에는 남지만 실행 중 리플렉션으로는 못 본다 | |
| `RUNTIME` | 실행 중에 리플렉션으로 읽을 수 있다 | `@Service`, `@GetMapping`, `@Transactional` 등 Spring 어노테이션 |

커스텀 어노테이션을 만들 때 `@Retention(RUNTIME)`을 붙이는 이유가 이것이다. 빼먹으면 기본값 `CLASS`가 되어 실행 중에 안 보이고, 에러 없이 **조용히 동작하지 않는다.**

```java
@Target(ElementType.PARAMETER)          // 어디에 붙일 수 있나
@Retention(RetentionPolicy.RUNTIME)     // 어디까지 살아남나
public @interface LoginUser { }

// ArgumentResolver가 실행 중에 확인한다
public boolean supportsParameter(MethodParameter parameter) {
    return parameter.hasParameterAnnotation(LoginUser.class);
}
```

리플렉션으로 읽는 모양은 클래스 → 메서드 → 어노테이션 → 값 순서다.

```java
Method m = OrderController.class.getDeclaredMethod("find", Long.class);
GetMapping gm = m.getAnnotation(GetMapping.class);   // 안 붙어 있으면 null
String[] paths = gm.value();                          // ["/api/orders/{id}"]
```

자바의 `getAnnotation()`은 **겉에 붙은 것만** 본다. `@GetMapping` 안에 든 `@RequestMapping(method = GET)`까지는 열어 보지 않는다(`m.getAnnotation(RequestMapping.class)`는 `null`). 그래서 Spring은 자기 도구(`AnnotatedElementUtils`, `MergedAnnotations`)로 안쪽까지 펼쳐서 합쳐 읽는다. `@Service`를 `@Component`로 알아보는 것도 같은 원리다.

### 4. 객체 만들기, 필드 넣기 (6번 복습)

```java
Constructor<?> ctor = OrderService.class.getDeclaredConstructors()[0];
Object bean = ctor.newInstance(orderRepository);   // 생성자로 만들기

Field f = OrderService.class.getDeclaredField("paymentClient");
f.setAccessible(true);                             // private 잠금 풀기
f.set(bean, paymentClient);                        // 필드 주입
```

빈을 만들 때 리플렉션으로 생성자·필드·메서드에 붙은 것을 읽고 그에 맞게 처리한다.

| 리플렉션으로 읽는 것 | 그걸로 하는 일 | 누가 |
| --- | --- | --- |
| 생성자와 파라미터 타입 | 필요한 빈을 넣어 객체 만들기 | 빈 팩토리 |
| 필드의 `@Autowired` | 그 필드에 빈 넣기 | `@Autowired` 담당 후처리기 |
| 메서드의 `@PostConstruct` | 그 메서드 부르기 | 초기화 담당 후처리기 |
| 메서드의 `@Transactional` | 프록시로 감쌀지 정하기 | 프록시 담당 후처리기 |
| 컨트롤러 메서드의 `@GetMapping` | 주소 표에 적기 | HandlerMapping (자기 초기화 때) |

클래스 위의 `@Service`, `@Lazy`, `@Primary`는 그보다 앞서 설계도를 적을 때 `.class` 바이트로 읽었다(리플렉션 ✕).

### 5. 메서드 부르기: 저장해 둔 `Method`로 `invoke`

HandlerMapping은 서버 켤 때 컨트롤러 메서드를 리플렉션으로 훑어 주소 표를 만들고, 표에 **`Method` 객체를 그대로** 저장한다. 요청 때는 어노테이션을 다시 읽지 않고, 저장한 `Method`로 `invoke`만 한다.

| 주소 표의 키 | 값 |
| --- | --- |
| `GET /api/orders/{id}` | `orderController` 빈 + `find(Long)` Method |
| `POST /api/orders/{shopId}` | `orderController` 빈 + `create(...)` Method |

```
요청: GET /api/orders/42
 ① HandlerMapping  : 주소 표에서 찾기 → "orderController 빈의 find(Long)"
 ② ArgumentResolver: 파라미터 값 준비 → 42L
 ③ HandlerAdapter  : method.invoke(orderController, 42L)   ← 리플렉션
 ④ 메시지 컨버터    : 리턴값을 JSON으로
```

실제로는 HandlerAdapter가 `InvocableHandlerMethod`를 통해 `invoke`한다. 리플렉션은 **Spring 코드에서 우리 코드로 들어가는 입구**에서만 쓴다. 컨트롤러 안에서 `orderService.find(id)`를 부르는 건 우리가 짠 평범한 자바 호출이다.

| Spring이 우리 메서드를 부르는 곳 | 언제 |
| --- | --- |
| 컨트롤러 메서드 (`@GetMapping` 등) | 요청 올 때마다 |
| `@PostConstruct` 메서드 | 빈 만들 때 한 번 |
| `@Scheduled` 메서드 | 정해진 시간마다 |
| `@EventListener` 메서드 | 이벤트가 날 때 |

### 6. 비용: 비싼 찾기는 켤 때 한 번, 요청 땐 `invoke`만

| 일 | 비용 | Spring에서 언제, 몇 번 |
| --- | --- | --- |
| 클래스를 훑어 메서드·필드 찾기, 어노테이션 읽기 | 비싸다 | 서버 켤 때, 클래스마다 한 번 |
| 찾은 결과 저장 | | 주소 표, 주입할 필드 목록, `@PostConstruct` 목록 등 |
| 저장된 `Method`로 `invoke` | 싸다 | 요청 올 때마다 |

- JVM은 자주 불리는 `invoke`를 점점 빠르게 만든다. JDK 18부터는 리플렉션 내부가 `MethodHandle` 기반으로 다시 구현됐다(JEP 416).
- 요청 하나의 시간은 대부분 DB나 외부 API가 쓴다. `invoke`는 대략 나노초, DB 쿼리는 밀리초 단위라 비교가 안 된다.
- 비용이 드러나는 곳은 **서버 시작 시간**이다. 그래서 Spring Boot 3에는 이 작업을 빌드할 때로 옮기는 AOT 처리와 GraalVM 네이티브 이미지가 있다.

3장에서 남긴 질문 "리플렉션 비용은 얼마나 되고, Spring은 어떻게 줄일까?"의 답이 이것이다.

### 7. Nest와 비교

- TypeScript 데코레이터는 클래스가 정의될 때 실행되어 메타데이터를 남긴다(`reflect-metadata`의 `Reflect.defineMetadata`). Nest는 그걸 `Reflect.getMetadata`나 `Reflector`로 읽는다. Spring이 리플렉션으로 어노테이션을 읽는 자리다.
- Nest도 앱을 띄울 때 컨트롤러를 훑어 라우트를 등록해 두고, 요청 때는 찾아서 부르기만 한다.
- 자바스크립트는 함수가 값이라 `Method.invoke` 같은 별도 API 없이 저장해 둔 핸들러 함수를 그냥 부른다.

## 헷갈렸던 점

| 처음의 생각 | 정리한 내용 |
| --- | --- |
| 리플렉션은 실행 중에 내가 모르는 코드를 "보려고" 만든 것 | 코드를 **짜는 시점에** 모르는 클래스를, 실행 중에 보고 · **만들고 · 부르기** 위한 것 |
| 리플렉션이 객체를 읽는다 | 읽는 건 **클래스 정보**다. 그걸로 객체를 만들거나 메서드를 부른다 |
| 이런 게 전부 AOP다 | 리플렉션은 바닥 도구, DI는 만들고 잇기, AOP는 부를 때마다 앞뒤에 일을 끼우는 것(감싸 주기)만이다 |
| AOP는 코드를 입혀 준다 | 원본 코드는 그대로 두고, 앞에 프록시를 세워서 앞뒤로 일을 끼운다 |
| 요청 과정의 메서드를 Spring이 전부 리플렉션으로 부른다 | 리플렉션은 입구(컨트롤러 메서드) 한 번이다. 그다음은 우리 코드의 평범한 호출이다 |
| (11/25) HandlerMapping이 어노테이션을 읽는 게 리플렉션이지? | 맞다. 서버 켤 때 한 번 읽어 주소 표에 `Method`까지 저장한다. 그 앞의 컴포넌트 스캔은 리플렉션이 아니다 |

### 확인 문제

**문제 1.** `@GetMapping("/api/orders/{id}") find(@PathVariable Long id)`가 있는 컨트롤러가 `orderService.find(id)`를 부른다. ① Spring은 `find()`에 `@GetMapping`이 붙은 걸 언제, 어떻게 알아내나? 커스텀 어노테이션에서 `@Retention(RUNTIME)`을 빼먹으면? ② `GET /api/orders/42`에서 `find()`가 실행되기까지 리플렉션은 어디서 쓰이나? `orderService.find(id)`도 리플렉션인가? ③ 리플렉션은 느리다는데 Spring은 왜 괜찮나?

내 답:

> 1. 그건 처음에 객체를 생성하면서 리플렉션이 발생하면서 알게 됩니다. 이때 어노테이션을 감지하고 mapping 이 표에 등록하는걸 돕습니다. 그리고 만약 커스텀 어노테이션에서 RUNTIME 을 빼먹게 되면 기본값인 CLASS 로 되면서 리플렉션에서 대상으로 잡히지 않습니다.
> 2. mapping 에서 리플렉션으로 인해 등록된 표로 찾아내는데 사용되고 그 메서드를 adapter가 method.invoke() 하면서 사용하게 됩니다. 그리고 컨트롤러에서는 실제 서비스의 코드를 호출합니다.
> 3. 리플렉션이 전체 코드를 훑어 작업하기 때문에 비싸기지만 최초에 빈을 등록하는 과정에서 전부 훑어서 파악을 끝내기 때문에 서버가 켜진 이후에는 그저 호출만 하는거라 괜찮습니다

채점: 통과. 정확히는 컨트롤러 객체를 만들 때가 아니라 **HandlerMapping 빈이 초기화될 때** 컨트롤러 메서드를 훑는다. `RUNTIME`을 빼먹으면 에러 없이 조용히 동작하지 않는다는 점까지 말하면 좋다. 요청 때 리플렉션이 실제로 일하는 건 `invoke` 한 번이고, 주소 표 찾기는 저장해 둔 결과를 꺼내는 것이다. 컨트롤러 안의 `orderService.find(id)`는 리플렉션이 아니라 평범한 호출이라고 분명히 말하면 좋다. 비용이 드러나는 곳은 서버 시작 시간이다.

## 예제와 확인 결과

- 코드는 모두 설명용이고 실행하지 않았다.
- `invoke`와 DB 쿼리의 시간 비교는 단위(나노초 vs 밀리초)만 비교한 대략값이다. 직접 재지 않았다.

## 남은 질문

- 컨트롤러가 부른 `orderService.find(id)`가 프록시를 거칠 때, 프록시는 어떻게 원본 메서드까지 갈까? (8번)

## 참고 자료

이번 공부는 Claude와 대화로 진행했다(2026-11-28).

- 얄코, 인프런 "제대로 파는 자바 (Java)" - [리플렉션](https://www.inflearn.com/courses/lecture?courseId=330942&unitId=158057)
- [Java SE 21 API - java.lang.reflect](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/reflect/package-summary.html)
- [Java SE 21 API - RetentionPolicy](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/annotation/RetentionPolicy.html)
- [JEP 416: Reimplement Core Reflection with Method Handles](https://openjdk.org/jeps/416)
- [Spring Framework - Using Meta-annotations and Composed Annotations](https://docs.spring.io/spring-framework/reference/core/beans/classpath-scanning.html#beans-meta-annotations)
- [Spring Framework - Ahead of Time Optimizations](https://docs.spring.io/spring-framework/reference/core/aot.html)

## 정리 이력

- 2026-11-28: 7번 리플렉션을 공부하고 확인 문제를 통과해 정리했다.
