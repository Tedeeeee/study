# 필터는 무엇을 걸러내고, Spring은 어디서부터일까?

[목차](README.md) · [이전: 요청은 어떻게 스레드 하나에게 갈까?](request-to-thread.md) · [다음: DispatcherServlet은 요청을 어떻게 내 메서드까지 보낼까?](dispatcher-servlet.md)

## 다루는 범위

작업 스레드가 요청을 들고 Spring MVC의 정문(DispatcherServlet)에 닿기 전에 지나는 필터를 정리한다. [전체 흐름](README.md#전체-흐름)의 2단계다.

- 필터는 Spring이 아니라 서블릿 표준이고, 톰캣이 돌린다
- `doFilter` 하나 안에 "들어갈 때"와 "나올 때"가 다 있는 이유
- 필터가 요청을 걸러내는 방법: `chain.doFilter()`를 부르지 않기
- Spring Security가 필터 자리에 끼어드는 방식
- `@RestControllerAdvice`가 닿는 경계선
- Nest의 미들웨어와 비교

필터에서 난 예외를 실제로 처리하는 방법은 [5장](response-and-exception.md#4-필터-예외를-처리하는-네-가지)에서 다룬다. 이 문서의 코드는 설명용이다.

## 시작한 질문

1. 요청이 컨트롤러에 닿기 전에 로그인 확인, 인코딩, 요청 제한 같은 일은 어디서 할까?
2. 필터에서 난 예외는 왜 `@RestControllerAdvice`가 잡지 못한다고 할까?

## 내가 이해한 내용

### 1. 필터는 서블릿 표준이고, 톰캣이 DispatcherServlet 앞에서 돌린다

- 톰캣은 서블릿 컨테이너다. Spring MVC의 정문인 `DispatcherServlet`도 서블릿 하나다.
- 필터(`jakarta.servlet.Filter`)는 그 서블릿 앞에 줄 세워 두는 장치다. 톰캣이 `FilterChain`으로 하나씩 부르고, 줄 끝에서 `DispatcherServlet`을 부른다.
- 그래서 이 단계에서는 요청이 어떤 컨트롤러로 갈지 아직 모른다. 그걸 찾는 건 DispatcherServlet 안의 HandlerMapping이다([3장](dispatcher-servlet.md)).

```
요청 → 인코딩 필터 → 보안 필터(Spring Security) → 요청 제한 필터 ┊ DispatcherServlet
                                                         ┊ ← 여기부터 Spring MVC
```

### 2. `doFilter` 하나에 "들어갈 때"와 "나올 때"가 다 있다

```java
public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) {
    // 1. 들어갈 때: 검사, 준비
    chain.doFilter(req, res);   // 2. 다음 필터로 (줄 끝은 DispatcherServlet)
    // 3. 나올 때: 응답 로그 등
}
```

`chain.doFilter()`는 다음 필터를 부르는 평범한 메서드 호출이다. 다음 필터가 또 다음을 부르고, 끝에서 DispatcherServlet이 내 컨트롤러까지 다 실행한 뒤에야 차례로 돌아온다. 그래서 그 아래 3번 코드는 **응답이 만들어져서 나오는 길에** 실행된다. 안으로 들어갔다가 같은 겹을 거꾸로 나오는 양파 모양이다. 이 모양이 응답이 거꾸로 나가는 이유이기도 하다([5장](response-and-exception.md#2-왜-거꾸로일까-전부-한-스레드의-메서드-호출이다)).

### 3. 걸러내기: `chain.doFilter()`를 부르지 않으면 거기서 끝난다

필터라는 이름 그대로, 핵심은 요청을 걸러내는 것이다.

```java
public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) {
    if (토큰이 없으면) {
        ((HttpServletResponse) res).setStatus(401);
        return;                 // 다음으로 넘기지 않는다 → 여기서 끝
    }
    chain.doFilter(req, res);   // 통과 → 다음으로
}
```

| 요청 | 결과 |
| --- | --- |
| `GET /api/health` | 모든 필터를 통과해서 DispatcherServlet에 도착한다 |
| 로그인 토큰이 없는 요청 | 보안 필터에서 막혀 401을 받는다 |
| 1초에 100번씩 보내는 요청 | 요청 제한 필터에서 막혀 429를 받는다 |

걸러진 요청은 DispatcherServlet에 닿지도 않는다. 컨트롤러는 실행조차 되지 않는다.

### 4. Spring Security도 필터 하나다

- 톰캣 눈에는 Spring Security가 필터 하나(`DelegatingFilterProxy`)로만 보인다.
- `DelegatingFilterProxy`는 서블릿 필터와 Spring 빈을 잇는 다리다. 실제 일은 Spring 빈인 `FilterChainProxy`에게 넘긴다.
- `FilterChainProxy` 안에서 인증, 권한 확인, 예외 변환 같은 Security 전용 필터 여러 개가 다시 차례로 돈다.
- 로그인 안 된 요청에 401, 권한 없는 요청에 403을 돌려주는 곳이 여기다.

### 5. 경계선: `@RestControllerAdvice`는 DispatcherServlet부터다

예외를 한곳에 모아 에러 JSON을 만드는 `@RestControllerAdvice`는 DispatcherServlet 안의 `try-catch`에서 출발한다. 필터는 그 바깥에 있다.

- DispatcherServlet 안(인터셉터, 컨트롤러, 서비스)에서 난 예외 → DispatcherServlet이 잡아서 `@RestControllerAdvice`로 넘긴다.
- 필터에서 난 예외 → 그냥 두면 거기까지 가지 않고 톰캣으로 올라가서 Spring Boot 기본 에러 응답(`/error`)이 나간다.

그래서 "필터 예외는 못 잡는다"보다 **"경계선이 DispatcherServlet 앞에 있다"**고 이해하는 게 정확하다. 바깥에서 난 예외도 필터가 직접 넘겨주면 `@ExceptionHandler`로 보낼 수 있다([5장](response-and-exception.md#4-필터-예외를-처리하는-네-가지)).

### 6. Nest와 비교

| | Spring | Nest |
| --- | --- | --- |
| 이 자리의 장치 | 서블릿 필터 | 미들웨어 (`consumer.apply()`) |
| 다음으로 넘기기 | `chain.doFilter(req, res)` | `next()` |
| 인증 | 보안 필터(Spring Security) | 보통 Guard가 따로 맡는다 |
| 여기서 난 예외 | 그냥 두면 `@RestControllerAdvice`까지 안 간다 | `consumer.apply()`로 등록한 미들웨어 예외는 전역 Exception Filter가 받는다. `app.use()`로 Express에 직접 붙인 것은 Nest 바깥이라 Express 기본 에러 처리로 간다 |

Express의 `next()` 아래 코드는 "응답이 끝난 뒤"가 아니라 다음 미들웨어가 돌아온 뒤에 실행되고, 핸들러가 비동기면 응답 전일 수 있다. 응답 뒤에 할 일은 `res.on('finish')`나 Nest Interceptor로 한다.

## 헷갈렸던 점

| 처음의 생각 | 정리한 내용 |
| --- | --- |
| 필터는 Spring의 기능이다 | 서블릿 표준이고 톰캣이 돌린다. Spring Security는 그 자리에 Spring 빈을 끼워 넣은 것이다 |
| 톰캣이 Spring 바깥에서 필터로 관리하는 건 알겠는데, 무엇을 거르는지 와닿지 않는다 | 핵심은 막기다. `chain.doFilter()`를 부르지 않고 응답을 쓰면 요청이 거기서 끝난다(401, 429) |
| 필터에서 난 예외는 절대 처리할 수 없다 | 그냥 두면 `@RestControllerAdvice`까지 안 간다는 뜻이다. 경계선이 DispatcherServlet 앞에 있을 뿐이고, 필터가 직접 넘기면 된다 |
| Spring Security는 필터 여러 개로 톰캣에 등록된다 | 톰캣에는 `DelegatingFilterProxy` 하나만 보인다. 여러 개는 그 안의 `FilterChainProxy`에서 돈다 |

### 공부하며 던진 질문

- **서블릿이 뭐야?** 요청을 받아 응답을 만드는 자바 표준 인터페이스다. `DispatcherServlet`이 Spring MVC가 만든 서블릿이다.
- **필터 예외를 억지로 잡는 방법이 있던데, 그럼 "못 잡는다"는 틀린 말이야?** 둘 다 맞다. Spring이 알아서 잡아주지는 않고, 필터가 잡아서 넘겨주는 다리를 우리가 놓는 것이다.

## 예제와 확인 결과

위 코드는 모두 설명용이다. 직접 실행하지 않았다.

## 남은 질문

- `OncePerRequestFilter`는 왜 필요할까? forward나 에러 디스패치로 같은 요청이 필터를 두 번 탈 때를 확인해 보기.
- Security 필터 체인의 순서는 어떻게 정해지고, 커스텀 JWT 필터는 어디에 넣어야 할까?

## 참고 자료

이번 공부는 발표("요청 하나, 스레드 하나", 2026-11-24)를 준비하며 Claude와 대화로 진행했다. 더 읽어볼 공식 문서:

- [Spring Security - Servlet 아키텍처](https://docs.spring.io/spring-security/reference/servlet/architecture.html)
- [Spring Boot - 서블릿 웹 애플리케이션](https://docs.spring.io/spring-boot/reference/web/servlet.html)

## 정리 이력

- 2026-11-23: 발표 준비로 공부한 내용을 노트로 정리했다.
