# Docker

[전체 목차](../README.md)

컨테이너가 OS 위에서 실제로 무엇인지부터 시작해 이미지, 데이터와 네트워크, 컨테이너 생명주기를 공부하고, Docker Compose와 Spring Boot 앱 실행으로 마무리한다. 네트워크 기초와 나중에 공부할 Kubernetes로 이어지는 흐름을 함께 잡는다.

## 읽는 순서

아직 통과한 주제가 없다.

## 공부 계획

주제를 통과하면 체크하고, 노트를 위 "읽는 순서"에 올린다.

**Part 1. 컨테이너는 정체가 뭘까?**

- [ ] 1. 도커는 왜 나왔을까? — "내 PC에선 되는데", VM과 컨테이너의 차이
- [ ] 2. 컨테이너는 그냥 프로세스일까? — namespace(격리), cgroup(자원 제한), 커널 공유
- [ ] 3. 도커 엔진은 어떻게 생겼을까? — CLI → dockerd → containerd → runc, Windows·Mac에서 돌아가는 방식(WSL2·VM)

**Part 2. 이미지**

- [ ] 4. 이미지와 컨테이너는 뭐가 다를까? — 읽기 전용 레이어, 쓰기 레이어, OverlayFS
- [ ] 5. Dockerfile 한 줄은 무엇이 될까? — 명령어별 레이어, 빌드 캐시, `CMD`와 `ENTRYPOINT`
- [ ] 6. 이미지를 어떻게 작고 빠르게 만들까? — 멀티 스테이지 빌드, 베이스 이미지 선택, `.dockerignore`
- [ ] 7. 이미지는 어디에 어떻게 저장될까? — 레지스트리, 태그와 digest

**Part 3. 실행 환경**

- [ ] 8. 컨테이너 안의 데이터는 왜 사라질까? — volume, bind mount, tmpfs
- [ ] 9. 컨테이너끼리, 그리고 밖에서 어떻게 통신할까? — bridge 네트워크, 포트 매핑, 컨테이너 DNS
- [ ] 10. 컨테이너는 어떻게 시작하고 끝날까? — PID 1, 시그널과 graceful shutdown, 재시작 정책, 로그

**Part 4. 실전**

- [ ] 11. 여러 컨테이너를 어떻게 한 번에 띄울까? — Docker Compose, `depends_on`과 healthcheck
- [ ] 12. Spring Boot 앱을 컨테이너로 띄우면 뭘 신경 써야 할까? — JVM 메모리와 cgroup, 레이어드 jar, 환경 변수로 설정 주입
- [ ] 13. 다음은 어디로 갈까? — 오케스트레이션(Kubernetes)이 필요한 이유

### 다른 주제와 이어지는 곳

- 9번에서 나오는 IP·서브넷·포트·NAT·DNS 같은 일반 개념은 [Network](../network/) 노트로 남기고, 도커에서 확인한 내용은 이 폴더에 남긴다.
- 2번 namespace·cgroup은 [Operating System](../operating-system/)의 프로세스 개념과 이어진다.
- 8~12번은 Kubernetes의 PersistentVolume, Pod 네트워크·Service, probe와 Pod 종료 과정, requests·limits로 이어진다.

## 기록 기준

- 각 문서는 시작한 질문, 이해한 내용, 헷갈렸던 점, 예제와 확인 결과, 남은 질문, 참고 자료를 연결한다.
- 헷갈렸던 점에는 공부하며 실제로 헷갈린 것과 확인 문제·내 답을 남긴다.
- 설명용 예제와 실제로 실행한 결과를 구분한다. 실행했다면 실행 환경과 관찰한 결과를 적는다.
- 일반적인 개념과 인프런 강의 실습에서 확인한 내용을 구분한다.
