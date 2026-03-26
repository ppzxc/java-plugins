# Refactoring Catalog (Refactoring + Design Patterns Applied)

## Table of Contents
1. Refactoring Principles
2. Code Smells → Refactoring Mapping
3. Key Refactoring Techniques (Java Examples)
4. Large-Scale Refactoring Strategies

---

## 1. Refactoring Principles

Refactoring is **improving the internal structure of code without changing its external behavior**.

### When to Refactor
- **Rule of Three**: Refactor when you do something similar for the third time
- **Before adding a feature**: Clean up the structure first so the new feature fits in easily
- **During code review**: When you detect smells while reading code
- **When fixing bugs**: Tidy up as you work to understand the code

### When NOT to Refactor
- If there are no tests, write tests first
- If the code is too broken to salvage, consider a rewrite
- If a deadline is imminent, record the tech debt and move on

### Safe Refactoring Procedure
1. Verify all tests pass before making changes
2. Make changes in small increments (one refactoring at a time)
3. Run tests after each change
4. Revert immediately on failure
5. Commit on success and move to the next step

---

## 2. Code Smells → Refactoring Mapping

### Unnecessary Complexity

| Smell | Description | Response |
|-------|-------------|----------|
| Dead Code | Unused variables, methods, classes | Delete (history lives in VCS) |
| Speculative Generality | "Might need it someday" | Remove / Inline (YAGNI) |
| Duplicated Code | Same code in multiple places | Extract Method → unify |
| Magic Number | Meaningless literal values | Replace Magic Number with Constant |

### Structural Issues

| Smell | Description | Response |
|-------|-------------|----------|
| Long Method | Method doing too much | Extract Method |
| Large Class | Class with too many responsibilities | Extract Class / Extract Subclass |
| Long Parameter List | More than 3 parameters | Introduce Parameter Object |
| Data Clumps | Groups of data that travel together | Extract Class |
| Primitive Obsession | Using String/int for domain concepts | Replace Primitive with Object |

### Change Resistance

| Smell | Description | Response |
|-------|-------------|----------|
| Shotgun Surgery | One change ripples across many classes | Move Method, Move Field → cohere |
| Divergent Change | One class changes for multiple reasons | Extract Class (apply SRP) |
| Feature Envy | Method overuses another class's data | Move Method |
| Inappropriate Intimacy | Excessive cross-referencing between classes | Move Method/Field, Extract Class |

### Conditional Logic

| Smell | Description | Response |
|-------|-------------|----------|
| Repeated Switch Statements | Same switch in multiple places | Replace Conditional with Polymorphism |
| Type Code | int/String to distinguish types | Replace Type Code with Subclass/Strategy |
| Repeated Null Checks | `if (obj != null)` everywhere | Introduce Null Object / Optional |

---

## 3. Key Refactoring Techniques (Java Examples)

### Extract Method
The most frequently used refactoring. Give a name to a code fragment.

```java
// Before
public void printOwing() {
    // Print banner
    System.out.println("**************************");
    System.out.println("***** Customer Owes ******");
    System.out.println("**************************");

    // Calculate outstanding amount
    double outstanding = 0.0;
    for (Order order : orders) {
        outstanding += order.getAmount();
    }

    // Print details
    System.out.println("name: " + name);
    System.out.println("amount: " + outstanding);
}

// After
public void printOwing() {
    printBanner();
    double outstanding = calculateOutstanding();
    printDetails(outstanding);
}

private void printBanner() { /* ... */ }
private double calculateOutstanding() { /* ... */ }
private void printDetails(double outstanding) { /* ... */ }
```

### Replace Conditional with Polymorphism
Replace type-based branching with polymorphism.

```java
// Before: switch smell
public double calculatePay(Employee employee) {
    switch (employee.getType()) {
        case COMMISSIONED:
            return calculateCommissionedPay(employee);
        case HOURLY:
            return calculateHourlyPay(employee);
        case SALARIED:
            return calculateSalariedPay(employee);
        default:
            throw new InvalidEmployeeType(employee.getType());
    }
}

// After: polymorphism
public abstract class Employee {
    public abstract Money calculatePay();
}

public class CommissionedEmployee extends Employee {
    @Override
    public Money calculatePay() {
        return basePay.add(commission.multiply(salesAmount));
    }
}

public class HourlyEmployee extends Employee {
    @Override
    public Money calculatePay() {
        return hourlyRate.multiply(hoursWorked);
    }
}
```

### Introduce Parameter Object
Bundle repeating parameter groups into an object.

```java
// Before
public List<Transaction> findTransactions(
    LocalDate startDate, LocalDate endDate,
    BigDecimal minAmount, BigDecimal maxAmount) { ... }

// After
public record TransactionFilter(
    LocalDate startDate, LocalDate endDate,
    BigDecimal minAmount, BigDecimal maxAmount
) {
    public boolean matches(Transaction tx) {
        return !tx.date().isBefore(startDate)
            && !tx.date().isAfter(endDate)
            && tx.amount().compareTo(minAmount) >= 0
            && tx.amount().compareTo(maxAmount) <= 0;
    }
}

public List<Transaction> findTransactions(TransactionFilter filter) {
    return transactions.stream()
        .filter(filter::matches)
        .collect(Collectors.toList());
}
```

### Replace Primitive with Domain Object
Promote primitives to domain objects.

```java
// Before: String representing an email
public void sendEmail(String email, String subject) { ... }
// "not-an-email" can slip in

// After: Type safety via domain object
public record Email(String value) {
    public Email {
        if (!value.matches("^[\\w.-]+@[\\w.-]+\\.[a-zA-Z]{2,}$")) {
            throw new IllegalArgumentException("Invalid email: " + value);
        }
    }
}

public void sendEmail(Email to, String subject) { ... }
```

### Extract Interface
Separate the contract from the implementation.

```java
// Extract interface from implementation-dependent code
public interface NotificationSender {
    void send(Notification notification);
}

public class EmailNotificationSender implements NotificationSender { ... }
public class SmsNotificationSender implements NotificationSender { ... }
public class SlackNotificationSender implements NotificationSender { ... }
```

---

## 4. Large-Scale Refactoring Strategies

### Strangler Fig Pattern
Gradually replace a legacy system.

1. Build new code alongside the existing system
2. Incrementally redirect traffic/calls to the new code
3. Remove the old code piece by piece
4. Repeat until fully replaced

### Branch by Abstraction
Use when replacing large-scale dependencies.

1. Create an abstraction layer (interface) over the replacement target
2. Change existing code to access through the abstraction
3. Build a new implementation behind the abstraction
4. Remove the old implementation

### Micro-Refactoring Habits
Continuously perform small refactorings as part of daily work:
- Rename a variable to a better name (immediately)
- Extract one paragraph from a long method (10 minutes)
- Extract a common method when duplication is found (15 minutes)
- Remove unnecessary comments and clarify the code (5 minutes)

The accumulation of these small improvements is the key to maintaining code quality.
Boy Scout Rule: "Leave the campground cleaner than you found it."
