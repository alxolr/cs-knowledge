# SOLID Principles

**Track:** Design Concepts
**Difficulty Tier:** Beginner
**Prerequisites:** [Object-Oriented Design Principles](../object-oriented-design-principles/README.md)

## Concept Overview

SOLID is an acronym for five design principles that guide developers toward writing maintainable, flexible, and understandable object-oriented code. Coined by Robert C. Martin, these principles address common sources of rigidity and fragility in software systems. They are: Single Responsibility (SRP), Open/Closed (OCP), Liskov Substitution (LSP), Interface Segregation (ISP), and Dependency Inversion (DIP).

Each principle tackles a specific kind of design pain. SRP keeps classes focused so that changes in one business rule don't ripple across unrelated code. OCP encourages extending behavior through new code rather than editing existing, tested code. LSP ensures that subtype substitution doesn't break expectations. ISP prevents clients from being forced to depend on methods they never call. DIP decouples high-level policy from low-level implementation details.

These five principles work together as a toolkit. Applying them individually improves code quality; applying them in concert produces systems that are easy to test, extend, and refactor. The problems in this module give you practice identifying violations of each principle and redesigning code to follow them.

---

## Problem 1 — Splitting a User Service (Single Responsibility)

**Difficulty:** Easy
**Estimated Time:** 30 minutes

### Problem Statement

A `UserService` class handles user registration, password hashing, email verification, and audit logging. Every time the email provider changes or the audit format is updated, the `UserService` class must be modified and retested. Redesign the class so that each responsibility lives in its own unit.

### Scenario

**Context:** A SaaS platform's `UserService` has grown to 500 lines. It validates registration input, hashes passwords with bcrypt, sends verification emails through an SMTP client, and writes audit entries to a log file. A recent switch from SMTP to a third-party email API required changes deep inside `UserService`, which accidentally broke the audit logging.

**Requirements:** Separate the class so that changes to email delivery, password hashing, or audit logging do not require modifying the registration logic. Each resulting class should have exactly one reason to change.

**Expected Approach:** Identify the distinct responsibilities, extract each into its own class, and have the registration orchestrator delegate to them.

<details>
<summary>Hints</summary>

1. Count the reasons the current class might change: email provider swap, hashing algorithm update, audit format change, registration rule change — that is four reasons, so the class has at least four responsibilities.
2. Extract each responsibility into its own class (`PasswordHasher`, `EmailVerifier`, `AuditLogger`) and let a slim `UserRegistrationService` orchestrate them.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Apply the Single Responsibility Principle by extracting each axis of change into a dedicated class.

1. Create `PasswordHasher` with a method `hash(plainText) -> hashedPassword`. It encapsulates the hashing algorithm choice (bcrypt, argon2, etc.).
2. Create `EmailVerifier` with a method `sendVerification(emailAddress, token)`. It encapsulates the email delivery mechanism.
3. Create `AuditLogger` with a method `log(event, metadata)`. It encapsulates the log format and destination.
4. Refactor `UserRegistrationService` to accept these three collaborators via constructor injection. Its only job is to orchestrate the registration flow:
   a. Validate input.
   b. Call `passwordHasher.hash(password)`.
   c. Persist the user record.
   d. Call `emailVerifier.sendVerification(email, token)`.
   e. Call `auditLogger.log("USER_REGISTERED", userMetadata)`.
5. Now each class has exactly one reason to change, and modifying the email provider cannot break audit logging.

</details>

---

## Problem 2 — Extending a Discount Engine (Open/Closed)

**Difficulty:** Easy
**Estimated Time:** 30 minutes

### Problem Statement

An e-commerce platform calculates discounts using a single method containing a long `if/else` chain: one branch for percentage discounts, one for buy-one-get-one, one for seasonal promotions, and so on. Every new promotion type requires editing this method. Redesign the discount engine so that new discount types can be added without modifying existing code.

### Scenario

