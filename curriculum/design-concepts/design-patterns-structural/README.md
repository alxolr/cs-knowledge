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