# 학습 단계와 실제 Spring 소스 대응

조사·재설계일: 2026-11-06. 이 문서는 기준 버전·조사 메서드·호출 흐름의 출처를 관리한다. 단계별 목적과 테스트는 [로드맵](../00-overview/03-implementation-roadmap.md), 남은 차이는 [구현 현황](02-implementation-status.md)에 둔다.

## 1. 기준 버전과 선정 이유

| 대상 | 고정 기준 | 이유 |
|---|---|---|
| Spring Framework | v6.2.1 | Boot 3.4.1이 지정하는 버전 |
| Spring Boot | v3.4.1 | 같은 기준에서 Framework와 Tomcat 수명을 함께 읽을 수 있음 |
| Apache Tomcat | 10.1.34 | 기존 simple-tomcat 문서가 사용하는 기준 |
| 자체 서버 | 재설계 시 HEAD cbfef0e7ce71d68be31cfb987c9e9dc931b3514d | Java 코드와 계획된 Servlet API의 차이를 재확인. 초기 조사 스냅샷은 연동 계약에 보존 |

S01의 Boot gradle.properties에서 Framework 6.2.1과 Tomcat 10.1.34를 직접 확인했다. 기존 simple-spring에는 고정 Spring 버전이 없었다. 이번 선택은 최신 버전 선정이나 운영 버전 권장이 아니라 기존 Tomcat 학습 기준과 일관된 **소스 연구 기준**이다. 버전을 바꾸면 이 표와 각 단계의 핵심 동작·실패 테스트 영향을 함께 갱신한다.

링크는 main/current가 아닌 태그를 고정한다. 표의 행마다 대표 메서드 위치에 line anchor를 붙였으며, 같은 행의 추가 메서드는 같은 파일 안에서 확인한다. 전체 클래스·모든 조건 분기를 검증했다는 뜻은 아니다.

## 2. 조사한 소스

단계 열은 전면 재설계한 현재 00~19·E01~E05를 따른다. S01~S38의 ID는 유지하고 S39~S42를 추가했다. history의 이전 단계 번호와 혼동하지 않는다. 02는 완성 singleton 수동 등록, 03부터 정의 기반 생성이며 07의 callback과 10의 BPP를 구분한다.

