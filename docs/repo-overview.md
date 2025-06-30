# Repository Overview: @mozilla/readability

## 1. Project Purpose

The `@mozilla/readability` package is a standalone version of the library used in Firefox's Reader View. Its primary purpose is to extract the main, readable content from a web page, stripping away clutter like navigation, ads, and sidebars. This allows developers to create clutter-free reading experiences for users, similar to services like Pocket or Instapaper.

The library can be used in both browser and Node.js environments. In Node.js, it requires a DOM implementation like `jsdom`.

## 2. Core Components

The repository consists of two main JavaScript files that form the core of its functionality:

### `Readability.js`

This is the heart of the library. The `Readability` class takes a DOM document as input and provides a `parse()` method. This method analyzes the document's structure, scores different elements based on heuristics (e.g., text length, class names, link density), and identifies the primary article content.

The `parse()` method returns an object containing:
*   `title`: The article title.
*   `content`: The processed article content as an HTML string.
*   `textContent`: The plain text content of the article.
*   `byline`: The author's name.
*   `excerpt`: A short summary or description.
*   And other metadata like `siteName`, `lang`, and `publishedTime`.

The library also exposes a helper function, `isProbablyReaderable()`, which provides a quick, preliminary check to see if a document is suitable for processing.

### `JSDOMParser.js`

This file provides a lightweight, safe DOM parser that can be used in environments without a native DOM, such as a web worker or Node.js. It is a minimal implementation, designed specifically to support the needs of `Readability.js`. It is not a complete DOM implementation and expects well-formed HTML/XML.

## 3. Project Structure

The project's structure is defined by its `package.json` file.

### Dependencies

The project has several development dependencies for testing, linting, and releasing:
*   **Testing**: `mocha`, `chai`, `sinon`, `jsdom`.
*   **Linting & Formatting**: `eslint`, `prettier`.
*   **Release Management**: `release-it`.

It has no production dependencies, as it is a standalone library.

### Scripts

The `package.json` file defines the following scripts:
*   `lint`: Runs ESLint and Prettier to check code quality and formatting.
*   `test`: Executes the test suite using Mocha.
*   `generate-testcase`: A utility script to help create new test cases.
*   `release`: Manages the release process using `release-it`.

## 4. Testing Setup

The project has a comprehensive test suite located in the `test/` directory.

*   **Test Runner**: The tests are run using `mocha`.
*   **Assertions**: Assertions are made using the `chai` library.
*   **Test Structure**:
    *   `test/test-readability.js`: Contains the main tests for the `Readability` class.
    *   `test/test-isProbablyReaderable.js`: Tests for the `isProbablyReaderable` function.
    *   `test/test-jsdomparser.js`: Tests for the custom `JSDOMParser`.
*   **Test Cases**: The `test/test-pages/` directory contains a large number of subdirectories, each representing a specific test case with:
    *   `source.html`: The original HTML of the test page.
    *   `expected.html`: The expected HTML output after processing by Readability.
    *   `expected-metadata.json`: The expected metadata (title, byline, etc.).
*   **DOM Environments**: The tests are run against two different DOM environments to ensure compatibility:
    1.  **`jsdom`**: The standard, more complete DOM implementation for Node.js.
    2.  **`JSDOMParser`**: The project's custom, lightweight parser.

This dual-testing approach ensures that the library works correctly in different environments and that the custom parser is a valid substitute for `jsdom` for the library's purposes.

## HTML to Reader View Conversion Process

### **High-Level Overview**

The `@mozilla/readability` library is a standalone module that extracts the primary, readable content from a webpage, stripping away clutter like ads, navigation bars, and sidebars. The process can be broken down into three main phases:

1.  **Parsing**: An HTML string is converted into a DOM (Document Object Model) tree that the library can manipulate.
2.  **Analysis & Scoring**: The DOM tree is traversed, and each element is scored based on heuristics to identify the node most likely to contain the main article content.
3.  **Cleaning & Serialization**: The chosen article node and its relevant siblings are isolated, cleaned of unnecessary elements and styling, and then serialized back into a clean HTML string.

### **Key Files and Their Roles**

1.  **[`JSDOMParser.js`](JSDOMParser.js)**: This file provides a lightweight, custom DOM parser. Its primary purpose is to create a DOM-like object structure from an HTML string in environments where a standard browser DOM is not available (e.g., a Web Worker or a basic Node.js environment). It implements the minimum set of DOM APIs required by [`Readability.js`](Readability.js), such as [`getElementsByTagName`](JSDOMParser.js), [`getAttribute`](JSDOMParser.js), and node manipulation methods ([`appendChild`](JSDOMParser.js), [`remove`](JSDOMParser.js), etc.). It does not support live [`NodeList`](JSDOMParser.js)s, meaning collections of nodes are static arrays.

