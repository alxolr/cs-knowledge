# Design Patterns (Behavioral)

**Track:** Design Concepts
**Difficulty Tier:** Beginner
**Prerequisites:** [SOLID Principles](../solid-principles/README.md), [Design Patterns (Structural)](../design-patterns-structural/README.md)

## Concept Overview

Behavioral design patterns focus on how objects communicate, delegate responsibilities, and coordinate workflows. While creational patterns address object construction and structural patterns address object composition, behavioral patterns address the assignment of responsibilities between objects and the algorithms that govern their interaction. They help keep complex control flow manageable by distributing behavior across collaborating objects rather than concentrating it in a single class.

The classic behavioral patterns include Observer, Strategy, Command, State, Template Method, Chain of Responsibility, Iterator, Mediator, and Visitor, among others. Observer lets one object notify many dependents when its state changes. Strategy encapsulates interchangeable algorithms behind a common interface. Command turns a request into a stand-alone object, enabling undo, queuing, and logging. State allows an object to change its behavior when its internal state changes. Template Method defines the skeleton of an algorithm in a base class and lets subclasses fill in specific steps.

Choosing the right behavioral pattern depends on where variability lives. If the variation is in an algorithm, Strategy is a natural fit. If the variation is in an object's lifecycle stages, State is more appropriate. If you need loose coupling between a producer of events and many consumers, Observer is the answer. The problems in this module present realistic scenarios where each pattern is the natural fit, giving you practice recognizing the communication problem and selecting the right tool.

---

## Problem 1 — Live Sports Score Dashboard (Observer)

**Difficulty:** Easy
**Estimated Time:** 30 minutes

### Problem Statement

Design a live sports score system where a central score service tracks the current state of ongoing matches and multiple display components — a web dashboard, a mobile push-notification sender, and a statistics aggregator — all update automatically whenever a score changes. Adding or removing display components at runtime must not require changes to the score service.

### Scenario

**Context:** A sports media company streams live match data. A `MatchScoreService` receives real-time score updates from stadium feeds. Three independent consumers need to react to every score change: a web dashboard refreshes its UI, a mobile service pushes notifications to subscribers, and a statistics engine recalculates running averages.

**Requirements:** The score service must not hold direct references to concrete display classes. New consumers must be attachable at runtime without modifying the score service. When a score changes, all registered consumers must be notified with the updated match data. Consumers must be removable at runtime (e.g., the mobile service unsubscribes during maintenance windows).

**Expected Approach:** Identify the one-to-many dependency between the score service and its consumers. Apply the Observer pattern so that the score service (subject) maintains a list of observers and notifies them generically, without knowing their concrete types.

<details>
<summary>Hints</summary>

1. Define an `Observer` interface with a method like `update(matchData)` that all consumers implement.
2. The `MatchScoreService` maintains a list of `Observer` references and provides `subscribe(observer)` and `unsubscribe(observer)` methods.
3. When a score changes, the service iterates through its observer list and calls `update()` on each one — it never needs to know whether it is talking to a dashboard, a push service, or a statistics engine.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Apply the Observer pattern so the score service broadcasts changes to all registered consumers without coupling to their concrete types.

1. Define an `Observer` interface with a single method `update(matchData)`.
2. Define a `MatchScoreService` (the subject) with:
   - A private list of `Observer` references.
   - `subscribe(observer)` — adds an observer to the list.
   - `unsubscribe(observer)` — removes an observer from the list.
   - `notifyAll(matchData)` — iterates through the list and calls `update(matchData)` on each observer.
   - `updateScore(matchId, newScore)` — updates internal state and calls `notifyAll()` with the new match data.
3. Implement concrete observers:
   - `WebDashboard` implements `Observer` — refreshes the UI with the latest scores.
   - `MobilePushService` implements `Observer` — sends push notifications to subscribed users.
   - `StatisticsAggregator` implements `Observer` — recalculates running averages and stores them.
4. At runtime, consumers register themselves: `scoreService.subscribe(dashboard)`. When a score changes, all three receive the update automatically.
5. During maintenance, a consumer can call `scoreService.unsubscribe(mobilePush)` and re-subscribe later.

