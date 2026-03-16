# Object-Oriented Design Principles

**Track:** Design Concepts
**Difficulty Tier:** Beginner
**Prerequisites:** None

## Concept Overview

Object-oriented design (OOD) is a methodology for structuring software around objects — self-contained units that bundle data and the behavior that operates on that data. The four foundational pillars of OOD are encapsulation, abstraction, inheritance, and polymorphism. Together they provide a mental framework for decomposing complex systems into manageable, reusable pieces.

Encapsulation hides internal state behind well-defined interfaces, reducing the surface area that other parts of the code depend on. Abstraction lets you model real-world entities at the right level of detail, exposing only what consumers need. Inheritance enables code reuse by letting specialized classes extend general ones, while polymorphism allows different object types to be treated uniformly through shared interfaces.

Mastering these principles early is essential because nearly every design pattern, architectural style, and framework you will encounter later in this curriculum builds on them. When applied well, OOD leads to code that is easier to read, test, extend, and maintain. When applied poorly — for example, deep inheritance hierarchies or leaky abstractions — it creates rigid, fragile systems. The problems in this module give you practice recognizing when and how to apply each pillar effectively.

---

## Problem 1 — Encapsulating a Bank Account

**Difficulty:** Easy
**Estimated Time:** 30 minutes

### Problem Statement

Design a `BankAccount` class that protects its internal balance from direct external manipulation. The class should support depositing funds, withdrawing funds (with insufficient-balance protection), and querying the current balance. No external code should be able to set the balance directly.

### Scenario

**Context:** A retail banking application allows customers to manage their checking accounts through a mobile app. Multiple parts of the codebase — the transfer service, the ATM module, and the statement generator — all interact with account objects.

**Requirements:** Ensure that the balance can never be set to an arbitrary value from outside the class. Deposits must be positive amounts. Withdrawals must be rejected when the balance is insufficient. The current balance must be readable but not writable.

**Expected Approach:** Identify which fields should be private, which operations form the public interface, and how validation logic lives inside the class rather than being scattered across callers.

<details>
<summary>Hints</summary>

1. Make the balance field private and expose it only through a read-only accessor (getter with no public setter).
2. The `deposit` and `withdraw` methods are the only paths that modify the balance, so all validation (positive amount, sufficient funds) belongs inside those methods.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Use encapsulation to guard the balance behind a controlled interface.

1. Declare a private field `balance` initialized to zero (or an opening amount passed to the constructor).
2. Provide a public `getBalance()` method that returns the current balance. Do not provide a `setBalance()` method.
3. Implement `deposit(amount)`:
   - Validate that `amount > 0`; reject otherwise.
   - Add `amount` to `balance`.
4. Implement `withdraw(amount)`:
   - Validate that `amount > 0` and `amount <= balance`; reject otherwise.
   - Subtract `amount` from `balance`.
5. All invariants (non-negative balance, positive transaction amounts) are enforced inside the class, so callers cannot put the account into an invalid state.

This keeps the balance consistent regardless of how many different modules interact with the account.

</details>

---

## Problem 2 — Abstracting a Notification System

**Difficulty:** Easy
**Estimated Time:** 30 minutes

### Problem Statement

Design an abstraction layer for a notification system that can deliver messages through multiple channels (email, SMS, push notification). The rest of the application should interact with a single, uniform interface without knowing which channel is being used.

### Scenario

**Context:** An e-commerce platform sends order confirmations, shipping updates, and promotional offers to customers. Today it only supports email, but the product team plans to add SMS next quarter and push notifications later.

**Requirements:** Define an abstract interface that all notification channels implement. The order service should send notifications without referencing any specific channel. Adding a new channel in the future must not require changes to the order service code.

**Expected Approach:** Identify the common operations shared by all channels, define them in an abstract type, and show how concrete channel classes implement that contract.

<details>
<summary>Hints</summary>

1. Think about what every notification channel has in common — they all need a recipient and a message body, and they all perform a "send" action.
2. Define an abstract class or interface with a `send(recipient, message)` method, then let each channel provide its own implementation.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Use abstraction to decouple the notification consumer from the delivery mechanism.

1. Define an interface (or abstract class) `NotificationChannel` with a method `send(recipient, message)`.
2. Create concrete classes `EmailChannel`, `SmsChannel`, and `PushChannel`, each implementing `send` with channel-specific logic (SMTP call, SMS gateway API, push service SDK).
3. The order service depends only on `NotificationChannel`, never on a concrete class. It receives the channel through constructor injection or a factory.
4. To add a new channel later, create a new class that implements `NotificationChannel` — no existing code changes.

