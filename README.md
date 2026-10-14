# Spring Boot Backend Code Review Sub-agent

Claude Code용 Spring Boot 백엔드 코드 리뷰 서브 에이전트입니다. Java 모범 사례 90개 항목 체크리스트와 Spring Boot 전용 체크리스트 37개 항목을 기준으로, 변경된 Java 코드와 그 주변 설정(application.yml, DB 마이그레이션)의 결함을 심각도별로 보고합니다.

## 구성

```
.
├── .claude/
│   └── agents/
│       └── spring-boot-code-reviewer.md   # 서브 에이전트 정의 (frontmatter + 시스템 프롬프트)
├── reference_books/                       # 참고 서적 (저작권 자료, git 추적 제외)
├── .gitignore
└── README.md
```

## 설치

### 이 저장소에서 바로 사용

저장소 루트에서 `claude`를 실행하면 `.claude/agents/spring-boot-code-reviewer.md`가 프로젝트 서브 에이전트로 자동 로드됩니다.

### 다른 프로젝트에서 사용

리뷰 대상 Spring Boot 프로젝트의 `.claude/agents/` 로 파일을 복사합니다.

```bash
mkdir -p /path/to/spring-project/.claude/agents
cp .claude/agents/spring-boot-code-reviewer.md /path/to/spring-project/.claude/agents/
```

모든 프로젝트에서 쓰려면 사용자 디렉터리에 둡니다.

```bash
mkdir -p ~/.claude/agents
cp .claude/agents/spring-boot-code-reviewer.md ~/.claude/agents/
```

같은 이름이 양쪽에 있으면 프로젝트(`.claude/agents/`)가 사용자(`~/.claude/agents/`)보다 우선합니다. 새 에이전트 파일은 Claude Code 세션 시작 시 로드되므로, 복사 후에는 세션을 다시 시작해야 합니다.

## 사용법

Claude Code 세션 안에서 다음 중 한 가지로 호출합니다.

| 방법 | 예 |
|---|---|
| 자연어 요청 | `spring-boot-code-reviewer 서브 에이전트로 방금 바꾼 코드 리뷰해줘` |
| @ 멘션 | `@agent-spring-boot-code-reviewer src/main/java/com/example/order/OrderService.java 리뷰해줘` |
| 자동 위임 | Spring Boot 프로젝트에서 Java 파일을 수정한 뒤 "리뷰해줘"라고 하면 description을 보고 Claude가 이 에이전트에 위임합니다 |
| 세션 전체 | `claude --agent spring-boot-code-reviewer` |

리뷰 범위를 지정하지 않으면 에이전트가 `git diff` → `git diff --cached` → `git diff HEAD~1` 순으로 변경 내용을 찾습니다. 변경된 `.java` 파일이 주 대상이고, 함께 바뀐 `application*.yml`, Flyway/Liquibase 마이그레이션, 빌드 파일도 읽습니다. 특정 파일, 브랜치, PR을 리뷰하려면 프롬프트에 명시하면 됩니다.

```
spring-boot-code-reviewer 서브 에이전트로 main 브랜치 대비 feature/payment 브랜치의 변경을 리뷰해줘
```

## 에이전트 설정 요약

`.claude/agents/spring-boot-code-reviewer.md`의 frontmatter입니다.

| 필드 | 값 | 이유 |
|---|---|---|
| `name` | `spring-boot-code-reviewer` | 호출 시 사용하는 식별자 |
| `description` | Spring Boot 백엔드 전문 리뷰어, `.java` 수정 직후 또는 Spring Boot PR·서비스·컨트롤러·엔티티 리뷰 요청 시 사용 | Claude가 자동 위임 여부를 판단하는 기준 |
| `tools` | `Read, Grep, Glob, Bash` | 코드 탐색과 `git diff` 실행에 필요한 최소 도구 |
| `disallowedTools` | `Edit, Write, NotebookEdit` | 리뷰어는 코드를 수정하지 않음 |
| `model` | `inherit` | 메인 세션과 같은 모델 사용 |
| `color` | `blue` | 터미널 표시 색 |

`Bash`는 시스템 프롬프트에서 `git`, `ls`, `find`, `./gradlew tasks` 등 읽기 전용 명령으로만 제한하도록 지시했습니다. 더 엄격하게 막으려면 `tools`에서 `Bash`를 빼거나 `PreToolUse` 훅을 추가하면 됩니다.

