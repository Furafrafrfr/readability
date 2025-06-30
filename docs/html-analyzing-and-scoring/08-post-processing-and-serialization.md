# Post-Processing and Serialization

This document provides a detailed analysis of the final content polishing logic within the [`_postProcessContent()`](Readability.js:282) method in `Readability.js`. This is the final step before serialization, where the extracted article content undergoes three critical transformations to ensure it's ready for display.

## Overview

The [`_postProcessContent()`](Readability.js:282) method performs three sequential operations on the article content:

1. **Relative URI Fixing**: Converts relative URLs to absolute URLs
2. **Nested Element Simplification**: Unwraps redundant nested elements
3. **Class Attribute Cleaning**: Removes class attributes (unless preserved)

```javascript
_postProcessContent(articleContent) {
  // Readability cannot open relative uris so we convert them to absolute uris.
  this._fixRelativeUris(articleContent);

  this._simplifyNestedElements(articleContent);

  if (!this._keepClasses) {
    // Remove classes.
    this._cleanClasses(articleContent);
  }
}
```

## 1. Relative URI Fixing

The [`_fixRelativeUris()`](Readability.js:457) method ensures all URLs in the article content are absolute, making them accessible regardless of where the content is displayed.

### Implementation Details

#### Base URI Resolution

The method first establishes the base URI context:

```javascript
var baseURI = this._doc.baseURI;
var documentURI = this._doc.documentURI;
```

It then defines a `toAbsoluteURI` helper function that handles the conversion logic:

```javascript
function toAbsoluteURI(uri) {
  // Leave hash links alone if the base URI matches the document URI:
  if (baseURI == documentURI && uri.charAt(0) == "#") {
    return uri;
  }

  // Otherwise, resolve against base URI:
  try {
    return new URL(uri, baseURI).href;
  } catch (ex) {
    // Something went wrong, just return the original:
  }
  return uri;
}
```

**Key behaviors:**
- **Hash link preservation**: If the base URI matches the document URI and the URI starts with `#`, it's left unchanged (for same-page navigation)
- **URL resolution**: Uses the browser's `URL` constructor to resolve relative URLs against the base URI
- **Error handling**: If URL resolution fails, returns the original URI unchanged

#### Link Processing

The method processes all anchor elements (`<a>` tags):

```javascript
var links = this._getAllNodesWithTag(articleContent, ["a"]);
this._forEachNode(links, function (link) {
  var href = link.getAttribute("href");
  if (href) {
    // Remove links with javascript: URIs
    if (href.indexOf("javascript:") === 0) {
      // Handle content preservation...
    } else {
      link.setAttribute("href", toAbsoluteURI(href));
    }
  }
});
```

**Special handling for JavaScript URLs:**
- **Single text node**: If the link contains only a single text node, it's replaced with a plain text node
- **Multiple children**: If the link has multiple children, they're wrapped in a `<span>` element
- **Regular URLs**: Converted to absolute URLs using `toAbsoluteURI`

#### Media Element Processing

The method processes media elements with various attributes:

```javascript
var medias = this._getAllNodesWithTag(articleContent, [
  "img", "picture", "figure", "video", "audio", "source"
]);

this._forEachNode(medias, function (media) {
  var src = media.getAttribute("src");
  var poster = media.getAttribute("poster");
  var srcset = media.getAttribute("srcset");

  // Convert each attribute to absolute URI...
});
```

**Attributes processed:**
- **`src`**: Direct source URL
- **`poster`**: Poster image for videos
- **`srcset`**: Responsive image source sets

**Srcset handling:**
The method uses a regex pattern to handle `srcset` attributes, which can contain multiple URLs with descriptors:

```javascript
if (srcset) {
  var newSrcset = srcset.replace(
    this.REGEXPS.srcsetUrl,
    function (_, p1, p2, p3) {
      return toAbsoluteURI(p1) + (p2 || "") + p3;
    }
  );
  media.setAttribute("srcset", newSrcset);
}
```

The regex [`REGEXPS.srcsetUrl`](Readability.js:163) pattern: `/(\S+)(\s+[\d.]+[xw])?(\s*(?:,|$))/g` captures:
- `p1`: The URL
- `p2`: Optional descriptor (width or pixel density)
- `p3`: Delimiter (comma or end of string)

## 2. Nested Element Simplification

The [`_simplifyNestedElements()`](Readability.js:538) method flattens redundant nested DIV and SECTION elements to create cleaner HTML structure.

### Implementation Logic

The method traverses the entire article content tree:

```javascript
_simplifyNestedElements(articleContent) {
  var node = articleContent;

  while (node) {
    if (
      node.parentNode &&
      ["DIV", "SECTION"].includes(node.tagName) &&
      !(node.id && node.id.startsWith("readability"))
    ) {
      // Process the node...
    }
    node = this._getNextNode(node);
  }
}
```

