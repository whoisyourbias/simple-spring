# 현재 지원 범위·중간 모델·차이·검증

기준일: 2026-11-06 재설계 후. 현재 학습 순서는 [00~19](../00-overview/03-implementation-roadmap.md), 선택 확장은 [E01~E05](../00-overview/05-extension-tracks.md)다. 아래 계획과 실제 구현 상태를 구분한다.

## 1. 실제 상태

| 영역 | 현재 상태 | 실행 검증 |
|---|---|---|
| 00~19 학습 과정 | 문서 전면 재설계 | 한계·개선·회귀의 예정 실험 정의 |
| 공통 학습 규약·확장 과정 | 문서 작성 | 문서 구조·연결 확인 대상 |
| 자체 Spring 코드 | src 파일 없음 | Java 테스트·빌드 없음 |
| 00의 HTTP 앱 | 미구현 | 실제 TCP 검증 없음 |
| BeanFactory·DI·Context·AOP | 미구현 | 통과한 기능 테스트 없음 |
| MVC·Boot식 실행·자동 구성 | 미구현 | 통과한 기능 테스트 없음 |
| E01~E05 | 모두 미구현 | 개요만으로 구현 진입 불가 |
| P0·Q·B·R 서버 조건 | 각각 미충족·미지원 | 연동 계약의 조건 참조 |

자체 Spring 동작이 실제 Spring과 같다고 실행으로 확인된 영역은 아직 없다. [서버의 현재 코드·기존 검증 출처](01-embedded-tomcat-contract.md)를 이번 Spring 구현 결과로 확대하지 않는다.

## 2. 중간 모델이 약속하는 범위

| 단계 | 제공하는 범위 | 아직 약속하지 않는 것 |
|---|---|---|
| 00~01 | 수동 조립 HTTP 앱, 생성자 DI의 수동 사용 | BeanFactory·자동 생성 |
| 02 | 완성 객체의 이름 등록·조회 | 정의 기반 생성·자동 init/destroy |
| 03~06 | 정의와 singleton 캐시, 점진적 DI·스캔·설정 | 모든 scope·전체 annotation·전체 configuration |
| 07 | 정의 기반 빈의 명시 init/destroy | 전체 annotation callback·모든 destroy 추론 |
| 08 | 필수 non-lazy 선행 생성·실패 정리, 선택 집계 빈은 명시 lazy 유지 | 자동 proxy·Boot 내장 서버 lifecycle |
| 09 | 명시 factory의 수동 proxy 제공, 모든 기존 경로의 기록 보존 | 신규 서비스 타입의 자동 proxy 적용 |
| 10 | 일반 생성 경로의 BPP·최종 proxy 주입 | early proxy·클래스 proxy 전체 |
| 11~13 | 제한된 handler 형태·정확 경로·명시 메서드 조건 | 임의 인자·반환값·전체 URI 패턴 |
| 14~17 | 명시 지원 인자·ResponseBody·예외·동기 interceptor | JSON 요청 body·View·async dispatch |
| 18~19 | 명시한 웹 lifecycle·일부 자동 구성 | Boot 전체 외부 설정·packager·모든 자동 구성 |

이 표는 핵심 동작을 틀리게 구현해도 된다는 허용이 아니다. 아직 도입하지 않은 기능은 API와 완료 범위에서 제외하고, 지원하는 입력의 객체 정체성·호출 순서·실패 처리는 실제 소스와 맞춘다. 핵심 차이를 허용한 사용자 예외 승인은 없다.

## 3. 실제 기술과 프로젝트 선택

