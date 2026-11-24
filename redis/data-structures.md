# 자료구조는 언제 무엇을 쓸까?

[목차](README.md) · [이전: Redis는 왜 빠를까?](why-redis-fast.md)

## 다루는 범위

Redis의 기본 자료구조 다섯 개를 하나씩 보고, 어떤 요구사항에 어떤 자료구조를 고르는지 정리한다.

- String: `SET`의 옵션(`EX`, `NX`)과 타입 덮어쓰기
- Hash: 필드 단위 수정, `HINCRBY`, 만료(`EXPIRE`, `HEXPIRE`, `HSETEX`)
- List: 양 끝 O(1), 최근 N개, 내부 구조 quicklist
- Set: 중복 없는 모음, `SADD` 반환값, 집합 연산
- Sorted Set: 점수 순 정렬, 해시 테이블과 스킵 리스트
- 시간복잡도와 큰 키(big key) 문제, `SCAN` 계열

TTL이 지워지는 방식과 메모리가 찼을 때의 동작은 3번 주제, 명령 여러 개를 원자적으로 묶는 방법은 5번 주제에서 다룬다. 이 문서의 명령과 출력은 모두 설명용이다.

## 시작한 질문

1. Redis가 `Map<String, 값>`이라면, 값 자리에는 무엇이 들어갈 수 있을까?
2. 기능을 만들 때 어떤 자료구조를 골라야 할까? 그냥 JSON 문자열로 넣으면 안 될까?
3. 같은 데이터라도 명령에 따라 왜 빠르기도 하고 위험하기도 할까?

## 내가 이해한 내용

### 0. Redis 전체는 `Map<String, 자료구조>`다

키는 항상 문자열이고, 값 자리에 다섯 가지 자료구조 중 하나가 들어간다. 명령 앞 글자를 보면 어떤 타입용인지 알 수 있다.

| 접두어 | 타입 | 넣기 | Java로 비유 | 안에 들어가는 것 |
|---|---|---|---|---|
| 없음 | String | `SET` | `String` | value |
| `H` | Hash | `HSET` | `HashMap<String, String>` | field, value |
| `L` / `R` | List | `LPUSH` / `RPUSH` | 작은 배열을 이은 리스트 | element |
| `S` | Set | `SADD` | `HashSet<String>` | member |
| `Z` | Sorted Set | `ZADD` | `HashMap` + 정렬된 `TreeSet` | score, member |

```java
Map<String, Object> redis = {
    "product:1"   → "키보드",                          // String
    "user:42"     → {name: "윤", point: "1500"},       // Hash
    "recent:42"   → ["상품C", "상품B", "상품A"],         // List
    "like:post:7" → {"user42", "user10"},              // Set
    "rank:game"   → {영희:2300, 철수:2100}              // Sorted Set
};
```

다른 타입의 명령을 쓰면 `WRONGTYPE` 에러가 난다. 키의 타입은 `TYPE key`로 확인한다.

### 1. String: 값 하나, 그리고 `SET`의 옵션

- 최대 512MB, 바이너리도 그대로 저장한다. 캐시 값, 카운터, JSON 덩어리를 넣는다.
- `GET`, `SET`, `INCR`은 O(1)이고, 여러 키는 `MGET`/`MSET`으로 한 번에 처리한다.

`SET`에는 옵션이 있다.

| 옵션 | 뜻 |
|---|---|
| `EX 초` / `PX 밀리초` | 넣으면서 만료시간도 건다 |
| `NX` | 키가 없을 때만 넣는다 |
| `XX` | 키가 있을 때만 넣는다 |
| `KEEPTTL` | 기존 만료시간을 유지한다 |

```text
SET lock:order:1 "server-A" NX EX 10
```

예전에는 `SETNX` 후 `EXPIRE`를 따로 보냈다. 그 사이에 앱 서버가 죽으면 만료 없는 키가 영원히 남았다. Redis 2.6.12부터 `SET` 하나에 옵션을 넣어 이 틈을 없앴다.

**`SET`은 String 전용 명령이다.** 그래서 키가 어떤 타입이든 확인하지 않고 String으로 덮어쓴다. 옵션 없이 쓰면 기존 TTL도 지운다.

