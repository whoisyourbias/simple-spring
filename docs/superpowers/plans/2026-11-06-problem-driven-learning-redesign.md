# 문제 해결을 따라가는 Spring 학습 단계 재설계 계획

작성일: 2026-11-06. 범위: 학습 과정·문서 전면 수정. Java 구현·Tomcat 수정은 포함하지 않는다.

**Goal:** 사용자가 명시한 “다음 단계가 이전 단계의 문제를 해결하면서 Spring 동작을 자연스럽게 익힌다”를 모든 단계의 필수 조건으로 만든다.

**Architecture:** 처음부터 하나의 HTTP 주문 앱을 유지한다. 객체 공유·조립·수명·프록시·요청 처리·기동의 남은 수작업을 한 번에 하나씩 옮긴다. 기능명 순서의 이전 로드맵은 역사 자료로 보존한다.

**Tech Stack:** 소스 기준 Framework v6.2.1·Boot v3.4.1·Tomcat 10.1.34 유지. 향후 Java 21·Maven·JUnit 환경을 제안하되 이번 작업에서 생성하지 않는다.

**Spec:** `docs/superpowers/specs/2026-11-06-problem-driven-learning-design.md`에 재설계 기준과 선택을 기록한다. 사용자는 이번 재설계를 직접 지시했다. 이를 Java 구현 착수 승인으로 확대하지 않는다.

## 수정 전 확인한 소스와 핵심 동작

기존 38개 소스 지도를 유지하고 이번에 아래 경로를 추가 확인했다. 모든 클래스의 전체 분기를 조사했다는 뜻은 아니다.

