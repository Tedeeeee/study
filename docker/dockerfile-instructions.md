# Dockerfile 한 줄은 무엇이 될까?

[목차](README.md) · [이전: 이미지와 컨테이너는 뭐가 다를까?](image-and-container.md)

## 다루는 범위

Dockerfile의 명령이 레이어를 만드는 것과 설정을 적는 것으로 나뉘는 기준, `RUN`이 레이어가 되는 과정, 빌드 캐시와 줄의 순서, `CMD`와 `ENTRYPOINT`의 차이, exec 형식과 shell 형식에 따라 PID 1이 달라지는 이유를 정리한다.

이미지를 작게 만드는 방법은 6번, 종료 신호와 정상 종료의 자세한 과정은 9번, Docker Compose는 14번에서 다룬다.

## 시작한 질문

Dockerfile의 명령은 각각 이미지의 무엇이 될까? 같은 내용인데 왜 줄의 순서에 따라 빌드 시간이 달라질까?

## 내가 이해한 내용

### 1. 명령은 "레이어를 만드는 것"과 "설정을 적는 것"으로 나뉜다

이미지는 레이어들 + 실행 설정이다. 기준은 **파일을 바꾸는가**다.

```dockerfile
FROM eclipse-temurin:21-jre                      # 레이어 (베이스 이미지의 것을 가져옴)
WORKDIR /app                                     # 설정
RUN apt-get update && apt-get install -y curl    # 레이어
COPY build/libs/app.jar app.jar                  # 레이어
ENV SPRING_PROFILES_ACTIVE=prod                  # 설정
EXPOSE 8080                                      # 설정
CMD ["java", "-jar", "app.jar"]                  # 설정
```

| | 명령 | 하는 일 |
| --- | --- | --- |
| 레이어를 만든다 | `FROM` | 베이스 이미지의 레이어들을 바닥에 깐다 |
| | `RUN` | 명령을 실행하고, 그 결과로 바뀐 파일들을 레이어로 남긴다 |
| | `COPY` | 내 프로젝트의 파일을 복사해 넣는다 |
| 설정을 적는다 | `WORKDIR` | 이후 명령과 컨테이너가 기준으로 삼을 폴더 |
| | `ENV` | 환경 변수 |
| | `EXPOSE` | 이 앱이 쓰는 포트 표시. 실제로 포트를 여는 건 아니다 |
| | `CMD` | 컨테이너의 시작 명령 |

위 Dockerfile을 풀어 쓰면 이렇다.

> `eclipse-temurin:21-jre`를 바닥에 깔고, 기준 폴더를 `/app`으로 정하고, `curl`을 설치하고, jar를 복사해 넣고, 환경 변수와 포트를 적고, 시작 명령을 적어 둔다. 여기까지가 이미지 만들기다. 나중에 컨테이너를 띄울 때 그 시작 명령이 실행된다.

```text
[docker build]  위에서 아래로 실행. CMD는 적기만 하고 실행하지 않는다 ⇒ 이미지 완성
[docker run]    이미지에 적힌 CMD를 실행한다 ⇒ java -jar app.jar 가 뜬다
```

### 2. `WORKDIR`은 컨테이너 안의 "일하는 폴더"다

`/app`은 서버도, 이미지가 저장된 위치도 아니다. **컨테이너 안에서 보이는 폴더 경로**다. `WORKDIR /app`은 `cd /app`과 같고, 폴더가 없으면 만든다.

```dockerfile
WORKDIR /app                      # 일하는 폴더는 /app
COPY build/libs/app.jar app.jar   # 거기에 jar를 넣고 → /app/app.jar
CMD ["java", "-jar", "app.jar"]   # 거기서 돌린다
```

- 빌드할 때는 그 아래 `COPY`, `RUN`의 기준 폴더가 되고, 띄울 때는 앱이 시작되는 폴더가 된다.
- 경로를 전부 절대 경로로 적으면 `WORKDIR` 없이도 된다. 반복을 줄이려고 위에 한 번 적는다.

### 3. `RUN`은 임시 컨테이너에서 실행되고, 결과만 레이어로 남는다