## 리뷰 기준

시스템 프롬프트는 영어로 작성했습니다. 자동 위임 판단에 쓰이는 description과 도구명·코드 용어가 영어라 라우팅이 더 일관되고 토큰도 줄어들기 때문입니다. 리뷰 결과는 한국어로 출력하도록 지시되어 있습니다.

지적마다 근거 번호가 붙습니다. Java 일반 항목은 `[Item NN]`, Spring Boot 항목은 `[SB-NN]`, 둘 다 아닌 일반 결함은 `[General]`입니다.

### Spring Boot 체크리스트 (`[SB-NN]`)

해당 스타터가 클래스패스에 있을 때만 적용합니다. 빌드 파일에서 Java 버전, Spring Boot 버전(2.x `javax` / 3.x `jakarta`), 사용 중인 스타터를 읽어 그에 맞는 수정안을 냅니다.

| 영역 | 항목 | 주요 점검 대상 |
|---|---|---|
| 빈 정의·주입·생명주기 | SB-1~7 | 생성자 주입, 싱글턴 빈의 가변 상태, 순환 의존, 자기 호출로 프록시 우회, `@ConfigurationProperties`, `@Bean` 함정 |
| 트랜잭션 | SB-8~12 | 검사 예외는 롤백 안 됨, 트랜잭션 안의 외부 호출, `@Transactional` 위치, `readOnly`/전파 속성, open-in-view |
| JPA / Spring Data | SB-13~21 | N+1, 기본 EAGER, 엔티티 `equals`/`hashCode`/`@Data`, 엔티티 직접 노출, cascade/orphanRemoval, `@Modifying`, 쿼리 정확성, `ddl-auto`·마이그레이션 불일치, 감사·시간 처리 |
| 웹 계층 | SB-22~25 | `@Valid` 누락, `@ControllerAdvice`·상태 코드, 컨트롤러 책임, 페이징 |
| 보안 | SB-26~29 | 메서드 보안·객체 수준 권한, CORS/CSRF, 설정 파일의 비밀·actuator 노출, 비밀번호·토큰 처리 |
| 비동기·스케줄·캐시·이벤트 | SB-30~33 | `@Async` void 예외 유실·실행자 미설정, `@Scheduled` 단일 스레드·분산 락, `@Cacheable` 키·evict, `@TransactionalEventListener` |
| Lombok·매핑 | SB-34~35 | 엔티티의 `@Data`, `@Builder.Default`, `@AllArgsConstructor` 순서, MapStruct 미매핑 |
| 테스트 | SB-36~37 | `@SpringBootTest` 남용, 테스트의 `@Transactional`이 숨기는 버그, H2 vs 운영 DB, flush/clear 없는 저장소 테스트 |

### Spring 관례와 충돌하는 일반 규칙의 예외 처리

올바른 Spring 코드를 결함으로 잡지 않도록 시스템 프롬프트 2절에 예외 규칙을 두었습니다.

- 프록시 대상 빈(`@Service`, `@Transactional`, `@Cacheable` 등)에는 `final`을 권고하지 않습니다. 오히려 `final`이 붙어 있으면 Critical로 보고합니다.
- JPA 엔티티는 불변성 항목(Item 17)에서 면제됩니다. 다만 public 필드, `@Data`, 가변 필드 기반 `equals`는 계속 지적합니다.
- 세션·캐시에 저장되는 클래스는 `Serializable` 금지 항목(Item 85/86)에서 면제됩니다.
- Javadoc·스레드 안전성 문서화(Item 56/82)는 라이브러리성 공개 API에만 적용합니다. 컨트롤러·서비스·DTO에는 요구하지 않습니다.
- 컨트롤러에서 `@Valid`로 검증하면 서비스에 같은 검증을 다시 요구하지 않습니다.
- 컨트롤러 핸들러, `@Bean` 메서드, 서비스 메서드가 public인 것은 지적하지 않습니다.

### Java 일반 체크리스트 (`[Item NN]`)

