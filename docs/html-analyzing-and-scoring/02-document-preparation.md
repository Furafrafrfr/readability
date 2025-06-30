# Document Pre-processing and Sanitization Logic

This document provides a detailed analysis of the document pre-processing and sanitization logic within the [`_prepDocument()`](../Readability.js:669) method in `Readability.js`. This analysis covers the three main operations: script/style/noscript tag removal, consecutive `<br>` tag replacement, and deprecated `<font>` tag conversion.

## Overview

The [`_prepDocument()`](../Readability.js:669) method is called early in the parsing process to clean and normalize the HTML document before content analysis begins. It performs essential sanitization operations to:

1. Remove potentially harmful or distracting elements (`<script>`, `<style>`, `<noscript>`)
2. Normalize presentation-focused markup into semantic elements
3. Prepare the document structure for accurate content scoring

## 1. Script, Style, and Noscript Tag Removal

### Script and Noscript Removal

**Method**: [`_removeScripts(doc)`](../Readability.js:1985)

```javascript
_removeScripts(doc) {
  this._removeNodes(this._getAllNodesWithTag(doc, ["script", "noscript"]));
}
```

**Purpose**:
- Remove all JavaScript code that could interfere with parsing or pose security risks
- Remove `<noscript>` fallback content that's no longer needed

**Implementation Details**:
- Uses [`_getAllNodesWithTag()`](../Readability.js:397) to find all `<script>` and `<noscript>` elements
- Removes all matching elements regardless of their attributes or content
- Called from the main [`parse()`](../Readability.js:2749) method before document preparation

**Behavior**:
- **Complete removal**: Both opening and closing tags, along with all content, are removed
- **No exceptions**: All script types are removed (JavaScript, VBScript, etc.)
- **Recursive**: Nested script tags within other elements are also removed

### Style Tag Removal

**Method**: Part of [`_prepDocument()`](../Readability.js:673)

```javascript
// Remove all style tags in head
this._removeNodes(this._getAllNodesWithTag(doc, ["style"]));
```

**Purpose**:
- Remove CSS styles that could interfere with content analysis
- Eliminate presentation-only markup that doesn't contribute to content scoring

**Implementation Details**:
- Removes all `<style>` elements from the entire document (not just head)
- Uses the same removal mechanism as script removal

**Behavior**:
- **Complete removal**: All `<style>` tags and their CSS content are removed
- **Document-wide**: Removes style tags from head, body, and any other location

## 2. Consecutive `<br>` Tag Replacement

### Method: [`_replaceBrs(elem)`](../Readability.js:706)

**Purpose**:
Convert sequences of `<br>` tags used for paragraph separation into proper `<p>` elements for better semantic structure and content analysis.

### Algorithm Overview

The replacement logic follows these steps:

1. **Find BR chains**: Identify sequences of 2 or more consecutive `<br>` elements
2. **Remove extras**: Remove all but the first `<br>` in each chain
3. **Create paragraphs**: Replace the remaining `<br>` with a `<p>` element
4. **Collect content**: Move following content into the new `<p>` until hitting another BR chain or block element

### Detailed Implementation

```javascript
_replaceBrs(elem) {
  this._forEachNode(this._getAllNodesWithTag(elem, ["br"]), function (br) {
    var next = br.nextSibling;
    var replaced = false;

    // Remove consecutive <br> elements, keeping only the first
    while ((next = this._nextNode(next)) && next.tagName == "BR") {
      replaced = true;
      var brSibling = next.nextSibling;
      next.remove();
      next = brSibling;
    }

    // If we found a chain, replace the first <br> with <p>
    if (replaced) {
      var p = this._doc.createElement("p");
      br.parentNode.replaceChild(p, br);

      // Collect following content into the new paragraph
      next = p.nextSibling;
      while (next) {
        // Stop if we hit another <br> chain
        if (next.tagName == "BR") {
          var nextElem = this._nextNode(next.nextSibling);
          if (nextElem && nextElem.tagName == "BR") {
            break;
          }
        }

        // Stop if we hit non-phrasing content
        if (!this._isPhrasingContent(next)) {
          break;
        }

        // Move this node into the paragraph
        var sibling = next.nextSibling;
        p.appendChild(next);
        next = sibling;
      }

      // Clean up trailing whitespace
      while (p.lastChild && this._isWhitespace(p.lastChild)) {
        p.lastChild.remove();
      }

      // Handle nested paragraph issue
      if (p.parentNode.tagName === "P") {
        this._setNodeTag(p.parentNode, "DIV");
      }
    }
  });
}
```

### Key Behaviors

**BR Chain Detection**:
- Uses [`_nextNode()`](../Readability.js:687) to skip whitespace-only text nodes
- Considers only actual `<br>` elements, ignoring whitespace between them
- Requires 2 or more consecutive `<br>` elements to trigger replacement