### 2. Hash: 객체를 필드 단위로

```text
HSET user:42 name "윤" grade "GOLD" point 1500
HGET user:42 point
HINCRBY user:42 point 100     ← 포인트만 원자적으로 +100
HGETALL user:42               ← 전부 조회, O(N)
```

**JSON String과 비교하면** JSON은 필드 하나를 바꿀 때도 `GET` → 앱에서 수정 → `SET`으로 통째로 다시 써야 한다. 왕복이 2번이고, 그 사이에 다른 요청이 끼면 한쪽 수정이 사라진다. Hash는 `HSET`, `HINCRBY`로 필드 하나를 명령 하나에 바꾼다.

**숫자는 문자열로 저장된다.** `HINCRBY`, `INCRBY`는 실행할 때마다 값을 64비트 정수로 해석해 본다.

- 해석되면 더하고, 안 되면 에러를 낸다.
- 키나 필드가 없으면 0으로 보고 시작한다.
- 소수는 `INCRBYFLOAT`, `HINCRBYFLOAT`를 쓴다.
- 작은 정수는 내부적으로 정수로 저장해 메모리를 아끼지만(`OBJECT ENCODING`이 `int`), 이건 최적화일 뿐 규칙은 같다.

**만료시간**

| 하고 싶은 것 | 방법 | 명령 수 |
|---|---|---|
| String 값과 키 만료 | `SET key value EX 600` | 1개 |
| Hash 필드와 필드 만료 | `HSETEX key EX 600 FIELDS 1 point 1500` (8.0+) | 1개 |
| 이미 있는 필드에 만료 | `HEXPIRE key 600 FIELDS 1 point` (7.4+) | 1개 |
| Hash 필드와 **키 전체** 만료 | `HSET` + `EXPIRE` | 2개 |

- `EX`는 `SET`의 옵션이고, `EXPIRE`는 이미 있는 키에 만료만 거는 별도 명령이다. `EXPIRE`는 모든 타입의 키에 쓸 수 있다.
- 단위는 짝이 맞는다. 초는 `EX` ↔ `EXPIRE`, 밀리초는 `PX` ↔ `PEXPIRE`다.
- Hash 키 전체 만료는 지금도 명령 2개라서, 사이에 앱 서버가 죽으면 만료 없는 키가 남을 수 있다. 묶는 방법(`MULTI/EXEC`, Lua)은 5번 주제에서 본다.
- Hash는 필드가 하나도 안 남으면 키가 사라진다.

### 3. List: 양 끝에서 넣고 빼는 순서 있는 목록

```text
LPUSH recent:42 "상품A"     ← 앞에 넣기 O(1)
LTRIM recent:42 0 9         ← 0~9번(10개)만 남기기
LRANGE recent:42 0 9        ← 최근 것부터 10개
RPOP queue:mail             ← 뒤에서 꺼내기 O(1)
```

- 정렬하지 않는다. **넣은 순서**를 유지한다. `LPUSH`로 앞에 넣으면 최신 것이 앞에 온다.
- 양 끝(`LPUSH`, `RPUSH`, `LPOP`, `RPOP`)은 O(1)이다.
- 중간 접근(`LINDEX`, `LSET`, `LPOS`, `LREM`)과 범위 조회(`LRANGE`)는 O(N)이다.
- 꺼내면서 지우는 건 양 끝에서만 된다. 원소에는 이름이 없고 위치(index)만 있다.

**쓰임**: 최근 본 상품(`LPUSH` + `LTRIM`), 최근 메시지 N개, 간단한 작업 큐(`LPUSH` → `RPOP`).

**내부 구조는 quicklist다.** 순수 연결 리스트도, 순수 배열도 아니다.

| | 문제 |
|---|---|
| 배열만 쓰면 | 앞에 넣을 때 전부 밀어야 하고(O(N)), 꽉 차면 통째로 복사한다. 싱글 스레드라 그동안 모두 멈춘다. |
| 연결 리스트만 쓰면 | 원소마다 포인터 두 개(16바이트)가 붙어 메모리가 아깝고, 노드가 흩어져 CPU 캐시를 못 쓴다. |