| 항목 | 실제 동작·근거 | 이번 선택 | 이유 | 사용자 코드 영향 | 후속 |
|---|---|---|---|---|---|
| API 호환 | 실제 MVC는 Jakarta Servlet 사용; S13·S37 | 자체 minispring/Servlet | 내부 동작을 직접 구현 | 실제 Spring JAR·annotation을 그대로 실행 못 함 | 완전 호환은 별도 프로젝트 |
| 수동 객체 등록 | registerSingleton은 이미 만든 객체 등록; S06 | 02에서 이 경로만 | 생성·관리 차이를 직접 경험 | 등록만으로 자동 callback 약속 없음 | 03·07에서 정의 기반 관리 |
| singleton 정체성 | factory의 이름/정의별 공유; S02·S06 | 03부터 동일 핵심 | 공유 문제의 실제 해결 | 같은 클래스도 다른 정의는 별개 | 정의·실패·동시 조회 검증 |
| DI | 실제 후보 선택·생성자 규칙은 다양; S04·S05 | 04는 공개 생성자 하나·유일 후보 입력 | 의존 관계 중복 제거에 집중 | primary·qualifier·컬렉션 미지원 | 지원 확장 전 세부 선택 규칙 조사 |
| 순환 | 생성자 순환은 실패, 일부 field/setter 순환은 early 참조; S03·S06 | constructor DI만 | 불완전 객체 노출 없는 출발 | field/setter 순환 해결 없음 | 해당 주입 방식 도입 전 별도 설계 |
| scope | singleton·prototype·웹/사용자 scope; S02 | 주 과정은 singleton, E01에서 prototype | 상태 섞임 문제에서 필요 도입 | 요청 scope 자동 제공 안 됨 | request/session/proxy 별도 조사 |
| scan | metadata·리소스·후보·이름 등록; S07·S39 | 05는 명시 후보→디렉터리 | 등록 누락부터 해결 | JAR 전체 스캔 미지원 | JAR·classloader 별도 요구 |
| configuration | factory method 정의와 full enhancement; S08·S09·S40 | 06은 lite, E02는 선택 | 외부 타입 등록과 직접 호출 문제 분리 | lite 직접 호출은 새 객체일 수 있음 | full은 CGLIB·분기 조사 선행 |
| 수명 | init 후 완성 객체, dependent 우선 파괴; S02·S03·S06 | 07은 명시 callback | main의 준비/해제 목록 제거 | 미지원 callback annotation은 실행 안 됨 | annotation 수명 소스 조사 후 |
| refresh | non-lazy 선행 생성·실패 정리; S04·S10 | 08에서 제한 Context, 선택 집계는 lazy | 첫 요청에 늦게 나는 필수 빈 오류 방지 | eager 소비자의 lazy 의존 빈은 생성됨 | 전체 이벤트·재refresh 별도 조사 |
| AOP | interceptor chain·proxy, 일반/early 경로; S11·S12 | 09 모든 기존 소비자에 수동 연결→10 신규 타입도 자동 JDK proxy | 포장 등록 누락과 최종 참조 경험 | 인터페이스 주입 기준, 자기 호출 우회 | 클래스/early proxy 별도 |
| BPP | before/init/after와 최종 wrapper; S03·S41 | 10에서 일반 빈 생성 전 준비 | 모든 소비자의 동일 proxy | 사후 등록으로 기존 빈 교체 안 함 | 자동 탐색 정렬 세부 조사 |
| handler 선택 | 여러 mapping/adapter·충돌·조건; S13~S16·S20 | 11~13 정확 경로부터 | Servlet 변경·타입 분기·등록 중복 차례 해결 | pattern 전체 호환 아님 | PathPattern/condition 조사 |
| 인자 | resolver·binder·검증·호출; S16·S42 | 14는 명시 필수 이름·지원 타입 | method의 parsing 제거 | 모든 default/빈 값/validation 미지원 | resolver별 규칙 확인 후 확장 |
| 응답 | 본문 converter와 View 구분; S18·S19 | 15는 ResponseBody, E04에서 View | 직렬화와 화면 요구 분리 | 일반 String을 무조건 본문으로 쓰지 않음 | negotiation·View 분기 조사 |
| 오류 | resolver 처리·미해결 전파; S13·S38 | 16은 미커밋 메모리 응답 범위 | method별 catch 제거 | commit 뒤 재응답·ERROR dispatch 미지원 | 서버 buffer/commit 계약 확장 |
| 전후 처리 | pre 정순, post/completion 역순; S17 | 17 동기 REQUEST | handler별 검사·완료 누락 해결 | Security·async 구현 아님 | async는 E05 |
| 요청 body | 실제 서버는 body framing·입력 지원 | 현 서버는 body 501 | 기존 서버의 명시 제외 | JSON 요청·업로드 사용 불가 | 서버 범위 변경 계획 후 |
| query | 실제 Servlet은 parameter parsing 제공 | Q를 14 진입 조건 | HTTP 파싱을 MVC에서 우회하지 않음 | 현재 query 바인딩 단계 미완료 | 서버 parsing/decoding 구현 |
| 서버 시작 | Boot는 준비와 수락을 분리; S22~S25 | B를 18 진입 조건 | 준비 전 업무 요청 방지·실패 정리 | 현재 Endpoint 즉시 bind와 같지 않음 | Tomcat 별도 계획 |
| 자동 구성 | imports·조건·back-off; S26·S27 | 19는 class/bean·핵심 기본값 | 사용자 정의 우선권 | Boot 전체 자동 구성 없음 | 정렬·property 조건 확장 전 조사 |
| 외부 설정 | PropertySource·ConfigData·binder | 주 과정은 명시 설정 입력 | 지원하지 않은 전체 규칙 회피 | Boot 설정 전체 호환 안 됨 | 독립 요구부터 설계 |
| packaging | Boot plugin/loader·실행 JAR | 고정 JAR 의존성과 일반 배포 | 학습 흐름과 배포 구현 분리 | nested JAR 지원 아님 | 별도 packaging 과제 |
| transaction | propagation·rollback·자원 공유; S28~S32 | E03 동기 REQUIRED·단일 DataSource | 주문/재고 부분 반영에서 출발 | 다른 전파·분산·reactive TX 없음 | 별도 소스 조사 |
| View | model/view 이름·resolver·render; S13·S19 | E04 직접 HTML 쓰는 View | JSP 없이 화면 책임 학습 | forward·JSP 미지원 | 엔진/서버 요구로 확장 |
| reactive | async·listener I/O·수요·완료; S33~S36 | E05와 서버 R 조건 | 느린 처리·소비자 수명부터 해결 | Future/Mono만으로 WebFlux 완성 아님 | 서버·Reactive Streams 상세 설계 |
| 생태계 프로젝트 | Security·Data·Cloud·Batch의 별도 역할 | 참고 자료 | 주 과정 목표 유지 | 대체 라이브러리 아님 | 선택할 때 새 요구부터 정의 |