| 순서 | 하는 일 |
| --- | --- |
| 1 | 지금까지 쌓인 레이어로 임시 컨테이너를 띄운다 |
| 2 | 그 안에서 명령을 실행한다. 바뀐 파일은 쓰기 층에 쌓인다 |
| 3 | 그 쓰기 층을 굳혀서 새 레이어로 저장한다 |
| 4 | 임시 컨테이너는 버린다 |

4번 주제의 쓰기 층과 같은 원리다. 그래서 레이어가 "바로 아래에서 달라진 파일들만" 담는다. 설치를 빌드 때 한 번 해서 굳혀 두니, 컨테이너를 띄울 때마다 설치하지 않는다.

`apt-get update`는 서버를 업데이트하는 게 아니라 설치 가능한 패키지 목록을 받아 오는 것이고, `install -y curl`이 설치다.

| | Dockerfile의 `RUN` | `docker run` 명령 |
| --- | --- | --- |
| 언제 | `docker build` 중 | 이미지가 만들어진 뒤 |
| 띄우는 것 | 임시 컨테이너 (끝나면 버림) | 진짜 컨테이너 |
| 목적 | 이미지에 넣을 준비 작업 | 앱 실행 |
| 남는 것 | 레이어 | 돌고 있는 컨테이너 |

| | `RUN` | `CMD` |
| --- | --- | --- |
| 언제 실행되나 | 이미지를 만들 때 | 컨테이너를 띄울 때 |
| 용도 | 설치, 준비 작업 | 앱 실행 |

`RUN java -jar app.jar`라고 쓰면 빌드 중에 앱이 실행돼서 빌드가 끝나지 않는다.

### 4. 순서는 결과와 빌드 속도 둘 다에 영향을 준다

Dockerfile은 위에서 아래로 실행되고, 레이어 명령과 설정 명령을 섞어서 필요한 순서대로 쓴다.

| 설정 명령의 종류 | 예 | 위치 |
| --- | --- | --- |
| 빌드 중의 아랫줄에도 영향을 준다 | `WORKDIR`, 빌드 중에 쓰는 `ENV` | 영향을 줄 줄보다 위 |
| 컨테이너를 띄울 때만 쓰인다 | `EXPOSE`, `CMD`, 앱이 읽는 `ENV` | 어디든 된다. 보통 아래 |

`FROM`은 바닥을 까는 명령이라 맨 위일 수밖에 없다.

빌더는 적힌 대로만 실행하고, 아랫줄을 보고 위로 돌아가 고쳐 주지 않는다. `WORKDIR`을 `COPY` 아래에 쓰면 jar는 `/app.jar`에 들어가고 앱은 `/app/app.jar`를 찾는다. 이때 **빌드는 성공하고, 띄울 때 실패한다**(`Unable to access jarfile app.jar`).

| 종류 | 언제 드러나나 |
| --- | --- |
| 명령 자체가 안 되는 것 (복사할 파일이 없다) | 빌드가 실패한다 |
| 명령은 되는데 앞뒤가 안 맞는 것 (경로가 어긋났다) | 빌드는 성공하고 띄울 때 실패한다 |

### 5. 빌드 캐시: 안 바뀐 줄은 다시 실행하지 않는다

| 규칙 | 내용 |
| --- | --- |
| ① 지난번과 같으면 다시 쓴다 | 명령과 재료가 같으면 지난번 레이어를 그대로 쓴다. `RUN`은 명령 문자열, `COPY`는 파일의 내용으로 판단한다 |
| ② 한 줄이 달라지면 그 아래는 전부 다시 한다 | 레이어 명령뿐 아니라 설정 명령(`WORKDIR` 등)이 달라져도 마찬가지다. 아랫줄이 그 값에 기대기 때문이다 |

```dockerfile
# 나쁜 순서                                        # 좋은 순서
FROM eclipse-temurin:21-jre                       FROM eclipse-temurin:21-jre
COPY build/libs/app.jar app.jar    # 매번 바뀜     RUN apt-get update && apt-get install -y curl   # 캐시 사용
RUN apt-get update && ...          # 매번 다시     COPY build/libs/app.jar app.jar                 # 이것만 다시
```

> 잘 안 바뀌는 것은 위에, 자주 바뀌는 것은 아래에 쓴다.

위쪽에 공통되고 안 바뀌는 것을 두면 이득이 두 가지다. 같은 앱을 다시 빌드할 때 캐시를 써서 빠르고, 여러 앱 사이에서 같은 레이어를 한 벌만 두고 같이 쓴다. 캐시된 레이어는 레지스트리에 올릴 때도 건너뛴다.