그래서 작은 배열(listpack) 여러 개를 연결 리스트로 잇는다. `LinkedList<String[]>`처럼 나눠 담은 리스트다. 원소가 적으면 listpack 하나로만 저장한다.

### 4. Set: 중복 없는 모음

```text
SADD like:post:7 "user42"
(integer) 1      ← 새로 들어감
SADD like:post:7 "user42"
(integer) 0      ← 이미 있음

SISMEMBER like:post:7 "user42"   ← 들어 있나? O(1)
SCARD like:post:7                ← 개수 O(1)
SREM like:post:7 "user42"        ← 빼기 O(1)
SMEMBERS like:post:7             ← 전부 O(N)
SPOP like:post:7                 ← 무작위로 하나 꺼내고 삭제
```

`like:post:7`은 "7번 글에 좋아요를 누른 사용자 모음"이다. 다른 사람이 누르면 같은 키에 멤버가 늘어나서 `Map<String, Set<String>>` 모양이 된다. 멤버 수가 곧 좋아요 수다.

**`SADD`의 반환값으로 판단한다.** 1이면 처음, 0이면 이미 있음이다. `SISMEMBER`로 확인한 뒤 `SADD`하면 명령이 2개로 나뉘어 사이에 같은 사용자의 요청이 끼어들 수 있다. 1장의 "`INCR` 결과로 판단하기"와 같은 방식이다.

**쓰임**: 좋아요·쿠폰 중복 방지, 오늘 방문자 수, 차단 사용자 목록 같은 포함 여부 확인.

**집합 연산**

```text
follow:A = {철수, 영희, 민수}
follow:B = {영희, 민수, 지훈}

SINTER follow:A follow:B   → {영희, 민수}               교집합 (공통 친구)
SUNION follow:A follow:B   → {철수, 영희, 민수, 지훈}     합집합
SDIFF  follow:A follow:B   → {철수}                     차집합 (A - B, 순서 중요)
SINTERSTORE common:A:B follow:A follow:B               결과를 새 키에 저장
```

- O(N)이라 큰 Set끼리 하면 싱글 스레드를 오래 붙잡는다.
- Cluster에서는 키가 다른 서버에 있으면 멀티 키 명령이 안 된다(9번 주제).
- 큰 데이터 비교는 배치로 미리 계산해 두거나, 복제본에서 돌리거나, 개수만 필요하면 `SINTERCARD ... LIMIT`(7.0+)을 쓰거나, DB 쿼리로 처리한다.
- 결국 집합 연산은 **데이터가 이미 Redis에 있고, 작고, 자주 빠르게 계산해야 할 때**만 의미가 있다.

### 5. Sorted Set: 점수 순으로 정렬된 Set

```text
ZADD rank:game 1500 "철수"
ZADD rank:game 2300 "영희"
ZINCRBY rank:game 600 "철수"            ← 점수 더하기
ZSCORE rank:game "철수"                 ← 내 점수 O(1)
ZREVRANK rank:game "철수"               ← 높은 점수 기준 등수 (0부터) O(log N)
ZRANGE rank:game 0 9 REV WITHSCORES     ← 상위 10명
ZRANGE jobs 0 1763900000 BYSCORE        ← 점수(시각) 범위 조회
```

- 멤버는 중복이 안 되고, 점수로 항상 정렬돼 있다. 점수가 같으면 멤버 이름의 사전순이다.
- 멤버를 key, 점수를 value로 보면 **value로 정렬**하는 셈이다.

**내부에 구조가 두 개 있다.**

```java
record Entry(double score, String member) {}
Map<String, Double> scoreOf;                       // 해시 테이블
TreeSet<Entry> sorted = new TreeSet<>(             // Redis는 스킵 리스트
    Comparator.comparingDouble(Entry::score).thenComparing(Entry::member));
```

| 명령 | 쓰는 구조 | 속도 |
|---|---|---|
| `ZSCORE` 내 점수 | 해시만 | O(1) |
| `ZRANGE` 상위 N명 | 스킵 리스트만 | O(log N + N) |
| `ZREVRANK` 내 등수 | 해시로 점수 확인 → 스킵 리스트에서 위치 계산 | O(log N) |
| `ZADD`, `ZINCRBY` | 둘 다 갱신 | O(log N) |