**Context:** The marketing team launches new promotion types every few weeks. Each time, a developer must add another branch to the `calculateDiscount` method, retest the entire method, and hope the new branch doesn't interfere with existing ones. Last month a holiday promotion branch accidentally overrode the loyalty discount for some customers.

**Requirements:** The existing discount types (percentage, buy-one-get-one, seasonal) must continue to work unchanged. Adding a new discount type (e.g., "spend $100 get $20 off") must not require modifying any existing discount class or the engine that applies them.

**Expected Approach:** Replace the conditional chain with a polymorphic design where each discount type is a separate class implementing a common interface, and the engine iterates over a collection of discount strategies.

<details>
<summary>Hints</summary>

1. Define a `DiscountStrategy` interface with a method like `apply(order) -> discountedOrder` or `calculate(order) -> discountAmount`.
2. Each promotion type becomes its own class implementing `DiscountStrategy`. The engine holds a list of strategies and applies them in sequence — no conditionals needed.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Apply the Open/Closed Principle by making the engine open for extension (new strategies) but closed for modification.

1. Define an interface `DiscountStrategy` with `calculate(order) -> discountAmount`.
2. Implement existing types:
   - `PercentageDiscount` — constructed with a percentage; returns `order.total * percentage`.
   - `BuyOneGetOneDiscount` — identifies eligible items and returns the value of the free items.
   - `SeasonalDiscount` — checks the current date against a promotion window and applies a fixed reduction.
3. Create a `DiscountEngine` that holds a `List<DiscountStrategy>`. Its `applyDiscounts(order)` method iterates over the list, calls `calculate` on each, and sums the results (or applies them in priority order).
4. To add "spend $100 get $20 off," create `ThresholdDiscount` implementing `DiscountStrategy` and register it with the engine. No existing class is touched.
5. The `if/else` chain is gone. Each strategy is independently testable, and the engine's logic is a simple loop.

</details>

---

## Problem 3 — Fixing a Broken Substitution (Liskov Substitution)

**Difficulty:** Easy
**Estimated Time:** 35 minutes

### Problem Statement

A file storage library defines a base class `ReadWriteFile` with methods `read()` and `write(data)`. A subclass `ReadOnlyFile` extends it but throws an exception in `write()`. Client code that accepts any `ReadWriteFile` crashes when it receives a `ReadOnlyFile`. Redesign the hierarchy so that substituting any subclass for its parent never violates the caller's expectations.

### Scenario

**Context:** A document management system stores files on disk. The `ReadWriteFile` base class works fine for normal documents. The team added `ReadOnlyFile` as a subclass for archived documents, overriding `write()` to throw an `UnsupportedOperationException`. A batch migration script that calls `write()` on every file in a folder now fails unpredictably whenever it encounters an archived file.

**Requirements:** Archived files must genuinely be read-only — they must not expose a `write` method at all. Editable files must support both reading and writing. Any code that accepts a type must be able to use all of that type's methods safely, without runtime surprises.

**Expected Approach:** Restructure the type hierarchy so that read-only and read-write capabilities are expressed through separate interfaces or a revised inheritance tree, eliminating the need to throw exceptions for unsupported operations.

<details>
<summary>Hints</summary>

1. The violation occurs because `ReadOnlyFile` cannot fulfill the full contract of `ReadWriteFile`. A subclass must be usable everywhere its parent is expected — throwing on an inherited method breaks that promise.
2. Split the hierarchy: define a `Readable` interface with `read()` and a `Writable` interface with `write(data)`. Compose them as needed rather than inheriting a combined class.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Apply the Liskov Substitution Principle by ensuring every subtype fully satisfies its parent's contract.

