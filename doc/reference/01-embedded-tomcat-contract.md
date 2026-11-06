# simple-tomcat 내장 연동 계약 조사

조사일: 2026-11-06. 이 문서는 **현재 코드에서 가능한 것**, **계획만 존재하는 것**, **새로 필요한 것**을 구분한다. 구현 순서는 [로드맵](../00-overview/03-implementation-roadmap.md), Spring 비교 근거는 [소스 지도](03-learning-stage-mapping.md)에 둔다.

## 현재 학습 과정의 진입 조건

2026-11-06 전면 재설계 후에는 웹을 후반에 추가하지 않는다. 00부터 같은 HTTP 주문 앱을 유지한다. 아래 조건은 기능 순서가 아니라 해당 학습 단계에 들어가기 위한 실제 서버 조건이다.

| 조건 | 필요한 시점 | 통과 기준 | 현재 |
|---|---|---|---|
| P0 | 00 이전 | 자체 Servlet·프로그램 등록·Wrapper 수명·내장 start/getPort/close·실제 TCP·종료 계약 검증 | 미충족, 현재 HTTP/Adapter까지 |
| Q | 14 이전 | query 원문과 디코딩 값 구분·문자셋·반복 값·잘못된 인코딩 처리 계약 | 미지원 |
| B | 18 이전 | 서버 준비·bind·수락 분리와 시작 실패 정리 | 생성자 즉시 bind와 차이 있음 |
| R | E05 이전 | async 수명·논블로킹 read/write·streaming·cancel/error/timeout·버퍼 정리 | 기존 계획의 제외 범위 |

P0 전에 순수 Java 객체 조립을 준비할 수는 있지만 그것을 00 HTTP 기준선 완료로 표시하지 않는다. 00~19의 기본 요청은 body 없는 읽기 요청이다. fixture/서비스에서 주문을 준비하고 GET에서 상태를 변경하지 않는다. JSON 응답은 가능 범위와 구별하며 JSON 요청은 별도 body 확장 뒤에만 다룬다.

재설계 시 서버 HEAD cbfef0e7ce71d68be31cfb987c9e9dc931b3514d와 Java 파일 목록을 재확인했다. 아래 초기 조사의 API 제약과 동일하며 서버 저장소에는 쓰기 작업을 하지 않았다.

## 1. 로컬 확인 범위

서버 경로: `C:/Users/SSAFY/Downloads/simple-tomcat`.

조사 시 HEAD는 `743e6c20340c7577ae261c257c59ca32ba660240`이었다. 작업 트리에 문서 변경이 있었으므로 이 기록은 해당 커밋의 pristine snapshot이라고 주장하지 않는다. 서버 저장소는 읽기만 했고 기존 변경을 수정하지 않았다.

최종 재확인 때 HEAD는 `cbfef0e7ce71d68be31cfb987c9e9dc931b3514d`로 바뀌었고 작업 트리는 clean이었다. 두 커밋 차이는 doc 아래 문서·구조도·문서용 검사 파일 6개이며 Java 소스 차이는 없었다. 아래 API·선행 조건 조사 결과에는 영향이 없다. 이 작업에서 서버 저장소에 쓰기나 커밋을 수행하지 않았다.

| 확인한 파일 | 실제 확인 내용 |
|---|---|
| `pom.xml` | `org.example:simple-tomcat:0.0.1-SNAPSHOT`, Java 21, JUnit 5.11.4 |
| `.mvn/wrapper/maven-wrapper.properties` | Wrapper 3.3.2, Maven 3.9.9 |
| `src/main/java/simpletomcat/util/net/Endpoint.java` | 생성자에서 소켓 bind, 호출 스레드의 순차 accept/처리, 요청 후 연결 close |
| `src/main/java/simpletomcat/coyote/http11/Http11Protocol.java` | Endpoint·Processor·Adapter 조립, getPort/serve/close |
| `src/main/java/simpletomcat/coyote/http11/Http11Processor.java` | HTTP 파싱·framing 검사·Adapter 호출·응답 쓰기, 본문 있는 요청 501 |
| `src/main/java/simpletomcat/coyote/Adapter.java` | `void service(Request, Response) throws Exception` 동기 경계 |
| `src/main/java/simpletomcat/coyote/Request.java` | method·원문 URI/query·headers, body·parameter map 없음 |
| `src/main/java/simpletomcat/coyote/Response.java` | status·단일 값 응답 header·String body 메모리 저장 |
| `doc/00-overview/01-progress.md` | 2단계까지 완료, 53개 테스트 등 기존 검증 기록 |
| `doc/plans/2026-11-05-step-03-servlet.md` 및 `step-07-embedded.md` | 자체 Servlet 수명과 SimpleTomcat 공개 API는 예정 |
| `doc/reference/02-scope-and-differences.md` | 본문·async·streaming 제외, 표준 Servlet 바이너리 호환 없음 |

