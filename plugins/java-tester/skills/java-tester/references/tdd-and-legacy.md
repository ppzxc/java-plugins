# Testing Strategies and Legacy Code (TDD + Working Effectively with Legacy Code)

## Table of Contents
1. TDD Core Cycle
2. Test Writing Principles
3. Java Testing Patterns
4. Dealing with Legacy Code
5. Test Doubles (Mock, Stub, Fake)
6. Making Untestable Code Testable

---

## 1. TDD Core Cycle

### Red → Green → Refactor

```
[Red]      Write a failing test
  ↓
[Green]    Write the minimum code to pass the test
  ↓
[Refactor] Remove duplication and improve structure
  ↓
(Repeat)
```

### The Three Laws of TDD
1. Do not write production code until you have a failing unit test
2. Write only enough of a unit test to fail (compilation failure counts)
3. Write only enough production code to pass the currently failing test

### TDD Rhythm
- Each cycle should complete within 1–5 minutes
- If a cycle takes longer, break the step into smaller pieces
- In the Green phase, write "minimum code" — it doesn't need to be clean yet
- In the Refactor phase, make it clean

---

## 2. Test Writing Principles

### F.I.R.S.T Principles
- **Fast**: Execute in milliseconds. Slow tests won't get run
- **Independent**: No order dependency between tests. Runnable in any order
- **Repeatable**: Same result in any environment. No network/DB dependencies
- **Self-Validating**: Automatically determines pass/fail. Manually scanning logs is not testing
- **Timely**: Written just before production code (TDD) or immediately after

### One Concept Per Test
Do not verify multiple concepts in a single test method.

```java
// Bad: Multiple concepts in one test
@Test
void testOrder() {
    Order order = new Order();
    order.addItem(new Item("A", 1000));
    assertEquals(1, order.getItemCount());     // item addition
    assertEquals(1000, order.getTotal());       // total calculation
    order.applyDiscount(10);
    assertEquals(900, order.getTotal());        // discount application
}

// Good: Separated by concept
@Test
void addItem_increasesItemCount() {
    order.addItem(new Item("A", 1000));
    assertThat(order.getItemCount()).isEqualTo(1);
}

@Test
void getTotal_returnsSumOfItemPrices() {
    order.addItem(new Item("A", 1000));
    order.addItem(new Item("B", 2000));
    assertThat(order.getTotal()).isEqualTo(3000);
}

@Test
void applyDiscount_reducesTotalByPercentage() {
    order.addItem(new Item("A", 1000));
    order.applyDiscount(10);
    assertThat(order.getTotal()).isEqualTo(900);
}
```

### Given-When-Then Structure
Clearly divide test code into three phases.

```java
@Test
void withdraw_withSufficientBalance_reducesBalance() {
    // Given: An account with a balance of 10,000
    BankAccount account = new BankAccount(Money.of(10_000));

    // When: Withdraw 3,000
    account.withdraw(Money.of(3_000));

    // Then: Balance is 7,000
    assertThat(account.getBalance()).isEqualTo(Money.of(7_000));
}

@Test
void withdraw_withInsufficientBalance_throwsException() {
    // Given
    BankAccount account = new BankAccount(Money.of(1_000));

    // When & Then
    assertThatThrownBy(() -> account.withdraw(Money.of(5_000)))
        .isInstanceOf(InsufficientFundsException.class)
        .hasMessageContaining("Insufficient funds");
}
```

### Test Naming Convention
The method name alone should tell you what is being tested.
Pattern: `methodUnderTest_condition_expectedResult`

```java
void calculateShipping_forOverseasOrder_addsInternationalFee()
void login_withExpiredToken_returnsUnauthorized()
void findUsers_withNoMatch_returnsEmptyList()
```

---

## 3. Java Testing Patterns

### Test Fixture Management
```java
class OrderServiceTest {
    private OrderService orderService;
    private OrderRepository mockRepository;
    private NotificationSender mockNotifier;

    @BeforeEach
    void setUp() {
        mockRepository = mock(OrderRepository.class);
        mockNotifier = mock(NotificationSender.class);
        orderService = new OrderService(mockRepository, mockNotifier);
    }
}
```

### Boundary Value Testing
Boundary conditions are where bugs hide.

```java
// If the age restriction is 18 and above
@ParameterizedTest
@ValueSource(ints = {17, 18, 19})
void ageRestriction_boundaryValues(int age) {
    User user = new User(age);
    if (age < 18) {
        assertThat(user.canPurchaseAlcohol()).isFalse();
    } else {
        assertThat(user.canPurchaseAlcohol()).isTrue();
    }
}
```

### Exception Testing
```java
@Test
void createUser_withBlankName_throwsValidationException() {
    assertThatThrownBy(() -> new User("", "test@mail.com"))
        .isInstanceOf(ValidationException.class)
        .hasFieldOrPropertyWithValue("field", "name");
}
```

---

## 4. Dealing with Legacy Code

Definition of legacy code: **code without tests** (Michael Feathers)

### The Legacy Code Change Algorithm
1. **Identify the change point**
2. **Find a test point**
3. **Break dependencies** (to make it testable)
4. **Write tests**
5. **Make changes and refactor**

### Key Dependency-Breaking Techniques

#### Extract and Override (The Safest Starting Point)
Extract hard-to-test dependencies into a method, then override in tests.

