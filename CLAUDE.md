# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 프로젝트 개요

"Get Your Hands Dirty on Clean Architecture" (2판) 책의 예제 코드. Spring Boot 3.1 + Java 17 기반의 송금(`buckpal`) 도메인을 헥사고날(포트 & 어댑터) 아키텍처 스타일로 구현한다. 코드 자체보다 **레이어 간 의존성 방향**과 **패키지 경계**가 핵심 학습 대상이다.

## 빌드·실행·테스트

Gradle Wrapper(`./gradlew`)를 사용한다.

- 전체 빌드 + 테스트: `./gradlew build`
- 테스트만 실행: `./gradlew test`
- 단일 테스트 클래스: `./gradlew test --tests "io.reflectoring.buckpal.application.domain.service.SendMoneyServiceTest"`
- 단일 테스트 메서드: `./gradlew test --tests "io.reflectoring.buckpal.application.domain.service.SendMoneyServiceTest.givenWithdrawalFails_thenOnlySourceAccountIsLockedAndReleased"`
- 애플리케이션 실행: `./gradlew bootRun` (in-memory H2 사용)
- 빌드 리포트: 실패 시 `build/reports/`

CI는 `.github/workflows/ci.yml`에서 Java 17로 `./gradlew build`를 수행한다.

## 아키텍처

### 패키지 토폴로지

```
io.reflectoring.buckpal
├── application
│   ├── domain
│   │   ├── model       ← 순수 도메인 엔티티(Account, Money, Activity, ActivityWindow)
│   │   └── service     ← 유스케이스 구현 (SendMoneyService 등)
│   └── port
│       ├── in          ← 인커밍 포트(유스케이스 인터페이스, Command record)
│       └── out         ← 아웃고잉 포트(LoadAccountPort, UpdateAccountStatePort, AccountLock)
├── adapter
│   ├── in.web          ← REST 컨트롤러
│   └── out.persistence ← JPA 어댑터, 매퍼, 엔티티
└── common              ← @UseCase, @WebAdapter, @PersistenceAdapter 스테레오타입 + Validation 유틸
```

루트(`BuckPalApplication`, `BuckPalConfiguration`, `BuckPalConfigurationProperties`)는 **configuration 레이어**로 취급된다. ArchUnit 테스트는 이 패키지를 `withConfiguration("configuration")`으로 등록하지만 실제 디렉터리는 루트(`io.reflectoring.buckpal`)에 위치한다.

### 의존성 규칙 (강제됨)

`DependencyRuleTests`가 ArchUnit으로 다음을 검증한다:

1. `application.domain.model..`은 자기 자신 + `lombok..` + `java..` 외에는 어떤 패키지에도 의존하지 않는다 (Spring 어노테이션조차 금지).
2. 도메인 → 어댑터 의존 금지.
3. 어댑터 ↔ 어댑터 상호 의존 금지.
4. 어댑터 → configuration 의존 금지.
5. application 레이어 → 어댑터/configuration 의존 금지.
6. `port.in`과 `port.out`은 서로 의존하지 않는다.

새 패키지/클래스를 추가할 때는 위 규칙을 깨지 않는지 먼저 확인할 것. 빌드가 깨지는 가장 흔한 이유가 이 테스트다.

### 커스텀 스테레오타입

`@UseCase`, `@WebAdapter`, `@PersistenceAdapter`는 모두 `@Component`를 메타-어노테이션으로 갖는 마커다. 새 서비스/어댑터를 만들 때 일반 `@Service`, `@Repository`, `@RestController` 대신 이 어노테이션들을 함께 사용해 아키텍처 의도를 코드에 드러낸다.

### 입력 검증 패턴

`SendMoneyCommand`(record)는 컴팩트 컨스트럭터 안에서 `Validation.validate(this)`를 호출해 Bean Validation 어노테이션(`@NotNull`, `@PositiveMoney`)을 즉시 평가한다. 새 Command를 만들 때 같은 패턴을 따르고, 도메인/포트 입력의 유효성을 어댑터 레이어에 미루지 말 것.

### 설정값

- `buckpal.transferThreshold` (`application.yml`에서 기본 10000) → `BuckPalConfigurationProperties` → `BuckPalConfiguration#moneyTransferProperties` Bean으로 노출되어 `SendMoneyService`의 송금 한도 검사에 쓰인다.

### 영속성

- H2 in-memory(런타임 + 테스트). 별도 스키마 파일 없이 JPA가 자동 생성.
- 시스템/통합 테스트는 `@Sql`로 픽스처를 주입한다 (`src/test/resources/io/reflectoring/buckpal/**/*.sql`).
- 도메인 ↔ JPA 엔티티 변환은 `AccountMapper`가 전담하며, 도메인 모델은 JPA 어노테이션을 알지 못한다.

### 동시성

`AccountLock`은 포트로만 정의되어 있고, 운영 구현은 의도적으로 `NoOpAccountLock`이다. 송금 시 락/해제 호출 순서(`SendMoneyService#sendMoney`)는 책의 트랜잭션 시나리오를 위한 것이지 실제 분산 락이 아니다.
