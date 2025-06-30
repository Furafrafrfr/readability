# Technical Modernization Plan for Readability.js

## 1. Overall Goal

The primary objective of this initiative is to refactor the monolithic `Readability.js` file into a collection of smaller, single-responsibility modules. This modernization will focus on making the codebase type-safe, highly testable, and significantly more maintainable for future development. By breaking down the large, complex file, we aim to improve code clarity, reduce cognitive load for developers, and enable more robust and predictable enhancements.

## 2. TypeScript for Type Safety

We will introduce TypeScript to enforce strict type safety throughout the codebase. This will start with two critical areas:

*   **`Readability` Constructor Options**: The options object passed to the `Readability` constructor will be defined with a strict TypeScript interface. This will eliminate runtime errors caused by invalid or misspelled options and provide clear, self-documenting code for consumers of the library.
*   **Article Object**: The final article object returned by the `parse()` method will also have a well-defined interface. This ensures that consumers of the parsed data can rely on a consistent and predictable structure, improving the overall robustness of applications that depend on `Readability.js`.

## 3. Decoupling with Dependency Injection

Currently, `Readability.js` directly instantiates `JSDOMParser`, creating a tight coupling between the core parsing logic and a specific DOM parser implementation.

To resolve this, we will apply the Dependency Injection (DI) pattern. A generic `Parser` interface will be defined, outlining the contract required for any parser to work with the core logic. The `Readability` class will then accept an instance conforming to this `Parser` interface in its constructor. This decouples the core algorithm from any specific parsing technology, allowing developers to "inject" different parsers (e.g., a browser's native `DOMParser`) without modifying the core `Readability` code.

## 4. Refactoring to Feature-Based Modules

The core of this refactoring effort lies in decomposing the large and complex `parse()` method. This monolithic method will be broken down into a pipeline of smaller, pure functions or classes, each responsible for a distinct stage of the article extraction process.

We will adopt a feature-based directory structure under `src/features/readability/`. This will house the new, granular modules:

*   `src/features/readability/metadata-extraction.ts`: Functions responsible for extracting metadata from the document (e.g., title, author, site name, excerpt).
*   `src/features/readability/dom-cleanup.ts`: A series of pure functions dedicated to cleaning the DOM, such as removing scripts, styles, and unlikely candidate elements before scoring.
*   `src/features/readability/content-scoring.ts`: Logic for traversing the DOM, scoring nodes based on readability heuristics, and identifying the primary content container.
*   `src/features/readability/article-assembly.ts`: Functions responsible for taking the selected content node and assembling the final, clean article HTML, including handling images, tables, and other elements.
*   `src/features/readability/index.ts`: The public entry point for the readability feature, which orchestrates the pipeline by calling the other modules in sequence.

This modular approach makes each step of the process independently testable and easier to understand and modify.

## 5. ES Modules for Code Organization

The single, large `Readability.js` file will be physically split into the new module structure outlined above. We will use modern ES Modules (`import`/`export` syntax) to manage dependencies between these files. This will provide a clear and standard way to organize the code, improve tree-shaking for bundlers, and formally define the public API of each module, preventing unintended access to internal implementation details.