```java
// Legacy code: Directly accesses the DB
public class ReportGenerator {
    public Report generate() {
        // Direct DB access (untestable)
        Connection conn = DriverManager.getConnection(DB_URL);
        List<Record> records = queryRecords(conn);
        return buildReport(records);
    }
}

// Step 1: Extract the dependency into a method
public class ReportGenerator {
    public Report generate() {
        List<Record> records = fetchRecords();  // Extracted!
        return buildReport(records);
    }

    protected List<Record> fetchRecords() {  // Changed to protected
        Connection conn = DriverManager.getConnection(DB_URL);
        return queryRecords(conn);
    }
}

// Step 2: Override in tests
class TestableReportGenerator extends ReportGenerator {
    private final List<Record> testRecords;

    TestableReportGenerator(List<Record> testRecords) {
        this.testRecords = testRecords;
    }

    @Override
    protected List<Record> fetchRecords() {
        return testRecords;  // Return test data
    }
}

// Step 3: Write the test
@Test
void generate_withRecords_producesCorrectReport() {
    var records = List.of(new Record("A", 100), new Record("B", 200));
    var generator = new TestableReportGenerator(records);

    Report report = generator.generate();

    assertThat(report.getTotalAmount()).isEqualTo(300);
}
```

#### Introduce Interface (Better Long-Term Approach)
Change from depending on concrete classes to depending on interfaces.

```java
// Extract interface
public interface RecordFetcher {
    List<Record> fetch();
}

// Production implementation
public class DatabaseRecordFetcher implements RecordFetcher {
    public List<Record> fetch() {
        // Actual DB access
    }
}

// Switch to constructor injection
public class ReportGenerator {
    private final RecordFetcher fetcher;

    public ReportGenerator(RecordFetcher fetcher) {
        this.fetcher = fetcher;
    }

    public Report generate() {
        return buildReport(fetcher.fetch());
    }
}
```

#### Characterization Test
A test that records the current behavior of the code. Used to build a safety net before refactoring.

```java
// "I don't know what this code does" → Write a characterization test
@Test
void characterize_calculateDiscount_currentBehavior() {
    // Record current behavior as-is
    LegacyPricingEngine engine = new LegacyPricingEngine();

    // Record current outputs for various inputs
    assertThat(engine.calculateDiscount(100, "VIP")).isEqualTo(15.0);
    assertThat(engine.calculateDiscount(100, "NORMAL")).isEqualTo(5.0);
    assertThat(engine.calculateDiscount(0, "VIP")).isEqualTo(0.0);
    assertThat(engine.calculateDiscount(-10, "VIP")).isEqualTo(0.0);  // Verify negative handling
}
```

---

## 5. Test Doubles (Mock, Stub, Fake)

### Terminology

| Type | Purpose | Example |
|------|---------|---------|
| **Stub** | Returns predetermined values | `when(repo.findById(1L)).thenReturn(user)` |
| **Mock** | Verifies interactions occurred | `verify(notifier).send(any())` |
| **Fake** | A working simplified implementation | `InMemoryRepository` |
| **Spy** | Wraps a real object, overriding only parts | `spy(realService)` |

### Usage Principles
- **Prefer state verification** (check result values). Use mock verification (behavior verification) only for side effects
- Too many mocks signal a design problem: too many dependencies
- Don't mock types you don't own: create an Adapter for external libraries and mock the Adapter

```java
// State verification (preferred)
@Test
void processOrder_calculatesTotalCorrectly() {
    Order order = orderService.process(items);
    assertThat(order.getTotal()).isEqualTo(Money.of(15_000));
}

// Behavior verification (for side effect confirmation)
@Test
void processOrder_sendsConfirmationEmail() {
    orderService.process(items);
    verify(emailSender).send(argThat(email ->
        email.getSubject().contains("Order Confirmation")
    ));
}
```

---

## 6. Making Untestable Code Testable

### Common Untestable Patterns and Solutions

| Problem | Solution |
|---------|----------|
| Static method calls | Wrap in a class/interface |
| Direct `new` instantiation | Constructor injection or Factory injection |
| Global state (Singleton) | Extract interface + DI |
| Time dependency (`LocalDateTime.now()`) | Inject `Clock` |
| File/network access | Extract interface + Fake implementation |

### Solving Time Dependencies
```java
// Bad: Untestable (depends on current time)
public class MembershipService {
    public boolean isExpired(Membership m) {
        return m.getExpiryDate().isBefore(LocalDate.now());
    }
}

// Good: Time control via Clock injection
public class MembershipService {
    private final Clock clock;

    public MembershipService(Clock clock) {
        this.clock = clock;
    }

    public boolean isExpired(Membership m) {
        return m.getExpiryDate().isBefore(LocalDate.now(clock));
    }
}

// Test
@Test
void isExpired_afterExpiryDate_returnsTrue() {
    Clock fixedClock = Clock.fixed(
        LocalDate.of(2025, 6, 1).atStartOfDay(ZoneId.systemDefault()).toInstant(),
        ZoneId.systemDefault()
    );
    var service = new MembershipService(fixedClock);
    var membership = new Membership(LocalDate.of(2025, 5, 31));

    assertThat(service.isExpired(membership)).isTrue();
}
```

### Finding Seams
A seam is a place where you can alter behavior without editing the code at that point.

Types of seams in Java:
1. **Object Seam**: Swap behavior via polymorphism (most common)
2. **Compile Seam**: Inject different implementations (DI / compile config)
3. **Link Seam**: Swap via classpath (different JAR at test time)

In legacy code, find seams first, then break dependencies at those points to insert tests.
