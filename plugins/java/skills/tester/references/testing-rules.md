# Java Testing Rules
> Sources: Test Driven Development (Kent Beck), Unit Testing Principles, Practices, and Patterns (Vladimir Khorikov)
> Format: RULE → WHEN → PATTERN → EXCEPTION

---

## TDD 워크플로우

RULE: Red-Green-Refactor 사이클 준수
WHEN: 새 기능 구현 시
PATTERN:
1. **Red** — 실패하는 테스트 먼저 작성 (컴파일 오류도 Red)
2. **Green** — 테스트를 통과시키는 최소한의 코드 작성 (지저분해도 됨)
3. **Refactor** — 동작을 유지하면서 코드 정리

RULE: 테스트 하나씩 작성 — 여러 테스트를 동시에 작성하지 않는다
WHEN: TDD 사이클 중
PATTERN: 한 번에 하나의 실패 테스트만 존재

---

## 테스트 분류 전략 (Khorikov)

RULE: 테스트를 4분면으로 분류하여 작성 전략 결정

```
                  빠름 & 격리성 높음
                         ↑
                    (1) 단위 테스트
                         |
복잡도 낮음 ——————————————+—————————————— 복잡도 높음
                         |
                  (3) E2E 테스트
                         ↓
                  느림 & 격리성 낮음
```

테스트 비율 목표:
- **단위 테스트 (Unit)**: 70% — 도메인 로직, 비즈니스 규칙
- **통합 테스트 (Integration)**: 25% — DB, 외부 API, 메시지 브로커
- **E2E 테스트**: 5% — 핵심 사용자 시나리오

RULE: 단위 테스트 대상 = 비즈니스 로직이 복잡한 코드
WHEN: 단위 테스트를 작성할지 결정
PATTERN: 도메인 Entity, Value Object, Domain Service
EXCEPTION: 단순 getter/setter, 단순 CRUD — 가치 없는 테스트 작성 금지

---

## Mock 전략 (Khorikov — London vs Classical)

RULE: Mock은 아키텍처 경계에서만 사용
WHEN: 외부 의존성(DB, HTTP 클라이언트, 이메일 서비스)
PATTERN:
```java
// GOOD — 외부 서비스 경계에서 Mock
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {
    @Mock EmailService emailService;  // 외부 의존성
    @InjectMocks OrderService orderService;

    @Test
    void confirm_sendsConfirmationEmail() {
        var order = Order.create("item", 1);
        orderService.confirm(order);
        verify(emailService).sendConfirmation(order.id());
    }
}
```

RULE: 도메인 내부 객체는 Mock 하지 않는다
WHEN: 같은 레이어/모듈 내 협력 객체
PATTERN: 실제 객체를 사용한다 (또는 fake 구현체)
```java
// BAD — 도메인 객체 Mock
@Mock Order mockOrder;
when(mockOrder.total()).thenReturn(Money.of(100));

// GOOD — 실제 객체 사용
Order order = Order.create("item", 1);
order.addLine(product, 2);
```

RULE: `@Mock` vs `@MockitoBean` 구분
WHEN: 테스트 유형에 따라
PATTERN:
- `@Mock` (Mockito): 스프링 컨텍스트 없는 단위 테스트 — @ExtendWith(MockitoExtension.class) 사용
- `@MockitoBean` (Spring Boot 4+): @WebMvcTest, @DataJpaTest 등 스프링 컨텍스트 내 Mock
  (Spring Boot 3.4에서 @MockBean Deprecated, Spring Boot 4.0에서 @MockitoBean으로 대체)
- `@MockitoSpyBean`: 실제 빈을 spy하여 일부 메서드만 stubbing
EXCEPTION: Spring Boot 3.3 이하는 @MockBean 사용

---

## 좋은 테스트의 4가지 속성 (Khorikov)

모든 테스트는 다음을 가져야 한다:

1. **퇴행(Regression) 방지**: 기능 변경 시 실패해야 함
2. **리팩토링 내성**: 구현 변경에 불필요하게 깨지지 않음
3. **빠른 피드백**: 가능한 한 빠르게 실행
4. **유지보수성**: 이해하기 쉽고 수정하기 쉬움

