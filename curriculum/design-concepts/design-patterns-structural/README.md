# Design Patterns (Structural)

**Track:** Design Concepts
**Difficulty Tier:** Beginner
**Prerequisites:** [SOLID Principles](../solid-principles/README.md), [Design Patterns (Creational)](../design-patterns-creational/README.md)

## Concept Overview

Structural design patterns are concerned with how classes and objects are composed to form larger structures. While creational patterns focus on how objects are made, structural patterns focus on how objects are assembled and connected. They simplify relationships between entities, making systems easier to understand, extend, and maintain.

The seven classic structural patterns are Adapter, Bridge, Composite, Decorator, Facade, Flyweight, and Proxy. Adapter converts the interface of an existing class into one that a client expects, enabling classes with incompatible interfaces to work together. Bridge separates an abstraction from its implementation so the two can vary independently. Composite lets you treat individual objects and groups of objects uniformly through a tree structure. Decorator attaches additional responsibilities to an object dynamically, providing a flexible alternative to subclassing. Facade provides a simplified interface to a complex subsystem. Flyweight minimizes memory usage by sharing common state across many similar objects. Proxy provides a surrogate or placeholder for another object to control access to it.

Choosing the right structural pattern requires understanding the relationships in your design. Wrapping an incompatible third-party library calls for Adapter; adding optional behaviors at runtime calls for Decorator; simplifying a tangled subsystem calls for Facade. The problems in this module present realistic scenarios where each pattern is the natural fit, helping you build intuition for when and how to apply them.

---

## Problem 1 — Integrating a Legacy Payment Gateway (Adapter)

**Difficulty:** Easy
**Estimated Time:** 30 minutes

### Problem Statement

Design an adapter that allows a modern e-commerce checkout system to work with a legacy payment gateway whose interface is incompatible with the system's expected payment processor interface.

### Scenario

**Context:** An e-commerce platform defines a standard `PaymentProcessor` interface that all payment integrations must implement. The interface expects methods like `charge(amount, currency, cardToken) -> TransactionResult`. A legacy payment gateway your company acquired uses a completely different interface: `submitPayment(xmlPayload) -> xmlResponse`, where the payload and response are XML strings with a proprietary schema.

**Requirements:** The checkout system must call the legacy gateway through the standard `PaymentProcessor` interface without modifying either the checkout code or the legacy gateway code. The adapter must translate method calls, convert data formats (structured parameters to XML and XML responses back to `TransactionResult`), and handle any error-code mapping between the two systems.

**Expected Approach:** Recognize that two incompatible interfaces need to collaborate without modification. Apply the Adapter pattern to wrap the legacy gateway behind the expected interface, translating calls and data formats in the adapter layer.

<details>
<summary>Hints</summary>

1. Create a class `LegacyGatewayAdapter` that implements the `PaymentProcessor` interface and holds a reference to the `LegacyPaymentGateway` instance.
2. In the `charge()` method, build the XML payload from the structured parameters, call `submitPayment()`, then parse the XML response into a `TransactionResult`.
3. Map the legacy gateway's error codes to the standard error types expected by the checkout system.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Apply the Adapter pattern (object adapter variant) to wrap the legacy gateway behind the standard interface.

1. Define the `PaymentProcessor` interface with method `charge(amount, currency, cardToken) -> TransactionResult`.
2. The `LegacyPaymentGateway` class already exists with method `submitPayment(xmlPayload) -> xmlResponse`. This class cannot be modified.
3. Create `LegacyGatewayAdapter` implementing `PaymentProcessor`:
   - Constructor accepts a `LegacyPaymentGateway` instance and stores it as a private field.
   - `charge(amount, currency, cardToken)`:
     - Build an XML string from the parameters: `<payment><amount>...</amount><currency>...</currency><card>...</card></payment>`.
     - Call `legacyGateway.submitPayment(xmlPayload)`.
     - Parse the XML response to extract the transaction ID, status, and error code.
     - Map legacy status codes to `TransactionResult` fields (e.g., legacy code `"00"` → success, `"05"` → declined).
     - Return a `TransactionResult` object.