### 6. 빌드와 실행은 별개의 명령이다

`docker run`에는 빌드가 들어 있지 않다. Dockerfile을 보지 않고, 이미지가 없으면 레지스트리에서 받아 올 뿐이다.

```text
./gradlew bootJar               # jar를 만든다
docker build -t my-app .        # 그 jar를 넣어 이미지를 만든다 (-t는 이름, .은 재료 폴더)
docker run my-app               # 띄운다
```

- Dockerfile을 고친 뒤 `docker build`를 다시 하지 않으면 예전 이미지가 그대로 뜬다.
- Dockerfile이 있는 폴더에서 명령 하나로 빌드와 실행이 됐던 건 `docker compose up`이다. Compose 파일에 `build:`가 있으면 이미지가 없을 때 빌드부터 한다.
- Compose는 길어진 `docker run` 옵션과 여러 컨테이너를 띄우는 순서를 파일 하나에 적어 두려고 나왔다. 환경 설치를 Dockerfile로 굳힌 것과 같은 발상이다.

### 7. `CMD`는 통째로 바뀌고, `ENTRYPOINT`는 뒤에 붙는다

| | `CMD` | `ENTRYPOINT` |
| --- | --- | --- |
| 성격 | 기본값 | 고정 |
| `docker run 이미지` 뒤에 적은 것은 | `CMD`를 대신한다 | `ENTRYPOINT` 뒤에 인자로 붙는다 |
| 어울리는 경우 | 범용 이미지, 다른 명령으로도 띄울 때 | 한 가지 앱 전용 이미지 |

| Dockerfile | `docker run my-app --server.port=9090` |
| --- | --- |
| `CMD ["java", "-jar", "app.jar"]` | `--server.port=9090`을 프로그램으로 실행하려다 실패한다 |
| `ENTRYPOINT ["java", "-jar", "app.jar"]` | `java -jar app.jar --server.port=9090` |

- 둘을 같이 쓰면 `ENTRYPOINT`가 고정된 명령, `CMD`가 기본 인자가 된다.
- `ENTRYPOINT`도 `--entrypoint` 옵션을 일부러 적으면 바꿀 수 있다. 실수로는 안 바뀌고 작정하면 바뀐다.
- 운영에서는 띄우는 명령이 Compose나 Kubernetes 파일에 고정돼 있어서 어느 쪽이든 큰 차이가 없다. 그래도 앱 전용 이미지는 `ENTRYPOINT`가 무난하다. 실제로는 둘 다 흔하다.

| 테스트하면서 바꾸는 것 | 편한 쪽 |
| --- | --- |
| 같은 앱에 인자만 바꿔 넣는다 | `ENTRYPOINT` |
| 앱 대신 다른 걸 실행해 본다 (`docker run -it my-app sh`) | `CMD` |

### 8. 대괄호로 써야 내 앱이 PID 1이 된다

| 이름 | 적는 법 | 실제로 실행되는 것 |
| --- | --- | --- |
| exec 형식 | `CMD ["java", "-jar", "app.jar"]` | 도커가 `java`를 바로 실행한다 |
| shell 형식 | `CMD java -jar app.jar` | 도커가 `sh`를 실행하고, `sh`가 `java`를 실행한다 |

```text
[exec 형식]    PID 1  java -jar app.jar
[shell 형식]   PID 1  /bin/sh -c "java -jar app.jar"
               PID 7  java -jar app.jar
```

`docker stop`은 PID 1에게만 종료 신호를 보낸다.

| | exec 형식 | shell 형식 |
| --- | --- | --- |
| 신호를 받는 쪽 | `java` | `sh` |
| 그 뒤 | JVM이 셧다운 훅을 실행한다. Spring Boot가 처리 중이던 요청을 마치고 연결을 닫은 뒤 종료한다 | `sh`가 `java`에게 전달하지 않는다. 10초 뒤 도커가 강제로 죽인다 |

| 신호 | 셧다운 훅 |
| --- | --- |
| 종료 요청 (SIGTERM) | 실행된다 |
| 강제 종료 (SIGKILL) | 실행되지 않는다. 커널이 그 자리에서 프로세스를 없앤다 |