RULE: 구현 세부사항을 테스트하지 않는다 (리팩토링 내성)
WHEN: 테스트 작성 시
PATTERN: 공개 API (결과)만 검증, private 메서드 직접 테스트 금지
```java
// BAD — 구현 세부사항 검증
verify(orderRepository).findById(orderId);  // 내부 호출 검증
// GOOD — 최종 결과 검증
assertThat(result.status()).isEqualTo(CONFIRMED);
```

---

## 테스트 구조 (AAA)

RULE: Arrange-Act-Assert 패턴 준수
WHEN: 모든 테스트 메서드
PATTERN:
```java
@Test
void cancel_confirmedOrder_changesStatusToCancelled() {
    // Arrange
    var order = Order.create("item", 1);
    order.confirm();

    // Act
    order.cancel("고객 요청");

    // Assert
    assertThat(order.status()).isEqualTo(CANCELLED);
    assertThat(order.cancelReason()).isEqualTo("고객 요청");
}
```

RULE: 테스트당 하나의 논리적 검증
WHEN: 테스트 메서드 작성 시
PATTERN: `assertThat()` 호출이 여러 개여도 괜찮지만, 하나의 동작에 대한 검증이어야 함
EXCEPTION: 같은 결과의 여러 속성 검증은 한 테스트에서 가능

---

## 테스트 네이밍

RULE: 테스트 이름은 시나리오와 예상 결과를 명시
PATTERN: `methodName_scenario_expectedResult`
```java
// GOOD
void confirm_emptyOrder_throwsIllegalState()
void cancel_cancelledOrder_throwsIllegalState()
void addLine_validProduct_increasesTotalPrice()

// BAD
void testConfirm()
void test1()
```

---

## 테스트 격리

RULE: 테스트 간 상태 공유 금지
WHEN: 테스트 클래스 설계
PATTERN: 각 테스트는 자신의 픽스처를 독립적으로 설정
```java
// GOOD — @BeforeEach에서 초기화
@BeforeEach
void setUp() {
    order = Order.create("item", 1);
}

// BAD — static 공유 상태
static Order order = Order.create("item", 1); // 다른 테스트에서 변경됨
```

---

## 레거시 코드 테스트 (Feathers)

RULE: 레거시 코드에 테스트 추가 시 Characterization Test 먼저
WHEN: 이해하지 못한 레거시 코드를 수정해야 할 때
PATTERN:
1. 현재 동작을 그대로 기록하는 테스트 작성 (올바른지 판단하지 않음)
2. 테스트가 통과되면 현재 동작이 문서화된 것
3. 이제 안전하게 수정 가능

RULE: Seam을 찾아 테스트 가능하게 만들기
WHEN: 테스트하기 어려운 코드
PATTERN:
- **Object Seam**: 생성자/메서드에 의존성 주입 추가
- **Interface Seam**: `new ConcreteClass()` → 인터페이스 주입으로 교체
```java
// BAD — 테스트 불가
public class ReportGenerator {
    public void generate() {
        EmailSender sender = new EmailSender(); // 직접 생성
        ...
    }
}
// GOOD — Seam 추가
public class ReportGenerator {
    private final EmailSender sender;
    public ReportGenerator(EmailSender sender) { this.sender = sender; }
}
```

---

## AssertJ 관용구

RULE: JUnit `assertEquals` 대신 AssertJ 사용
WHEN: 검증 작성 시
PATTERN:
```java
// 기본
assertThat(actual).isEqualTo(expected);
assertThat(actual).isNotNull();
assertThat(actual).isInstanceOf(Order.class);

// 컬렉션
assertThat(list).hasSize(3).contains("A", "B").doesNotContain("X");

// 예외
assertThatThrownBy(() -> order.confirm())
    .isInstanceOf(IllegalStateException.class)
    .hasMessageContaining("already confirmed");

// 소프트 어설션 (여러 검증을 한 번에)
assertSoftly(softly -> {
    softly.assertThat(order.status()).isEqualTo(CONFIRMED);
    softly.assertThat(order.total()).isEqualByComparingTo("100.00");
});
```