1. Define an interface `Readable` with `read() -> data`.
2. Define an interface `Writable` with `write(data)`.
3. Create `ReadOnlyFile` implementing only `Readable`. It has no `write` method, so there is nothing to throw on.
4. Create `ReadWriteFile` implementing both `Readable` and `Writable`.
5. The batch migration script accepts `Writable` (or `Readable & Writable`) — it will never receive a `ReadOnlyFile` because `ReadOnlyFile` does not implement `Writable`.
6. Code that only needs to read accepts `Readable` and works with both file types.
7. No method ever throws `UnsupportedOperationException`. Every object fully honors the contract of every type it claims to implement.

</details>

---

## Problem 4 — Slimming a Bloated Interface (Interface Segregation)

**Difficulty:** Medium
**Estimated Time:** 45 minutes

### Problem Statement

A project management tool defines a single `ProjectMember` interface with methods for viewing tasks, creating tasks, assigning tasks, managing sprints, generating reports, and administering user roles. Every role in the system — viewer, developer, scrum master, admin — must implement this interface, even though most roles use only a subset of the methods. Redesign the interface structure so that each role depends only on the capabilities it actually needs.

### Scenario

**Context:** The tool has four user roles. Viewers can only see tasks. Developers can view and create tasks. Scrum masters can view tasks, manage sprints, and generate reports. Admins can do everything. Today all four role classes implement `ProjectMember`, stubbing out unused methods with empty bodies or `throw new UnsupportedOperationException()`. When a new method is added to `ProjectMember`, all four classes must be updated — even if only one role uses the new method.

**Requirements:** Each role class should implement only the interfaces whose methods it genuinely uses. Adding a capability (e.g., "export to PDF") should affect only the roles that gain that capability, not every role in the system. No method should ever need a stub or exception-throwing implementation.

**Expected Approach:** Break the fat interface into several focused interfaces aligned with capabilities, then have each role implement the combination it needs.

<details>
<summary>Hints</summary>

1. Group the methods by capability: viewing, task authoring, sprint management, reporting, and administration. Each group becomes its own interface.
2. A role class implements only the interfaces matching its permissions. The `Viewer` class implements `TaskViewing`; the `Admin` class implements all five interfaces.
3. Client code that only needs to display tasks accepts `TaskViewing`, not the entire `ProjectMember` mega-interface.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Apply the Interface Segregation Principle by splitting the fat interface into cohesive, role-aligned interfaces.

1. Define focused interfaces:
   - `TaskViewing` — `viewTasks()`, `viewTaskDetails(taskId)`
   - `TaskAuthoring` — `createTask(task)`, `assignTask(taskId, userId)`
   - `SprintManagement` — `createSprint(sprint)`, `closeSprint(sprintId)`
   - `Reporting` — `generateReport(type, dateRange)`
   - `UserAdministration` — `addUser(user)`, `removeUser(userId)`, `changeRole(userId, role)`
2. Implement role classes:
   - `Viewer` implements `TaskViewing`.
   - `Developer` implements `TaskViewing`, `TaskAuthoring`.
   - `ScrumMaster` implements `TaskViewing`, `SprintManagement`, `Reporting`.
   - `Admin` implements `TaskViewing`, `TaskAuthoring`, `SprintManagement`, `Reporting`, `UserAdministration`.
3. No class contains stub methods or exception-throwing overrides. Each class fully implements every method of every interface it declares.
4. Client code depends on the narrowest interface it needs. The task board UI accepts `TaskViewing`; the sprint planning page accepts `SprintManagement`.
5. Adding a new capability (e.g., `Exporting` with `exportToPdf()`) means creating a new interface and adding it only to the roles that need it — other roles are untouched.

</details>

---

## Problem 5 — Inverting a Tightly Coupled Order Processor (Dependency Inversion)

**Difficulty:** Hard
**Estimated Time:** 60 minutes

### Problem Statement

An order processing service directly instantiates a MySQL database client, a Stripe payment gateway, and an SMTP email sender inside its constructor. This makes the service impossible to unit-test without live external services and forces a rewrite whenever the company switches vendors. Redesign the service so that high-level business logic depends on abstractions rather than concrete implementations, and low-level details are injected from the outside.