2.  **[`Readability.js`](Readability.js)**: This is the core of the library, containing all the logic for content extraction and cleaning. The main class, [`Readability`](Readability.js), takes a parsed document object in its constructor and exposes a single public method, [`parse()`](Readability.js), which executes the entire process.

3.  **[`index.js`](index.js)**: This is the main entry point for the npm package. It exports the [`Readability`](Readability.js) class from [`Readability.js`](Readability.js) and the [`isProbablyReaderable`](Readability-readerable.js) helper function from [`Readability-readerable.js`](Readability-readerable.js), making them available to other Node.js modules.

4.  **[`package.json`](package.json)**: This file defines the project's metadata, dependencies, and scripts. It shows a dependency on [`jsdom`](package.json) for testing, which provides a more complete DOM implementation for the test environment.

### **Step-by-Step Conversion Flow**

The entire process is orchestrated by the [`Readability.prototype.parse()`](Readability.js) method. Here is a breakdown of its execution flow:

#### **Phase 1: Initialization and Pre-processing**

1.  **Document Parsing (External to [`Readability.js`](Readability.js))**: Before [`Readability`](Readability.js) is even instantiated, the consumer of the library must parse the raw HTML into a document object. This can be done using a browser's [`DOMParser`](Readability.js), [`jsdom`](package.json), or the library's own [`JSDOMParser.js`](JSDOMParser.js).

2.  **Instantiate [`Readability`](Readability.js)**: An instance is created, passing the `doc` object.
    ```javascript
    const reader = new Readability(doc);
    const article = reader.parse();
    ```

3.  **Unwrap [`<noscript>`](Readability.js) Images ([`_unwrapNoscriptImages()`](Readability.js))**: The parser first looks for lazy-loaded images often found within [`<noscript>`](Readability.js) tags. It replaces placeholder [`<img>`](Readability.js) tags with the higher-quality versions from inside the [`<noscript>`](Readability.js), ensuring the best images are preserved.

4.  **Metadata Extraction**:
    *   **JSON-LD ([`_getJSONLD()`](Readability.js))**: It searches for [`<script type="application/ld+json">`](Readability.js) tags to extract structured metadata like title, author, and publication date, prioritizing [`Article`](Readability.js) schema types.
    *   **HTML Meta Tags ([`_getArticleMetadata()`](Readability.js))**: It then scrapes standard [`<meta>`](Readability.js) tags (e.g., Open Graph, Twitter, Dublin Core) for the same information. This data is used to populate the final output and to help identify the article's title within the document.
    *   **Article Title ([`_getArticleTitle()`](Readability.js))**: The definitive article title is determined through a series of heuristics, starting with [`doc.title`](Readability.js) and refining it by removing site name prefixes/suffixes and comparing it against [`<h1>`](Readability.js) and [`<h2>`](Readability.js) tags.

5.  **Prepare the Document ([`_prepDocument()`](Readability.js))**: The entire document is sanitized before the main content analysis begins.
    *   All [`<script>`](Readability.js) and [`<style>`](Readability.js) tags are removed ([`_removeScripts()`](Readability.js)).
    *   Successive [`<br>`](Readability.js) tags are intelligently converted into [`<p>`](Readability.js) tags to create proper paragraphs ([`_replaceBrs()`](Readability.js)).
    *   Deprecated [`<font>`](Readability.js) tags are replaced with [`<span>`](Readability.js)s.

#### **Phase 2: Content Analysis and Scoring ([`_grabArticle`](Readability.js))**

This is the most complex phase, where the algorithm identifies the main content. It's an iterative process that may run multiple times with different settings.

1.  **Initial DOM Traversal and Cleanup**: The method walks the entire DOM tree from the [`documentElement`](Readability.js).
    *   **Remove Unlikely Candidates**: Elements with IDs or classes matching a "negative" regex (e.g., `ad-`, `comment`, `sidebar`, `footer` in [`REGEXPS.unlikelyCandidates`](Readability.js)) are removed. This is controlled by the [`FLAG_STRIP_UNLIKELYS`](Readability.js) flag.
    *   **Remove Hidden Elements**: Any element that is not visible (e.g., `display: none`, `hidden` attribute) is removed ([`_isProbablyVisible()`](Readability.js)).
    *   **Convert [`div`](Readability.js) to [`p`](Readability.js)**: [`<div>`](Readability.js) elements that contain only phrasing content (inline elements like [`<span>`](Readability.js), [`<a>`](Readability.js), [`<em>`](Readability.js)) are converted into [`<p>`](Readability.js) tags. This helps to correctly score content that isn't marked up semantically.

