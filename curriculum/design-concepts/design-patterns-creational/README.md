# Design Patterns (Creational)

**Track:** Design Concepts
**Difficulty Tier:** Beginner
**Prerequisites:** [Object-Oriented Design Principles](../object-oriented-design-principles/README.md), [SOLID Principles](../solid-principles/README.md)

## Concept Overview

Creational design patterns deal with object creation mechanisms, aiming to create objects in a manner that is flexible, decoupled, and appropriate to the situation. Rather than instantiating classes directly with `new`, creational patterns introduce indirection that gives a program more control over which objects are created, how they are composed, and when they come into existence.

The five classic creational patterns are Singleton, Factory Method, Abstract Factory, Builder, and Prototype. Singleton ensures a class has only one instance and provides a global access point to it. Factory Method delegates instantiation to subclasses so that the client code works with an interface rather than a concrete class. Abstract Factory extends this idea to families of related objects. Builder separates the construction of a complex object from its representation, letting the same construction process produce different results. Prototype creates new objects by cloning an existing instance rather than building from scratch.

Understanding when to apply each pattern — and when not to — is a core skill in software design. Overusing Singleton can introduce hidden global state; choosing Builder when a simple constructor suffices adds unnecessary complexity. The problems in this module present realistic scenarios where each pattern is the natural fit, giving you practice identifying the right tool for the job.

---

## Problem 1 — Managing a Configuration Registry (Singleton)

**Difficulty:** Easy
**Estimated Time:** 30 minutes

### Problem Statement

Design a configuration registry that loads application settings from a file at startup and provides read access to those settings throughout the application. The registry must guarantee that only one instance exists at any time, and all modules share the same configuration state.

### Scenario

**Context:** A web application reads database credentials, feature flags, and logging levels from a YAML configuration file when it starts. Several modules — the database connection pool, the feature-flag evaluator, and the logging framework — all need access to these settings.

**Requirements:** Ensure exactly one instance of the configuration registry exists across the entire application. Any module that requests the registry must receive the same instance. The registry should load the configuration file once and cache the parsed settings. It must be safe to access from multiple modules without re-reading the file.

**Expected Approach:** Identify why uncontrolled instantiation is problematic here (duplicate file reads, inconsistent state). Apply the Singleton pattern to guarantee a single shared instance. Consider how the instance is created lazily or eagerly and how the constructor is hidden from external callers.

<details>
<summary>Hints</summary>

1. Make the constructor private so no external code can call `new ConfigRegistry()`.
2. Provide a static method like `getInstance()` that creates the instance on first call and returns the cached instance on subsequent calls.
3. Store the single instance in a private static field within the class.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Apply the Singleton pattern to ensure a single, shared configuration registry.

1. Define a class `ConfigRegistry` with a private constructor that accepts a file path, reads the YAML file, and stores the parsed key-value pairs in a private map.
2. Declare a private static field `instance` initialized to `null`.
3. Implement a public static method `getInstance()`:
   - If `instance` is `null`, create a new `ConfigRegistry` with the default config path and assign it to `instance`.
   - Return `instance`.
4. Provide public read-only methods like `getString(key)`, `getInt(key)`, and `getBoolean(key)` that look up values in the internal map.
5. No public setter for the instance — the only way to obtain the registry is through `getInstance()`.

**Why Singleton fits:** Configuration is inherently global and read-heavy. A single instance avoids redundant file I/O and guarantees every module sees the same settings. The private constructor prevents accidental duplicate registries.

</details>

---

## Problem 2 — Creating Document Exporters (Factory Method)

**Difficulty:** Easy
**Estimated Time:** 30 minutes

### Problem Statement

Design a document export system where the application can produce documents in multiple formats (PDF, HTML, Markdown). The core export workflow — gathering content, applying a template, and writing the output — is the same for every format, but the rendering step differs. New formats should be addable without modifying the existing export logic.

### Scenario

**Context:** A content management system lets authors write articles and export them for distribution. Today it supports PDF only. The product team wants to add HTML and Markdown export, with more formats likely in the future.