**Content Collection**:
- Only collects [phrasing content](https://developer.mozilla.org/en-US/docs/Web/Guide/HTML/Content_categories#Phrasing_content) into paragraphs
- Stops at block-level elements or another BR chain
- Preserves inline formatting within collected content

**Whitespace Handling**:
- Removes trailing whitespace from created paragraphs
- Uses [`_isWhitespace()`](../Readability.js:2052) to identify whitespace-only nodes

**Nested Paragraph Prevention**:
- Converts parent `<p>` elements to `<div>` if a new `<p>` would be nested inside

### Example Transformation

**Input**:
```html
<div>
  Lorem ipsum<br/>dolor sit<br/> <br/><br/>amet, consectetur
</div>
```

**Output**:
```html
<div>
  Lorem ipsum<br/>dolor sit<p>amet, consectetur</p>
</div>
```

## 3. Font Tag Conversion

### Method: Part of [`_prepDocument()`](../Readability.js:679)

```javascript
this._replaceNodeTags(this._getAllNodesWithTag(doc, ["font"]), "SPAN");
```

**Purpose**:
Convert deprecated `<font>` elements to modern `<span>` elements while preserving all attributes for styling compatibility.

### Implementation Details

**Tag Replacement Process**:
1. **Find all font tags**: Uses [`_getAllNodesWithTag()`](../Readability.js:397) to locate all `<font>` elements
2. **Replace with spans**: Uses [`_replaceNodeTags()`](../Readability.js:327) to convert each `<font>` to `<span>`
3. **Preserve attributes**: All attributes (face, size, color, etc.) are maintained on the new `<span>` elements
4. **Maintain content**: All child nodes and text content remain unchanged

### The [`_replaceNodeTags()`] Method

```javascript
_replaceNodeTags(nodeList, newTagName) {
  // Avoid ever operating on live node lists
  if (this._docJSDOMParser && nodeList._isLiveNodeList) {
    throw new Error("Do not pass live node lists to _replaceNodeTags");
  }
  for (const node of nodeList) {
    this._setNodeTag(node, newTagName);
  }
}
```

### The [`_setNodeTag()`](../Readability.js:762) Method

```javascript
_setNodeTag(node, tag) {
  this.log("_setNodeTag", node, tag);
  if (this._docJSDOMParser) {
    // For JSDOM parser, just update the tag names
    node.localName = tag.toLowerCase();
    node.tagName = tag.toUpperCase();
    return node;
  }

  // For real DOM, create a new element and transfer everything
  var replacement = node.ownerDocument.createElement(tag);
  while (node.firstChild) {
    replacement.appendChild(node.firstChild);
  }
  node.parentNode.replaceChild(replacement, node);

  // Preserve readability scoring data
  if (node.readability) {
    replacement.readability = node.readability;
  }

  // Copy all attributes
  for (var i = 0; i < node.attributes.length; i++) {
    replacement.setAttributeNode(node.attributes[i].cloneNode());
  }
  return replacement;
}
```

### Key Behaviors

**Attribute Preservation**:
- All attributes from `<font>` tags are copied to the new `<span>` elements
- Includes deprecated attributes like `face`, `size`, and `color`
- Maintains styling compatibility for CSS that might target these attributes

**Content Preservation**:
- All child elements and text content are moved to the new `<span>`
- Nested elements maintain their structure and attributes
- No content is lost during the conversion

**DOM Compatibility**:
- Handles both JSDOM parser (faster, in-place modification) and real DOM (creates new elements)
- Preserves any readability scoring data attached to elements

### Example Transformation

**Input**:
```html
<font face="Arial" size="2">
  <font face="Times" size="10">Lorem</font> ipsum dolor
</font>
```

**Output**:
```html
<span face="Arial" size="2">
  <span face="Times" size="10">Lorem</span> ipsum dolor
</span>
```

## Integration with Overall Processing

The document preparation operations occur early in the parsing workflow:

1. **JSON-LD extraction** (before script removal)
2. **Script removal** ([`_removeScripts()`](../Readability.js:1985))
3. **Document preparation** ([`_prepDocument()`](../Readability.js:669))
   - Style tag removal
   - BR replacement
   - Font tag conversion
4. **Content analysis and scoring**

This ordering ensures that:
- Metadata is extracted before scripts are removed
- Document structure is normalized before content analysis
- Sanitization doesn't interfere with content scoring algorithms

## Error Handling and Edge Cases

**Live NodeList Protection**:
- [`_replaceNodeTags()`](../Readability.js:327) includes protection against live NodeLists to prevent modification during iteration

**Whitespace Handling**:
- BR replacement includes careful whitespace management to avoid empty paragraphs
- Uses specialized [`_isWhitespace()`](../Readability.js:2052) detection

**Nested Elements**:
- Font conversion preserves complex nested structures
- BR replacement prevents invalid nested paragraph structures

**Performance Considerations**:
- Operations work with static NodeLists to avoid live collection issues
- JSDOM optimization for tag replacement when available
- Efficient traversal using specialized helper methods

This pre-processing logic ensures that the document is in an optimal state for content analysis while preserving semantic meaning and avoiding common HTML parsing pitfalls.
