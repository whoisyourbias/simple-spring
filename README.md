# 문제를 해결하며 만드는 작은 Spring

직접 만든 simple-tomcat 위에서 같은 주문 앱을 조금씩 발전시키며 Spring 내부를 배운다. **매 다음 단계는 이전 앱에서 재현한 문제를 해결한다.** 기능 이름보다 그 기능이 필요해진 이유와 옮겨 간 책임을 먼저 이해한다.

현재는 학습 과정과 소스 조사를 문서화한 상태다. src에는 구현 코드가 없으며 실행 가능한 Spring 또는 HTTP 앱은 아직 없다.

## 어디서 시작하는가

먼저 [문서 목차](doc/README.md)와 [00~19 학습 순서](doc/00-overview/03-implementation-roadmap.md)를 읽는다. 출발점은 BeanFactory가 아니라, Servlet 안에서 직접 객체를 만들고 /orders/count를 보여 주는 작은 앱이다.

```text
직접 만든 두 화면이 다른 주문을 본다
 → 같은 객체를 수동 주입한다
 → 여러 main에 복사한 조립을 한곳에 등록한다
 → 안 쓰는 객체까지 만드는 문제를 생성 정의로 해결한다
 → recipe에 다시 적은 의존 관계를 constructor에서 읽는다
 → 같은 앱의 등록·수명·proxy·요청 처리 수작업을 하나씩 옮긴다
 → 반복되는 시작과 기본 구성을 Boot식 실행으로 옮긴다
```

## 학습의 큰 흐름

| 범위 | 해결해 나갈 문제 |
|---|---|
| P0·00 | 서버 계약 확인과 프레임워크 없는 HTTP 기준선 |
| 01~06 | 서로 다른 상태·복사한 조립·불필요한 생성·의존 관계 중복·등록 누락·외부 객체 등록 |
| 07~10 | 수동 자원 관리·늦은 시작 오류·업무 코드 중복·원본과 proxy 참조 불일치 |
| 11~17 | Servlet 분기·handler 타입 분기·URL 등록 중복·인자/응답/예외/전후 처리 반복 |
| 18~19 | 복사한 시작 순서·기본 구성 반복과 사용자 설정 충돌 |
| E01~E05 | 주 과정의 새 요구에서 출발하는 scope·full configuration·트랜잭션·View·reactive |

단계마다 **이전 버전의 한계 실험 → 직접 고쳐 보기 → 최소 책임 추출 → 개선 확인 → 기존 계약 회귀 → 다음 문제**를 기록한다. [학습 규약](doc/00-overview/04-learning-workflow.md)을 따른다. 구체적 문제·완료 기준은 로드맵 한 곳에서 관리한다.

## 서버와 소스 기준

소스 연구 기준은 Framework v6.2.1·Boot v3.4.1·Tomcat 10.1.34다. Java 21·Maven Wrapper·JUnit Jupiter 환경을 제안한다. 실제 빌드 설정은 아직 만들지 않았다.

simple-tomcat은 현재 HTTP/Adapter까지 구현되었으며 자체 Servlet·내장 API는 예정이므로 P0가 미충족이다. 준비 전에 Coyote Adapter 실험을 Servlet MVC 완료로 표시하지 않는다. query 인자와 Boot 초기 수명에는 추가 서버 조건도 있다. [서버 연동 계약](doc/reference/01-embedded-tomcat-contract.md)을 참고한다.

## 관련 문서

- [Spring·Framework·Boot·MVC·WebFlux의 관계](doc/00-overview/02-spring-ecosystem.md)
- [현재 상태와 다음 작업](doc/00-overview/01-progress.md)
- [남은 문제에서 출발하는 확장 과정](doc/00-overview/05-extension-tracks.md)
- [실제 Spring 소스와 단계 대응](doc/reference/03-learning-stage-mapping.md)
- [지원 범위·차이·미확인](doc/reference/02-implementation-status.md)
- [전면 재설계 계획과 검증 결과](docs/superpowers/plans/2026-11-06-problem-driven-learning-redesign.md)
- [이전 학습안](doc/history/README.md)