**Requirements:** The export workflow (fetch content → apply template → render → write file) must be defined once. Each format provides its own rendering logic. Adding a new format must not require changes to the base export class or existing format classes. Client code should request an exporter by format name without knowing the concrete class.

**Expected Approach:** Recognize that the workflow is fixed but one step (rendering) varies by format. Use the Factory Method pattern to let subclasses decide which renderer to create while the base class controls the overall process.

<details>
<summary>Hints</summary>

1. Define an abstract class `DocumentExporter` with a template method `export(content)` that calls a sequence of steps, one of which — `createRenderer()` — is abstract.
2. Each concrete exporter (e.g., `PdfExporter`, `HtmlExporter`) overrides `createRenderer()` to return the appropriate renderer.
3. The factory method is `createRenderer()` — it is the "hook" that subclasses override to supply format-specific behavior.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Use the Factory Method pattern so each exporter subclass provides its own renderer.

1. Define an interface `Renderer` with a method `render(templateData) -> bytes`.
2. Implement `PdfRenderer`, `HtmlRenderer`, and `MarkdownRenderer`, each producing output in its format.
3. Define an abstract class `DocumentExporter` with:
   - A concrete method `export(content)` that: fetches content, applies a template, calls `createRenderer().render(templateData)`, and writes the result to a file.
   - An abstract method `createRenderer() -> Renderer` (the factory method).
4. Create concrete subclasses:
   - `PdfExporter` overrides `createRenderer()` to return a `PdfRenderer`.
   - `HtmlExporter` overrides `createRenderer()` to return an `HtmlRenderer`.
   - `MarkdownExporter` overrides `createRenderer()` to return a `MarkdownRenderer`.
5. Client code works with `DocumentExporter` and never references a concrete renderer directly.

**Why Factory Method fits:** The overall algorithm is stable; only the object-creation step varies. The pattern localizes the variation in a single overridable method, keeping the base class closed for modification.

</details>

---

## Problem 3 — Building Cross-Platform UI Components (Abstract Factory)

**Difficulty:** Easy
**Estimated Time:** 35 minutes

### Problem Statement

Design a UI toolkit that creates families of visual components (buttons, text fields, checkboxes) styled for different operating systems (Windows, macOS). The application code should build forms using a uniform API without knowing which OS-specific components it is creating.

### Scenario

**Context:** A desktop application must run on both Windows and macOS. Each platform has its own look-and-feel guidelines — buttons, text fields, and checkboxes render differently. The application logic for building forms (login screen, settings panel) should be identical on both platforms; only the visual components change.

**Requirements:** Define a family of related component interfaces (Button, TextField, Checkbox). Provide platform-specific implementations for each. The form-building code must create components through a single factory interface, never referencing platform-specific classes. Switching the entire look-and-feel should require changing only which factory is supplied, not the form code.

**Expected Approach:** Recognize that you need to create families of related objects that must be used together (a Windows button with a Windows text field, not mixed). The Abstract Factory pattern provides a factory interface whose concrete implementations produce a consistent family of components.

<details>
<summary>Hints</summary>

1. Define component interfaces: `Button`, `TextField`, `Checkbox` — each with methods like `render()`.
2. Define a `UIFactory` interface with methods `createButton()`, `createTextField()`, `createCheckbox()`.
3. Implement `WindowsUIFactory` and `MacUIFactory`, each returning the platform-appropriate component implementations.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Use the Abstract Factory pattern to produce families of platform-consistent UI components.

1. Define component interfaces:
   - `Button` with `render()`, `onClick(handler)`
   - `TextField` with `render()`, `getValue()`
   - `Checkbox` with `render()`, `isChecked()`
2. Implement platform-specific classes:
   - `WindowsButton`, `WindowsTextField`, `WindowsCheckbox`
   - `MacButton`, `MacTextField`, `MacCheckbox`
3. Define an abstract factory interface `UIFactory`:
   - `createButton() -> Button`
   - `createTextField() -> TextField`
   - `createCheckbox() -> Checkbox`
4. Implement `WindowsUIFactory` (returns Windows components) and `MacUIFactory` (returns Mac components).
5. The form-building code receives a `UIFactory` and calls `factory.createButton()`, etc. It never imports a platform-specific class.
6. At application startup, detect the OS and inject the appropriate factory.

