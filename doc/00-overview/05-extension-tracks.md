# 남은 문제에서 출발하는 확장 과정

기준일: 2026-11-06. 필수 주 과정은 [00~19](03-implementation-roadmap.md)다. 아래 과정은 특정 단계의 산출물에 **새 요구를 추가했을 때** 시작한다. 번호 순으로 모두 구현해야 한다는 뜻이 아니다. 현재 전부 미구현이며 세부 구현 전에 별도 설계·소스 조사·계획이 필요하다.

공통 소스 기준은 Framework v6.2.1·Boot v3.4.1·Tomcat 10.1.34다. S번호는 [소스 대응표](../reference/03-learning-stage-mapping.md)를 따른다.

## E01 — 보고서 두 번의 상태가 섞이면 scope를 구분한다

- **출발 산출물:** 03 이후의 이름별 singleton factory. 아직 singleton만 제공한다.
- **한계 재현:** 누적 값을 갖는 ReportAccumulator를 조회해 보고서를 두 번 만든다. 같은 객체를 다시 사용해 두 번째 작업에 앞 작업 값이 남는 상황을 기록한다.
- **손으로 해볼 해결:** 매번 reset하거나 호출 지점에서 new한다. reset 누락과 컨테이너 밖 생성으로 돌아가는 비용을 비교한다.
- **최소 변경:** 조회마다 생성하는 prototype 정의를 지원한다. 공유 repository는 singleton, 작업별 accumulator는 prototype으로 선택한다.
- **전후 검증:** singleton은 assertSame, prototype은 assertNotSame. 두 보고서 값은 분리되고 constructor·init은 조회마다 호출된다.
- **경계·실패:** prototype을 singleton constructor에 한 번 주입하면 그 참조가 매 작업 자동 교체되지는 않는다. 필요한 시점의 명시 lookup과 고정 주입을 비교한다. prototype destroy는 컨테이너가 자동 관리하지 않는다.
- **Spring 대응:** S02·S03·S06. 스코프별 생성과 자동 파괴 등록 제외를 대조한다.
- **남은 문제·차이:** request/session scope·scoped proxy·Provider는 별도 문제와 소스 조사가 필요하다. “요청마다 새 객체”와 prototype을 같은 뜻으로 쓰지 않는다.

## E02 — 설정 method 직접 호출이 별도 객체를 만들면 full configuration을 조사한다

- **출발 산출물:** 06의 proxyBeanMethods=false에 대응하는 설정 method 등록.
- **한계 재현:** 설정 method 안에서 다른 설정 method를 직접 호출한다. 직접 반환한 객체와 getBean으로 받은 singleton 참조를 비교한다. lite configuration에서 다를 수 있는 정상 동작을 버그라고 부르지 않는다.
- **손으로 해볼 해결:** method 인자 주입으로 바꾸어 관리 빈을 받는다. 대체로 이 방법만으로 요구를 충족하는지 먼저 확인한다.
- **최소 변경:** 직접 호출도 관리 빈 조회로 연결해야 하는 별도 요구가 있을 때에만 configuration enhancement를 설계한다. class proxy와 일반 AOP proxy의 목적 차이를 확인한다.
- **전후 검증:** lite의 직접 호출과 인자 주입, full의 method 간 호출을 각각 검증한다. 이름별 singleton 정체성과 factory method의 실제 실행 횟수를 확인한다.
- **경계·실패:** final method/class, factory 자체 생성 중 호출, FactoryBean·scope와 결합한 분기는 추가 조사 전 지원하지 않는다.
- **Spring 대응:** S08·S09·S40. ConfigurationClassEnhancer.BeanMethodInterceptor가 실제 factory 호출과 bean 참조 호출을 구분하는 흐름을 읽는다.
- **남은 문제·차이:** CGLIB 도입·범위·버전은 별도 결정이다. 단순 리플렉션 호출을 full configuration이라고 선언하지 않는다.

## E03 — 주문만 반영되고 재고는 실패하면 트랜잭션 경계를 만든다

- **출발 산출물:** 10의 자동 service proxy 또는 19의 완주 앱. 저장소는 아직 인메모리이므로 JDBC로 옮기는 새 요구를 명시한다.
- **한계 재현:** 주문 저장 후 재고 차감에서 예외를 발생시키고 새 연결로 결과를 조회한다. 주문만 남은 부분 반영을 확인한다.
- **손으로 해볼 해결:** 먼저 같은 JDBC Connection으로 두 작업을 수동 commit/rollback한다. 다음으로 repository마다 연결을 따로 여는 방식에서는 이 해결이 유지되지 않는 것을 비교한다.
- **최소 변경:** DataSource별 자원 holder와 동기 TransactionManager로 연결 공유를 만들고, 반복되는 경계 코드를 service interceptor로 옮긴다. 어노테이션은 그 뒤에 붙인다.
- **전후 검증:** 한 연결·한 최외곽 commit, 실패 시 주문·재고 모두 원복, 두 repository의 연결 정체성을 검증한다. proxy 이전/이후 업무 코드의 수동 commit·rollback 중복 제거를 확인한다.
- **경계·실패:** 기본 RuntimeException/Error rollback·checked exception commit, 내부 REQUIRED 실패를 외부가 잡은 뒤 rollback-only와 최종 UnexpectedRollback, 자기 호출의 proxy 우회, 실패 후 unbind·설정 복원·반환을 검증한다.
- **Spring 대응:** S28~S32. TransactionAspectSupport→TransactionManager→DataSourceTransactionManager→DataSourceUtils 흐름이다.
- **남은 문제·차이:** REQUIRES_NEW·NESTED·JTA·reactive TX·동기화 전체는 별도 조사다. DB 드라이버 버전·fixture·simple-jorm 통합 여부도 구현 전에 고정한다.

