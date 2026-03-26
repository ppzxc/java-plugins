# Clean and Pragmatic Code (Clean Code + Pragmatic Programmer + Code Complete)

## Table of Contents
1. The Art of Naming
2. Function/Method Design
3. Comments and Self-Documenting Code
4. Error Handling
5. Classes and Data Structures
6. Formatting and Code Organization
7. Pragmatic Development Habits

---

## 1. The Art of Naming

Names are the single most important factor that determines code readability.

### Variable Names
```java
// Bad: Intent is hidden
int d;  // elapsed time in days
List<int[]> list1;

// Good: Intent is clear
int elapsedTimeInDays;
List<Cell> flaggedCells;
```

### Method Names
- Start with a verb or verb phrase: `save()`, `calculateTotal()`, `sendNotification()`
- Boolean-returning methods: use `is`, `has`, `can`, `should` prefixes
- Follow accessor/mutator/predicate conventions
```java
account.setBalance(amount);    // mutator
account.getBalance();           // accessor
account.isOverdrawn();          // predicate
```

### Class Names
- Use nouns or noun phrases: `Customer`, `OrderProcessor`, `PaymentGateway`
- Minimize vague suffixes like `Manager`, `Processor`, `Data`, `Info`
- Meaningful distinctions: `ProductInfo` and `ProductData` should not coexist

### Consistency
- Same word for same concept: choose one of `fetch`/`retrieve`/`get` and use it project-wide
- Similar names must not carry different meanings: `add` should not mean "append to collection" in one place and "sum values" in another

---

## 2. Function/Method Design

### Size
- Smaller is better. Ideally 5–15 lines
- Should fit on one screen
- Consider splitting when exceeding 20 lines

### Do One Thing
How to tell if a function does one thing: if you can extract a meaningfully named function from it, it's doing more than one thing.

```java
// Bad: Does multiple things at once
public void processEmployee(Employee emp) {
    validateEmployee(emp);
    calculateSalary(emp);
    saveToDB(emp);
    sendPayslip(emp);
    updateDashboard(emp);
}

// Good: Each does one thing, orchestrated by a higher-level function
public void onboardEmployee(Employee emp) {
    validateEmployee(emp);
    processPayroll(emp);
    notifyStakeholders(emp);
}
```

### Uniform Abstraction Level (Stepdown Rule)
Do not mix abstraction levels within a single function.
High-level concepts and low-level details should not live in the same function.

```java
// Bad: Mixed abstraction levels
public String renderPage() {
    String html = "<html>";
    html += getHeader();  // high level
    html += "<div class=\"content\">";  // low level
    html += getBody();  // high level
    html += "</div>";  // low level
    return html;
}

// Good: Uniform abstraction level
public String renderPage() {
    return buildHtmlDocument(
        getHeader(),
        wrapInContentDiv(getBody()),
        getFooter()
    );
}
```

### Parameters
- 0 (niladic) is best, 1 (monadic) is good, up to 2 (dyadic) is acceptable
- 3+ parameters → wrap into an object
- **Flag arguments are forbidden**: `render(boolean isSuite)` → `renderForSuite()`, `renderForSingle()`
- Avoid output arguments: instead of `appendFooter(report)`, use `report.appendFooter()`

### No Side Effects
A function must do only what its name promises. `checkPassword()` should never initialize a session.

---

## 3. Comments and Self-Documenting Code

### Good Comments (only when necessary)
- Legal comments (copyright, license)
- Intent explanation: why this approach was chosen
- Warnings: `// Not thread-safe`
- TODO: future work to address (should not be permanent)
- Javadoc for public APIs

### Bad Comments (replaceable with code)
```java
// Bad: Comment restates the code
// Returns the employee's name
public String getName() { return name; }

// Bad: Change log (version control handles this)
// 2024-01-15: Added discount logic - John Doe

// Bad: Commented-out code (leave it to version control)
// public void oldMethod() { ... }
```

### Self-Documenting Code
When you feel a comment is needed, first ask if you can make the code clearer instead.
```java
// Code that needs a comment
// Check if the employee is eligible for benefits
if (employee.age > 65 && employee.yearsOfService > 20 && employee.isActive) { ... }

// Self-documenting code
if (employee.isEligibleForBenefits()) { ... }
```

---

## 4. Error Handling

### Exception Usage Principles
- Use exceptions instead of return codes
- Use checked exceptions only when necessary: when the caller can reasonably recover
- Default to unchecked exceptions (programming errors, unrecoverable situations)
- Include context information in exceptions