4. The checkout system creates `LegacyGatewayAdapter(legacyGateway)` and uses it as any other `PaymentProcessor` — no changes to checkout logic.

**Why Adapter fits:** Two existing interfaces are incompatible and neither can be modified. The adapter sits between them, translating calls and data formats so they can collaborate seamlessly.

</details>

---

## Problem 2 — Building a File System Tree (Composite)

**Difficulty:** Easy
**Estimated Time:** 30 minutes

### Problem Statement

Design a file system representation where files and directories are treated uniformly. A directory can contain files and other directories, and operations like calculating total size or listing contents should work the same way whether applied to a single file or an entire directory tree.

### Scenario

**Context:** A backup utility needs to scan a directory tree, calculate the total size of all files, and display a hierarchical listing. The utility should be able to ask any node — whether it is a single file or a deeply nested directory — for its total size, and the answer should include everything contained within it.

**Requirements:** Define a common interface that both files and directories implement. A file reports its own size. A directory reports the sum of sizes of all its children (files and subdirectories, recursively). The utility code that calculates sizes and prints listings should not need to distinguish between files and directories — it calls the same methods on both. Directories must support adding and removing children.

**Expected Approach:** Recognize the part-whole hierarchy: files are leaves and directories are composites that contain other components. Apply the Composite pattern so that clients treat individual files and directory trees uniformly through a shared interface.

<details>
<summary>Hints</summary>

1. Define a `FileSystemNode` interface with methods `getSize() -> int` and `getName() -> string`.
2. `File` implements `FileSystemNode` and returns its own size from `getSize()`.
3. `Directory` implements `FileSystemNode`, holds a list of `FileSystemNode` children, and computes `getSize()` by summing the sizes of all children recursively.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Use the Composite pattern to model the file system as a tree where files and directories share a common interface.

1. Define interface `FileSystemNode`:
   - `getName() -> string`
   - `getSize() -> int`
   - `display(indent) -> void` — prints the node with indentation for hierarchy.
2. Implement `File`:
   - Constructor takes `name` and `size`.
   - `getSize()` returns the file's size directly.
   - `display(indent)` prints the file name and size at the given indentation level.
3. Implement `Directory`:
   - Constructor takes `name`. Internally holds a list of `FileSystemNode` children.
   - `add(node)` and `remove(node)` manage children.
   - `getSize()` iterates over children, calling `child.getSize()` on each, and returns the sum. This naturally recurses into subdirectories.
   - `display(indent)` prints the directory name, then calls `child.display(indent + 1)` for each child.
4. Client code:
   ```
   root = Directory("project")
   src = Directory("src")
   src.add(File("main.py", 1200))
   src.add(File("utils.py", 800))
   root.add(src)
   root.add(File("README.md", 300))
   print(root.getSize())   // 2300
   root.display(0)          // hierarchical listing
   ```
5. The backup utility calls `getSize()` and `display()` on any `FileSystemNode` without checking its type.

**Why Composite fits:** The file system is a natural tree structure with a part-whole relationship. Composite lets clients treat leaves (files) and branches (directories) uniformly, eliminating type-checking conditionals throughout the codebase.

</details>

---

## Problem 3 — Extending a Beverage Ordering System (Decorator)

**Difficulty:** Easy
**Estimated Time:** 35 minutes

### Problem Statement

Design a beverage ordering system where customers can add optional extras — milk, whipped cream, caramel syrup, extra espresso shot — to a base drink. Each extra modifies the drink's description and price. The system must support any combination of extras without creating a separate class for every possible combination.

### Scenario

**Context:** A coffee shop application offers base drinks (espresso, house blend, dark roast) and a growing list of optional add-ons. The original design used subclasses for each combination (`EspressoWithMilk`, `EspressoWithMilkAndCaramel`, etc.), resulting in a class explosion. The shop frequently introduces new add-ons, making the subclass approach unmaintainable.

**Requirements:** Define a common interface for all beverages. Base drinks implement this interface directly. Each add-on wraps an existing beverage, augmenting its description and adding its cost to the wrapped beverage's cost. Add-ons must be stackable — a customer can add milk, then whipped cream, then caramel to the same drink. Adding a new add-on must not require modifying any existing class. The final beverage object must report the complete description and total cost including all add-ons.

