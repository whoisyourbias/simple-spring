# simple-tomcat 기반 Spring 학습 프레임워크 조사 계획과 결과

작성일: 2026-11-06 (Asia/Seoul). 범위: 조사·문서화. 제품 코드 구현 계획의 확정이나 구현 착수를 의미하지 않는다.

> 당시 조사 기록이다. 이후 사용자 지시로 학습 단계를 전면 재설계했다. 아래 00~13·문서 9개·소스 38개 등은 초기 조사 시점의 기록이며, 현재 순서·검증 결과는 [새 재설계 계획](2026-11-06-problem-driven-learning-redesign.md)을 따른다.

## 목적과 요청

`../simple-tomcat`을 내장 서버로 사용하는 학습용 Spring을 만들기 위해 기술 관계, 실제 Spring 호출 흐름, 서버 선행 조건, 구현 순서와 검증 기준을 조사한다. 기존 README의 주문 예제와 `minispring` 패키지 제안을 출발점으로 삼는다.

## 문서 작업 전에 확인한 기준 소스

- Spring Framework **v6.2.1**, Spring Boot **v3.4.1**, Apache Tomcat **10.1.34**.
- [Boot gradle.properties](https://github.com/spring-projects/spring-boot/blob/v3.4.1/gradle.properties): `springFrameworkVersion=6.2.1`, `tomcatVersion=10.1.34` 확인. 기존 simple-tomcat의 소스 기준과 일치하는 조합이므로 선택했다. 최신 릴리스나 운영용 권장 버전이라는 뜻은 아니다.
- [AbstractBeanFactory.doGetBean](https://github.com/spring-projects/spring-framework/blob/v6.2.1/spring-beans/src/main/java/org/springframework/beans/factory/support/AbstractBeanFactory.java): 캐시 조회 → 스코프별 생성 → 반환.
- [AbstractAutowireCapableBeanFactory.doCreateBean/initializeBean](https://github.com/spring-projects/spring-framework/blob/v6.2.1/spring-beans/src/main/java/org/springframework/beans/factory/support/AbstractAutowireCapableBeanFactory.java): 인스턴스화 → 주입 → 초기화 전 후처리 → init → 초기화 후 후처리.
- [AbstractApplicationContext.refresh](https://github.com/spring-projects/spring-framework/blob/v6.2.1/spring-context/src/main/java/org/springframework/context/support/AbstractApplicationContext.java): 정의 후처리 → 빈 후처리기 등록 → onRefresh → 나머지 싱글턴 초기화 → finishRefresh. 실패 시 생성된 싱글턴 정리.
- [DispatcherServlet.doDispatch](https://github.com/spring-projects/spring-framework/blob/v6.2.1/spring-webmvc/src/main/java/org/springframework/web/servlet/DispatcherServlet.java): Mapping → Adapter → preHandle → handler → postHandle → 결과·예외 처리 → afterCompletion.
- [ServletWebServerApplicationContext.onRefresh/createWebServer](https://github.com/spring-projects/spring-boot/blob/v3.4.1/spring-boot-project/spring-boot/src/main/java/org/springframework/boot/web/servlet/context/ServletWebServerApplicationContext.java), [WebServerStartStopLifecycle.start](https://github.com/spring-projects/spring-boot/blob/v3.4.1/spring-boot-project/spring-boot/src/main/java/org/springframework/boot/web/servlet/context/WebServerStartStopLifecycle.java): refresh 중 서버 준비와 lifecycle 단계의 요청 수락 시작을 구별한다.
- [ServletHttpHandlerAdapter.service](https://github.com/spring-projects/spring-framework/blob/v6.2.1/spring-web/src/main/java/org/springframework/http/server/reactive/ServletHttpHandlerAdapter.java): startAsync → reactive 요청·응답 래핑 → subscribe. ReadListener/WriteListener 사용도 요청·응답 소스에서 확인했다.
- AOP, 트랜잭션, 설정 클래스, MVC 반환값·오류의 추가 조사 위치는 [소스 대응표](../../../doc/reference/03-learning-stage-mapping.md)에 모은다. 조사한 메서드 외 모든 분기의 동등성을 주장하지 않는다.

## 작업 순서와 산출물

1. simple-spring의 파일·Git 상태와 simple-tomcat의 POM·현재 Java 코드·단계 문서를 읽는다. 완료 여부는 코드와 문서를 구별한다.
2. 위 태그의 실제 Spring 소스를 읽고, 공식 문서를 계약 설명에 보조로 사용한다.
3. Spring 생태계의 관계를 `doc/00-overview/02-spring-ecosystem.md`에 정리한다.
4. bottom-up 단계, 목적·정상·실패 완료 기준을 `doc/00-overview/03-implementation-roadmap.md`에 제안한다.
5. 현재 서버와 미래 내장 서버의 차이·선행 계약을 `doc/reference/01-embedded-tomcat-contract.md`에 기록한다.
6. 현황·차이 목록·소스 대응·목차를 지정된 문서 구조에 작성한다. 빈 학습 단계 폴더는 만들지 않는다.
7. 문서 링크·소스 태그·변경 범위를 검증하고 이 계획에 실제 결과를 덧붙인다.

## 일치시킬 핵심 동작과 향후 검증 기준

| 핵심 | 정상 기준 | 경계·실패 기준 |
|---|---|---|
| 빈 정의·스코프 | 같은 이름의 singleton은 같은 최종 객체, prototype은 매 조회 생성 | 같은 클래스의 다른 정의는 별개; 생성 실패 객체를 캐시하지 않음 |
| 생성자 DI | 의존 빈부터 생성해 주입 | 후보 없음·해결 불가능한 복수 후보·생성자 순환은 실패; 원인과 주입 지점 보존 |
| refresh·생명주기 | 정의 후처리 후 빈 후처리기 등록, init 후 최종 객체 노출 | refresh 실패 정리; dependent를 dependency보다 먼저 파괴; 반복 close |
| 프록시 | 최종 프록시를 조회·주입, advice chain을 거쳐 대상 호출 | 자기 호출은 프록시 우회; 대상 예외가 반사 래퍼로 바뀌지 않음 |
| Servlet·MVC | 서버는 Servlet 선택, MVC는 업무 handler 선택; Controller는 빈으로 조회 | 중복 매핑 시작 실패, 404/405 구별, 중단·예외별 interceptor 완료 순서 |
| Boot식 실행 | 서버 준비와 요청 수락을 구별, refresh 완료 경로에서 시작 | 빈 초기화·포트 충돌 시 서버·스레드·빈 정리; factory 0개·복수 실패 |
| 자동 구성 | 사용자 정의를 먼저 반영하고 조건이 맞을 때 기본 정의 등록 | 생성된 객체만 보고 누락을 판단하지 않음; 사용자 빈이 기본값을 대체 |
| JDBC 트랜잭션 | REQUIRED 참여·동일 DataSource 연결 공유·최외곽 commit | 기본 unchecked rollback, checked commit, 내부 rollback-only 후 외부 완료 실패·자원 정리 |
| reactive 확장 | 비동기 수명·수요 기반 전송·취소 전파 | service 반환 후 연결 유지, 수요 없는 전송 금지, 취소·오류·종료 시 정리 |

이 표는 미래 구현의 검증 요구사항이다. 이번 조사에서 Java 테스트가 통과했다는 기록이 아니다.

## 차이·생략 관리와 구현 진입 조건

- 자체 Servlet API는 Jakarta Servlet 바이너리 호환이 아니다. 실제 Spring JAR을 현재 서버에 끼우는 방식을 전제로 삼지 않는다.
- 기존 simple-tomcat은 요청 본문과 비동기를 의도적으로 제외했다. JSON 요청·스트리밍·WebFlux를 넣으려면 서버 쪽 별도 소스 조사·계획과 구현이 먼저 필요하다. 이를 단순화로 숨기지 않는다.
- 직접 재구현 / 실제 Spring 연결 / 혼합 방식의 대안을 조사하고, 추천과 사용자 확정 사항을 구분한다. 이번 요청은 조사이므로 추천안을 확정 설계로 표현하지 않는다.
- 차이별 실제 동작·선택·이유·영향·후속은 [구현 현황](../../../doc/reference/02-implementation-status.md)에 기록한다.
- 요청·세션 스코프, CGLIB 전체 동작, 전체 Servlet 사양, 환경 속성 우선순위 전체, 실제 Reactive Streams 구현 세부는 후속 조사 대상이다. 해당 기능 구현 전 소스를 추가 확인한다.
- 후속 구현은 해당 단계의 파일·API·테스트를 정한 설계와 상세 구현 계획을 먼저 저장한 뒤 진행한다.

## 이번 조사 완료 기준

- Spring/Framework/Boot/MVC/WebFlux 관계와 관련 프로젝트의 역할을 구분할 수 있다.
- 현재 가능한 학습 작업과 Tomcat 선행 작업을 구별한다.
- 시작·요청·종료·트랜잭션 흐름에 실제 클래스·메서드·태그 링크가 있다.
- 단계마다 정상·경계·실패 검증 기준이 있고 차이와 미확인 사항이 드러난다.
- 문서의 상대 링크가 유효하고 구현 완료를 오기하지 않는다.

## 실행·검증 결과

2026-11-06 조사·문서 작성 완료. 제품 코드 작성, 빌드 설정 변경, 의존성 설치, simple-tomcat 수정은 수행하지 않았다.

### 실제 산출물

- 기존 README에 새 조사 문서로 가는 안내를 추가하고 기존 주문 실습안은 보존했다.
- doc 목차, 진행 요약, 생태계 정리, 00~13단계 로드맵, 내장 서버 연동 계약, 구현 현황, 소스 대응표를 작성했다.
- 소스 대응표는 고정 태그의 38개 파일·대표 위치를 연결하며 빈 생성·Context·MVC·Boot·트랜잭션·WebFlux 흐름을 설명한다.
- 이 문서는 이번 조사 계획과 실제 결과를 기록한다. 이후 구현에 사용할 상세 실행 계획을 대신하지 않는다.

### 확인과 결과

| 확인 | 실행·관찰 결과 |
|---|---|
| 로컬 코드·계획 조사 | simple-spring src 파일 없음, simple-tomcat 현재 코드와 3~7단계 계획의 구별 확인 |
| 버전 조합 | Boot v3.4.1 gradle.properties에서 Framework 6.2.1·Tomcat 10.1.34 직접 확인 |
| 소스 위치 | 연결 도구로 읽은 태그 소스에서 대표 38개 anchor의 실제 줄·선언 확인 |
| 문서 검증 | README·doc·docs의 Markdown 9개, 상대 링크 30개, 끊어진 링크·code fence·문자 인코딩 오류 0개 |
| 변경 범위 | 이 작업은 simple-spring의 Markdown만 작성·수정. 서버 최종 HEAD는 cbfef0e이며 최초 HEAD와 문서 관련 6개 파일만 차이, Java 소스 차이 없음. 서버에 쓰기·커밋하지 않음 |
| Java 테스트·빌드 | 실행하지 않음. 제품 코드·빌드 파일이 없고 이번 범위는 조사임 |

문서 검증은 PowerShell here-string으로 아래 Python을 `python -`에 전달해 수행했다. 원격 소스 anchor는 위 연결 도구 조회 결과와 대조했으며 이 로컬 검사만으로 원격 링크를 검증한 것은 아니다.

```python
from pathlib import Path
import re
from urllib.parse import unquote
root = Path.cwd()
files = [root / 'README.md', *sorted((root/'doc').rglob('*.md')),
         *sorted((root/'docs').rglob('*.md'))]
links = 0
for path in files:
    content = path.read_text(encoding='utf-8')
    assert '\ufffd' not in content
    assert len(re.findall(r'^```', content, re.MULTILINE)) % 2 == 0
    assert content.endswith('\n')
    assert not any(line.rstrip() != line for line in content.splitlines())
    for target in re.findall(r'\]\(([^)]+)\)', content):
        target = target.strip('<>')
        if '://' in target or target.startswith('#'):
            continue
        target = unquote(target.split('#', 1)[0])
        if target:
            links += 1
            assert (path.parent / target).resolve().exists(), (path, target)
print(f'Markdown {len(files)} files; {links} local links; PASS')
```

### 남은 차이·미확인

현재 자체 Spring 구현은 없으며 테스트로 확인한 Spring 동등 동작도 없다. 자체 Servlet 비호환, Tomcat의 본문·async 미지원, 초기 bind 수명 차이와 세부 미조사 영역을 구현 현황에 명시했다. 추천 범위·구현 방식은 확정 설계로 기록하지 않았다. 이후 첫 구현은 00~01단계의 구체적 설계·계획을 저장한 뒤 진행한다.