```java
// Bad: Exception without context
throw new RuntimeException("Lookup failed");

// Good: Sufficient context
throw new OrderNotFoundException(
    String.format("Order ID %d not found. Customer: %s", orderId, customerId)
);
```

### Null Handling
- **Do not return null**: return Optional or empty collections instead
- **Do not pass null**: do not allow null as a method argument

```java
// Bad
public List<Employee> findByDepartment(String dept) {
    if (noResult) return null;
}

// Good
public List<Employee> findByDepartment(String dept) {
    if (noResult) return Collections.emptyList();
}

// For single-item lookups
public Optional<Employee> findById(Long id) {
    return Optional.ofNullable(repository.get(id));
}
```

### Structuring try-catch
- Write the try block first (think transactionally)
- Never swallow exceptions in catch blocks (at minimum, log them)
- Leverage try-with-resources

```java
try (var reader = new BufferedReader(new FileReader(path))) {
    return reader.lines()
        .map(this::parseLine)
        .collect(Collectors.toList());
} catch (IOException e) {
    throw new DataLoadException("Failed to load file: " + path, e);
}
```

---

## 5. Classes and Data Structures

### Class Organization Order (Java Convention)
1. Static constants
2. Static variables
3. Instance variables (public → protected → private)
4. Constructors
5. Public methods
6. Private methods (placed directly below the public method that calls them)

### Cohesion
Minimize instance variables, and have each method use as many instance variables as possible.
Low cohesion is a signal to split the class.

### Data Transfer Objects (DTO)
- Carrying data without behavior is fine (DTO pattern)
- But domain objects should carry behavior (avoid Anemic Domain Model)

```java
// DTO: For data transport — leverage records
public record OrderSummaryDto(
    Long orderId,
    String customerName,
    BigDecimal totalAmount,
    LocalDateTime createdAt
) {}

// Domain Object: Has behavior
public class Order {
    private List<OrderItem> items;
    private OrderStatus status;

    public Money calculateTotal() { /* business logic */ }
    public void cancel() { /* apply cancellation rules */ }
    public boolean canBeModified() { /* modification condition */ }
}
```

---

## 6. Formatting and Code Organization

### Vertical Formatting
- Recommended file length: 200–500 lines (consider splitting beyond 500)
- Related code stays close together, unrelated code stays apart
- Callers are placed above callees (newspaper article order)
- Separate concepts with blank lines

### Horizontal Formatting
- Stay within 120 characters per line (follow team rules)
- Never skip indentation
- Use spacing around operators to visualize precedence

### Package Organization
```
com.company.project
├── domain/          # Domain models, business logic
├── application/     # Use cases, services
├── infrastructure/  # DB, external API integration
├── presentation/    # Controllers, DTOs
└── common/          # Shared utilities
```

---

## Cross-References

- For Java-specific idioms that complement these principles (static factory, Builder, Optional) → `effective-java.md`
- For DDD domain language that informs naming and domain object design → `domain-driven-design.md`

---

## 7. Pragmatic Development Habits

### Orthogonality
Maintain independence between modules. If changing A requires modifying B, they are not orthogonal.
- Always ask when writing code: "What is the blast radius of this change?"

### Tracer Bullet Development
In uncertain projects, build a minimal working implementation that pierces through the entire architecture first.
Connect a single line from UI → Service → DB end-to-end, then flesh it out.

### Estimation
- Provide ranges: "2–3 days" (not precise numbers)
- Mind the units: 150 days → "about 6 months", 15 days → "about 3 weeks"
- Keep estimation records and compare with actuals to improve accuracy

### Domain Language
Reflect the business domain's language in the code.
For an accounting system, use `entry`, `ledger`, `debit`, `credit` in the code as well.

### Design by Contract
- **Preconditions**: Conditions that must hold before calling a function
- **Postconditions**: Guaranteed state after a function completes
- **Class invariants**: Conditions that must always hold true

```java
public class BankAccount {
    private BigDecimal balance;  // Invariant: balance >= 0

    public void withdraw(BigDecimal amount) {
        // Precondition
        if (amount.compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("Withdrawal amount must be positive");
        }
        if (amount.compareTo(balance) > 0) {
            throw new InsufficientFundsException("Insufficient funds");
        }

        balance = balance.subtract(amount);

        // Postcondition (defensive programming)
        assert balance.compareTo(BigDecimal.ZERO) >= 0 : "Balance became negative";
    }
}
```