## E04 — 같은 주문 데이터를 HTML로도 보여 줘야 하면 View를 분리한다

- **출발 산출물:** 15의 ResponseBody 문자열·JSON 응답과 16의 오류 처리.
- **한계 재현:** HTML 주문 화면 요구를 추가하면 method에 HTML 조립이 들어가고, 일반 Controller의 문자열 view 이름을 본문과 구분할 필요가 생긴다.
- **손으로 해볼 해결:** HTML 조립을 함수로 분리하고 Model과 view 이름을 반환해 본다.
- **최소 변경:** 일반 문자열 반환의 view 이름 처리와 ModelAndView·ViewResolver·View.render 경계를 추가한다. 첫 View는 응답에 직접 HTML을 쓰며 JSP·forward를 요구하지 않는다.
- **전후 검증:** 같은 주문 fixture로 JSON과 HTML을 각각 제공한다. ResponseBody 문자열은 본문, 일반 Controller 문자열은 view 이름으로 처리한다. rendering은 업무 method 밖에서 수행한다.
- **경계·실패:** 이름에 맞는 View 없음, 렌더링 실패, 오류 View와 이미 처리한 응답을 구별한다. 16·17의 resolver/completion 회귀를 유지한다.
- **Spring 대응:** S13의 render/resolveViewName·S19. 반환값 처리와 최종 렌더링은 같은 사건이 아니다.
- **남은 문제·차이:** 템플릿 엔진·locale 전체·redirect·forward·JSP는 별도 지원 범위다. 해당 기능이 없는 서버를 표준 MVC view 배포와 동일하다고 설명하지 않는다.

## E05 — 느린 응답과 소비자가 작업자를 점유하면 비동기 수명과 수요를 다룬다

- **출발 산출물:** 18 이후의 동기 Servlet 앱. Tomcat의 제한 worker가 있는 상태에서 실험한다.
- **한계 재현:** 완료를 기다리는 응답과 느린 클라이언트를 유한 시간·latch로 제어한다. worker 점유·다른 요청 대기·전송 버퍼 누적을 관찰한다. 실제 수치를 재기 전에 WebFlux가 더 빠르다고 결론내리지 않는다.
- **손으로 해볼 해결:** 별도 실행기/Future로 처리를 옮긴다. service 반환 때 연결이 닫히거나 취소 후 작업이 남는다면 실행기 이동만으로 수명 문제가 해결되지 않았음을 확인한다.
- **최소 변경의 순서:** (1) Servlet async의 service 반환/응답 완료 분리, (2) read/write readiness와 스트리밍·취소·timeout 계약, (3) 네트워크 없는 Publisher 수요 실험, (4) HttpHandler·exchange/filter 경계, (5) DispatcherHandler의 mapping/adapter/result 처리로 연결한다. 각 작업의 다음 입력은 앞에서 해결하지 못한 수명·수요 문제다.
- **전후 검증:** service 반환 뒤 응답 유지, demand 이하 전송, 느린 소비자에서 유계 버퍼, 취소·오류·종료 시 subscription·버퍼·연결 정리를 검증한다. 기존 동기 요청 회귀를 유지한다.
- **경계·실패:** listener readiness 경합·중복 완료·중간 전송 실패·취소 후 신호를 확인한다. blocking JDBC를 같은 event loop에서 실행해 전체가 논블로킹이라고 주장하지 않는다.
- **Spring 대응:** S33~S36. ServletHttpHandlerAdapter의 startAsync·subscribe, 요청/응답 listener, DispatcherHandler의 reactive 결과 연결을 대조한다.
- **진입 조건·미확인:** 현재 Tomcat은 async·논블로킹 API·스트리밍을 의도적으로 제외했다. 서버 범위를 바꾸는 별도 계획이 선행되어야 한다. Reactive Streams 상세·Reactor 연산자·버퍼 소유권·HttpHandler 전체 조립은 아직 미확인이므로 이 개요만으로 구현하지 않는다.

## 확장 과정의 완료 기록

선택한 확장의 목표·출발 checkpoint·새 요구·소스·상세 작업·검증을 별도 설계와 계획에 저장한다. 실행 후에만 현재 현황을 갱신한다. Security·Data·Cloud·Batch는 역할을 이해하는 자료로 유지하며 구현할 때는 같은 방식으로 새 문제부터 정의한다.