This is the core benefit of abstraction: consumers program against a contract, not an implementation.

</details>

---

## Problem 3 — Modeling a Vehicle Hierarchy with Inheritance

**Difficulty:** Easy
**Estimated Time:** 35 minutes

### Problem Statement

Design a class hierarchy for a fleet management system that tracks different vehicle types (cars, trucks, motorcycles). Each vehicle shares common attributes (make, model, year, mileage) but has type-specific behavior for calculating maintenance schedules.

### Scenario

**Context:** A logistics company operates a mixed fleet of cars, trucks, and motorcycles. The maintenance department needs a unified dashboard that lists every vehicle and its next scheduled service date, but the maintenance rules differ by vehicle type — trucks require service every 10,000 miles, cars every 15,000 miles, and motorcycles every 5,000 miles.

**Requirements:** Shared attributes and common behavior should be defined once in a base class. Each vehicle type must provide its own maintenance-interval logic. The dashboard code should be able to iterate over a collection of vehicles and call a single method to get the next service mileage, regardless of type.

**Expected Approach:** Decide what belongs in the base class versus the subclasses. Consider whether the base class should be abstract or concrete, and how the maintenance calculation is overridden per type.

<details>
<summary>Hints</summary>

1. Place shared fields (`make`, `model`, `year`, `mileage`) and any truly common methods in a base `Vehicle` class.
2. Declare `nextServiceMileage()` as an abstract method in the base class so each subclass is forced to provide its own interval logic.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Use inheritance to share common structure while allowing specialized behavior.

1. Define an abstract class `Vehicle` with fields `make`, `model`, `year`, `mileage` and a concrete method `describe()` that returns a summary string.
2. Declare an abstract method `nextServiceMileage()` in `Vehicle`.
3. Create subclasses `Car`, `Truck`, and `Motorcycle`, each implementing `nextServiceMileage()`:
   - `Truck`: return the next multiple of 10,000 above current mileage.
   - `Car`: return the next multiple of 15,000 above current mileage.
   - `Motorcycle`: return the next multiple of 5,000 above current mileage.
4. The dashboard iterates over a list of `Vehicle` objects and calls `nextServiceMileage()` on each — it never needs to know the concrete type.

This avoids duplicating shared fields across three classes while still allowing each type to customize its maintenance logic.

</details>

---

## Problem 4 — Designing a Polymorphic Payment Processor

**Difficulty:** Medium
**Estimated Time:** 45 minutes

### Problem Statement

Design a payment processing module for an online store that supports multiple payment methods (credit card, PayPal, bank transfer). The checkout flow should process any payment method through a uniform interface, and the system must be easy to extend when new payment methods are added.

### Scenario

**Context:** An online marketplace currently accepts credit cards only. The business wants to add PayPal immediately and bank transfers next month. The checkout service is deeply coupled to the credit card SDK — every function directly calls credit-card-specific methods, making it painful to add alternatives.

**Requirements:** Refactor the design so the checkout service depends on an abstraction rather than a concrete payment method. Each payment method must handle authorization, charging, and refunding through the same interface. Adding a new payment method must not require modifying the checkout service. The design should also handle the fact that different methods have different data requirements (e.g., credit card needs a card number and CVV; PayPal needs an email; bank transfer needs an account and routing number).

**Expected Approach:** Apply polymorphism to let the checkout service treat all payment methods uniformly. Consider how method-specific configuration data is supplied without leaking into the shared interface.

<details>
<summary>Hints</summary>

1. Define a `PaymentProcessor` interface with methods like `authorize(amount)`, `charge(amount)`, and `refund(transactionId, amount)`.
2. Each concrete processor (CreditCardProcessor, PayPalProcessor, BankTransferProcessor) is constructed with its own configuration data, so the shared interface stays clean.
3. A factory or registry can map a payment-method identifier to the correct processor instance at runtime.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Use polymorphism combined with a factory to decouple checkout from payment details.

1. Define an interface `PaymentProcessor` with:
   - `authorize(amount) -> AuthResult`
   - `charge(amount) -> ChargeResult`
   - `refund(transactionId, amount) -> RefundResult`
2. Implement concrete classes:
   - `CreditCardProcessor` — constructed with card number, expiry, CVV; calls the card network SDK internally.
   - `PayPalProcessor` — constructed with PayPal email/token; calls the PayPal API internally.
   - `BankTransferProcessor` — constructed with account and routing numbers; calls the ACH API internally.