**Why Observer fits:** The score service has a one-to-many relationship with consumers that must stay loosely coupled. Observer decouples the subject from its dependents, allowing consumers to be added or removed at runtime without modifying the service.

</details>

---

## Problem 2 — Flexible Shipping Cost Calculator (Strategy)

**Difficulty:** Easy
**Estimated Time:** 30 minutes

### Problem Statement

Design a shipping cost calculator for an e-commerce platform that supports multiple shipping methods — standard ground, express air, and same-day courier. Each method uses a different pricing algorithm based on package weight, distance, and service-specific surcharges. The platform must be able to switch shipping methods at runtime (e.g., when a user changes their selection in the checkout flow) and new shipping methods must be addable without modifying the calculator.

### Scenario

**Context:** An e-commerce checkout page lets customers choose a shipping method. Standard ground charges a flat rate per kilogram plus a distance fee. Express air uses a tiered pricing table based on weight brackets. Same-day courier charges a premium base fee plus a per-kilometer rate. The business adds new shipping partners regularly.

**Requirements:** The checkout service must calculate shipping cost by delegating to the selected method's algorithm. Switching between methods must not require conditional branching (no `if/else` or `switch` on method type). Adding a new shipping method must only require creating a new class, not modifying existing code. All shipping methods must share a common interface so the calculator treats them uniformly.

**Expected Approach:** Recognize that the variation is in the pricing algorithm, not in the overall checkout workflow. Encapsulate each algorithm behind a common interface and inject the chosen strategy into the calculator at runtime.

<details>
<summary>Hints</summary>

1. Define a `ShippingStrategy` interface with a method `calculateCost(weight, distance) -> cost`.
2. Implement `GroundShipping`, `ExpressAirShipping`, and `SameDayCourier`, each with its own pricing logic inside `calculateCost()`.
3. The `ShippingCalculator` holds a reference to a `ShippingStrategy` and delegates cost computation to it. The strategy can be swapped at runtime via a setter method.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Use the Strategy pattern to encapsulate each shipping pricing algorithm behind a common interface, allowing runtime selection.

1. Define a `ShippingStrategy` interface with method `calculateCost(weight, distance) -> decimal`.
2. Implement concrete strategies:
   - `GroundShipping`: cost = (weight × ratePerKg) + (distance × distanceRate).
   - `ExpressAirShipping`: look up weight bracket in a tiered table, add distance surcharge.
   - `SameDayCourier`: cost = basePremiumFee + (distance × perKmRate).
3. Define `ShippingCalculator` with:
   - A private field `strategy` of type `ShippingStrategy`.
   - `setStrategy(strategy)` — allows runtime switching.
   - `calculate(weight, distance) -> decimal` — delegates to `strategy.calculateCost(weight, distance)`.
4. In the checkout flow:
   ```
   calculator = ShippingCalculator()
   calculator.setStrategy(GroundShipping())
   groundCost = calculator.calculate(weight, distance)
   calculator.setStrategy(ExpressAirShipping())
   expressCost = calculator.calculate(weight, distance)
   ```
5. Adding a new method (e.g., `DroneDelivery`) requires only a new class implementing `ShippingStrategy` — no changes to `ShippingCalculator` or existing strategies.

**Why Strategy fits:** The algorithms are interchangeable and selected at runtime. Strategy eliminates conditional branching, keeps each algorithm in its own class, and makes the system open for extension without modification.

</details>

---

## Problem 3 — Undo/Redo for a Text Editor (Command)

**Difficulty:** Easy
**Estimated Time:** 35 minutes

### Problem Statement

Design an undo/redo system for a simple text editor. The editor supports operations such as inserting text, deleting text, and replacing text. Each operation must be reversible. The user can undo multiple operations in sequence and then redo them. The system must also support macro recording — grouping a sequence of operations into a single undoable unit.

### Scenario

**Context:** A desktop text editor allows users to type, delete, and find-and-replace. Users expect Ctrl+Z to undo the last operation and Ctrl+Y to redo it. A power-user feature lets users record a macro (a sequence of edits) and replay it as a single action. If the macro is undone, all its constituent edits are reversed in one step.