메모리를 조금 더 쓰는 대신 조회 종류마다 가장 빠른 구조를 쓴다. 원소가 적으면 listpack 하나로 압축해 저장한다.

**넣을 때 정렬된다.** `ZADD`는 명령 안에서 스킵 리스트의 맞는 자리를 찾아 끼워 넣고 응답한다. 백그라운드 작업은 없고, 전체를 다시 정렬하지도 않는다. 원소 100만 개여도 약 20단계라 µs 수준이다.

**DB `ORDER BY`와 비교**

| | DB | Sorted Set |
|---|---|---|
| 정렬 시점 | 조회할 때마다 | 넣을 때 제자리에 |
| 상위 10명 | 매번 정렬·인덱스 탐색 | 앞에서 10개 읽기 |
| 내 등수 | `COUNT(*) WHERE score > 내점수` → O(N) | `ZREVRANK` → O(log N) |
| 점수 올리기 | `UPDATE` + 인덱스 갱신 + 로그 | `ZINCRBY` 하나 |

**쓰임**: 랭킹, 인기 상품 TOP N, 점수에 타임스탬프를 넣은 시간순 목록과 예약 작업.

### 6. 자료구조 고르는 법

요구사항의 핵심 단어로 고른다.

| 요구사항에 이런 말이 있으면 | 자료구조 |
|---|---|
| 최근 N개, 넣은 순서대로, 큐 | List |
| 한 번만, 중복 없이, 포함돼 있나? | Set |
| 순위, TOP N, 점수, 시간 범위 | Sorted Set |
| 객체의 필드 하나만 바꾸기 | Hash |
| 단순한 값, 카운터, 통째로 읽고 쓰는 캐시 | String |

"사용자마다 다르다"는 키를 사용자별로 나누면 된다(`recent:42`, `recent:43`).

### 7. 시간복잡도와 큰 키

| 타입 | 빠른 명령 | 느린 명령 (O(N)) |
|---|---|---|
| String | `GET`, `SET`, `INCR` | — |
| Hash | `HGET`, `HSET`, `HINCRBY` | `HGETALL` |
| List | `LPUSH`, `RPOP` | `LRANGE` 전체, `LINDEX`, `LREM` |
| Set | `SADD`, `SISMEMBER`, `SCARD` | `SMEMBERS`, `SINTER` |
| Sorted Set | `ZADD`, `ZSCORE`, `ZREVRANK` | 큰 범위의 `ZRANGE` |

**큰 키**는 키 하나의 값이 너무 큰 것이다. 수십 MB짜리 String처럼 크기가 크거나, 필드 500만 개짜리 Hash처럼 원소가 많은 경우다.

```text
HGETALL user:points   ← 필드 500만 개, 요청은 1번
  ├─ 그 명령 하나를 처리하는 동안 싱글 스레드가 묶임 → 다른 모든 요청 대기
  ├─ 수백 MB 응답 → 네트워크와 앱 메모리 부담
  └─ DEL로 지워도 메모리 해제하는 동안 또 멈춤
```

**명령 하나가 오래 걸리면 그 뒤에 줄 선 모든 요청이 같이 느려진다.** 1장의 `KEYS *`와 같다. 그래서 명령을 고를 때 O(1)인지 O(N)인지, N이 얼마나 커질 수 있는지 본다.

**피하는 방법**

| 방법 | 예 |
|---|---|
| 키를 잘게 나누기 | `user:points` 하나 대신 사용자별 `user:42`의 `point` 필드 |
| 나눠서 가져오기 | `HGETALL` → `HSCAN`, `SMEMBERS` → `SSCAN`, `KEYS` → `SCAN` |
| 개수 제한하기 | `LTRIM 0 49`로 50개만 유지 |
| 지울 때 `UNLINK` | 메모리 해제를 백그라운드 스레드에서 한다 |
| 전체 목록은 DB에서 | 관리자 화면처럼 전체가 필요하면 DB에서 페이지 단위로 |

```text
SSCAN coupon:event:7 0 COUNT 500
1) "1832"                     ← 다음 커서
2) ["user1", "user9", ...]
SSCAN coupon:event:7 1832 COUNT 500   ← 커서가 0이 되면 끝
```