3. Create a `PaymentProcessorFactory` that accepts a payment-method identifier (e.g., `"credit_card"`, `"paypal"`) and the associated configuration, then returns the appropriate `PaymentProcessor` instance.
4. The checkout service calls `factory.create(methodId, config)` to obtain a processor, then calls `authorize` and `charge` without knowing which concrete class it holds.
5. Adding a new payment method means writing a new class and registering it in the factory — zero changes to checkout.

Method-specific data stays in each constructor, keeping the shared interface free of payment-method details.

</details>

---

## Problem 5 — Refactoring a Monolithic Report Generator

**Difficulty:** Hard
**Estimated Time:** 60 minutes

### Problem Statement

A reporting module generates financial reports in a single, 800-line class that mixes data retrieval, business-rule calculations, and output formatting (PDF, CSV, HTML). Redesign this module using object-oriented principles so that each responsibility is isolated, new output formats can be added without modifying existing code, and the business rules can be tested independently of data access and formatting.

### Scenario

**Context:** A fintech startup's `ReportGenerator` class fetches transaction data from the database, applies tax calculations and currency conversions, then renders the result as a PDF. A new requirement asks for CSV and HTML output. The team also wants to unit-test the tax logic without needing a live database connection.

**Requirements:**
- Separate data retrieval, business logic, and formatting into distinct classes.
- The business-logic layer must not depend on concrete data-access or formatting classes.
- Adding a new output format (e.g., Excel) must not require changes to existing formatting classes or the business-logic layer.
- The tax-calculation and currency-conversion logic must be testable with in-memory data (no database required).
- The design should use encapsulation, abstraction, inheritance, and/or polymorphism where appropriate — justify each choice.

**Expected Approach:** Identify the three responsibilities tangled in the monolith. Define abstractions for data access and formatting. Apply dependency injection so the business-logic layer depends on interfaces. Show how the classes collaborate to produce a report.

<details>
<summary>Hints</summary>

1. Start by identifying the three axes of change: where data comes from, how it is processed, and how it is rendered. Each axis should map to its own class or interface.
2. Define a `DataSource` interface (with a method like `fetchTransactions(dateRange)`) so the business layer can work with any data provider — real database, mock, or file.
3. Define a `ReportFormatter` interface (with a method like `render(reportData)`) so new output formats are just new implementations.
4. The business-logic class (`ReportEngine`) takes a `DataSource` and a `ReportFormatter` through its constructor (dependency injection) and orchestrates the flow: fetch → calculate → format.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Decompose the monolith into three layers connected by abstractions and dependency injection.

1. **Data Access Layer**
   - Define an interface `DataSource` with `fetchTransactions(dateRange) -> List<Transaction>`.
   - Implement `DatabaseSource` for production and `InMemorySource` for testing.
   - *Principle used:* Abstraction — the rest of the system programs against `DataSource`, not a database client.

2. **Business Logic Layer**
   - Create a `TaxCalculator` class with a method `applyTax(transactions) -> List<TaxedTransaction>`. Encapsulate tax rules (rates, thresholds) as private fields.
   - Create a `CurrencyConverter` class with `convert(amount, fromCurrency, toCurrency) -> convertedAmount`. Encapsulate exchange-rate lookup internally.
   - Create a `ReportEngine` class that:
     a. Receives a `DataSource`, `TaxCalculator`, `CurrencyConverter`, and `ReportFormatter` via constructor injection.
     b. Calls `dataSource.fetchTransactions(dateRange)`.
     c. Passes transactions through `taxCalculator.applyTax(...)`.
     d. Converts amounts via `currencyConverter.convert(...)`.
     e. Calls `formatter.render(processedData)` to produce output.
   - *Principles used:* Encapsulation (tax rules hidden inside `TaxCalculator`), Dependency Injection (engine depends on abstractions).

3. **Formatting Layer**
   - Define an interface `ReportFormatter` with `render(reportData) -> output`.
   - Implement `PdfFormatter`, `CsvFormatter`, `HtmlFormatter`.
   - To add Excel later, create `ExcelFormatter` implementing `ReportFormatter` — no existing class changes.
   - *Principles used:* Polymorphism (engine calls `render` without knowing the format), Inheritance/Interface implementation (each formatter fulfills the contract).

4. **Collaboration Flow**
   ```
   ReportEngine(dataSource, taxCalc, currencyConv, formatter)
       transactions = dataSource.fetchTransactions(dateRange)
       taxed = taxCalc.applyTax(transactions)
       converted = currencyConv.convertAll(taxed, targetCurrency)
       output = formatter.render(converted)
       return output
   ```

5. **Testability:** Unit-test `TaxCalculator` and `CurrencyConverter` with plain in-memory data. Test `ReportEngine` by injecting `InMemorySource` and a simple `TestFormatter` that captures output — no database or file system needed.

</details>