| 영역 | 항목 | 주요 점검 대상 |
|---|---|---|
| 객체 생성과 파괴 | 1~9 | 빌더, 의존 객체 주입, 불필요한 객체 생성, 메모리 누수, try-with-resources |
| 공통 메서드 | 10~14 | equals/hashCode 규약, toString, clone, Comparable |
| 클래스와 인터페이스 | 15~25 | 접근 제한, 불변성, 상속 vs 컴포지션, 태그 달린 클래스, 정적 멤버 클래스 |
| 제네릭 | 26~33 | 로 타입, 비검사 경고, 배열 vs 리스트, 와일드카드(PECS) |
| 열거 타입과 애너테이션 | 34~41 | int 상수, ordinal, EnumSet/EnumMap, @Override |
| 람다와 스트림 | 42~48 | 익명 클래스, 표준 함수형 인터페이스, 스트림 부작용, 병렬 스트림 |
| 메서드 | 49~56 | 매개변수 검증, 방어적 복사, 오버로딩, null 반환, Optional |
| 일반 원칙 | 57~68 | 지역변수 범위, for-each, double 금액 계산, 박싱 타입, 문자열 연결 |
| 예외 | 69~77 | 검사/비검사 예외, 예외 번역, 실패 원자성, 예외 무시 |
| 동시성 | 78~84 | 동기화 누락, 과도한 동기화, 실행자, 동시성 유틸리티, 지연 초기화 |
| 직렬화 | 85~90 | 직렬화 대안, serialVersionUID, readObject 방어 |
| 일반 `[General]` | - | 정확성, null 안전성, 보안(SQL 주입, 민감 정보 로깅 등), 성능, 테스트, API 설계 |

### 테스트 코드

`src/test/**`에는 가벼운 기준만 적용합니다. 변경된 동작을 검증하는지, 단언이 있는지, `Thread.sleep` 기반 타이밍이 없는지, `@SpringBootTest` 대신 슬라이스 테스트로 충분한지만 봅니다. 매개변수 수, Javadoc, 접근 제한, 불변성 항목은 테스트에 적용하지 않습니다.

### 심각도

| 등급 | 기준 | Spring Boot 예 |
|---|---|---|
| Critical | 데이터 손상, 보안 취약점, 스레드 안전성 위반, 리소스 누수, 명백한 런타임 오류. 머지 전 수정 필수 | 프록시 빈에 `final`, 검사 예외로 트랜잭션이 커밋됨, 변경 엔드포인트에 권한 검사 없음, SQL 주입 |
| Major | 특정 조건에서 잘못된 동작, 유지보수를 심각하게 해치는 설계 결함 | N+1, EAGER `@OneToMany`, 엔티티에 `@Data`, 엔티티를 요청 본문으로 사용, 필드 주입 |
| Minor | 동작은 맞지만 관용구·API 설계·가독성 개선점 | `readOnly = true` 누락, open-in-view 기본값, `@Value` 남발 |
| Nit | 취향 수준. 한 줄로만 언급 | 네이밍, 람다 vs 메서드 참조 |

### 리뷰 원칙

- 지적 전에 해당 코드와 호출자/피호출자를 실제로 읽습니다. 엔티티를 바꿨으면 저장소와 마이그레이션도 읽습니다. 추측이면 "확인 필요"로 분류합니다.
- 포맷팅(들여쓰기, import 순서)은 지적하지 않습니다.
- 확신이 낮은 지적은 보고하지 않습니다. 지적 수를 채우기 위한 Nit을 만들지 않습니다.
- 수정안은 컴파일 가능한 Java 코드로, 감지된 Spring Boot 버전에 맞춰 제시합니다.
- 프로젝트 전반의 기존 관례는 존중하고, 관례 자체를 바꾸는 제안은 등급을 낮춥니다.

## 출력 예