**Why Abstract Factory fits:** The problem requires creating multiple related objects that must belong to the same family. A single Factory Method would handle one product; Abstract Factory handles an entire product family, ensuring consistency.

</details>

---

## Problem 4 — Constructing Complex Meal Orders (Builder)

**Difficulty:** Medium
**Estimated Time:** 45 minutes

### Problem Statement

Design an order construction system for a restaurant that assembles complex meal orders with many optional components — main course, side dishes, drinks, dessert, dietary modifications, and special instructions. The system must support building different meal configurations (kids' meal, combo deal, custom order) through a step-by-step process, and the final order object should be immutable once constructed.

### Scenario

**Context:** A restaurant chain's ordering system handles diverse meal configurations. A kids' meal always includes a small main, a side, a drink, and a toy. A combo deal includes a main, two sides, and a large drink. A custom order can have any combination. Currently, the `MealOrder` constructor takes 10+ parameters, most of which are optional, leading to confusing calls with many `null` values and frequent bugs where parameters are passed in the wrong order.

**Requirements:** Replace the telescoping constructor with a step-by-step building process. Each step should be optional and chainable. The builder must validate that the final order meets minimum requirements (at least a main course). Pre-configured builder sequences for kids' meals and combo deals should be easy to define without duplicating logic. The resulting `MealOrder` object must be immutable — no modification after construction.

**Expected Approach:** Apply the Builder pattern to separate the construction of a `MealOrder` from its representation. Consider using a director class or pre-built builder configurations for standard meal types. Think about how validation is enforced at build time.

<details>
<summary>Hints</summary>

1. Create a `MealOrderBuilder` class with methods like `setMain(item)`, `addSide(item)`, `setDrink(item)`, `setDessert(item)`, `addModification(mod)`, and `setInstructions(text)` — each returning `this` for chaining.
2. The `build()` method validates the order (e.g., main course is required) and returns an immutable `MealOrder` object.
3. A `MealDirector` class can define methods like `buildKidsMeal(builder)` and `buildCombo(builder)` that call the builder steps in a predefined sequence.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Use the Builder pattern with an optional Director to construct complex, immutable meal orders step by step.

1. Define an immutable `MealOrder` class with private fields for main course, list of sides, drink, dessert, list of dietary modifications, and special instructions. The constructor is package-private or accessible only to the builder.
2. Create `MealOrderBuilder`:
   - Private fields mirroring `MealOrder`.
   - `setMain(item) -> this` — sets the main course.
   - `addSide(item) -> this` — appends to the sides list.
   - `setDrink(item) -> this` — sets the drink.
   - `setDessert(item) -> this` — sets the dessert.
   - `addModification(mod) -> this` — appends a dietary modification.
   - `setInstructions(text) -> this` — sets special instructions.
   - `build() -> MealOrder` — validates that a main course is set, then constructs and returns an immutable `MealOrder`.
3. Create a `MealDirector` with convenience methods:
   - `buildKidsMeal(builder)` — calls `builder.setMain(smallBurger).addSide(fries).setDrink(juice)` and adds a toy flag.
   - `buildCombo(builder)` — calls `builder.setMain(burger).addSide(fries).addSide(coleslaw).setDrink(largeSoda)`.
4. Client code for a custom order:
   ```
   order = MealOrderBuilder()
       .setMain("Grilled Chicken")
       .addSide("Salad")
       .setDrink("Water")
       .addModification("No gluten")
       .build()
   ```
5. The `MealOrder` object has no setters — once built, it cannot be changed.

**Why Builder fits:** The object has many optional parts and multiple valid configurations. Builder avoids telescoping constructors, makes construction readable, and enforces validation at the final `build()` step. The Director reuses builder sequences for standard configurations without duplicating code.

</details>

---

## Problem 5 — Cloning Game World Templates (Prototype)

**Difficulty:** Hard
**Estimated Time:** 60 minutes

### Problem Statement

Design a system for a game engine that creates new game entities (enemies, power-ups, terrain tiles) by cloning pre-configured template objects rather than constructing each one from scratch. Templates are loaded from a data file at startup and stored in a registry. At runtime, the game spawns new entities by cloning the appropriate template and optionally customizing the clone. The design must handle entities that contain nested objects (e.g., an enemy with an inventory of items) and ensure that cloning produces fully independent copies.

### Scenario

**Context:** A 2D game engine loads dozens of entity templates from a configuration file at startup — each template defines an entity's sprite, hit points, speed, attack patterns, and an inventory of items. During gameplay, the engine spawns hundreds of entities per level. Constructing each entity from scratch by re-parsing the config file is too slow, and sharing mutable template objects between spawned entities causes bugs when one entity's state changes affect others.

**Requirements:**
- Define a `Prototype` interface with a `clone()` method that all game entities implement.
- Implement deep cloning so that nested objects (e.g., inventory items inside an enemy) are fully independent copies, not shared references.
- Provide a `PrototypeRegistry` that stores named templates and returns clones on request.
- Support post-clone customization — after cloning, the caller can adjust specific fields (e.g., position, difficulty scaling) without affecting the template.
- The design must handle at least three entity types (Enemy, PowerUp, TerrainTile) with different internal structures.

**Expected Approach:** Apply the Prototype pattern to avoid expensive construction. Distinguish between shallow and deep cloning and explain why deep cloning is necessary here. Design the registry as a lookup table of named prototypes. Consider how the clone method is implemented for objects with nested collections.

<details>
<summary>Hints</summary>

1. Define a `Prototype` interface with a single method `clone() -> Prototype`.
2. Each entity class implements `clone()` by creating a new instance and copying all fields. For nested objects and collections, recursively clone each element (deep copy) rather than copying references (shallow copy).
3. The `PrototypeRegistry` is a map from string names to `Prototype` instances. Its `get(name)` method returns `template.clone()`, not the template itself.
4. For post-clone customization, the caller receives the cloned object and calls setters or a configuration method before using it.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Use the Prototype pattern with deep cloning and a registry to efficiently spawn independent game entities from templates.

1. **Define the Prototype interface:**
   - `Prototype` with method `clone() -> Prototype`.

2. **Implement entity classes:**
   - `Enemy` with fields: name, sprite, hitPoints, speed, attackPattern, inventory (list of `Item` objects).
     - `clone()`: create a new `Enemy`, copy primitive fields, deep-clone the inventory by cloning each `Item`.
   - `PowerUp` with fields: name, sprite, effectType, duration, magnitude.
     - `clone()`: create a new `PowerUp`, copy all fields (all primitives or immutable, so shallow copy suffices).
   - `TerrainTile` with fields: name, sprite, isPassable, damageEffect (nullable nested object with type and amount).
     - `clone()`: create a new `TerrainTile`, copy primitives, clone `damageEffect` if present.
   - `Item` with fields: name, weight, effect.
     - `clone()`: create a new `Item`, copy all fields.

3. **Deep vs. shallow cloning decision:**
   - `Enemy` requires deep cloning because its inventory is a mutable collection of mutable `Item` objects. A shallow copy would share the same list and items, causing cross-entity mutation bugs.
   - `PowerUp` can use shallow cloning because all fields are primitives or immutable.
   - `TerrainTile` needs to deep-clone the optional `damageEffect` object to avoid shared mutable state.

4. **Build the PrototypeRegistry:**
   - Internal storage: `Map<String, Prototype>`.
   - `register(name, prototype)` — stores a template.
   - `get(name) -> Prototype` — looks up the template by name and returns `template.clone()`.
   - At startup, parse the config file, construct template objects, and register each one.

5. **Runtime spawning with customization:**
   ```
   enemy = registry.get("goblin")          // returns a deep clone
   enemy.setPosition(x, y)                 // customize without affecting template
   enemy.scaleHitPoints(difficultyFactor)
   world.addEntity(enemy)
   ```

6. **Why Prototype fits:** Constructing entities from config files is expensive (file I/O, parsing). Cloning a pre-built template is fast and avoids re-parsing. Deep cloning ensures each spawned entity is fully independent, preventing shared-state bugs. The registry provides a clean lookup mechanism for templates by name.

</details>