**Expected Approach:** Recognize that behavior (description and cost) needs to be added dynamically at runtime in any combination. Subclassing every combination is impractical. Apply the Decorator pattern to wrap beverages with add-on layers, each adding its own contribution to description and cost.

<details>
<summary>Hints</summary>

1. Define a `Beverage` interface with `getDescription() -> string` and `getCost() -> float`.
2. Create an abstract `BeverageDecorator` class that implements `Beverage` and holds a reference to another `Beverage` (the wrapped component).
3. Each concrete decorator (e.g., `MilkDecorator`) delegates to the wrapped beverage and adds its own contribution: `getDescription()` returns `wrapped.getDescription() + ", Milk"` and `getCost()` returns `wrapped.getCost() + 0.50`.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Use the Decorator pattern to dynamically layer add-ons onto base beverages.

1. Define interface `Beverage`:
   - `getDescription() -> string`
   - `getCost() -> float`
2. Implement base drinks:
   - `Espresso`: description `"Espresso"`, cost `2.00`.
   - `HouseBlend`: description `"House Blend"`, cost `1.50`.
   - `DarkRoast`: description `"Dark Roast"`, cost `1.75`.
3. Define abstract class `BeverageDecorator` implementing `Beverage`:
   - Constructor takes a `Beverage` instance and stores it as `wrappedBeverage`.
   - Subclasses override `getDescription()` and `getCost()` to add their contribution.
4. Implement concrete decorators:
   - `MilkDecorator`: adds `", Milk"` to description, adds `0.50` to cost.
   - `WhipDecorator`: adds `", Whipped Cream"` to description, adds `0.70` to cost.
   - `CaramelDecorator`: adds `", Caramel"` to description, adds `0.60` to cost.
   - `ExtraShotDecorator`: adds `", Extra Shot"` to description, adds `0.80` to cost.
5. Client code stacks decorators:
   ```
   drink = Espresso()
   drink = MilkDecorator(drink)
   drink = CaramelDecorator(drink)
   print(drink.getDescription())  // "Espresso, Milk, Caramel"
   print(drink.getCost())         // 3.10
   ```
6. Adding a new add-on (e.g., `OatMilkDecorator`) requires only one new class — no existing classes change.

**Why Decorator fits:** The number of possible add-on combinations is combinatorial, making subclassing impractical. Decorator composes behaviors at runtime by wrapping objects in layers, each adding a single responsibility. It follows the Open/Closed Principle — new add-ons extend behavior without modifying existing code.

</details>

---

## Problem 4 — Simplifying a Home Automation Subsystem (Facade)

**Difficulty:** Medium
**Estimated Time:** 45 minutes

### Problem Statement

Design a facade that provides a simple, unified interface for controlling a complex home automation subsystem. The subsystem includes separate components for lighting, climate control, a security system, and an entertainment system. Users should be able to execute common multi-step routines — like "leaving home" or "movie night" — with a single method call instead of coordinating each component individually.

### Scenario

**Context:** A smart home application controls dozens of devices through individual component APIs. Turning on "movie night" mode requires dimming the lights via the `LightingController`, setting the thermostat via the `ClimateController`, disarming the security cameras in the living room via the `SecuritySystem`, and powering on the projector and speakers via the `EntertainmentSystem`. Currently, the mobile app's UI code calls each subsystem directly, resulting in tightly coupled, duplicated sequences scattered across multiple screens.

**Requirements:** Create a `HomeAutomationFacade` that exposes high-level routines: `leaveHome()`, `arriveHome()`, `movieNight()`, and `goodNight()`. Each routine orchestrates the appropriate calls to the underlying subsystem components. The facade must not prevent direct access to individual components when fine-grained control is needed. Adding a new routine should only require changes to the facade, not to the subsystem components or the client code that uses existing routines.

**Expected Approach:** Recognize that the client is overwhelmed by the complexity of coordinating multiple subsystems. Apply the Facade pattern to provide a simplified interface that encapsulates common interaction sequences while leaving the subsystem accessible for advanced use cases.

<details>
<summary>Hints</summary>