`SCAN` 계열은 내 작업 전체는 왕복이 늘어 더 오래 걸리지만, 명령 하나하나가 짧아서 그 사이사이에 다른 요청이 처리된다. 대신 같은 멤버가 두 번 나올 수 있고, 도중에 추가·삭제된 멤버는 나올 수도 안 나올 수도 있다. 처음부터 끝까지 있던 멤버는 반드시 한 번 이상 나온다.

**키가 많은 것은 큰 키 문제가 아니다.** 작은 키 500만 개는 각각 O(1)로 접근하므로 속도 문제는 없다. 다만 키마다 관리 정보가 붙어 메모리를 쓰고, `KEYS *`가 더 위험해진다. 메모리는 TTL과 `maxmemory`로 관리한다(3번 주제).

## 헷갈렸던 점

| 처음의 생각 | 정리한 내용 |
|---|---|
| `SET`은 Set 타입이다 | `SET`은 String에 값을 넣는 명령이다. Set 타입은 `SADD`로 넣는다. |
| `SET`은 모든 자료구조에 쓰는 "넣기" 명령이다 | String 전용이다. 다른 타입은 `HSET`, `LPUSH`, `SADD`, `ZADD`처럼 접두어가 붙는다. |
| `NX`와 `EX`를 한 명령에 넣은 이유 | 맞게 이해했다. 나눠 보내면 사이에 서버가 죽거나 다른 명령이 끼어드는 틈이 생긴다. |
| `INCRBY` 대상이 숫자가 아니면? 1500은 숫자로 저장되나? | 문자열로 저장된다. 실행할 때마다 64비트 정수로 해석해 보고, 안 되면 에러다. |
| 숫자인지 아닌지 Redis가 어떻게 아나 | 저장할 때 타입을 기억하는 게 아니라, 숫자 명령을 실행할 때 해석해 본다. |
| Hash도 만료시간을 정할 수 있나 | 키 전체는 `EXPIRE`, 필드는 `HEXPIRE`(7.4+)나 `HSETEX`(8.0+)로 건다. |
| Hash는 `EX` 대신 왜 `EXPIRE`를 쓰나 | `EX`는 `SET`의 옵션이고, `EXPIRE`는 이미 있는 키에 거는 별도 명령이다. `HSET`에는 키 만료 옵션이 없다. |
| `HSETEX`가 있으니 Hash도 한 번에 된다 | 필드 만료는 된다. 키 전체 만료는 여전히 `HSET` + `EXPIRE` 2개다. |
| List는 `LinkedList`다 | 동작은 비슷하지만 실제는 작은 배열(listpack)을 연결한 quicklist다. |
| List는 양 끝에서만 꺼낼 수 있나 | 꺼내며 지우는 건 양 끝만 된다. 조회·수정·삭제는 index나 값으로 중간도 되지만 O(N)이다. |
| `SADD key v`의 `v`는 키 이름 같다 | `key`는 Set의 이름, `member`는 그 안에 들어가는 값이다. (설명할 때 `v`로 줄여 써서 헷갈렸다) |
| 좋아요를 여러 명이 누르면? | 같은 키에 멤버가 늘어난다. `Map<String, Set<String>>`이고, `SCARD`가 좋아요 수다. |
| 집합 연산은 큰 데이터 비교에 쓴다 | O(N)이라 싱글 스레드를 붙잡는다. 큰 데이터는 배치, 복제본, DB에서 한다. |
| Sorted Set의 `TreeSet`은 key 정렬이다 | `TreeSet`은 원소 자체를 정렬한다. Sorted Set은 (점수, 멤버)를 점수로, 같으면 멤버로 정렬한다. |
| Sorted Set은 넣고 나서 백그라운드로 정렬한다 | 명령 안에서 맞는 자리에 끼워 넣고 응답한다. O(log N)이라 기다리는 시간은 µs 수준이다. |
| 해시로 찾고 순위로 이어서 찾나 | 명령마다 필요한 구조만 쓴다. 등수 조회만 해시 → 스킵 리스트로 이어서 쓴다. |
| 큰 키는 데이터가 큰 것이다 | 맞다. 크기가 크거나 원소가 많은 키 하나다. 키 개수가 많은 것은 다른 문제(메모리)다. |
| `SCAN`으로 나누면? | 내 작업은 느려져도 다른 요청을 막지 않는다. |