## 4. 미확인과 구현 전 확인할 것

- DI의 모든 후보 우선순위·여러 생성자·제네릭 선택, field/setter 순환.
- annotation callback 전체 순서·destroy 추론·full configuration/FactoryBean·scoped proxy의 모든 분기.
- classpath 스캔의 JAR·module·custom loader와 전체 naming/충돌 규칙.
- MVC URI 패턴 비교·인자 default/빈 값·validation·미디어 협상·View 전체 분기.
- Boot ConfigData·PropertySource 우선순위·binder·자동 구성 정렬, graceful timeout·재시도.
- Reactor operator·Reactive Streams 상세·read/write 경합·버퍼 소유권·reactive transaction.

일부 소스를 조회했다는 사실과 기능 전체를 구현할 만큼 검증했다는 사실을 구별한다. 위 사항에 의존하는 코드는 소스 확인과 계획 보완 뒤 작성한다.

## 5. 검증 기록의 위치

이번에는 문서·단계 전환·소스 대응을 검토한다. 실제 명령과 결과·남은 제약은 [재설계 계획](../../docs/superpowers/plans/2026-11-06-problem-driven-learning-redesign.md)에 기록한다. 이전 조사 결과는 당시 계획에 보존한다. 단계별 Java 기능은 구현 후 실제 실행 결과가 생겼을 때만 완료로 바꾼다.
