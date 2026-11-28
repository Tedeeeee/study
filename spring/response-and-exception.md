# 응답과 예외는 어떤 길로 나갈까?

[목차](README.md) · [이전: @Transactional은 DB 연결을 어떻게 붙잡고 있을까?](transaction-and-db.md) · [다음: 빈은 어떻게 만들어질까?](bean-creation.md)

## 다루는 범위

`health()`가 리턴한 뒤 응답이 사용자에게 나가기까지, 그리고 그 길 중간에 예외가 났을 때를 정리한다. [전체 흐름](README.md#전체-흐름)의 9·10단계다.

- 응답은 들어온 길을 거꾸로 나간다
- 왜 거꾸로인가: 전부 한 스레드의 메서드 호출, 호출 스택
- 예외는 DispatcherServlet 안이냐 밖이냐로 갈린다
- 필터에서 난 예외를 처리하는 네 가지
- 돌아보기: 요청 하나 = 스레드 하나, 우리가 짠 건 7단계뿐
- Nest의 Exception Filter와 비교

코드는 설명용이다.

## 시작한 질문

1. `Map`이 JSON이 된 다음, 응답은 어떤 길로 사용자에게 갈까?
2. 그 길 중간에 예외가 나면 누가 받아서 에러 응답을 만들까? 필터에서 난 예외는 왜 다르게 취급될까?

## 내가 이해한 내용

### 1. 응답은 들어온 길을 거꾸로 나간다 (10단계)

지금까지 지나온 곳은 바깥에서 안쪽으로 겹쳐 있다. 톰캣 안에 필터, 그 안에 DispatcherServlet, 그 안에 인터셉터, 맨 안쪽에 어댑터가 있다.

| 겹 (바깥 → 안쪽) | 들어올 때 (위에서 아래로) | 나갈 때 (아래에서 위로) |
| --- | --- | --- |
| 톰캣 | 요청 받기 | 응답 보내기 |
| 필터 | `doFilter` 앞쪽 | `doFilter` 뒤쪽 |
| DispatcherServlet | 핸들러 찾기 | `afterCompletion` 부르기 |
| 인터셉터 | `preHandle` | `postHandle` |
| 어댑터 (맨 안쪽) | `health()` 실행 | `Map` → JSON |

응답은 새 길로 나가지 않는다. 들어올 때 지난 겹을 반대 순서로 하나씩 다시 지나고, 그때마다 그 겹의 "나머지 코드"가 실행된다.

1. 어댑터: `Map` → JSON (직렬화)
2. 인터셉터: `postHandle`
3. DispatcherServlet: 마무리하면서 `afterCompletion`을 부른다
4. 필터: `chain.doFilter()` 아래쪽 코드 ([2장](filter.md#2-dofilter-하나에-들어갈-때와-나올-때가-다-있다))
5. 톰캣: 응답을 사용자에게 보낸다

응답을 보내고 나면 exec-3은 스레드 풀로 돌아가 다음 요청을 기다린다.

### 2. 왜 거꾸로일까: 전부 한 스레드의 메서드 호출이다

이 겹들은 따로 도는 장치가 아니라 **exec-3 한 스레드 안에서 서로를 부르는 메서드**다.

```
톰캣 → filterChain.doFilter()
         → 필터A.doFilter() {
               앞쪽 코드
               chain.doFilter() → … → DispatcherServlet.doDispatch() {
                                          preHandle()
                                          adapter.handle() → health()   ← 맨 안쪽
                                          postHandle()
                                          afterCompletion()
                                      }
               뒤쪽 코드      ← 안쪽이 return해야 실행된다
           }
```

안으로 들어갈 때 호출 스택이 쌓이고, 맨 안쪽 `health()`가 return하면 스택이 위에서부터 풀린다. 바깥 메서드의 남은 코드는 안쪽이 return해야 실행되니까, 나갈 때는 자연스럽게 반대 순서가 된다. 인터셉터가 여러 개일 때 나올 때 거꾸로 불리는 것도 같은 모양이다.

### 3. 예외도 같은 길을 throw로 올라간다. 다른 건 누가 잡느냐다 (9단계)

응답이 return으로 바깥에 나갔듯이, 예외는 throw로 같은 호출 스택을 거꾸로 올라간다.

**DispatcherServlet 안에서 난 예외** (인터셉터, 컨트롤러, 서비스)

1. 서비스에서 예외가 나면 `@Transactional` 프록시가 먼저 롤백하고 연결을 반납한 뒤 예외를 다시 던진다([4장](transaction-and-db.md)).
2. 예외는 컨트롤러, 어댑터를 지나 위로 올라간다. 인터셉터의 `postHandle`은 건너뛴다.
3. DispatcherServlet의 `doDispatch()` 안 `try-catch`가 예외를 잡는다.
4. 예외 처리기(`HandlerExceptionResolver`)에게 넘기고, 그중 하나가 우리가 만든 `@RestControllerAdvice`의 `@ExceptionHandler`를 찾아 실행한다 → **우리가 정한 모양의 에러 JSON**.
5. `afterCompletion`은 예외를 들고 불린다. 그다음은 평소 응답처럼 필터와 톰캣을 지나 나간다.

우리가 처리하지 않은 예외도 Spring 기본 처리기가 알맞은 상태 코드를 붙인다(지원하지 않는 HTTP 메서드 405, 검증 실패 400 등).

**DispatcherServlet 밖, 필터에서 난 예외**

1. DispatcherServlet의 `try-catch` 바깥이라 잡아줄 곳이 없다.
2. 예외는 바깥 필터들을 지나 톰캣까지 올라간다.
3. 톰캣이 에러 페이지인 `/error`로 요청을 다시 보내고, Spring Boot의 `BasicErrorController`가 **기본 에러 JSON**(`timestamp`, `status`, `error`, `path`)을 만든다. 우리가 정한 모양이 아니다.

그래서 필터 쪽 예외는 필터 안에서 직접 처리한다. Spring Security가 401, 403을 자기 필터 안에서 직접 만드는 이유가 이거다. [2장](filter.md#5-경계선-restcontrolleradvice는-dispatcherservlet부터다)의 경계선 이야기가 바로 이것이다.

### 4. 필터 예외를 처리하는 네 가지

| | 방법 | 특징 |
| --- | --- | --- |
| 1 | 필터 안에서 직접 응답 쓰기 | 가장 단순하다. 필터마다 같은 코드가 반복된다 |
| 2 | 예외 전용 필터를 맨 바깥에 두기 | 안쪽 필터에서 올라오는 예외를 한곳에서 잡는다(양파 구조) |
| 3 | `@RestControllerAdvice`로 넘기기 | **추천.** 에러 JSON 모양을 한곳에서 관리한다 |
| 4 | Spring Security는 정해진 자리에 | 인증 실패 401은 `AuthenticationEntryPoint`, 권한 없음 403은 `AccessDeniedHandler` |

3번은 DispatcherServlet이 안에서 쓰는 예외 처리기를 필터가 주입받아 직접 넘기는 방식이다. Spring이 알아서 잡아주는 게 아니라, 우리가 다리를 놓는 것이다.

```java
@Component
class JwtFilter extends OncePerRequestFilter {
    private final HandlerExceptionResolver resolver;

    JwtFilter(@Qualifier("handlerExceptionResolver") HandlerExceptionResolver resolver) {
        this.resolver = resolver;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain)
            throws ServletException, IOException {
        try {
            verify(req);                       // 토큰 검사
            chain.doFilter(req, res);
        } catch (Exception e) {
            resolver.resolveException(req, res, null, e);   // → @ExceptionHandler로
        }
    }
}
```

4번의 주의점: 이 둘을 불러주는 `ExceptionTranslationFilter`는 **자기보다 안쪽 필터의 예외만** 받는다. 그보다 앞에 둔 커스텀 JWT 필터의 예외는 1~3번 방법으로 처리해야 한다.

### 5. 돌아보기: 요청 하나 = 스레드 하나

- 1단계부터 10단계까지 작업 스레드 exec-3 하나가 요청을 처음부터 끝까지 맡았다. 트랜잭션을 `ThreadLocal`에 묶을 수 있는 것도, 응답이 거꾸로 나가는 것도, 스레드가 200개뿐이라 외부 API를 조심해야 하는 것도 모두 여기서 나온다.
- 이 긴 길에서 우리가 직접 짠 건 사실상 7단계, 비즈니스 로직뿐이다. 나머지는 Spring과 톰캣이 해 주고, 그 설정은 Spring Boot가 꾸려 준다. 처음 질문 "`health()`는 누가 부르고, JSON은 누가 만드나"의 답도 결국 여기 있다.

### 6. Nest와 비교

- Nest는 예외를 Exception Filter(`@Catch`)에 모아 처리한다. Guard, Interceptor, Pipe, 컨트롤러에서 난 예외가 다 여기로 온다. `consumer.apply()`로 등록한 미들웨어 예외도 전역 Exception Filter가 받는다.
- `app.use()`로 Express에 직접 붙인 미들웨어는 Nest 바깥이라 Express 기본 에러 처리로 간다. Spring의 필터와 비슷한 처지다.
- 응답을 보내고 나면 돌려줄 스레드가 없다. 이벤트 루프는 바로 다음 일을 처리한다.

## 헷갈렸던 점

| 처음의 생각 | 정리한 내용 |
| --- | --- |
| 응답은 "들어올 때 한 줄, 나갈 때 한 줄"로 가고 오는 것 | 같은 겹을 반대 순서로 다시 지난다. 한 스레드의 호출 스택이 풀리는 것이라 순서가 자연스럽게 반대가 된다 |
| `afterCompletion`은 인터셉터가 스스로 부른다 | DispatcherServlet이 마무리하면서 부른다 |
| 필터에서 난 예외는 못 잡는다 vs 억지로 잡는 방법이 있다 | 둘 다 맞다. 그냥 두면 경계 밖이라 `@RestControllerAdvice`까지 안 가고, 필터가 직접 넘기면 `@ExceptionHandler`로 보낼 수 있다 |
| Security의 401·403도 `@RestControllerAdvice`가 만든다 | Security 필터 안에서 `AuthenticationEntryPoint`와 `AccessDeniedHandler`가 직접 만든다 |
| 서비스에서 예외가 나면 롤백과 에러 응답이 한 번에 된다 | 두 단계다. 프록시가 롤백하고 예외를 다시 던지면, DispatcherServlet이 잡아 `@RestControllerAdvice`로 응답을 만든다 |

### 공부하며 던진 질문

- **그럼 필터의 예외 처리는 어떻게 하면 좋을까?** 위 네 가지 중, 에러 모양을 한곳에서 관리하려면 3번이 깔끔하다. Security를 쓰면 4번을 기본으로 하고, 그 앞단 필터는 1~3번으로 보완한다.
- **2장(필터)에서 "못 잡는다"고 한 건 틀린 말이야?** 경계선을 설명한 말이다. "그냥 두면" 안 간다는 뜻으로 읽어야 한다.

## 예제와 확인 결과

위 코드는 모두 설명용이다. 직접 실행하지 않았다.

## 남은 질문

- `ProblemDetail`(RFC 9457)로 에러 응답 모양을 표준화하면 어떻게 달라질까?
- 예외 타입이 여러 `@ExceptionHandler`에 걸리면 어떤 게 먼저일까?
- `@Async`나 `CompletableFuture`처럼 다른 스레드로 넘어간 작업에서 난 예외와 `ThreadLocal`은 어떻게 될까?

## 참고 자료

이번 공부는 발표("요청 하나, 스레드 하나", 2026-11-24)를 준비하며 Claude와 대화로 진행했다. 더 읽어볼 공식 문서:

- [Spring Framework - 예외 처리](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-servlet/exceptionhandlers.html)
- [Spring Boot - 에러 처리](https://docs.spring.io/spring-boot/reference/web/servlet.html#web.servlet.spring-mvc.error-handling)
- [Spring Security - Servlet 아키텍처](https://docs.spring.io/spring-security/reference/servlet/architecture.html)

## 정리 이력

- 2026-11-23: 발표 준비로 공부한 내용을 노트로 정리했다.
