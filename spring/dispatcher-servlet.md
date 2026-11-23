# DispatcherServlet은 요청을 어떻게 내 메서드까지 보낼까?

[목차](README.md) · [이전: 필터는 무엇을 걸러내고, Spring은 어디서부터일까?](filter.md)

## 다루는 범위

필터를 통과한 요청이 Spring MVC의 정문 `DispatcherServlet`에 들어와 내 컨트롤러 메서드가 실행되기까지를 정리한다. [전체 흐름](README.md#전체-흐름)의 3~6단계와 8단계다.

- DispatcherServlet 안의 순서: 찾기 → preHandle → 실행 → postHandle → 마무리 → afterCompletion
- 찾기: HandlerMapping의 주소 표와 HandlerExecutionChain
- 인터셉터의 세 시점과 등록 방식
- 실행: HandlerAdapter
- 들어올 때(6단계): ArgumentResolver와 역직렬화
- 나갈 때(8단계): 메시지 컨버터와 직렬화
- Nest의 Guard, Interceptor, Pipe와 비교

메서드 안(7단계, 서비스와 DB)은 [4장](transaction-and-db.md)에서, 응답이 나가는 길과 예외는 [5장](response-and-exception.md)에서 다룬다. 코드는 설명용이다.

## 시작한 질문

1. `health()`를 부르는 코드가 없는데, 누가 이 메서드를 찾아서 부를까?
2. `health()`는 `Map`을 리턴했을 뿐인데, 받는 쪽에는 JSON이 도착한다. 누가 바꿔줄까?

## 내가 이해한 내용

### 0. DispatcherServlet 안의 순서

```
찾기 → preHandle → 실행 → postHandle → 마무리 → afterCompletion
(HandlerMapping) (인터셉터) (HandlerAdapter) (인터셉터)        (인터셉터)
```

모든 요청이 지나는 정문이라서 앞 컨트롤러(front controller)라고 부른다. DispatcherServlet은 직접 일하지 않고 순서를 진행하면서 각 일을 맡은 부품을 부른다.

### 1. 찾기: HandlerMapping의 주소 표

**서버가 켜질 때 표를 미리 만든다.**

1. 컴포넌트 스캔으로 `@Component`가 붙은 클래스를 찾는다. `@RestController` 안에도 `@Component`가 들어 있다.
2. 찾은 클래스로 객체(빈)를 만든다. `HealthController` 객체가 여기서 생긴다.
3. `RequestMappingHandlerMapping`이 빈의 메서드를 리플렉션으로 훑어 `@GetMapping` 같은 어노테이션을 읽고, "HTTP 메서드 + 주소 → 핸들러" 표에 적는다.

`@GetMapping`은 안을 열어 보면 `@RequestMapping(method = GET)`이 붙은 어노테이션이라, Spring은 결국 `@RequestMapping` 하나만 읽으면 된다. 표를 켜질 때 만들기 때문에 같은 주소를 두 번 등록하면 시작할 때 바로 에러가 난다. Actuator의 `/actuator/mappings`로 표 전체를 볼 수 있다.

**요청이 오면 표에서 꺼낸다.**

| 요청 | 결과 |
| --- | --- |
| `GET /api/health` | `HealthController` 객체 + `health()` 메서드 (= 핸들러) |
| `POST /api/health` | 주소는 있는데 HTTP 메서드가 다름 → 405 |
| `GET /api/nothing` | 주소가 없음 → 404 |

찾은 핸들러에 이 주소에 걸린 인터셉터 목록을 묶어서 돌려준다. 이 묶음이 `HandlerExecutionChain`이다.

```
HandlerExecutionChain
 ├ 핸들러: HealthController 객체 + health()
 └ 인터셉터: [로그인 확인, …]
```

HandlerMapping의 일은 여기서 끝이다. 찾아만 주고 실행하지 않는다.

### 2. 인터셉터: 실행을 앞뒤로 감싼다

| | 언제 | 예외가 나면 | 주로 하는 일 |
| --- | --- | --- | --- |
| `preHandle` | 실행 전 | `false`를 돌려주면 실행 자체를 막는다 | 로그인, 권한 확인 |
| `postHandle` | 실행 직후 | 불리지 않는다 | 화면에 넘길 데이터 손보기 |
| `afterCompletion` | 응답 마무리까지 다 끝난 뒤 | 그래도 불린다 | 시간 로그, 스레드에 붙인 값 정리 |

- 인터셉터가 여러 개면 `preHandle`은 A → B 순서, 나머지는 B → A 거꾸로 불린다. 들어갈 땐 순서대로, 나올 땐 거꾸로다.
- **`afterCompletion`에서 정리하는 이유**: 작업 스레드는 요청이 끝나면 풀로 돌아가 다음 요청에 다시 쓰인다. 스레드에 붙여 둔 값(`ThreadLocal`, 로그용 MDC 등)을 여기서 지우지 않으면 다음 요청이 그 값을 물려받는다.
- `@RestController` 메서드는 응답 본문을 어댑터 안에서 이미 써 버리기 때문에, `postHandle`에서는 JSON 본문을 바꿀 수 없다.
- 인터셉터는 DispatcherServlet 안에 있어서 어떤 핸들러로 갈지 알고, 여기서 난 예외는 `@RestControllerAdvice`가 잡는다. 필터와 다른 점이다([2장](filter.md#5-경계선-restcontrolleradvice는-dispatcherservlet부터다)).

**인터셉터는 어노테이션이 아니라 설정으로 등록한다.**

```java
@Configuration
class WebConfig implements WebMvcConfigurer {
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new LoginCheckInterceptor())
                .addPathPatterns("/api/**")          // 어디에 걸지
                .excludePathPatterns("/api/health"); // 어디는 뺄지
    }
}
```

메서드 위에 어노테이션을 붙여서 쓰고 싶다면, 커스텀 어노테이션은 "표시"로만 쓰고 `preHandle`이 그 표시를 확인하게 만든다.

```java
public boolean preHandle(HttpServletRequest req, HttpServletResponse res, Object handler) {
    if (handler instanceof HandlerMethod hm && hm.hasMethodAnnotation(LoginRequired.class)) {
        // 로그인 확인
    }
    return true;
}
```

메서드 위에 붙여서 바로 동작하는 `@Transactional`, `@PreAuthorize`는 인터셉터가 아니라 AOP 프록시가 감싸는 방식이다.

### 3. 실행: HandlerAdapter

DispatcherServlet은 직접 실행하지 않고 어댑터들에게 묻는다.

```java
for (HandlerAdapter adapter : handlerAdapters)
    if (adapter.supports(handler))
        return adapter;   // "응, 내가 할게"
```

컨트롤러 메서드를 실행하는 건 `RequestMappingHandlerAdapter`다. 이 어댑터가 하는 일은 세 가지다.

1. 파라미터 준비: 요청 JSON → 객체 **(역직렬화)**
2. 호출: `method.invoke(객체, 파라미터)` (리플렉션)
3. 리턴값 처리: 객체 → JSON **(직렬화)**

[전체 흐름](README.md#전체-흐름)의 6, 7, 8단계가 전부 이 안에서 일어난다. 그래서 첫 번째 질문의 답은 이거다. **`health()`는 DispatcherServlet이 (HandlerMapping으로) 찾고, HandlerAdapter가 부른다.**

### 4. 들어올 때(6단계): ArgumentResolver가 값을 꺼내 넣는다

```
POST /api/orders/42?coupon=WELCOME
{"productId": 7, "quantity": 2}
```

```java
@PostMapping("/api/orders/{shopId}")
OrderResponse order(
    @PathVariable Long shopId,             // 42      ← 주소에서
    @RequestParam String coupon,           // WELCOME ← 쿼리 스트링에서
    @Valid @RequestBody OrderRequest req)  // 본문 JSON → 객체 (Jackson)
```

- 어댑터는 파라미터를 하나씩 보면서, 파라미터마다 맞는 `HandlerMethodArgumentResolver`에게 "요청의 어디서 값을 꺼낼지"를 맡긴다.
- `@RequestBody`는 본문을 메시지 컨버터(`HttpMessageConverter`, JSON이면 Jackson)가 `OrderRequest` 객체로 바꾼다(역직렬화).
- `@Valid` 검증에 실패하면 메서드는 실행되지도 않고 `MethodArgumentNotValidException`으로 400이 나간다.

### 5. 나갈 때(8단계): 메시지 컨버터가 JSON으로 쓴다

`@RestController`(= `@Controller` + `@ResponseBody`)면 리턴값 처리기(`ReturnValueHandler`)가 메시지 컨버터를 불러 `Map.of("status", "ok")`를 `{"status":"ok"}`로 바꿔 응답 본문에 쓴다(직렬화).

들어올 때 JSON을 객체로 바꾼 것도, 나갈 때 객체를 JSON으로 바꾸는 것도 같은 부품, 메시지 컨버터다. 그래서 두 번째 질문의 답은 **메시지 컨버터, Jackson**이다.

### 6. Nest와 비교

| Spring MVC | Nest | 비고 |
| --- | --- | --- |
| HandlerMapping (켜질 때 어노테이션을 읽어 표 작성) | 켜질 때 `@Get()` 같은 데코레이터 메타데이터를 읽어 라우터에 등록 | 둘 다 시작할 때 주소 표를 만든다 |
| 인터셉터 `preHandle` | Guard | 실행 여부를 정한다 |
| 인터셉터 앞뒤 | Interceptor | Nest Interceptor는 RxJS로 응답 값까지 바꿀 수 있어 AOP에 더 가깝다 |
| ArgumentResolver + `@Valid` | 파라미터 데코레이터(`@Param`, `@Query`, `@Body`) + Pipe(`ValidationPipe`) | 검증 실패 시 둘 다 400 |
| 메시지 컨버터(Jackson) | 리턴 객체를 Express `res.json()`으로 직렬화 | |

## 헷갈렸던 점

| 처음의 생각 | 정리한 내용 |
| --- | --- |
| 인터셉터는 메서드 위에 어노테이션으로 붙인다 | `WebMvcConfigurer`에 경로 패턴으로 등록한다. 커스텀 어노테이션은 `preHandle`이 확인하는 표시일 뿐이다 |
| HandlerMapping이 메서드를 실행한다 | 찾기만 한다. 실행은 HandlerAdapter가 한다 |
| 결국 실행하는 건 어댑터가 맞나 | 맞다. DispatcherServlet은 순서를 진행하고, 실제 호출(`method.invoke`)은 어댑터가 한다 |
| ArgumentResolver가 하는 일이 막연했다 | 파라미터 하나마다 요청의 어디서 값을 꺼낼지 정한다. 주소(42) → `@PathVariable`, 쿼리(WELCOME) → `@RequestParam`, 본문 → `@RequestBody` |
| JSON → 객체가 직렬화, 객체 → JSON이 역직렬화다 | 반대다. 객체 → JSON이 직렬화, JSON → 객체가 역직렬화다 |
| 주소 표는 요청이 올 때 만든다 | 서버가 켜질 때 만든다. 그래서 같은 주소를 두 번 등록하면 시작할 때 에러가 난다 |

### 공부하며 던진 질문

- **어노테이션은 어떻게 읽어?** 리플렉션으로 클래스와 메서드에 붙은 어노테이션을 꺼내 본다. 빈 생성, 리플렉션, AOP는 따로 공부하기로 했다([공부 계획](README.md#공부-계획)).
- **HandlerExecutionChain은 뭐야?** HandlerMapping이 돌려주는 묶음이다. 찾은 핸들러와 그 주소에 걸린 인터셉터 목록이 들어 있다.
- **인터셉터 순서는 어떻게 정해져?** `addInterceptors`에 등록한 순서다. 들어갈 때 그 순서, 나올 때 거꾸로다.

## 예제와 확인 결과

위 코드는 모두 설명용이다. 직접 실행하지 않았다.

## 남은 질문

- 빈은 정확히 어떤 순서로 만들어질까? (컴포넌트 스캔 → 빈 정의 → 생성 → 의존성 주입 → 초기화)
- 리플렉션으로 어노테이션을 읽고 메서드를 부르는 비용은 얼마나 될까? Spring은 이를 어떻게 줄일까?
- `postHandle`이 쓸모 있는 경우는 뷰(템플릿)를 그릴 때뿐일까?

## 참고 자료

이번 공부는 발표("요청 하나, 스레드 하나", 2026-11-24)를 준비하며 Claude와 대화로 진행했다. 더 읽어볼 공식 문서:

- [Spring Framework - DispatcherServlet](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-servlet.html)
- [Spring Framework - 인터셉터](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-servlet/handlermapping-interceptor.html)
- [Spring Framework - 컨트롤러 메서드 인자](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-controller/ann-methods/arguments.html)
- [Spring Framework - 메시지 컨버터](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-config/message-converters.html)

## 정리 이력

- 2026-11-23: 발표 준비로 공부한 내용을 노트로 정리했다.