### Processing Conditions

**Target elements**: Only DIV and SECTION elements are processed, excluding those with Readability-generated IDs (which start with "readability").

**Two simplification scenarios:**

#### 1. Empty Element Removal

```javascript
if (this._isElementWithoutContent(node)) {
  node = this._removeAndGetNext(node);
  continue;
}
```

Elements are considered empty if they:
- Have no text content after trimming
- Have no children, OR only contain `<br>` and `<hr>` elements

#### 2. Single Child Unwrapping

```javascript
else if (
  this._hasSingleTagInsideElement(node, "DIV") ||
  this._hasSingleTagInsideElement(node, "SECTION")
) {
  var child = node.children[0];
  // Copy all attributes from parent to child
  for (var i = 0; i < node.attributes.length; i++) {
    child.setAttributeNode(node.attributes[i].cloneNode());
  }
  node.parentNode.replaceChild(child, node);
  node = child;
  continue;
}
```

**Unwrapping process:**
1. **Condition check**: The element must have exactly one DIV or SECTION child with no meaningful text content
2. **Attribute preservation**: All attributes from the parent are copied to the child
3. **DOM replacement**: The parent is replaced with its child in the DOM tree
4. **Continuation**: Processing continues with the child element

### Helper Methods

**[`_isElementWithoutContent()`](Readability.js:2012)**:
```javascript
_isElementWithoutContent(node) {
  return (
    node.nodeType === this.ELEMENT_NODE &&
    !node.textContent.trim().length &&
    (!node.children.length ||
      node.children.length ==
        node.getElementsByTagName("br").length +
          node.getElementsByTagName("hr").length)
  );
}
```

**[`_hasSingleTagInsideElement()`](Readability.js:1997)**:
```javascript
_hasSingleTagInsideElement(element, tag) {
  // Must have exactly 1 element child with given tag
  if (element.children.length != 1 || element.children[0].tagName !== tag) {
    return false;
  }

  // And no text nodes with real content
  return !this._someNode(element.childNodes, function (node) {
    return (
      node.nodeType === this.TEXT_NODE &&
      this.REGEXPS.hasContent.test(node.textContent)
    );
  });
}
```

## 3. Class Attribute Cleaning

The [`_cleanClasses()`](Readability.js:418) method removes CSS class attributes from all elements unless the `keepClasses` option is enabled.

### Implementation Details

```javascript
_cleanClasses(node) {
  var classesToPreserve = this._classesToPreserve;
  var className = (node.getAttribute("class") || "")
    .split(/\s+/)
    .filter(cls => classesToPreserve.includes(cls))
    .join(" ");

  if (className) {
    node.setAttribute("class", className);
  } else {
    node.removeAttribute("class");
  }

  // Recursively process all child elements
  for (node = node.firstElementChild; node; node = node.nextElementSibling) {
    this._cleanClasses(node);
  }
}
```

### Processing Logic

**Class preservation:**
1. **Split classes**: The existing class attribute is split on whitespace
2. **Filter preservation**: Only classes in `_classesToPreserve` are kept
3. **Rejoin**: Preserved classes are joined back with spaces

**Attribute handling:**
- **If preserved classes exist**: Set the filtered class attribute
- **If no preserved classes**: Remove the class attribute entirely

**Default preserved classes:**
The [`CLASSES_TO_PRESERVE`](Readability.js:265) array contains:
```javascript
CLASSES_TO_PRESERVE: ["page"]
```

Additional classes can be preserved through the `classesToPreserve` option in the constructor.

### Recursive Processing

The method recursively processes all child elements using a simple traversal:
```javascript
for (node = node.firstElementChild; node; node = node.nextElementSibling) {
  this._cleanClasses(node);
}
```

This ensures that every element in the article content tree is processed, regardless of nesting depth.

## Integration and Flow

The post-processing step occurs after the main article extraction but before serialization:

1. **Article extraction** ([`_grabArticle()`](Readability.js:1041)) identifies and assembles the main content
2. **Content preparation** ([`_prepArticle()`](Readability.js:792)) cleans and refines the content
3. **Post-processing** ([`_postProcessContent()`](Readability.js:282)) performs final polishing
4. **Serialization** uses the configured serializer to generate the final output

This ensures that the final article content is:
- **Portable**: All URLs are absolute and will work in any context
- **Clean**: Redundant nesting is eliminated
- **Minimal**: Unnecessary CSS classes are removed (unless preservation is requested)

The result is article content that's ready for display in any environment while maintaining its semantic structure and accessibility.