- [DefaultSingletonBeanRegistry.registerSingleton/addSingleton](https://github.com/spring-projects/spring-framework/blob/v6.2.1/spring-beans/src/main/java/org/springframework/beans/factory/support/DefaultSingletonBeanRegistry.java): 완성된 객체 등록과 생성 정의 기반 관리는 다르다. 초기 수동 객체 등록 단계에서는 자동 init/destroy를 약속하지 않는다.
- [AbstractAutowireCapableBeanFactory.initializeBean](https://github.com/spring-projects/spring-framework/blob/v6.2.1/spring-beans/src/main/java/org/springframework/beans/factory/support/AbstractAutowireCapableBeanFactory.java#L1795): 초기화 전 후처리 → init → 초기화 후 후처리 → 최종 객체. 수동 프록시에서 자동 프록시로 이동할 때 이 순서를 보존한다.
- [AbstractApplicationContext.refresh](https://github.com/spring-projects/spring-framework/blob/v6.2.1/spring-context/src/main/java/org/springframework/context/support/AbstractApplicationContext.java#L587): 정의 후처리 → 빈 후처리기 등록 → onRefresh → 나머지 singleton 생성 → lifecycle 시작. refresh 실패 시 생성된 singleton 정리.
- [ClassPathScanningCandidateComponentProvider](https://github.com/spring-projects/spring-framework/blob/v6.2.1/spring-context/src/main/java/org/springframework/context/annotation/ClassPathScanningCandidateComponentProvider.java): metadata 읽기 → 후보 판단 → 정의 수집. 스캔을 인스턴스 생성과 섞지 않는다.
- [ConfigurationClassBeanDefinitionReader](https://github.com/spring-projects/spring-framework/blob/v6.2.1/spring-context/src/main/java/org/springframework/context/annotation/ConfigurationClassBeanDefinitionReader.java): factory method·factory bean 이름·초기화 메타데이터를 정의에 등록. 외부 타입 등록의 필요에서 설정 메서드를 도입한다.
- [BeanPostProcessor](https://github.com/spring-projects/spring-framework/blob/v6.2.1/spring-beans/src/main/java/org/springframework/beans/factory/config/BeanPostProcessor.java): plain factory의 명시 등록과 context의 자동 탐색을 구분한다. 반환된 wrapper를 이후 빈 생성에 사용한다.
- [InvocableHandlerMethod.invokeForRequest/getMethodArgumentValues](https://github.com/spring-projects/spring-framework/blob/v6.2.1/spring-web/src/main/java/org/springframework/web/method/support/InvocableHandlerMethod.java): 인자 해결 → 검증 → 대상 호출. resolver 없는 인자를 임의 null로 넘기지 않는다.
- [DispatcherServlet.getHandlerAdapter/processHandlerException/render](https://github.com/spring-projects/spring-framework/blob/v6.2.1/spring-webmvc/src/main/java/org/springframework/web/servlet/DispatcherServlet.java): adapter 선택·예외 resolver·View 렌더링은 서로 다른 책임이다. 학습 단계도 문제별로 나눈다.
- [ServletWebServerApplicationContext](https://github.com/spring-projects/spring-boot/blob/v3.4.1/spring-boot-project/spring-boot/src/main/java/org/springframework/boot/web/servlet/context/ServletWebServerApplicationContext.java): refresh 중 서버 준비, lifecycle에서 요청 수락, 실패 시 서버 정리. 단순 main 래퍼와 구분한다.

그 밖의 후보 선택·AOP 호출·MVC 매핑·자동 구성·트랜잭션·reactive 근거는 기존 소스 지도의 고정 태그를 따른다. 버전 변경은 없다.

## Global Constraints

- 매 단계는 이전 산출물, 한계 재현, 손으로 해볼 해결, 최소 변경, 개선 검증, 다음 문제, Spring 대응을 가진다.
- 전후 비교는 같은 업무 fixture와 요청으로 수행한다. 리팩터링은 응답 보존과 수정 범위 감소도 증거다.
- 새 요구 때문에 한계가 드러난 경우 기존 기능의 버그와 구별한다. 기능을 고의로 틀리게 구현하지 않는다.
- 기본 과정은 본문 없는 읽기 요청을 사용한다. 주문 생성은 fixture/서비스 호출로 준비하고 GET 요청으로 상태를 변경하지 않는다.
- P0의 자체 Servlet·내장 API가 준비되기 전 HTTP 단계는 완료할 수 없다. 순수 Java 실험은 준비 작업일 뿐 단계 완료로 표시하지 않는다.
- 본문·query decoding·Boot 지연 bind·reactive 서버 역량은 명시적 서버 선행 조건이다.
- 핵심 동작에 영향을 주는 임시 모델은 실제 Spring 완성으로 표현하지 않는다. 미확인에 의존하는 구현은 진행하지 않는다.

## Review Focus

1. 단계 사이의 다음 문제와 다음 단계 재현이 실제로 이어지는가: 00~19 전환 표와 각 본문을 교차 대조한다.
2. 생명주기·후처리의 선후 관계가 Spring과 어긋나지 않는가: BPP 도입 전 빈은 빈 후처리기 없는 부분집합이며 도입 후 refresh 순서를 명시한다.
3. 웹이 중간에 갑자기 추가되거나 현재 없는 Tomcat API를 이미 있다고 전제하지 않는가: P0와 00 기동 조건을 검토한다.
4. 인자·응답·예외를 한 단계에 다시 묶지 않았는가: 각 단계가 없애는 수작업과 검증 fixture가 독립적인지 본다.
5. 새 단계 번호와 이전 기록이 혼동되는가: 현재 문서·소스 지도는 새 번호로, 이전 문서는 history와 당시 계획으로 구분한다.

## 작업 순서

### Task 1 — 이전안 보존과 재설계 기준 저장

- [x] 이전 로드맵·초기 README를 doc/history에 복사하고 상대 링크를 보정한다.
- [x] 새 설계 문서에 목표·선택·범위·서버 전제·완료 기준을 적는다.
- [x] 학습 단계 규약을 문서화하고 AGENTS.md에 사용자 목표를 지속 규칙으로 반영한다.

### Task 2 — 단계 전면 재작성

- [x] 로드맵을 기존 00~13 번호에서 새 00~19의 인과관계로 재작성한다.
- [x] 각 단계에 전후 실험과 유지해야 할 정상·실패 계약, 다음 단계로 넘길 구체적인 문제를 적는다.
- [x] prototype·full configuration·트랜잭션·View·WebFlux는 출발 단계와 재현 문제를 가진 E01~E05 확장 과정으로 분리한다.

### Task 3 — 안내·소스·현황 동기화

- [x] README·목차·진행 요약을 새 과정의 단일 진입점으로 바꾼다.
- [x] 구현 현황의 번호·지원 범위와 서버 연동 계약의 P0·query·Boot 조건을 갱신한다.
- [x] 기존 소스 ID를 유지한 채 단계 열을 새 번호로 바꾸고 이번 추가 소스의 ID를 등록한다.
- [x] 기존 조사 계획은 당시 실행 기록으로 보존하고 새 계획으로의 안내를 추가한다.

### Task 4 — 검토와 검증

- [x] 독립 검토로 단계 연결·핵심 동작·범위 누락을 확인하고 지적을 반영한다.
- [x] Markdown 링크·번호·필수 단계 항목·소스 ID·고정 태그를 검증한다.
- [x] src 파일 없음·서버 읽기 전용·기존 비문서 파일 보존을 확인한다.
- [x] 실제 변경·검증 결과·남은 미확인을 이 계획에 기록한다.

## 차이·생략 기록

Spring은 완성된 범용 프레임워크지만 학습 과정은 수동 객체 등록→생성 정의→DI→수명→후처리의 중간 모델을 거친다. 중간 모델에는 아직 도입하지 않은 API를 노출하지 않으며, 지원하는 계약에서는 실제 Spring의 객체 정체성·호출 순서·실패 전파를 보존한다. 이름만 같은 기능을 먼저 채우지 않기 위한 선택이다.

사용자 코드는 단계별 수동 조립에서 점차 프레임워크 선언으로 이동한다. 자체 API는 표준 Spring/Servlet 바이너리 호환이 아니며, full configuration·전체 binder·전체 설정·Reactive Streams 세부는 조사 완료 후 별도로 구현한다. 세부 영향·후속 여부는 구현 현황과 확장 과정에 기록한다.

## 실행·검증 결과

2026-11-06 문서 재설계 완료. Java 구현·빌드 설정 생성·의존성 설치·서버 수정·Git 커밋은 수행하지 않았다.

### 실제 변경

- 기존 00~13을 현재 순서에서 제거하고 수동 HTTP 기준선 00부터 조건부 자동 구성 19까지 같은 주문 앱으로 연결했다.
- 모든 주 단계에 이전 산출물·한계 재현·수동 해결·최소 변경·Spring 대응·전후 검증·실패·다음 문제·선행 조건을 작성했다.
- scope·full configuration·transaction·View·reactive는 출발 단계와 새 요구가 있는 E01~E05로 분리했다.
- 초기 로드맵·README는 history에 보존하고 README·목차·진행·현황·서버 계약을 현재 과정에 맞췄다.
- 사용자 최우선 목표를 AGENTS.md와 공통 학습 규약에 반영했다.
- S01~S38의 ID와 기준 버전을 유지하고 S39~S42를 추가했다. 현재 소스 지도는 42개 출처다.

### 독립 검토와 반영

brainstorming의 builder check로 별도 검토자가 설계·로드맵·확장·서버 계약·현황·소스 지도를 읽었다. 사용자에게 추가 결정을 요구할 필요 없이 다음을 문서에서 해결했다.

1. 02에서 공통 조립 함수만으로 풀리는 중복과, 고정 반환 묶음의 확대로 드러나는 선택적 조회 필요를 구분했다.
2. 08에서 선택 집계 빈을 명시 lazy로 유지하고 non-lazy 소비자의 lazy 의존 빈은 생성되는 경계를 추가했다.
3. 09에서 모든 기존 HTTP 소비자에 수동 proxy를 제공해 기록을 보존했다. 10은 새 서비스의 포장 등록 누락을 해결하며 수동 포장 제거·이중 proxy를 검증한다.
4. 소스 표의 S38~S39 사이 빈 줄을 제거했다.

재검토 결과, 위 범위에서 단계 연결·Spring 핵심 동작에 대한 추가 차단 사항이 없다는 응답을 받았다. 미지원·미확인 기능의 구현 준비가 끝났다는 의미는 아니다.

### 검증 방법과 결과

PowerShell에서 Python 3를 `python -X utf8 -`로 실행해 문서를 읽고 아래 조건을 확인했다.

| 확인 | 실제 결과 |
|---|---|
| Markdown 범위 | README·AGENTS·doc·docs의 17개 파일 |
| 상대 링크 | 65개 대상 경로 확인, 끊어진 링크 0개 |
| 형식 | UTF-8 대체 문자·닫히지 않은 code fence·줄 끝 공백 0개 |
| 주 단계 구조 | 00~19 연속 20개, 단계별 필수 9항목 총 180항목 |
| 확장 구조 | E01~E05 연속 5개 |
| 소스 표 | S01~S42 중복·누락 없음, 모두 고정 태그와 line anchor, 연속 Markdown 표 |
| 참조 | 주 과정·확장의 모든 S번호가 소스 지도에 존재, 소스 표의 단계가 현재 번호 범위 안에 있음 |
| 상태 | src 파일 0개, 기존 비문서 파일에 쓰기 작업 없음 |
| 서버 | 최종 HEAD·작업 트리 확인 명령으로 읽기 전용 범위를 재확인 |
| Java 검증 | 실행하지 않음. 이번 산출물은 학습 문서이며 Java 구현 없음 |

검사는 local link의 경로 존재와 문서 구조를 확인한다. 브라우저 렌더링 전체·원격 링크의 모든 HTTP 상태·아직 작성하지 않은 Java 코드의 동작을 검증한 것이 아니다. 추가 소스는 실제 연결 도구 조회 내용에 근거하며 source anchor는 조회한 범위와 대조했다.

### 남은 조건

P0·Q·B·R 서버 조건은 미충족이며 새로운 단계 설계가 이 기능을 자동으로 구현하지 않는다. 전체 DI 선택·annotation 수명·full configuration·binder/negotiation·Boot 설정·reactive 세부는 미확인 목록에 남겼다.

다음 구현 작업은 P0의 실제 서버 API를 확인하고 00의 파일·계약·실행 명령을 구체화한 설계·계획을 먼저 저장하는 것이다. 문서 단계 완료와 Java 기능 완료를 구분한다.
