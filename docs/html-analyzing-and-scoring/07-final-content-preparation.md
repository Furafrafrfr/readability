# Final Content Preparation

The [`_prepArticle()`](../Readability.js:792) method represents the final phase of content processing in Readability.js. This is where the selected article content undergoes comprehensive cleaning and normalization to create the final readable output. The method performs four main categories of operations:

## 1. Undesirable Element Removal

### Form and Interactive Elements
The algorithm removes various undesirable elements that don't contribute to reading comprehension:

```javascript
// Clean out junk from the article content
this._cleanConditionally(articleContent, "form");
this._cleanConditionally(articleContent, "fieldset");
this._clean(articleContent, "object");
this._clean(articleContent, "embed");
this._clean(articleContent, "footer");
this._clean(articleContent, "link");
this._clean(articleContent, "aside");

// Later, remove more interactive elements
this._clean(articleContent, "iframe");
this._clean(articleContent, "input");
this._clean(articleContent, "textarea");
this._clean(articleContent, "select");
this._clean(articleContent, "button");
```

#### The `_clean()` Method
The [`_clean()`](../Readability.js:2192) method completely removes all elements of a specified tag type, with special exceptions for video content:

- **Video Preservation**: Elements containing YouTube, Vimeo, or other video content are preserved
- **Attribute Checking**: For embed elements (`<object>`, `<embed>`, `<iframe>`), the method checks all attributes against [`this._allowedVideoRegex`](../Readability.js:65)
- **Content Checking**: For `<object>` tags, it also checks the inner HTML for video indicators

#### The `_cleanConditionally()` Method
The [`_cleanConditionally()`](../Readability.js:2444) method applies more sophisticated heuristics to determine whether elements should be removed:

- **Data Table Protection**: Never removes elements that are data tables or contained within data tables
- **Code Block Protection**: Preserves elements within `<code>` tags
- **Content Analysis**: Uses multiple metrics including:
  - Character count (comma-separated content)
  - Paragraph-to-image ratio
  - List item density
  - Input field frequency
  - Heading density
  - Link density
  - Embed count
  - Overall content length

### Share Elements
Special handling for social sharing elements:

```javascript
var shareElementThreshold = this.DEFAULT_CHAR_THRESHOLD;

this._forEachNode(articleContent.children, function (topCandidate) {
  this._cleanMatchedNodes(topCandidate, function (node, matchString) {
    return (
      this.REGEXPS.shareElements.test(matchString) &&
      node.textContent.length < shareElementThreshold
    );
  });
});
```

- **Pattern Detection**: Uses [`REGEXPS.shareElements`](../Readability.js:156): `/(\b|_)(share|sharedaddy)(\b|_)/i`
- **Length Threshold**: Only removes share elements with content shorter than [`DEFAULT_CHAR_THRESHOLD`](../Readability.js:133) (500 characters)
- **Scope Limitation**: Only applied to top-level candidates, not nested deeply within content

## 2. Attribute and Style Cleaning

### The `_cleanStyles()` Method
The [`_cleanStyles()`](../Readability.js:2098) method removes all presentational attributes and styling information:

```javascript
_cleanStyles(e) {
  if (!e || e.tagName.toLowerCase() === "svg") {
    return;
  }

  // Remove `style` and deprecated presentational attributes
  for (var i = 0; i < this.PRESENTATIONAL_ATTRIBUTES.length; i++) {
    e.removeAttribute(this.PRESENTATIONAL_ATTRIBUTES[i]);
  }

  if (this.DEPRECATED_SIZE_ATTRIBUTE_ELEMS.includes(e.tagName)) {
    e.removeAttribute("width");
    e.removeAttribute("height");
  }

  var cur = e.firstElementChild;
  while (cur !== null) {
    this._cleanStyles(cur);
    cur = cur.nextElementSibling;
  }
}
```

#### Removed Attributes
The method removes all attributes defined in [`PRESENTATIONAL_ATTRIBUTES`](../Readability.js:202):
- `align` - Text and element alignment
- `background` - Background images
- `bgcolor` - Background colors
- `border` - Border styling
- `cellpadding` - Table cell padding
- `cellspacing` - Table cell spacing
- `frame` - Table frame styling
- `hspace` - Horizontal spacing
- `rules` - Table rules
- `style` - Inline CSS styles
- `valign` - Vertical alignment
- `vspace` - Vertical spacing

#### Size Attribute Removal
For elements in [`DEPRECATED_SIZE_ATTRIBUTE_ELEMS`](../Readability.js:217) (`TABLE`, `TH`, `TD`, `HR`, `PRE`):
- Removes `width` and `height` attributes
- Preserves semantic structure while removing presentation constraints

#### Recursive Processing
- **SVG Protection**: Skips SVG elements to preserve their styling
- **Recursive Traversal**: Processes all child elements recursively
- **Element-First Traversal**: Uses `firstElementChild` and `nextElementSibling` for efficiency

## 3. Heading Normalization

### H1 to H2 Conversion
All `<h1>` elements within the article content are converted to `<h2>` elements:

```javascript
// replace H1 with H2 as H1 should be only title that is displayed separately
this._replaceNodeTags(
  this._getAllNodesWithTag(articleContent, ["h1"]),
  "h2"
);
```

#### Implementation Details
The [`_replaceNodeTags()`](../Readability.js:327) method:
- **Finds All H1s**: Uses `_getAllNodesWithTag()` to locate all `<h1>` elements
- **Tag Replacement**: Calls `_setNodeTag()` for each element
- **Attribute Preservation**: Maintains all existing attributes during conversion
- **Content Preservation**: Keeps all child nodes and text content intact

#### Rationale
- **Semantic Hierarchy**: Ensures the article title (typically an H1) remains the primary heading
- **Consistent Structure**: Creates a uniform heading hierarchy within the article content
- **Screen Reader Compatibility**: Provides better accessibility by maintaining logical heading structure

### Header Cleaning
Additional header cleaning via [`_cleanHeaders()`](../Readability.js:2669):

```javascript
_cleanHeaders(e) {
  let headingNodes = this._getAllNodesWithTag(e, ["h1", "h2"]);
  this._removeNodes(headingNodes, function (node) {
    let shouldRemove = this._getClassWeight(node) < 0;
    if (shouldRemove) {
      this.log("Removing header with low class weight:", node);
    }
    return shouldRemove;
  });
}
```

- **Weight-Based Removal**: Removes headers with negative class weights
- **Quality Assessment**: Uses the same class weight system as content scoring
- **Selective Removal**: Only removes headers that appear to be navigational or promotional

## 4. Whitespace and Paragraph Cleanup

### Empty Paragraph Removal
Removes paragraphs that contain no meaningful content:

```javascript
this._removeNodes(
  this._getAllNodesWithTag(articleContent, ["p"]),
  function (paragraph) {
    var contentElementCount = this._getAllNodesWithTag(paragraph, [
      "img",
      "embed",
      "object",
      "iframe",
    ]).length;
    return (
      contentElementCount === 0 && !this._getInnerText(paragraph, false)
    );
  }
);
```

#### Criteria for Removal
- **No Media Content**: Must contain no images, embeds, objects, or iframes
- **No Text Content**: Must have no meaningful text after trimming whitespace
- **Preservation Logic**: Paragraphs with media content are kept even if they have no text

### Break Tag Cleanup
Removes redundant `<br>` tags that appear before paragraphs:

```javascript
this._forEachNode(
  this._getAllNodesWithTag(articleContent, ["br"]),
  function (br) {
    var next = this._nextNode(br.nextSibling);
    if (next && next.tagName == "P") {
      br.remove();
    }
  }
);
```

#### Logic
- **Adjacent Detection**: Uses `_nextNode()` to find the next non-whitespace element
- **Paragraph Proximity**: Removes `<br>` tags immediately followed by `<p>` tags
- **Whitespace Handling**: Ignores whitespace-only text nodes between elements

### Single-Cell Table Conversion
Converts single-cell tables to simpler block elements:

```javascript
this._forEachNode(
  this._getAllNodesWithTag(articleContent, ["table"]),
  function (table) {
    var tbody = this._hasSingleTagInsideElement(table, "TBODY")
      ? table.firstElementChild
      : table;
    if (this._hasSingleTagInsideElement(tbody, "TR")) {
      var row = tbody.firstElementChild;
      if (this._hasSingleTagInsideElement(row, "TD")) {
        var cell = row.firstElementChild;
        cell = this._setNodeTag(
          cell,
          this._everyNode(cell.childNodes, this._isPhrasingContent)
            ? "P"
            : "DIV"
        );
        table.parentNode.replaceChild(cell, table);
      }
    }
  }
);
```

#### Conversion Logic
- **Single-Cell Detection**: Checks for tables with exactly one tbody → tr → td chain
- **Content Analysis**: Determines whether cell content is phrasing content
- **Tag Selection**:
  - Converts to `<p>` if content is all phrasing content (inline elements)
  - Converts to `<div>` if content contains block elements
- **Structure Simplification**: Removes unnecessary table markup while preserving content

## Processing Order

The operations are performed in a specific order to maximize effectiveness:

1. **Style Cleaning**: First removes all presentational attributes
2. **Data Table Marking**: Identifies and protects legitimate data tables
3. **Lazy Image Fixing**: Converts lazy-loaded images to standard format
4. **Conditional Cleaning**: Removes form elements and fieldsets with heuristics
5. **Absolute Cleaning**: Removes objects, embeds, footers, links, asides
6. **Share Element Cleaning**: Removes small social sharing elements
7. **Interactive Element Cleaning**: Removes iframes, inputs, textareas, selects, buttons
8. **Header Cleaning**: Removes low-quality headers
9. **Structural Cleaning**: Conditionally removes tables, lists, and divs
10. **Heading Normalization**: Converts H1s to H2s
11. **Paragraph Cleanup**: Removes empty paragraphs
12. **Break Tag Cleanup**: Removes redundant line breaks
13. **Table Simplification**: Converts single-cell tables to block elements

This careful ordering ensures that elements are properly evaluated and that removal decisions are made with complete context about the content structure.