- 정리하고 끄는 건 데몬 스레드가 아니라 **셧다운 훅**이다. 데몬 스레드는 오히려 JVM이 종료할 때 기다려 주지 않는 스레드다.
- exec 형식은 셸을 거치지 않아 `$JAVA_OPTS` 같은 변수를 풀어 주지 않는다. 변수가 필요하면 `ENTRYPOINT exec java $JAVA_OPTS -jar app.jar`처럼 `exec`로 셸이 자기 자리를 `java`에게 넘기게 한다.
- `RUN`은 빌드 중에 잠깐 실행되고 끝나서 PID 1을 신경 쓸 일이 없다. `&&`를 쓰려면 shell 형식이 편하다.

## 헷갈렸던 점

| 처음 생각 | 고친 생각 |
| --- | --- |
| 레이어 명령을 먼저 적고 설정을 나중에 적는다 | 섞어서 필요한 순서대로 쓴다. 아랫줄에 영향을 주는 설정은 그 위에 있어야 한다. |
| `WORKDIR /app`은 우리의 업무 서버 `/app`에 설정하는 것이다 | `/app`은 서버가 아니라 컨테이너 안의 폴더다. 앱이 그 폴더를 기준으로 돌게 한다는 뜻이다. |
| `RUN apt-get update`는 실행하기 전에 서버를 업데이트하는 것이다 | 컨테이너를 띄울 때가 아니라 이미지를 만들 때 돈다. 패키지 목록을 받고 `curl`을 설치해 레이어로 굳힌다. |
| Dockerfile을 읽으며 마지막에 `CMD`가 실행된다 | 빌드 때는 적히기만 하고, `docker run` 때 실행된다. |
| "이미지 안의 폴더"는 이미지가 저장된 위치를 말한다 | 이미지에 든 파일 구조 안의 경로, 즉 컨테이너에서 보이는 경로다. 도커 저장 공간의 위치와는 상관없다. |
| `FROM`을 다른 앱과 같이 쓰려고 맨 위에 올린다 | `FROM`이 맨 위인 건 바닥이라서 그럴 수밖에 없다. 다만 공통된 것을 위에 두면 레이어를 같이 쓴다는 생각은 `FROM` 아래 줄들에 그대로 맞다. |
| 설정은 전부 밑에 모아 쓰면 되는데 왜 `WORKDIR`만 가운데에 끼나 | `WORKDIR`은 빌드 중의 아랫줄(`COPY`, `RUN`)에도 영향을 준다. `EXPOSE`, `CMD`처럼 띄울 때만 쓰이는 설정은 실제로 아래에 모아 쓴다. |
| `WORKDIR`이 밑에서 바뀌면 빌드를 실패시킨다 | 빌드는 성공한다. 빌더는 뜻을 헤아리지 않고 적힌 대로만 한다. 띄울 때 앱이 jar를 못 찾아 죽는다. |
| `docker run`에 빌드 과정이 들어 있다 | 없다. Dockerfile이 있는 폴더에서 한 번에 됐던 건 `docker compose up`이었을 가능성이 높다. |
| 예민한 앱이 아니면 대부분 `ENTRYPOINT`를 쓴다 | 고정돼야 하는 정도가 클수록 `ENTRYPOINT`가 어울린다. 실제로는 둘 다 흔하다. |
| 개발이나 테스트 중에는 `ENTRYPOINT`가 좋다 | 인자만 바꾸면 `ENTRYPOINT`, 앱 대신 다른 걸 실행해 보려면 `CMD`가 편하다. |
| 자바를 끌 때 데몬 스레드가 나머지를 정리한다 | 셧다운 훅이다. 강제 종료되면 셧다운 훅이 실행되지 않는다는 이해는 맞다. |

### 확인 문제

**문제 1.** Dockerfile이 `FROM` → `WORKDIR` → `COPY app.jar` → `RUN apt-get ... curl` → `CMD` 순서다. 코드를 한 줄만 고쳐도 `curl` 설치가 매번 다시 돈다. ① 왜 그런가 ② 어떻게 고치나 ③ 고친 뒤에는 어느 줄이 다시 실행되고 어느 줄을 건너뛰나

내 답:

