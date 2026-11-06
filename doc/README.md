# simple-spring 학습 문서

2026-11-06 단계 전면 재설계. **매 다음 단계가 이전 앱의 구체적인 문제를 해결하며 Spring 동작을 발견한다.** 현재는 문서 단계이며 Java 구현은 없다.

## 읽는 순서

| 순서 | 문서 | 담당 내용 |
|---|---|---|
| 1 | [진행 상황](00-overview/01-progress.md) | 현재 구현·서버 전제·다음 작업 |
| 2 | [기술 관계](00-overview/02-spring-ecosystem.md) | Spring 생태계의 위치와 역할 |
| 3 | [문제 해결 학습 순서](00-overview/03-implementation-roadmap.md) | P0와 00~19의 재현·해결·전후 검증·다음 문제 |
| 4 | [학습 단계 규약](00-overview/04-learning-workflow.md) | 매 단계가 갖춰야 할 증거와 기록 방식 |
| 선택 | [확장 과정](00-overview/05-extension-tracks.md) | E01~E05의 출발 단계·새 요구·진입 조건 |
| 참고 | [내장 Tomcat 계약](reference/01-embedded-tomcat-contract.md) | 서버의 실제 상태·소유권·선행 변경 |
| 참고 | [구현 현황](reference/02-implementation-status.md) | 지원·미지원·중간 모델·차이 |
| 참고 | [소스 대응표](reference/03-learning-stage-mapping.md) | 단계에 대응하는 고정 태그 클래스·메서드·흐름 |
| 작업 | [재설계 문서](../docs/superpowers/specs/2026-11-06-problem-driven-learning-design.md), [작업 계획·결과](../docs/superpowers/plans/2026-11-06-problem-driven-learning-redesign.md) | 이번 선택·변경·검증 |
| 기록 | [이전 학습안](history/README.md) | 폐기한 기능별 순서와 초기 예제의 보존 |

## 구조와 번호

```text
doc/
├─ README.md
├─ 00-overview/
│  ├─ 01-progress.md
│  ├─ 02-spring-ecosystem.md
│  ├─ 03-implementation-roadmap.md
│  ├─ 04-learning-workflow.md
│  └─ 05-extension-tracks.md
├─ reference/
│  ├─ 01-embedded-tomcat-contract.md
│  ├─ 02-implementation-status.md
│  └─ 03-learning-stage-mapping.md
└─ history/
   ├─ README.md
   ├─ 01-feature-first-roadmap.md
   └─ 02-initial-order-guide.md
```

P0는 서버·환경 선행 점검, 00은 수동 앱 기준선, 01~19는 누적 해결 순서다. E번호는 주 과정의 특정 상태에서 분기하는 선택 확장이다. 초기 00~13 순서는 현재 과정에서 사용하지 않는다.

학습 폴더 번호는 bottom-up 학습 순서, 파일 번호는 폴더 안 읽는 순서다. 구현을 시작하며 실제 설명이 필요할 때 해당 단계 폴더를 만든다. 빈 폴더는 만들지 않았다. 단계 이름은 기능명보다 해결할 문제를 드러내도록 정한다.

설계는 docs/superpowers/specs, 계획·실행 결과는 docs/superpowers/plans, 이전 상태는 history, 독립 실험은 필요할 때 lab, 원시 결과는 lab/results에 둔다. 읽는 순서는 simple-jorm 구조를 참고하고 역할은 AGENTS.md를 따른다.

## 기준 소스

Framework v6.2.1·Boot v3.4.1·Tomcat 10.1.34를 유지한다. [선정 근거](reference/03-learning-stage-mapping.md)를 참고한다. 실제 구현 전에는 해당 단계의 세부 분기를 추가 확인한다.