| ID | 로드맵 단계 | 실제 파일·클래스·메서드 | 확인한 흐름·범위 |
|---|---|---|---|
| S01 | P0·18 | [gradle.properties](https://github.com/spring-projects/spring-boot/blob/v3.4.1/gradle.properties#L22) | Framework·Tomcat 버전 고정 값 |
| S02 | 03·04·07·E01 | [AbstractBeanFactory.doGetBean / registerDisposableBeanIfNecessary](https://github.com/spring-projects/spring-framework/blob/v6.2.1/spring-beans/src/main/java/org/springframework/beans/factory/support/AbstractBeanFactory.java#L239) | 캐시·스코프별 생성, prototype의 자동 파괴 제외 |
| S03 | 04·07·10 | [AbstractAutowireCapableBeanFactory.doCreateBean / initializeBean](https://github.com/spring-projects/spring-framework/blob/v6.2.1/spring-beans/src/main/java/org/springframework/beans/factory/support/AbstractAutowireCapableBeanFactory.java#L554) | 인스턴스화·주입·초기화·최종 객체, early reference 분기 |
| S04 | 04·08 | [DefaultListableBeanFactory.doResolveDependency / preInstantiateSingletons](https://github.com/spring-projects/spring-framework/blob/v6.2.1/spring-beans/src/main/java/org/springframework/beans/factory/support/DefaultListableBeanFactory.java#L1524) | 후보 해결과 실패, non-lazy singleton 선행 생성 |
| S05 | 04·06 | [ConstructorResolver.autowireConstructor](https://github.com/spring-projects/spring-framework/blob/v6.2.1/spring-beans/src/main/java/org/springframework/beans/factory/support/ConstructorResolver.java#L135) | 생성자 후보·매개변수 해결 경로 |
| S06 | 02·03·04·07·08 | [DefaultSingletonBeanRegistry.registerSingleton / beforeSingletonCreation / destroySingletons / destroyBean](https://github.com/spring-projects/spring-framework/blob/v6.2.1/spring-beans/src/main/java/org/springframework/beans/factory/support/DefaultSingletonBeanRegistry.java#L419) | 생성 중 재진입 검사·의존 빈 우선 파괴 |
| S07 | 05 | [ClassPathBeanDefinitionScanner.doScan](https://github.com/spring-projects/spring-framework/blob/v6.2.1/spring-context/src/main/java/org/springframework/context/annotation/ClassPathBeanDefinitionScanner.java#L273) | 후보 → 이름·scope → 정의 등록 |
| S08 | 06·08·19 | [ConfigurationClassPostProcessor.postProcessBeanDefinitionRegistry / postProcessBeanFactory](https://github.com/spring-projects/spring-framework/blob/v6.2.1/spring-context/src/main/java/org/springframework/context/annotation/ConfigurationClassPostProcessor.java#L278) | 설정 정의 처리와 configuration enhancement |
| S09 | 06·E02 | [ConfigurationClassEnhancer.BeanMethodInterceptor.intercept](https://github.com/spring-projects/spring-framework/blob/v6.2.1/spring-context/src/main/java/org/springframework/context/annotation/ConfigurationClassEnhancer.java#L301) | 설정 factory 호출과 사용자 직접 호출을 구별 |
| S10 | 08·10·18 | [AbstractApplicationContext.refresh / finishRefresh / doClose](https://github.com/spring-projects/spring-framework/blob/v6.2.1/spring-context/src/main/java/org/springframework/context/support/AbstractApplicationContext.java#L587) | 후처리·onRefresh·일반 빈 생성·lifecycle·종료 |
| S11 | 10 | [AbstractAutoProxyCreator.postProcessAfterInitialization / wrapIfNecessary](https://github.com/spring-projects/spring-framework/blob/v6.2.1/spring-aop/src/main/java/org/springframework/aop/framework/autoproxy/AbstractAutoProxyCreator.java#L315) | 일반 빈 초기화 후 적용 대상 판정·프록시 생성 |
| S12 | 09·10·E03 | [JdkDynamicAopProxy.invoke](https://github.com/spring-projects/spring-framework/blob/v6.2.1/spring-aop/src/main/java/org/springframework/aop/framework/JdkDynamicAopProxy.java#L167) | 인터셉터 체인·대상 호출·반환과 예외 경계 |
| S13 | 11·12·15·16·17·E04 | [DispatcherServlet.doDispatch / processDispatchResult / processHandlerException / render](https://github.com/spring-projects/spring-framework/blob/v6.2.1/spring-webmvc/src/main/java/org/springframework/web/servlet/DispatcherServlet.java#L1048) | mapping·adapter·interceptor·결과·예외·완료 |
| S14 | 13 | [AbstractHandlerMethodMapping.MappingRegistry.validateMethodMapping](https://github.com/spring-projects/spring-framework/blob/v6.2.1/spring-webmvc/src/main/java/org/springframework/web/servlet/handler/AbstractHandlerMethodMapping.java#L668) | 동일 조건에 서로 다른 handler 등록 시 실패 |
| S15 | 13 | [RequestMappingHandlerMapping.getMappingForMethod](https://github.com/spring-projects/spring-framework/blob/v6.2.1/spring-webmvc/src/main/java/org/springframework/web/servlet/mvc/method/annotation/RequestMappingHandlerMapping.java#L316) | 클래스·메서드 매핑 조건 결합 |
| S16 | 12·14·15 | [RequestMappingHandlerAdapter.invokeHandlerMethod](https://github.com/spring-projects/spring-framework/blob/v6.2.1/spring-webmvc/src/main/java/org/springframework/web/servlet/mvc/method/annotation/RequestMappingHandlerAdapter.java#L936) | argument/return resolver·binder·handler 실행 연결 |
| S17 | 17 | [HandlerExecutionChain.applyPreHandle / applyPostHandle / triggerAfterCompletion](https://github.com/spring-projects/spring-framework/blob/v6.2.1/spring-webmvc/src/main/java/org/springframework/web/servlet/HandlerExecutionChain.java#L143) | 정순 pre, 역순 post, 성공한 pre만 역순 completion |
| S18 | 15 | [RequestResponseBodyMethodProcessor.handleReturnValue](https://github.com/spring-projects/spring-framework/blob/v6.2.1/spring-webmvc/src/main/java/org/springframework/web/servlet/mvc/method/annotation/RequestResponseBodyMethodProcessor.java#L190) | requestHandled 표시와 message converter 쓰기 |
| S19 | 15·E04 | [ViewNameMethodReturnValueHandler.handleReturnValue](https://github.com/spring-projects/spring-framework/blob/v6.2.1/spring-webmvc/src/main/java/org/springframework/web/servlet/mvc/method/annotation/ViewNameMethodReturnValueHandler.java#L78) | CharSequence 반환값을 view name으로 처리 |
| S20 | 13·14·15 | [RequestMappingInfoHandlerMapping.handleNoMatch](https://github.com/spring-projects/spring-framework/blob/v6.2.1/spring-webmvc/src/main/java/org/springframework/web/servlet/mvc/method/RequestMappingInfoHandlerMapping.java#L248) | 경로 부분 일치 후 method/consumes/produces 불일치 구별 |
| S21 | 18 | [SpringApplication.run](https://github.com/spring-projects/spring-boot/blob/v3.4.1/spring-boot-project/spring-boot/src/main/java/org/springframework/boot/SpringApplication.java#L301) | environment·context·refresh·runners·실패 처리 |
| S22 | 18 | [ServletWebServerApplicationContext.onRefresh / createWebServer / doClose](https://github.com/spring-projects/spring-boot/blob/v3.4.1/spring-boot-project/spring-boot/src/main/java/org/springframework/boot/web/servlet/context/ServletWebServerApplicationContext.java#L159) | factory 단일 선택·서버 준비·lifecycle 등록·destroy |
| S23 | 18 | [TomcatServletWebServerFactory.getWebServer](https://github.com/spring-projects/spring-boot/blob/v3.4.1/spring-boot-project/spring-boot/src/main/java/org/springframework/boot/web/embedded/tomcat/TomcatServletWebServerFactory.java#L199) | Tomcat·connector·context 구성과 initializer 연결 |
| S24 | 18 | [TomcatWebServer.initialize / start](https://github.com/spring-projects/spring-boot/blob/v3.4.1/spring-boot-project/spring-boot/src/main/java/org/springframework/boot/web/embedded/tomcat/TomcatWebServer.java#L110) | connector 분리·초기 bind 지연, start에서 복원 |
| S25 | 18 | [WebServerStartStopLifecycle.start / stop](https://github.com/spring-projects/spring-boot/blob/v3.4.1/spring-boot-project/spring-boot/src/main/java/org/springframework/boot/web/servlet/context/WebServerStartStopLifecycle.java#L43) | lifecycle에서 WebServer 시작·중단 |
| S26 | 19 | [AutoConfigurationImportSelector.getAutoConfigurationEntry / getCandidateConfigurations](https://github.com/spring-projects/spring-boot/blob/v3.4.1/spring-boot-project/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/AutoConfigurationImportSelector.java#L136) | imports 후보·중복·제외·필터 경로 |
| S27 | 19 | [OnBeanCondition.getMatchOutcome / evaluateConditionalOnMissingBean](https://github.com/spring-projects/spring-boot/blob/v3.4.1/spring-boot-project/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/condition/OnBeanCondition.java#L122) | bean 조건과 missing-bean back-off 판정 |
| S28 | E03 | [TransactionAspectSupport.invokeWithinTransaction](https://github.com/spring-projects/spring-framework/blob/v6.2.1/spring-tx/src/main/java/org/springframework/transaction/interceptor/TransactionAspectSupport.java#L342) | 경계 시작·대상 호출·성공/실패 완료·정보 복원 |
| S29 | E03 | [AbstractPlatformTransactionManager.getTransaction / commit / processRollback](https://github.com/spring-projects/spring-framework/blob/v6.2.1/spring-tx/src/main/java/org/springframework/transaction/support/AbstractPlatformTransactionManager.java#L373) | REQUIRED 참여, rollback-only와 UnexpectedRollback |
| S30 | E03 | [DefaultTransactionAttribute.rollbackOn](https://github.com/spring-projects/spring-framework/blob/v6.2.1/spring-tx/src/main/java/org/springframework/transaction/interceptor/DefaultTransactionAttribute.java#L185) | 기본 RuntimeException/Error rollback 규칙 |
| S31 | E03 | [DataSourceTransactionManager.doBegin / doCleanupAfterCompletion](https://github.com/spring-projects/spring-framework/blob/v6.2.1/spring-jdbc/src/main/java/org/springframework/jdbc/datasource/DataSourceTransactionManager.java#L262) | Connection 획득·설정·thread binding·복원·반환 |
| S32 | E03 | [DataSourceUtils.doGetConnection](https://github.com/spring-projects/spring-framework/blob/v6.2.1/spring-jdbc/src/main/java/org/springframework/jdbc/datasource/DataSourceUtils.java#L103) | 동일 DataSource의 thread-bound 연결 재사용 |
| S33 | E05 | [DispatcherHandler.handle / handleResultMono / handleResult](https://github.com/spring-projects/spring-framework/blob/v6.2.1/spring-webflux/src/main/java/org/springframework/web/reactive/DispatcherHandler.java#L142) | reactive mapping·adapter·result handler 연결 |
| S34 | E05 | [ServletHttpHandlerAdapter.service](https://github.com/spring-projects/spring-framework/blob/v6.2.1/spring-web/src/main/java/org/springframework/http/server/reactive/ServletHttpHandlerAdapter.java#L157) | async 시작·래핑·구독·완료/오류 listener 연결 |
| S35 | E05 | [ServletServerHttpRequest.RequestBodyPublisher.registerReadListener](https://github.com/spring-projects/spring-framework/blob/v6.2.1/spring-web/src/main/java/org/springframework/http/server/reactive/ServletServerHttpRequest.java#L337) | read readiness listener 등록 |
| S36 | E05 | [ServletServerHttpResponse 생성자 / ResponseBodyWriteListener](https://github.com/spring-projects/spring-framework/blob/v6.2.1/spring-web/src/main/java/org/springframework/http/server/reactive/ServletServerHttpResponse.java#L97) | write readiness listener 등록 |
| S37 | P0·00·18 | [Tomcat.addServlet / start](https://github.com/apache/tomcat/blob/10.1.34/java/org/apache/catalina/startup/Tomcat.java#L374) | 전달받은 Servlet을 Wrapper로 감싸 Context 등록 |
| S38 | 13·14·16 | [DefaultHandlerExceptionResolver.doResolveException / handleErrorResponse](https://github.com/spring-projects/spring-framework/blob/v6.2.1/spring-webmvc/src/main/java/org/springframework/web/servlet/mvc/support/DefaultHandlerExceptionResolver.java#L555) | 표준 MVC 예외의 상태·header 응답 처리 |
| S39 | 05 | [ClassPathScanningCandidateComponentProvider.scanCandidateComponents](https://github.com/spring-projects/spring-framework/blob/v6.2.1/spring-context/src/main/java/org/springframework/context/annotation/ClassPathScanningCandidateComponentProvider.java#L452) | metadata 탐색과 빈 인스턴스 생성을 분리 |
| S40 | 06·E02 | [ConfigurationClassBeanDefinitionReader.loadBeanDefinitionsForBeanMethod](https://github.com/spring-projects/spring-framework/blob/v6.2.1/spring-context/src/main/java/org/springframework/context/annotation/ConfigurationClassBeanDefinitionReader.java#L180) | factory 위치·method·반환 타입·callback metadata를 정의에 등록 |
| S41 | 10 | [BeanPostProcessor.postProcessBeforeInitialization / postProcessAfterInitialization](https://github.com/spring-projects/spring-framework/blob/v6.2.1/spring-beans/src/main/java/org/springframework/beans/factory/config/BeanPostProcessor.java#L100) | 명시 등록과 자동 탐색, 초기화 전후 wrapper 반환 계약 |
| S42 | 14 | [InvocableHandlerMethod.invokeForRequest / getMethodArgumentValues](https://github.com/spring-projects/spring-framework/blob/v6.2.1/spring-web/src/main/java/org/springframework/web/method/support/InvocableHandlerMethod.java#L177) | 인자 해결·지원 검증·대상 호출, resolver 없음 실패 |

## 3. 빈 생성·노출 흐름

```text
getBean
 → AbstractBeanFactory.doGetBean
   → singleton cache 확인
   → scope에 따른 생성 경로
   → AbstractAutowireCapableBeanFactory.createBean / doCreateBean
     → createBeanInstance
       → 필요 시 ConstructorResolver → resolveDependency → 의존 빈 getBean
     → populateBean
     → initializeBean
       → Aware
       → 초기화 전 BeanPostProcessor
       → init callbacks
       → 초기화 후 BeanPostProcessor
         → 일반적인 auto-proxy 경로
     → 파괴 callback 등록
   → 최종 객체 반환·singleton 등록
```

이 도식은 초기 인스턴스화 이전의 shortcut·FactoryBean·순환 노출 등 모든 경로를 나열한 것이 아니다. S03에는 early reference 노출과 최종 proxy 일관성 검사도 존재한다. 초기 학습은 순환 없는 생성자 주입을 지원 입력으로 삼고 그 밖을 같은 동작이라고 선언하지 않는다.

S06의 파괴는 단순 역등록 순서만이 아니다. destroySingletons의 역순 순회에 더해 destroyBean이 해당 빈을 의존하는 다른 빈(dependent)을 먼저 정리한다. prototype 파괴는 S02의 등록 경로에서 제외된다.

## 4. Context 시작·종료 흐름

```text
AbstractApplicationContext.refresh
 → prepareRefresh / obtainFreshBeanFactory / prepareBeanFactory
 → postProcessBeanFactory
 → invokeBeanFactoryPostProcessors
 → registerBeanPostProcessors
 → message source / event multicaster 준비
 → onRefresh
 → registerListeners
 → finishBeanFactoryInitialization
   → 나머지 non-lazy singleton 생성
 → finishRefresh
   → lifecycleProcessor.onRefresh
   → ContextRefreshedEvent
```

refresh 도중 실패하면 생성된 singleton을 파괴하고 활성 상태를 취소한 뒤 예외를 전파한다. close는 ContextClosedEvent → lifecycle 중단 → singleton 파괴 → factory 닫기·추가 정리 순서다. 세부 callback 종류·재시도·반복 refresh 계약은 아직 전체 조사하지 않았다.

## 5. MVC 요청과 완료 흐름

```text
DispatcherServlet.doDispatch
 → getHandler(HandlerMapping)
 → getHandlerAdapter
 → applyPreHandle
 → HandlerAdapter.handle
   → annotated handler: argument resolver / binder
   → 관리 Controller 메서드
   → return value handler
     → ResponseBody: message converter
     → View 경로: ModelAndView
 → applyPostHandle
 → processDispatchResult
   → 필요 시 exception resolver
   → 필요 시 render
 → triggerAfterCompletion
```

pre는 등록 순서, post와 completion은 역순이다. pre가 false면 해당 interceptor 자신은 completion 대상이 아니며 앞서 true를 반환한 interceptor만 정리한다. handler 예외는 post를 건너뛰고 resolver 경로로 간다. 예외가 해결되면 정상 완료 경로의 completion 인자는 null일 수 있고, 미해결이면 예외를 전달하며 바깥으로 전파한다. S13·S17의 분기를 구별해 검증한다.

S14는 동일 mapping의 등록 충돌을 검사한다. S20은 경로 부분 일치 뒤 method·consumes·produces 불일치를 구별한다. 모든 경로 매칭·동률 판정 알고리즘을 조사한 것은 아니다.

## 6. Boot 내장 서버와 자동 구성 흐름

```text
SpringApplication.run
 → prepareEnvironment
 → createApplicationContext / prepareContext
 → refreshContext
   → onRefresh: ServletWebServerApplicationContext.createWebServer
     → ServletWebServerFactory 하나 선택
     → TomcatServletWebServerFactory.getWebServer
       → Tomcat/Connector/Context 구성
       → TomcatWebServer.initialize
         → connector 분리, 초기 bind 지연
         → 내부 Tomcat 시작으로 initializer 실행
     → WebServer lifecycle 등록
   → 나머지 일반 singleton 초기화
   → finishRefresh의 lifecycle
     → WebServerStartStopLifecycle.start
       → TomcatWebServer.start: connector 복원·시작 확인
 → started listeners / runners / ready listeners
```

서버 초기화와 실제 요청 수락 시작의 차이를 보존해야 한다. onRefresh 중 initializer 처리에서 일부 빈 생성이 일찍 발생할 수 있다. 모든 빈을 먼저 만든 뒤 단순히 main에서 서버를 띄우는 모델과 동일하지 않다.

자동 구성은 별도의 정의 등록 흐름이다. S26에서 imports 후보 → 중복·exclude·필터를 확인했고 S27에서 bean 조건 판정을 확인했다. 사용자 정의가 보이는 시점과 자동 구성 간 순서가 결과에 영향을 준다. 세부 정렬·모든 condition 구현은 추가 조사 대상이다.

## 7. 동기 JDBC 트랜잭션 흐름

```text
서비스 proxy
 → TransactionAspectSupport.invokeWithinTransaction
 → TransactionManager.getTransaction
   → 기존 REQUIRED 자원 참여 또는 새 자원 시작
   → DataSourceTransactionManager.doBegin
     → Connection 확보 / autoCommit 등 조정 / thread binding
 → target service
   → Repository: DataSourceUtils.doGetConnection
     → 동일 DataSource의 bound Connection 재사용
 → 성공 commit 또는 실패 rollback 규칙 판정
 → transaction 정보 복원 / Connection unbind·설정 복원·반환
```

S30의 기본 rollback 대상은 RuntimeException과 Error다. S29에서 참여 실패의 rollback-only 표시와 최외곽 commit 시 UnexpectedRollback 처리를 확인했다. thread별 Connection 하나라는 설명만으로는 부족하며 DataSource별 자원과 참여·소유자의 구별이 필요하다. reactive transaction은 같은 ThreadLocal 설계를 그대로 적용하지 않는다.

## 8. WebFlux와 Servlet 어댑터 흐름

```text
ServletHttpHandlerAdapter.service
 → request.startAsync
 → ServletServerHttpRequest / Response
   → read/write listener
 → HttpHandler.handle(...).subscribe(...)
 → WebHandler / filter·exception 처리 구성
 → DispatcherHandler
   → reactive HandlerMapping
   → reactive HandlerAdapter
   → HandlerResultHandler
 → 비동기 완료·오류·취소와 응답 수명 정리
```

S34~S36에서 async·listener의 필요를 확인했고 S33에서 reactive dispatch를 확인했다. 중간 HttpHandler/WebHandler 조립 전체와 Reactive Streams 상세 규칙은 이번 조사에서 모두 추적하지 않았다. 해당 계층을 구현하기 전 추가 조사해야 한다.

## 9. 보조 자료와 조사 한계

- [Framework 공식 개요](https://docs.spring.io/spring-framework/reference/6.2/overview.html): 프로젝트·모듈 관계 설명.
- [WebFlux 공식 비교](https://docs.spring.io/spring-framework/reference/6.2/web/webflux/new-framework.html): MVC와 실행 모델·서버 선택 비교.
- [Boot 자동 구성 설명](https://docs.spring.io/spring-boot/3.4/reference/using/auto-configuration.html): 사용자 설정과 기본 구성의 관계.
- 생태계 프로젝트의 소개는 [생태계 문서](../00-overview/02-spring-ecosystem.md)의 공식 프로젝트 링크를 참고한다. Security·Data·Cloud·Batch 내부를 소스 조사한 것으로 확대하지 않는다.

공식 문서의 버전 계열 URL은 패치가 바뀔 수 있으므로 동작 구현의 최종 근거는 위 태그의 소스다. 남은 미확인은 [구현 현황](02-implementation-status.md)에 관리한다.