> 1. dockerfile 은 새로 변경된 내용을 기준으로 하위의 작업을 모두 새로 작업하는데 이때 수정된 코드가 있는 app.jar 가 copy 에 있게 되면서 하위의 run 을 무조건 작업을 하게 되고 update 나 intall 은 오래 걸리면 몇 분 씩 걸리는 작업이기 때문에 수정하는 것이 좋을거 가아
> 2. from -> workdir -> run -> copy 방식으로 변경된 코드를 하위로 내리면 훨씬 빠르다
> 3. copy 줄은 실행되겠지만 오래 걸리는 run 이 위로 올라가면서 생략될 것이다

채점: 통과. 세 가지 모두 맞다. 건너뛰는 것을 "캐시를 쓴다"고 하고, 빌드 화면에 `CACHED`로 나온다는 점을 보충했다. `FROM`, `WORKDIR`, `RUN`이 캐시를 쓰고 `COPY`부터 다시 실행된다.

**문제 2.** 마지막 줄이 `CMD java -jar app.jar`다. ① `docker stop`을 하면 10초쯤 멈췄다 꺼지고 처리 중이던 요청이 끊긴다. 왜 그런가 ② `docker run my-app --server.port=9090`을 치면 앱이 안 뜬다. 왜 그런가 ③ 어떻게 고치나

내 답:

> 1. 이건 단순 cmd 를 exec 가 아니라 sh 방식으로 진행을 해서 그런거야 이렇게 되면 처음에 jar 를 실행하는 명령이 중간에 shell 에게 부탁하는 과정이 생기면서 문제가 생긴다 그래서 살짝 오래 걸릴 수 있다
> 2. 기존의 cmd 로 dockerfile 을 설정하면 그 내용은 사용자가 docker run <추가내용> 을 입력하게 되면 기존의 cmd 명령어는 모두 생략된다.
> 3. 그래서 기존의 cmd 대신 entrypoint 라는 명령어를 사용하고 그냥 명령어를 치는거보다 [ ] 를 이용해서 명령어를 실행하면 훨씬 빠르게 문제 없이 실행될 것이댜

채점: 통과. 2번과 3번은 맞다. 1번은 shell 형식이라 셸이 중간에 낀다는 원인은 맞게 짚었지만, 그래서 **PID 1이 `sh`가 되고, `docker stop`의 종료 신호를 `sh`가 받아 `java`에게 전달하지 않으며, 10초 뒤 강제로 죽는다**는 연결이 빠졌다. "오래 걸린다"와 3번의 "훨씬 빠르게"는 속도 문제가 아니라, 앱이 종료 신호를 직접 받아 정리할 수 있느냐의 문제다.

## 예제와 확인 결과

이번 주제는 이론으로만 공부했고 실행한 것은 없다. Dockerfile과 빌드 출력(`CACHED`), 에러 메시지는 설명용이다.

## 남은 질문

- 아래 레이어에서 지운 파일이 이미지 크기에 남는다면, `RUN`으로 설치하고 정리하는 건 어떻게 써야 하나? (6번)
- jar 하나가 통째로 한 레이어면 코드 한 줄에도 jar 전체가 다시 올라간다. 더 잘게 나눌 수 있나? (6번, 15번)
- `RUN apt-get update`는 명령 문자열이 같으면 캐시를 쓴다는데, 그러면 오래된 패키지 목록이 계속 쓰이지 않나?
- `docker stop`의 대기 시간 10초는 어디서 바꾸고, Spring Boot의 graceful shutdown 설정과는 어떻게 맞추나? (9번)
- `docker build`의 끝에 적는 `.`(재료 폴더)에는 무엇이 넘어가고, 불필요한 파일은 어떻게 빼나? (6번)

## 참고 자료

- 인프런 「Docker가 쉬워지는 운영체제 이야기」: 이 노트는 강의를 보기 전에 썼다. 이어지는 섹션은 "Todolist 미니 프로젝트 실습"의 이미지 생성 강의들이다.
- [Dockerfile reference](https://docs.docker.com/reference/dockerfile/)
- [Docker — Build cache](https://docs.docker.com/build/cache/)
- [Dockerfile reference — Shell and exec form](https://docs.docker.com/reference/dockerfile/#shell-and-exec-form)
- [Spring — Spring Boot with Docker](https://spring.io/guides/gs/spring-boot-docker)