### Scenario

**Context:** A food delivery startup's `OrderProcessor` class creates `new MySqlClient(...)`, `new StripeGateway(...)`, and `new SmtpMailer(...)` in its constructor. The class orchestrates the order flow: validate the order, charge the customer, persist the order, and send a confirmation email. The team wants to switch from Stripe to a new payment provider, replace SMTP with a transactional email service, and write unit tests that run in milliseconds without hitting any external system.

**Requirements:**
- The `OrderProcessor` must not instantiate any concrete infrastructure class directly.
- Each external dependency (database, payment, email) must be represented by an abstraction that the processor depends on.
- Concrete implementations must be supplied from outside the processor (via constructor injection).
- It must be possible to run the full order-processing logic in a unit test using in-memory fakes for all three dependencies.
- Swapping from Stripe to a new payment provider must require only a new implementation class and a configuration change — zero modifications to `OrderProcessor`.

**Expected Approach:** Define interfaces for each infrastructure concern, refactor the processor to accept them through its constructor, and demonstrate how production wiring and test wiring each supply different concrete implementations.

<details>
<summary>Hints</summary>

1. Identify the three external dependencies: database persistence, payment charging, and email sending. Each one should become an interface.
2. The `OrderProcessor` constructor should accept these three interfaces as parameters instead of creating concrete objects with `new`.
3. For testing, create lightweight in-memory implementations (e.g., `InMemoryOrderStore`, `FakePaymentGateway`, `FakeMailer`) that record calls and return canned responses.
4. For production, a composition root (e.g., `main` function or DI container) wires the real implementations (`MySqlOrderStore`, `StripePaymentGateway`, `SmtpMailer`) into the processor.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Apply the Dependency Inversion Principle so that the high-level order logic depends on abstractions, and low-level infrastructure details are injected.

1. **Define abstractions:**
   - `OrderStore` interface — `save(order)`, `findById(orderId) -> Order`
   - `PaymentGateway` interface — `charge(amount, paymentDetails) -> PaymentResult`
   - `NotificationSender` interface — `sendConfirmation(email, orderSummary)`

2. **Refactor `OrderProcessor`:**
   - Constructor accepts `OrderStore`, `PaymentGateway`, and `NotificationSender`.
   - `processOrder(order)` method:
     a. Validate the order (business rules live here).
     b. Call `paymentGateway.charge(order.total, order.paymentDetails)`.
     c. If payment succeeds, call `orderStore.save(order)`.
     d. Call `notificationSender.sendConfirmation(order.customerEmail, order.summary())`.
     e. Return the result.
   - The processor has zero `import` or `require` statements for MySQL, Stripe, or SMTP.

3. **Production wiring (composition root):**
   ```
   store = new MySqlOrderStore(connectionConfig)
   gateway = new StripePaymentGateway(apiKey)
   mailer = new SmtpMailer(smtpConfig)
   processor = new OrderProcessor(store, gateway, mailer)
   ```

4. **Test wiring:**
   ```
   store = new InMemoryOrderStore()
   gateway = new FakePaymentGateway(alwaysSucceeds=true)
   mailer = new FakeMailer()
   processor = new OrderProcessor(store, gateway, mailer)
   processor.processOrder(testOrder)
   assert store.savedOrders.contains(testOrder)
   assert mailer.sentEmails.size == 1
   ```

5. **Vendor swap:** To replace Stripe, create `NewProviderPaymentGateway` implementing `PaymentGateway`, update the composition root to instantiate it, and deploy. `OrderProcessor` is never modified.

6. **Key insight:** Both the high-level module (`OrderProcessor`) and the low-level modules (`MySqlOrderStore`, `StripePaymentGateway`) depend on the abstractions (`OrderStore`, `PaymentGateway`). The abstractions are owned by the high-level layer, not the infrastructure layer — this is the "inversion" in Dependency Inversion.

</details>