**Requirements:** Each edit operation must be encapsulated as an object that knows how to execute itself and how to reverse itself. An undo stack stores executed commands; a redo stack stores undone commands. Executing a new command clears the redo stack. A macro command groups multiple commands and executes/undoes them as a batch. The editor itself should not contain undo logic — it simply executes command objects.

**Expected Approach:** Recognize that turning operations into objects enables storing, undoing, and composing them. Apply the Command pattern to decouple the editor from the operations it performs. Use a composite command for macros.

<details>
<summary>Hints</summary>

1. Define a `Command` interface with `execute()` and `undo()` methods.
2. Each operation (insert, delete, replace) is a concrete command that stores enough state to reverse itself (e.g., `InsertCommand` stores the position and text so `undo()` can delete it).
3. The `CommandHistory` manages two stacks: `undoStack` and `redoStack`. Executing a command pushes it onto the undo stack and clears the redo stack. Undoing pops from undo, calls `undo()`, and pushes onto redo.
4. A `MacroCommand` holds a list of commands and delegates `execute()` and `undo()` to each one in order (and reverse order for undo).

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Use the Command pattern to encapsulate each editor operation as an object with execute and undo capabilities.

1. Define a `Command` interface:
   - `execute()` — performs the operation.
   - `undo()` — reverses the operation.
2. Implement concrete commands:
   - `InsertCommand(editor, position, text)`: `execute()` inserts text at position; `undo()` deletes the same range.
   - `DeleteCommand(editor, position, length)`: `execute()` saves the deleted text and removes it; `undo()` re-inserts the saved text at position.
   - `ReplaceCommand(editor, position, oldText, newText)`: `execute()` replaces oldText with newText; `undo()` replaces newText with oldText.
3. Implement `MacroCommand(commands: List<Command>)`:
   - `execute()` calls `execute()` on each command in order.
   - `undo()` calls `undo()` on each command in reverse order.
4. Define `CommandHistory`:
   - `undoStack: Stack<Command>`, `redoStack: Stack<Command>`.
   - `executeCommand(cmd)` — calls `cmd.execute()`, pushes onto undoStack, clears redoStack.
   - `undo()` — pops from undoStack, calls `undo()`, pushes onto redoStack.
   - `redo()` — pops from redoStack, calls `execute()`, pushes onto undoStack.
5. The `TextEditor` class holds the document buffer and exposes mutation methods. It does not manage history — the `CommandHistory` orchestrates undo/redo.

**Why Command fits:** Each operation becomes a first-class object that can be stored, reversed, and composed. This cleanly separates the editor's document model from the undo/redo mechanism and enables macro recording as a composite command.

</details>

---

## Problem 4 — Order Lifecycle Management (State)

**Difficulty:** Medium
**Estimated Time:** 45 minutes

### Problem Statement

Design the order lifecycle for an e-commerce fulfillment system where an order transitions through several states — Pending, Paid, Shipped, Delivered, and Cancelled. The behavior of operations like `pay()`, `ship()`, `deliver()`, and `cancel()` depends entirely on the order's current state. For example, an order can only be shipped after it is paid, and a delivered order cannot be cancelled. The system must enforce valid transitions and make it easy to add new states (e.g., "Returned") without modifying existing state logic.

### Scenario

**Context:** An e-commerce platform processes thousands of orders daily. Each order moves through a lifecycle: it starts as Pending, becomes Paid when payment is confirmed, transitions to Shipped when the warehouse dispatches it, and finally becomes Delivered. At certain points the order can be Cancelled (before shipping only). Customer service frequently requests new states like "On Hold" or "Returned," and the current implementation — a large `switch` statement inside the `Order` class — is fragile and error-prone.

**Requirements:** Each state must encapsulate the behavior for all operations available in that state. Invalid operations (e.g., shipping an unpaid order) must be rejected cleanly with a meaningful error, not silently ignored. Transitioning to a new state must replace the order's behavior entirely. Adding a new state must require only a new class, not modifications to existing state classes or the `Order` class. The `Order` class itself should delegate all state-dependent behavior to its current state object.

