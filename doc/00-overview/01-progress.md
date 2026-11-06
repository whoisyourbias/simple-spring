# 방향·진행 상황·다음 작업

기준일: 2026-11-06. 상세 단계는 [로드맵](03-implementation-roadmap.md), 공통 규칙은 [학습 규약](04-learning-workflow.md), 지원 범위는 [현황](../reference/02-implementation-status.md)에 둔다.

## 최우선 목표와 이번 변경

사용자는 “다음 단계가 이전 단계의 문제를 해결하면서 Spring Framework의 동작을 자연스럽게 익힌다”를 최우선으로 명시하고 단계 전면 재창조를 지시했다. 이에 기존 기능별 00~13 순서를 폐기하고 **같은 HTTP 주문 앱을 유지하는 00~19 과정**으로 재설계했다.

처음부터 Servlet과 직접 조립한 서비스·저장소로 시작한다. 객체 공유·조립·생성·수명·proxy 문제를 해결한 뒤에도 같은 앱을 유지하며 요청 처리·시작·기본 구성의 수작업을 옮긴다. 새 단계의 시작 문제와 직전 단계의 다음 문제를 연결했다.

세부 단계 분할·외부 fixture·범위는 그 요구에 따른 설계자의 선택이다. 단계 문서 수정은 완료했으나 Java 구현 착수·기능 검증 완료를 뜻하지 않는다.

## 실제 상태

| 대상 | 확인한 상태 |
|---|---|
| simple-spring | src 파일·빌드 파일 없음, Java 구현과 실행 결과 없음 |
| 현재 문서 | 새 00~19와 E01~E05, 전후 검증·실패 기준·다음 문제 작성 |
| simple-tomcat | HEAD cbfef0e7ce71d68be31cfb987c9e9dc931b3514d 재확인. HTTP/Adapter 구현, Servlet·내장 API 예정 |
| P0 | 서버의 실제 Servlet·프로그램 등록·수명·내장 실행 API가 아직 없어 미충족 |
| Q 조건 | 14의 query parameter parsing·decoding 계약 미지원 |
| B 조건 | 18의 서버 준비·bind·수락 분리 계약 미지원 |
| 확장 | E01 scope·E02 full configuration·E03 transaction·E04 View·E05 reactive는 모두 미구현 |

서버의 기존 53개 테스트·HTTP 검증은 그 저장소 문서의 기록이다. 이번 재설계에서 서버 테스트를 재실행하지 않았다. 서버 저장소는 읽기만 했다. [연동 계약](../reference/01-embedded-tomcat-contract.md)

## 다음 작업

1. P0에서 소비할 Tomcat 아티팩트·실제 Servlet API·기동/종료·자원 소유권을 고정한다. 현 단계에서 가상의 SimpleTomcat method를 확정하지 않는다.
2. 00의 직접 조립 HTTP 앱과 회귀 fixture를 구현할 구체적 파일·API·테스트·명령을 설계·계획에 저장한다.
3. 실제 HTTP 기준선을 검증한 뒤, 00 끝에 남긴 “두 endpoint의 다른 repository” 실험으로 01에 들어간다.

P0 전에 순수 Java 객체 조립이나 빌드 준비를 실험할 수는 있지만 00의 HTTP 완료로 기록하지 않는다. 현재 서버 제약을 숨기려고 Coyote Adapter 직결을 Servlet MVC로 대체하지 않는다.

## 문서 관리

기존 로드맵과 초기 README는 [history](../history/README.md)에 보존했다. 기존 조사 계획은 당시 상태를 보존하고 현재 단계 번호는 새 로드맵을 따른다.

이번 재설계의 선택·결과는 [설계](../../docs/superpowers/specs/2026-11-06-problem-driven-learning-design.md)와 [계획](../../docs/superpowers/plans/2026-11-06-problem-driven-learning-redesign.md)에 기록한다. 각 단계의 코드·테스트·학습 문서 checkpoint는 구현할 때 생성한다.