## 2. 현재 경계와 목표 경계

현재 실제 흐름:

```text
사용자 thread
  → Http11Protocol.serve
  → Endpoint.serve
  → Http11Processor.process
  → Adapter.service(Coyote Request, Coyote Response)
  → 메모리 응답의 UTF-8 직렬화
  → 연결 close
```

연동 후 목표 흐름:

```text
HTTP / 소켓
  → Coyote Processor
  → Catalina Adapter / Servlet 요청·응답 래핑
  → Context·Wrapper 선택
  → FilterChain
  → minispring DispatcherServlet.service
  → HandlerMapping
  → HandlerAdapter / argument resolver
  → 컨테이너가 관리하는 Controller → Service → Repository
  → return value handler / converter 또는 View
  → Servlet 응답 → HTTP 전송
```

아래 흐름은 현재 존재하지 않는 Servlet·매핑·필터 계층을 포함한 목표다. 지금 Adapter에 lambda로 업무 라우팅을 넣는 실험은 가능하지만, 그것을 Servlet 기반 MVC 완료라고 부를 수 없다. 실제 Apache Tomcat은 프로그램 등록 API와 Wrapper를 제공하며 Spring은 Servlet 계약 위에서 동작한다. [Tomcat.addServlet/start](https://github.com/apache/tomcat/blob/10.1.34/java/org/apache/catalina/startup/Tomcat.java)

## 3. 서로 다른 소유권

| 자원·상태 | 소유자 | 필요한 계약 |
|---|---|---|
| listener·연결·서버 worker | simple-tomcat | 시작 실패·stop/close에서 정리, 요청 처리 중 종료 조정 |
| Servlet 인스턴스의 init/service/destroy | Tomcat Wrapper | 초기화 성공 후 service, 종료 시 destroy 한 번 |
| 업무 Controller·Service·Repository | Spring BeanFactory/Context | 생성·주입·후처리·callback·파괴 |
| MVC mapping·handler metadata | Spring MVC 계층 | 시작 시 검증, 요청 때 관리 빈 사용 |
| 내장 서버 구성·Context 실행 조정 | Mini Boot 계층 | 서버 factory 선택·Servlet 등록·lifecycle 연결 |
| transaction Connection | transaction manager / DataSource | 요청 객체와 구분, 동기 트랜잭션 범위 공유·반환 |

DispatcherServlet을 Spring 빈으로도 등록한다면 Servlet destroy와 빈 destroy가 같은 자원을 이중 해제하지 않게 해야 한다. Tomcat이 요청 처리를 멈추는 경로와 업무 빈을 파괴하는 경로의 순서도 검증해야 한다. ServletContext와 ApplicationContext는 역할이 다른 객체다.

## 4. 동기 MVC 연동을 위한 선행 조건

| 조건 | 현재 | 필요한 결과 |
|---|---|---|
| 자체 Servlet 계약 | 예정 | 최소 init/service/destroy와 요청·응답 API 확정 |
| Wrapper 수명 | 예정 | init 실패·활성 호출·중복 종료 계약 |
| Context와 Servlet 매핑 | 예정 | 업무 URL 여러 개가 기본 매핑 `/`의 디스패처에 전달 |
| FilterChain | 예정 | 디스패처 바깥의 전후 처리와 예외 정리 |
| 내장 API | 예정 | start/getPort/close와 프로그램 등록이 실제로 동작 |
| 실행 수명 | 현재 호출자가 serve thread 소유 | 누가 실행·join·실패 전달을 맡는지 명시 |
| query parameter | 원문 query만 있음 | 디코딩 책임·문자셋·반복 값·오류를 먼저 확정 |
| 배포 의존성 | SNAPSHOT POM만 확인 | 소비할 JAR 버전·커밋·공개 API 고정, 별도 앱에서 실행 |

최초 MVC smoke test는 요청 body 없이 GET `/hello`, `/orders/count`를 사용한다. 상태·header·본문의 실제 TCP 결과와 요청 예외 후 복구를 검증한다. 현재 서버의 HEAD 본문 억제·UTF-8 Content-Length 처리는 HTTP 계층 책임으로 유지한다.

`simple-tomcat`의 계획된 SimpleTomcat 클래스나 Servlet 인터페이스의 이름·메서드를 현재 존재하는 API처럼 import하는 구현 계획을 작성해서는 안 된다. 해당 단계가 구현된 시점에 이 표와 consumer 파일 목록을 갱신한다.

## 5. Boot식 시작의 핵심 차이

실제 Boot v3.4.1 흐름은 다음과 같다.

1. SpringApplication.run이 environment·context를 준비하고 refresh한다.
2. AbstractApplicationContext.refresh의 onRefresh에서 ServletWebServerApplicationContext가 factory를 찾아 서버를 만든다.
3. TomcatWebServer의 초기화가 내부 Tomcat을 시작해 등록 listener 등을 실행한다. 이때 connector를 분리하고 초기 바인딩을 지연한다.
4. 나머지 non-lazy singleton 초기화를 마친 후 finishRefresh의 lifecycle에서 WebServerStartStopLifecycle.start가 WebServer.start를 호출한다.
5. TomcatWebServer.start가 connector를 복원하고 시작 상태를 확인한다. 서버 준비와 업무 요청 수락 시작은 같은 사건이 아니다.

근거는 소스 지도 S10·S21~S25다. 빈 생성 전 서버 객체 생성이 절대로 없다고 설명하거나, 포트가 열렸다는 사실만으로 앱 초기화 성공을 판정하면 이 순서를 놓친다.

현재 Endpoint는 **생성자에서 즉시 bind**하고 close는 활성 소켓을 닫되 serve thread 종료까지 기다리지 않는다. 따라서 실제 Boot와 같은 시작·실패 계약을 만들 때는 다음이 필요하다.

- 준비, 바인딩, 요청 수락의 상태를 서버 쪽에서 구별한다. 준비 중 아직 업무 요청이 실행되지 않도록 한다.
- factory/initializer 실패, 빈 초기화 실패, 포트 충돌 때 원인을 보존하고 이미 획득한 자원을 정리한다.
- 종료 시 새 요청을 멈추고 실행 중 요청을 정리한 뒤 Servlet·업무 빈 수명을 끝낸다.
- 제한 시간 초과·종료 실패·재시도의 정확한 정책은 Spring/Boot와 Tomcat의 해당 소스를 추가 확인하고 계획에 명시한다. 이번 조사에서는 모든 graceful-shutdown 분기를 확인하지 않았다.

이 항목은 Spring 쪽 래퍼만 추가해서 해결되었다고 기록할 수 없다. Tomcat 프로젝트의 기존 lifecycle 계획과 충돌 여부를 확인한 별도 변경이 필요하다.

## 6. 본문·JSON·WebFlux 확장 조건

| 하고 싶은 것 | 현재 막는 조건 | 구현 전 필요한 조사·변경 |
|---|---|---|
| query 인자 바인딩 | 원문 query만 제공 | parameter parsing·charset·반복 값·decode 실패 |
| JSON 응답 | String 메모리 body만 있음 | text 기반 JSON converter는 가능, content type·Accept·UTF-8 검증 |
| JSON 요청 | 양수 Content-Length 또는 Transfer-Encoding 요청은 501 | HTTP framing·크기 제한·입력 스트림·오류 정책, 그 뒤 message converter |
| binary·중복 응답 header | String body·단일 값 Map | byte/stream response·다중 header·commit 계약 |
| SSE·streaming | 전송 전 전체 buffering, 연결당 한 요청 | flush/commit·stream framing·disconnect·수명 관리 |
| Servlet async | service 반환 후 연결 종료 | AsyncContext에 해당하는 수명·완료·timeout·error dispatch |
| Tomcat 기반 WebFlux | blocking transport·listener API 없음 | read/write readiness·listener·demand·cancel·버퍼 정리 |

기존 Tomcat 문서는 body·async를 **의도적 제외**로 둔다. 위 기능은 이미 제공되는 약속이 아니며, 범위를 바꾸는 신규 작업이다. HTTP 요청 body의 불완전한 프레이밍을 MVC에서 우회하지 않는다.

WebFlux v6.2.1의 ServletHttpHandlerAdapter는 service 안에서 startAsync를 하고 반환된 Publisher를 subscribe한다. 요청·응답은 ReadListener·WriteListener를 사용한다. [어댑터](https://github.com/spring-projects/spring-framework/blob/v6.2.1/spring-web/src/main/java/org/springframework/http/server/reactive/ServletHttpHandlerAdapter.java), [요청](https://github.com/spring-projects/spring-framework/blob/v6.2.1/spring-web/src/main/java/org/springframework/http/server/reactive/ServletServerHttpRequest.java), [응답](https://github.com/spring-projects/spring-framework/blob/v6.2.1/spring-web/src/main/java/org/springframework/http/server/reactive/ServletServerHttpResponse.java)

## 7. 저장소 간 통합 제안

처음부터 서버 코드를 Spring 저장소로 복사하거나 simple-tomcat이 Spring 코드를 참조하게 만들지 않는다. Tomcat의 공개 API가 준비되면 버전이 고정된 아티팩트로 소비한다. 로컬 개발에서는 Maven install 등을 사용할 수 있지만, 재현 시점의 서버 커밋·아티팩트 버전을 기록한다.

최소 배포는 앱과 서버 JAR의 classpath로 검증한다. `java -jar` 사용 여부와 packaging 방법은 별도 선택이며 실제 Boot의 nested-JAR loader를 복제할 필요는 없다. 이번 조사에서는 빌드·설치·서버 수정·통합 실행을 수행하지 않았다.