**Expected Approach:** Recognize that the order's behavior changes completely based on its current state, and that a growing `switch` statement violates the Open/Closed Principle. Apply the State pattern to encapsulate each state's behavior in its own class and let the order delegate to the current state object.

<details>
<summary>Hints</summary>

1. Define an `OrderState` interface with methods `pay(order)`, `ship(order)`, `deliver(order)`, and `cancel(order)`. Each method either performs the action and transitions the order to a new state, or throws an error for invalid transitions.
2. Implement concrete states: `PendingState`, `PaidState`, `ShippedState`, `DeliveredState`, `CancelledState`. Each class defines which operations are valid and which are not.
3. The `Order` class holds a reference to its current `OrderState` and delegates all operations to it. A `setState(newState)` method allows the state object to trigger transitions.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Use the State pattern to encapsulate each order lifecycle phase in its own class, eliminating conditional logic from the `Order` class.

1. Define an `OrderState` interface:
   - `pay(order)`, `ship(order)`, `deliver(order)`, `cancel(order)`.
   - Default implementations can throw "invalid operation in current state" errors.
2. Implement concrete states:
   - `PendingState`:
     - `pay(order)` — processes payment, calls `order.setState(PaidState)`.
     - `cancel(order)` — marks order cancelled, calls `order.setState(CancelledState)`.
     - `ship()` and `deliver()` — throw "order must be paid first."
   - `PaidState`:
     - `ship(order)` — records shipment, calls `order.setState(ShippedState)`.
     - `cancel(order)` — initiates refund, calls `order.setState(CancelledState)`.
     - `pay()` — throw "already paid."
     - `deliver()` — throw "order must be shipped first."
   - `ShippedState`:
     - `deliver(order)` — confirms delivery, calls `order.setState(DeliveredState)`.
     - `cancel()` — throw "cannot cancel a shipped order."
     - `pay()` and `ship()` — throw appropriate errors.
   - `DeliveredState`: all operations throw "order already delivered" (terminal state).
   - `CancelledState`: all operations throw "order is cancelled" (terminal state).
3. Define the `Order` class:
   - Private field `state` of type `OrderState`, initialized to `PendingState`.
   - `setState(newState)` — replaces the current state.
   - `pay()` delegates to `state.pay(this)`.
   - `ship()` delegates to `state.ship(this)`.
   - `deliver()` delegates to `state.deliver(this)`.
   - `cancel()` delegates to `state.cancel(this)`.
4. Adding a new state like `ReturnedState` requires only a new class implementing `OrderState` and a transition from `DeliveredState` — no changes to other states or the `Order` class.

**Why State fits:** The order's behavior is entirely determined by its current state, and transitions between states are well-defined. State eliminates sprawling conditional logic, localizes each state's rules in its own class, and makes the system open for extension when new states are introduced.

</details>

---

## Problem 5 — Data Pipeline Processing Framework (Template Method)

**Difficulty:** Hard
**Estimated Time:** 60 minutes

### Problem Statement

Design a data pipeline framework that processes data from various sources (CSV files, database tables, REST APIs) through a fixed sequence of steps: connect to the source, extract raw data, validate and clean the data, transform it into a target schema, and load it into a data warehouse. The overall pipeline algorithm is identical for every source, but each step's implementation varies by source type. The framework must allow teams to add new source types by implementing only the steps that differ, while the pipeline orchestration logic remains in one place and cannot be accidentally overridden.

### Scenario

**Context:** A data engineering team builds ETL (Extract-Transform-Load) pipelines for a business intelligence platform. Every pipeline follows the same five-phase workflow, but the specifics differ: a CSV pipeline opens a file and parses rows, a database pipeline opens a JDBC connection and runs a query, and an API pipeline authenticates via OAuth and paginates through JSON responses. Currently, each pipeline is a standalone script with duplicated orchestration logic. When the team added a logging step between extract and validate, they had to update all three scripts — and missed one, causing a production bug.