```markdown
# Spring Boot 코드 리뷰 결과

**범위**: src/main/java/com/example/order/OrderService.java, OrderController.java
**Java / Spring Boot 버전**: Java 17, Spring Boot 3.2.5, Spring Data JPA, Spring Security
**요약**: Critical 1건, Major 2건, Minor 1건. 트랜잭션 롤백 규칙과 N+1 보강이 필요합니다.

## Critical

### 1. 검사 예외 발생 시 트랜잭션이 커밋됨 `[SB-8]`
- **위치**: `src/main/java/com/example/order/OrderService.java:42`
- **문제**: `placeOrder`가 `throws PaymentException`(검사 예외)을 선언했지만 `@Transactional`은 기본적으로 RuntimeException에만 롤백합니다. 결제 실패 후 주문 행이 그대로 저장됩니다.
- **근거**: SB-8. Spring 선언적 트랜잭션의 기본 롤백 규칙은 unchecked 예외에만 적용된다.
- **수정안**:
  ```java
  @Transactional(rollbackFor = PaymentException.class)
  public Order placeOrder(PlaceOrderCommand command) throws PaymentException {
  ```
...
```

## 커스터마이징

- **기준 추가/삭제**: `spring-boot-code-reviewer.md`의 `# 3. Java review checklist` 절에 항목을 추가하거나 제거합니다. Spring Boot 항목은 3.13절, 예외 규칙은 2절의 `Spring Boot exemptions`에 있습니다.
- **출력 언어 변경**: 시스템 프롬프트 상단의 "Write the review report in **Korean**" 문장을 수정합니다.
- **모델 고정**: `model: inherit`를 `sonnet`, `opus` 등으로 바꿉니다.
- **도구 제한 강화**: `tools`에서 `Bash`를 제거하면 `git diff`를 직접 실행하지 못하므로, 호출 시 리뷰 대상 파일을 프롬프트에 명시해야 합니다.
- **다른 프레임워크**: 3.13절을 교체하면 Quarkus, Micronaut 등 다른 스택용 리뷰어로 바꿀 수 있습니다. 1절의 범위 규칙과 2절의 예외 규칙도 함께 조정해야 합니다.

## 참고 자료

### Claude Code

- 서브 에이전트 문서: https://code.claude.com/docs/en/sub-agents

### Spring Boot 체크리스트 근거

- Spring Framework 선언적 트랜잭션 `@Transactional` 설정: https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/annotations.html
- Spring Framework 트랜잭션 롤백 규칙(기본은 unchecked 예외만 롤백): https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/rolling-back.html
- Spring Framework AOP 프록시 메커니즘(CGLIB, `final` 제약, 자기 호출): https://docs.spring.io/spring-framework/reference/core/aop/proxying.html
- Spring Framework 컴포넌트 스캔과 빈 정의: https://docs.spring.io/spring-framework/reference/core/beans/classpath-scanning.html
- Spring Framework Bean Validation(`@Valid`, `@Validated`): https://docs.spring.io/spring-framework/reference/core/validation/beanvalidation.html
- Spring Framework `@Async`, `@Scheduled`: https://docs.spring.io/spring-framework/reference/integration/scheduling.html
- Spring Boot 외부 설정과 `@ConfigurationProperties`: https://docs.spring.io/spring-boot/reference/features/external-config.html
- Spring Boot Actuator 엔드포인트 노출 설정: https://docs.spring.io/spring-boot/reference/actuator/endpoints.html
- Spring Boot SQL 데이터베이스(JPA, `ddl-auto`, open-in-view): https://docs.spring.io/spring-boot/reference/data/sql.html
- Spring Data JPA 쿼리 메서드, `@EntityGraph`, `@Modifying`: https://docs.spring.io/spring-data/jpa/reference/jpa/query-methods.html
- Spring Data JPA 트랜잭션: https://docs.spring.io/spring-data/jpa/reference/jpa/transactions.html
- Spring Data JPA 엔티티 영속화와 저장: https://docs.spring.io/spring-data/jpa/reference/jpa/entity-persistence.html
- Spring Security 메서드 보안(`@PreAuthorize`, `@EnableMethodSecurity`): https://docs.spring.io/spring-security/reference/servlet/authorization/method-security.html
- Spring Security CORS: https://docs.spring.io/spring-security/reference/servlet/integrations/cors.html
- Spring Boot 3.0 마이그레이션 가이드(`javax` → `jakarta`): https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-3.0-Migration-Guide
- Hibernate ORM 6.6 User Guide(fetch 전략, cascade, equals/hashCode): https://docs.jboss.org/hibernate/orm/6.6/userguide/html_single/Hibernate_User_Guide.html
- Jakarta Bean Validation 3.0: https://beanvalidation.org/3.0/
- Lombok `@Data`: https://projectlombok.org/features/Data

### Java 일반 체크리스트 근거

- Joshua Bloch, 『이펙티브 자바 3판』(인사이트, 2018). 원서 *Effective Java, 3rd Edition* (Addison-Wesley, 2018)

`reference_books/` 디렉터리는 저작권이 있는 자료를 담으므로 `.gitignore`로 버전 관리에서 제외되어 있습니다.