2.  **Scoring Candidates**:
    *   The algorithm iterates through all scorable elements (like [`<p>`](Readability.js), [`<td>`](Readability.js), [`pre`](Readability.js)) and calculates a [`contentScore`](Readability.js) for them and their ancestors.
    *   **Text-based Score**: The score is increased based on the text length and the number of commas (as an indicator of sentence structure).
    *   **Class/ID Weighting ([`_getClassWeight()`](Readability.js))**: Elements get a score boost for "positive" class names/IDs (`article`, `content`, `post`) and a penalty for "negative" ones (`comment`, `share`, `widget`). This is controlled by the [`FLAG_WEIGHT_CLASSES`](Readability.js) flag.
    *   **Score Propagation**: The score of a node is added to its parent and ancestors, but divided by a factor at each level. This allows container [`<div>`](Readability.js)s to accumulate high scores if they contain many content-rich children.

3.  **Finding the Top Candidate**:
    *   All potential candidate nodes are ranked by their final [`contentScore`](Readability.js), adjusted for link density (nodes with too many links are penalized via [`_getLinkDensity()`](Readability.js)).
    *   The node with the highest score is selected as the [`topCandidate`](Readability.js).

4.  **Assembling the Final Content**:
    *   A new [`<div>`](Readability.js) element, [`articleContent`](Readability.js), is created to hold the final result.
    *   The [`topCandidate`](Readability.js) is appended to [`articleContent`](Readability.js).
    *   The algorithm then inspects the **siblings** of the [`topCandidate`](Readability.js). If a sibling has a score above a certain threshold or appears to be a paragraph of text, it is also appended to [`articleContent`](Readability.js). This crucial step stitches together content that may have been split by ads or other non-content elements.

5.  **Iterative Refinement**: If the resulting [`articleContent`](Readability.js) has a text length below a certain threshold ([`_charThreshold`](Readability.js)), the entire [`_grabArticle`](Readability.js) process is re-run with one of the cleanup flags disabled. This allows for a less aggressive cleaning pass if the first one failed. It will try again in this order:
    1.  With [`FLAG_STRIP_UNLIKELYS`](Readability.js) disabled.
    2.  With [`FLAG_WEIGHT_CLASSES`](Readability.js) disabled.
    3.  With [`FLAG_CLEAN_CONDITIONALLY`](Readability.js) disabled.
    If all attempts fail, it returns the content from the attempt that yielded the most text.

#### **Phase 3: Post-processing and Serialization**

Once [`_grabArticle`](Readability.js) returns the final [`articleContent`](Readability.js) element, a final cleanup pass occurs.

1.  **Final Content Preparation ([`_prepArticle()`](Readability.js))**:
    *   Removes any remaining undesirable tags like [`<form>`](Readability.js), [`<footer>`](Readability.js), [`aside`](Readability.js), and [`iframe`](Readability.js).
    *   Cleans remaining presentational attributes and inline styles ([`_cleanStyles()`](Readability.js)).
    *   Removes empty paragraphs and extra [`<br>`](Readability.js) tags.
    *   Replaces any [`<h1>`](Readability.js) tags inside the article with [`<h2>`](Readability.js) to preserve the main article title's hierarchy.

2.  **Final Polish ([`_postProcessContent()`](Readability.js))**:
    *   **Fix Relative URIs ([`_fixRelativeUris()`](Readability.js))**: All relative [`href`](Readability.js) and [`src`](Readability.js) attributes are converted to absolute URLs using the document's base URI.
    *   **Simplify Nested Elements ([`_simplifyNestedElements()`](Readability.js))**: Unwraps redundant nested elements (e.g., a [`<div>`](Readability.js) containing only another [`<div>`](Readability.js)).
    *   **Clean Classes ([`_cleanClasses()`](Readability.js))**: If the [`keepClasses`](Readability.js) option is false, it removes all [`class`](Readability.js) attributes from elements, except for a preserved list (e.g., `page`).

3.  **Serialization and Return**: The final, cleaned [`articleContent`](Readability.js) element is serialized to an HTML string using the provided [`_serializer`](Readability.js) function (which defaults to [`el.innerHTML`](Readability.js)). The [`parse()`](Readability.js) method returns a result object containing the [`content`](Readability.js), [`title`](Readability.js), [`byline`](Readability.js), [`excerpt`](Readability.js), and other extracted metadata.

### **Conclusion**

The conversion process is a sophisticated pipeline of parsing, heuristic-based scoring, and aggressive cleaning.

*   **[`JSDOMParser.js`](JSDOMParser.js)** acts as the entry point for the raw HTML, providing a necessary DOM abstraction for the core logic to work on.
*   **[`Readability.js`](Readability.js)** is the engine that performs the heavy lifting. It intelligently analyzes the structure and semantics of the document to distinguish between content and noise, reconstructs the article from potentially fragmented parts, and sanitizes it to produce a clean, readable output.