**Requirements:**
- The five-step sequence (connect → extract → validate → transform → load) must be defined exactly once in a base class and must not be overridable by subclasses.
- Each step is a method that subclasses can override to provide source-specific behavior.
- The `validate` step has a sensible default implementation (null-check and type-check) that subclasses may optionally override for source-specific validation rules.
- The framework must support optional hook methods — `beforeExtract()` and `afterLoad()` — that do nothing by default but can be overridden for cross-cutting concerns like logging or metrics.
- Adding a new source type (e.g., an S3 Parquet pipeline) must require only subclassing and implementing the abstract steps, not modifying the base class.

**Expected Approach:** Recognize that the algorithm's structure is fixed but individual steps vary. Apply the Template Method pattern to define the skeleton in a base class, with abstract methods for mandatory steps and hook methods for optional extension points. Consider how to prevent subclasses from changing the step order.

<details>
<summary>Hints</summary>

1. Define an abstract class `DataPipeline` with a `final` (non-overridable) method `run()` that calls the five steps in order, plus the hook methods.
2. Declare `connect()`, `extract()`, `transform()`, and `load()` as abstract methods that subclasses must implement.
3. Provide a default implementation for `validate()` that performs basic null and type checks — subclasses can override it if they need source-specific validation.
4. Define `beforeExtract()` and `afterLoad()` as empty (no-op) methods that subclasses may optionally override for logging, metrics, or cleanup.
5. Making `run()` final ensures no subclass can alter the step sequence.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Use the Template Method pattern to define the pipeline algorithm once in a base class, with abstract and hook methods for subclass customization.

1. **Define the abstract base class `DataPipeline`:**
   ```
   abstract class DataPipeline:
       final method run():          // template method — not overridable
           connect()
           beforeExtract()          // hook — no-op by default
           rawData = extract()
           cleanData = validate(rawData)
           transformed = transform(cleanData)
           load(transformed)
           afterLoad()              // hook — no-op by default

       abstract method connect()
       abstract method extract() -> RawData
       method validate(data) -> CleanData:    // default implementation
           for each record in data:
               remove records with null required fields
               cast fields to expected types
           return cleanedData
       abstract method transform(data) -> TransformedData
       abstract method load(data)

       method beforeExtract():      // hook — empty by default
           pass
       method afterLoad():          // hook — empty by default
           pass
   ```

2. **Implement `CsvPipeline`:**
   - `connect()` — opens the CSV file handle.
   - `extract()` — parses rows into a list of records.
   - `transform()` — maps CSV column names to the warehouse schema.
   - `load()` — bulk-inserts records into the warehouse.
   - Inherits the default `validate()`.

3. **Implement `DatabasePipeline`:**
   - `connect()` — opens a JDBC connection with credentials.
   - `extract()` — executes a SQL query and fetches the result set.
   - `validate(data)` — overrides default to add referential integrity checks specific to the source database.
   - `transform()` — maps source table columns to the warehouse schema.
   - `load()` — uses batch upsert into the warehouse.
   - Overrides `beforeExtract()` to log the query being executed.

4. **Implement `ApiPipeline`:**
   - `connect()` — authenticates via OAuth and stores the access token.
   - `extract()` — paginates through the REST API, collecting all JSON responses.
   - `transform()` — flattens nested JSON into tabular rows matching the warehouse schema.
   - `load()` — streams records into the warehouse.
   - Overrides `afterLoad()` to emit a metric recording the number of records loaded and the elapsed time.

5. **Adding a new source (e.g., `S3ParquetPipeline`):**
   - Subclass `DataPipeline`, implement the four abstract methods for S3/Parquet specifics.
   - Optionally override `validate()` or the hooks. The step sequence is guaranteed by the `final run()` method.

**Why Template Method fits:** The algorithm's structure (connect → extract → validate → transform → load) is invariant, but individual steps vary by source type. Template Method defines the skeleton once, prevents subclasses from reordering steps, and provides hooks for optional cross-cutting concerns. This eliminates the duplicated orchestration logic that caused the production bug.

</details>