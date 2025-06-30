# JSDOMParser Rewrite: TDD Implementation Plan

This document outlines the Test-Driven Development (TDD) strategy for modernizing `JSDOMParser.js`. The plan follows the Red-Green-Refactor cycle and is structured according to the feature-based architecture defined in the modernization plan.

## Guiding Principles

- **Test First:** No production code is written without a failing test that defines its specification.
- **Red-Green-Refactor:** Each step follows the TDD mantra to ensure quality and design clarity.
- **Incremental Development:** We build the system from the inside out, starting with core contracts and moving towards the final integration.

---

## Phase 1: Foundation & Contracts (Setup)

**Goal:** Establish the project structure and define the core interfaces that will govern the entire feature. These interfaces are the "contract" that all subsequent components will adhere to.

1.  **Task: Create Directory Structure**
    - **Action:** Create the `src/features/parsing/` directory and its subdirectories (`interfaces`, `dom`, `core`, `usecases`, `implementation`).

2.  **Task: Define Core Interfaces (The "What")**
    - **Action:** Create the interface files. This is a "Green" step as it involves no logic, only type definitions.
    - **File:** `src/features/parsing/interfaces/dom.interface.ts`
      - **Content:** Define `INode`, `IElement`, `IDocument`.
    - **File:** `src/features/parsing/interfaces/parser.interface.ts`
      - **Content:** Define `IParser` and the `Result<T, E>` type for error handling.

3.  **Task: Setup Test Environment**
    - **Action:** Configure the testing framework (e.g., Jest, Deno.test) to recognize files in the new feature directory.

---

## Phase 2: DOM Implementation (The Building Blocks)

**Goal:** Build the concrete DOM classes. Each class will be developed using a strict TDD cycle.

1.  **Module: `Node`**
    - **Red:** In `dom/node.test.ts`, write a test asserting that a `new Node()` has the correct `nodeType`. The test fails as `Node` doesn't exist.
    - **Green:** In `dom/node.ts`, create the `Node` class, implementing the `INode` interface with the minimal code to make the test pass.
    - **Refactor:** Review and clean up the `Node` class and its test.

2.  **Module: `Element`**
    - **Red:** In `dom/element.test.ts`, write a test asserting that `new Element("p")` has the correct `tagName` and inherits from `Node`. The test fails.
    - **Green:** In `dom/element.ts`, create the `Element` class that `extends Node` and implements `IElement`.
    - **Refactor:** Add tests for attributes and children, following the Red-Green-Refactor cycle for each piece of functionality.

3.  **Module: `Document`**
    - **Red:** In `dom/document.test.ts`, write a test for creating a `Document` node. The test fails.
    - **Green:** In `dom/document.ts`, create the `Document` class.
    - **Refactor:** Add tests for document-specific properties, iterating with the TDD cycle.

---

## Phase 3: Core Parsing Logic (The Engine)

**Goal:** Develop the tokenizer and the tree builder. These are pure logic modules, making them ideal for TDD.

1.  **Module: `Tokenizer`**
    - **Red:** In `core/tokenizer.test.ts`, write a test that provides a simple HTML string (e.g., `<p>Hi</p>`) and expects an array of specific token objects. The test fails.
    - **Green:** In `core/tokenizer.ts`, implement the simplest possible logic to parse the given string and produce the expected tokens.
    - **Refactor:** Clean up the implementation.
    - **Iterate:** Add more tests for increasing complexity: attributes, self-closing tags, comments, text nodes, etc. For each new case, write a failing test first.

2.  **Module: `Builder`**
    - **Red:** In `core/builder.test.ts`, write a test that provides an array of tokens (from the tokenizer step) and expects a specific `IDocument` object tree. The test fails.
    - **Green:** In `core/builder.ts`, implement the logic to consume the tokens and construct the corresponding DOM tree using the classes from Phase 2.
    - **Refactor:** Improve the tree construction logic.
    - **Iterate:** Add tests for more complex token sequences to handle nested elements, siblings, and edge cases.

---

## Phase 4: Concrete Parser & Usecase (The Orchestration)

**Goal:** Assemble the core logic into a functioning parser and expose it through a clean usecase.

1.  **Module: `JSDOMParser` Implementation**
    - **Red:** In `implementation/jsdom-parser.test.ts`, write an integration test. Call `new JSDOMParser().parse("<html>...</html>")` and assert that the returned `Result` contains the correctly structured document. The test fails.
    - **Green:** In `implementation/jsdom-parser.ts`, implement the `IParser` interface. The `parse` method will instantiate and call the `Tokenizer` and `Builder` to do the actual work.
    - **Refactor:** Clean up the orchestration logic within the `parse` method.

2.  **Module: `parse-html.usecase`**
    - **Red:** In `usecases/parse-html.usecase.test.ts`, write a test that calls the usecase. This test should inject a mock `IParser` to isolate the usecase logic.
    - **Green:** In `usecases/parse-html.usecase.ts`, create the usecase function that takes the parser as a dependency and calls its `parse` method.
    - **Refactor:** Ensure dependency injection is clean and the logic is minimal.

---

## Phase 5: Public API & Integration (The Final Product)

**Goal:** Expose the new feature through a public factory and integrate it into the main application, using the application's existing test suite as the final validation.

1.  **Module: Public Factory (`index.ts`)**
    - **Red:** Write a test that calls the factory function from `features/parsing/index.ts` and asserts that it returns a valid `IParser` instance.
    - **Green:** Implement the factory in `index.ts` to create and return a new `JSDOMParser` with its dependencies.
    - **Refactor:** Clean up the factory implementation.

2.  **Integration with `Readability.js`**
    - **Red:** Modify `Readability.js` to import and use the new parser factory. Run the entire existing `Readability.js` test suite. Expect many tests to fail. This validates the existing test suite as a safety net.
    - **Green:** Debug and fix the new parser's logic until all `Readability.js` tests pass. This ensures 100% functional parity.
    - **Refactor:** With all tests passing, refactor the new parser code for clarity and performance. Measure code coverage and add tests to meet the >90% target. Profile performance against the original and optimize if necessary.