### 확인 문제와 내 답

| 문제 | 내 답의 요지 | 결과 |
|---|---|---|
| 1. 기능 4개에 자료구조 고르기 | ① 최근 본 상품: JSON String ② 좋아요 1회: 모르겠다 ③ 인기 TOP 10: Sorted Set ④ 프로필·포인트: Hash | 부분 통과. ①은 List(`LPUSH` + `LTRIM`), ②는 Set(`SADD` 반환값). ③은 Set에 넣고 따로 정렬하는 게 아니라 Sorted Set 하나가 둘 다 한다. |
| 2. Hash 키에 `SET`을 보내면 | ① 에러 없이 덮어쓰고 데이터는 크게 안 바뀐다 ② 만료 설정이 사라진다 ③ `HGET`은 잘 된다 | 부분 통과. 타입이 String으로 바뀌고, TTL이 -1이 되어 영원히 남고, `HGET`은 `WRONGTYPE` 에러다. |
| 3. 필드 500만 개 Hash에 `HGETALL` | ① 500만 번 요청을 보내 처음과 끝의 간격이 크다 ② 사용자별 키로 나눈다 | 부분 통과. 요청은 1번이고, 그 명령 하나가 오래 걸려 다른 모든 요청이 기다린다. ②는 맞다. |
| 4. 실시간 랭킹을 DB 대신 Sorted Set으로 | ① Redis 통신이 단순하고 빠르다 ② `TreeSet` 규칙을 따른다 | 부분 통과. 핵심은 넣을 때 이미 정렬돼 있다는 것과 등수 조회가 O(log N)이라는 것. O(1) 점수 조회는 해시 테이블 덕분이다. |
| 5. 재시험: 쿠폰 1회 + 수 / 최근 메시지 50개 / 300만 명 `SMEMBERS` | ① Set, `SCARD` ② List, `LRANGE` ③ 전부 조회해 오래 걸리고 뒤 요청이 밀린다 | 통과. `SADD` 반환값으로 판단하기, 50개 유지에는 `LTRIM 0 49`, 해결은 `SSCAN`이나 DB 조회를 보충했다. |

## 예제와 확인 결과

- 이 문서의 명령, 출력, 커서 값, 타임스탬프는 모두 설명용이다. 실제로 실행한 결과는 아니다.
- 실습할 때는 아래처럼 띄우고, `INFO server`의 `redis_version`과 함께 기록한다. `HSETEX`는 8.0 이상, `HEXPIRE`는 7.4 이상에서만 된다.

```bash
docker run -d --name redis -p 6379:6379 redis:8
docker exec -it redis redis-cli
```

## 남은 질문

- 키 전체 만료가 필요한 `HSET` + `EXPIRE`를 `MULTI/EXEC`나 Lua로 어떻게 묶을까? (5번 주제)
- TTL이 지난 키는 정확히 언제 지워질까? 메모리가 차면 어떤 키부터 지울까? (3번 주제)
- 좋아요처럼 DB가 원본인 데이터를 Redis Set과 어떻게 맞출까? (4번 주제)
- `BLPOP` 같은 블로킹 명령으로 List를 큐로 쓰면 Kafka나 Streams와 무엇이 다를까? (6번 주제)
- 실제 Redis에서 `OBJECT ENCODING`으로 listpack이 quicklist/hashtable/skiplist로 바뀌는 순간을 확인해 보기
- 운영 중인 Redis에서 큰 키를 어떻게 찾을까? (`redis-cli --bigkeys`, `MEMORY USAGE`)

## 참고 자료

- [Redis — Data types](https://redis.io/docs/latest/develop/data-types/)
- [Redis — Commands](https://redis.io/docs/latest/commands/)
- 이전 노트: [Redis는 왜 빠를까?](why-redis-fast.md) — 싱글 스레드, `KEYS *`, `INCR` 결과로 판단하기

## 정리 이력

- 2026-11-24: Claude와의 학습 대화와 확인 문제를 바탕으로 정리했다. 예시는 설명용으로 작성했다.