1. The facade class holds references to each subsystem component: `LightingController`, `ClimateController`, `SecuritySystem`, `EntertainmentSystem`.
2. Each high-level method (e.g., `movieNight()`) calls a specific sequence of methods on the subsystem components — the facade knows the correct order and parameters.
3. The facade does not replace the subsystem APIs. Clients can still access individual components directly for custom scenarios not covered by the facade's routines.
4. Think about what happens when a step in the sequence fails — should the facade roll back previous steps or report a partial failure?

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Use the Facade pattern to wrap the complex home automation subsystem behind a simple, high-level interface.

1. **Identify subsystem components** (these already exist and are not modified):
   - `LightingController`: `setRoom(room, brightness)`, `allOff()`, `allOn(brightness)`.
   - `ClimateController`: `setTemperature(temp)`, `setMode(mode)`, `off()`.
   - `SecuritySystem`: `arm()`, `disarm()`, `disarmZone(zone)`, `getStatus()`.
   - `EntertainmentSystem`: `powerOn()`, `powerOff()`, `setSource(source)`, `setVolume(level)`.

2. **Create `HomeAutomationFacade`:**
   - Constructor accepts instances of all four subsystem components.
   - `leaveHome()`:
     - `lighting.allOff()`
     - `climate.setTemperature(18)` (energy-saving mode)
     - `security.arm()`
     - `entertainment.powerOff()`
   - `arriveHome()`:
     - `security.disarm()`
     - `lighting.allOn(70)` (comfortable brightness)
     - `climate.setTemperature(22)`
   - `movieNight()`:
     - `lighting.setRoom("living_room", 15)` (dim)
     - `climate.setTemperature(21)`
     - `security.disarmZone("living_room")` (disable motion alerts)
     - `entertainment.powerOn()`
     - `entertainment.setSource("HDMI1")`
     - `entertainment.setVolume(40)`
   - `goodNight()`:
     - `entertainment.powerOff()`
     - `lighting.allOff()`
     - `climate.setTemperature(19)`
     - `security.arm()`

3. **Error handling:** If a subsystem call fails, the facade logs the failure and continues with remaining steps, returning a status report indicating which steps succeeded and which failed. This avoids leaving the home in an inconsistent state (e.g., lights off but security not armed).

4. **Client code:**
   ```
   facade = HomeAutomationFacade(lighting, climate, security, entertainment)
   facade.movieNight()       // one call replaces 6 subsystem calls
   // Direct access still available:
   lighting.setRoom("bedroom", 50)
   ```

5. Adding a new routine (e.g., `partyMode()`) means adding one method to the facade — no changes to subsystem components or existing client code.

**Why Facade fits:** The subsystem is complex with many interdependent components. The facade provides a simple entry point for common use cases, reducing coupling between client code and subsystem internals. It does not hide the subsystem — it offers a convenient shortcut while preserving full access for advanced scenarios.

</details>

---

## Problem 5 — Implementing an Access-Controlled Document Service (Proxy)

**Difficulty:** Hard
**Estimated Time:** 60 minutes

### Problem Statement

Design a proxy for a document storage service that adds access control, usage logging, and lazy loading without modifying the original service. The proxy must enforce role-based permissions before allowing operations, log every access attempt for auditing, and defer loading large document content from disk until it is actually requested.

### Scenario

**Context:** A document management system has a `DocumentService` that stores and retrieves documents from a file system. The service was built without security or auditing features. The company now requires that only authorized users can read or modify documents based on their role (viewer, editor, admin), that every access attempt is recorded in an audit log, and that large documents are not loaded into memory until their content is explicitly requested (currently, listing documents loads all content eagerly, causing memory spikes).

**Requirements:**
- Define a `DocumentService` interface with methods: `getDocument(id) -> Document`, `listDocuments() -> List<DocumentSummary>`, `saveDocument(id, content) -> void`, and `deleteDocument(id) -> void`.
- The real implementation (`FileSystemDocumentService`) already exists and must not be modified.
- The proxy must check the current user's role before each operation: viewers can only call `getDocument` and `listDocuments`; editors can also call `saveDocument`; admins can call all methods including `deleteDocument`. Unauthorized calls throw an `AccessDeniedException`.
- Every method call — whether authorized or denied — must be logged with the user, operation, document ID, timestamp, and result (success/denied).
- `getDocument` must use lazy loading: the proxy returns a lightweight `Document` shell on first call and loads the full content from the real service only when `document.getContent()` is invoked.
- The proxy must be a drop-in replacement for the real service — client code should not know it is talking to a proxy.

**Expected Approach:** Recognize three distinct cross-cutting concerns (access control, logging, lazy loading) that must be added to an existing service without modifying it. Apply the Proxy pattern — specifically a protection proxy for access control, a logging proxy for auditing, and a virtual proxy for lazy loading. Consider whether to implement these as a single proxy class or as layered proxies.

<details>
<summary>Hints</summary>

1. The proxy implements the same `DocumentService` interface as the real service, so clients cannot tell the difference.
2. For access control, the proxy checks the current user's role against a permission map before delegating to the real service. If the role lacks permission, throw `AccessDeniedException` without calling the real service.
3. For logging, wrap every method entry and exit (or denial) with a log statement that records the user, operation, document ID, timestamp, and outcome.
4. For lazy loading in `getDocument`, return a `LazyDocument` object that stores the document ID but does not load content until `getContent()` is called. `LazyDocument` holds a reference to the real service and calls it on first access, caching the result.
5. Consider composing the three concerns in a single proxy class with clearly separated private methods, or stacking three proxy layers (each adding one concern). A single class is simpler for this scope; layered proxies are more flexible if concerns evolve independently.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Use the Proxy pattern to wrap the existing document service with access control, audit logging, and lazy loading — all transparent to client code.

1. **Define the shared interface:**
   - `DocumentService` with methods: `getDocument(id)`, `listDocuments()`, `saveDocument(id, content)`, `deleteDocument(id)`.

2. **The real service** (`FileSystemDocumentService`) implements `DocumentService` and reads/writes files on disk. It is not modified.

3. **Define permission rules:**
   - `viewer`: `getDocument`, `listDocuments`.
   - `editor`: all viewer permissions plus `saveDocument`.
   - `admin`: all permissions including `deleteDocument`.
   - Store as a map: `Map<Role, Set<Operation>>`.

4. **Create `DocumentServiceProxy` implementing `DocumentService`:**
   - Constructor takes: `realService: DocumentService`, `userContext: UserContext` (provides current user and role), `auditLogger: AuditLogger`.
   - Each method follows the same pattern:
     a. **Log** the attempt: user, operation, document ID, timestamp.
     b. **Check permission**: look up the user's role in the permission map. If the operation is not allowed, log the denial and throw `AccessDeniedException`.
     c. **Delegate** to `realService` for the actual operation.
     d. **Log** the success.

5. **Lazy loading for `getDocument`:**
   - Instead of returning the full `Document` from the real service, return a `LazyDocument`:
     - `LazyDocument` implements the `Document` interface.
     - Constructor takes `id`, `metadata` (title, author — lightweight), and a reference to `realService`.
     - `getContent()`: on first call, invokes `realService.getDocument(id).getContent()`, caches the result in a private field, and returns it. Subsequent calls return the cached content.
   - `listDocuments()` returns `DocumentSummary` objects (already lightweight), so no lazy loading is needed there.

6. **Client code is unchanged:**
   ```
   // Before (direct):
   service = FileSystemDocumentService(storagePath)
   // After (proxied):
   service = DocumentServiceProxy(
       FileSystemDocumentService(storagePath),
       userContext,
       auditLogger
   )
   // Client code uses `service` identically in both cases
   doc = service.getDocument("doc-123")
   // Content is NOT loaded yet — only metadata
   content = doc.getContent()
   // NOW the content is fetched from disk
   ```

7. **Why Proxy fits:** The proxy shares the same interface as the real service, making it a transparent drop-in replacement. It adds three cross-cutting concerns — access control (protection proxy), logging (logging proxy), and deferred loading (virtual proxy) — without modifying the original service. This respects the Open/Closed Principle and keeps each concern isolated within the proxy layer.

</details>
