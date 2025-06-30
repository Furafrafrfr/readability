# Initial DOM Cleanup Phase

This document analyzes the initial DOM traversal and cleanup logic within the [`_grabArticle()`](../Readability.js:1041) method in Readability.js. This is the first stage of the content analysis phase that prepares the document for scoring by removing unlikely candidates, checking visibility, and normalizing element structure.

## Overview

The initial DOM cleanup phase occurs at the beginning of the [`_grabArticle()`](../Readability.js:1041) method and consists of three main operations that run during the primary DOM traversal loop starting at line 1069:

1. **Unlikely Candidate Removal**: Strips elements that are unlikely to contain article content
2. **Visibility Check**: Removes elements that are not visible to users
3. **DIV to P Conversion**: Normalizes document structure by converting appropriate `<div>` elements to `<p>` tags

## 1. Unlikely Candidate Removal

### Overview

The unlikely candidate removal logic is controlled by the [`FLAG_STRIP_UNLIKELYS`](../Readability.js:112) flag. When enabled, this process removes elements whose class names or IDs suggest they contain non-article content like advertisements, comments, or navigation elements.

### Implementation Details

The logic is implemented in the main traversal loop of [`_grabArticle()`](../Readability.js:1127):

```javascript
if (stripUnlikelyCandidates) {
  if (
    this.REGEXPS.unlikelyCandidates.test(matchString) &&
    !this.REGEXPS.okMaybeItsACandidate.test(matchString) &&
    !this._hasAncestorTag(node, "table") &&
    !this._hasAncestorTag(node, "code") &&
    node.tagName !== "BODY" &&
    node.tagName !== "A"
  ) {
    // Remove the node
    node = this._removeAndGetNext(node);
    continue;
  }
}
```

Where `matchString` is constructed as: `node.className + " " + node.id` (line 1074).

### Regular Expressions Used

The system uses two primary regular expressions defined in the [`REGEXPS`](../Readability.js:137) object:

#### Unlikely Candidates Pattern

The [`REGEXPS.unlikelyCandidates`](../Readability.js:140) pattern matches elements that are unlikely to contain article content:

```javascript
/-ad-|ai2html|banner|breadcrumbs|combx|comment|community|cover-wrap|disqus|extra|footer|gdpr|header|legends|menu|related|remark|replies|rss|shoutbox|sidebar|skyscraper|social|sponsor|supplemental|ad-break|agegate|pagination|pager|popup|yom-remote/i
```

**Patterns matched:**
- `-ad-` - Ad-related content (dash-enclosed to avoid false positives)
- `ai2html` - AI2HTML generated content
- `banner` - Banner advertisements or headers
- `breadcrumbs` - Navigation breadcrumbs
- `combx` - Combination boxes, widgets
- `comment` - Comment sections
- `community` - Community/social features
- `cover-wrap` - Cover image wrappers
- `disqus` - Disqus comment system
- `extra` - Extra/supplemental content
- `footer` - Footer content
- `gdpr` - GDPR notices and banners
- `header` - Header content
- `legends` - Chart/image legends
- `menu` - Navigation menus
- `related` - Related content sections
- `remark` - Remarks/comments
- `replies` - Reply sections
- `rss` - RSS feed content
- `shoutbox` - Shoutbox widgets
- `sidebar` - Sidebar content
- `skyscraper` - Skyscraper ad format
- `social` - Social media widgets
- `sponsor` - Sponsored content
- `supplemental` - Supplemental content
- `ad-break` - Advertisement breaks
- `agegate` - Age verification gates
- `pagination` - Pagination controls
- `pager` - Pager controls
- `popup` - Popup content
- `yom-remote` - Yahoo remote content

#### Positive Candidates Pattern

The [`REGEXPS.okMaybeItsACandidate`](../Readability.js:142) pattern provides exceptions for elements that might contain article content despite matching the unlikely pattern:

```javascript
/and|article|body|column|content|main|mathjax|shadow/i
```

**Patterns that override unlikely removal:**
- `and` - Conjunctions in class names (e.g., "header-and-content")
- `article` - Article containers
- `body` - Body content
- `column` - Content columns
- `content` - Main content areas
- `main` - Main content sections
- `mathjax` - MathJax mathematical content
- `shadow` - Shadow DOM content

### Additional Protection Rules

Beyond the regex patterns, several additional rules protect certain elements from removal:

1. **Ancestor Protection**: Elements inside `<table>` or `<code>` tags are not removed
2. **Tag Protection**: `<body>` and `<a>` elements are never removed regardless of class/ID
3. **Role-based Removal**: Elements with certain ARIA roles are also removed

#### Unlikely ARIA Roles

The [`UNLIKELY_ROLES`](../Readability.js:178) array defines ARIA roles that indicate non-content elements:

```javascript
["menu", "menubar", "complementary", "navigation", "alert", "alertdialog", "dialog"]
```

These roles are checked separately at lines 1141-1150.

## 2. Visibility Check

### Overview

The visibility check removes elements that are not visible to users, preventing hidden content from affecting the scoring algorithm. This is crucial for handling modern web practices where content might be hidden via CSS or accessibility attributes.

### Implementation

The [`_isProbablyVisible()`](../Readability.js:2704) method is called for every node at line 1076:

```javascript
_isProbablyVisible(node) {
  // Have to null-check node.style and node.className.includes to deal with SVG and MathML nodes.
  return (
    (!node.style || node.style.display != "none") &&
    (!node.style || node.style.visibility != "hidden") &&
    !node.hasAttribute("hidden") &&
    //check for "fallback-image" so that wikimedia math images are displayed
    (!node.hasAttribute("aria-hidden") ||
      node.getAttribute("aria-hidden") != "true" ||
      (node.className &&
        node.className.includes &&
        node.className.includes("fallback-image")))
  );
}
```

### Visibility Criteria

An element is considered **not visible** (and thus removed) if it has any of the following:

1. **CSS Display**: `style.display` set to `"none"`
2. **CSS Visibility**: `style.visibility` set to `"hidden"`
3. **HTML Hidden Attribute**: The `hidden` attribute is present
4. **ARIA Hidden**: `aria-hidden="true"` attribute is present

### Special Exceptions

The method includes a special exception for **Wikimedia fallback images**:
- Elements with `aria-hidden="true"` are kept if they have a class containing `"fallback-image"`
- This preserves mathematical content and other fallback representations

### Null Safety

The method includes null checks for `node.style` and `node.className.includes` to handle:
- **SVG elements**: Which may not have standard style properties
- **MathML nodes**: Which have different DOM interfaces

## 3. DIV to P Conversion

### Overview

The DIV to P conversion process normalizes document structure by converting `<div>` elements that contain only phrasing content (inline elements like `<a>`, `<em>`, `<strong>`) into `<p>` tags. This improves the accuracy of the scoring algorithm by ensuring semantic consistency.

### Implementation

The conversion logic runs when a `<div>` element is encountered in the main loop (lines 1174-1214):

```javascript
if (node.tagName === "DIV") {
  // Put phrasing content into paragraphs.
  var p = null;
  var childNode = node.firstChild;
  while (childNode) {
    var nextSibling = childNode.nextSibling;
    if (this._isPhrasingContent(childNode)) {
      if (p !== null) {
        p.appendChild(childNode);
      } else if (!this._isWhitespace(childNode)) {
        p = doc.createElement("p");
        node.replaceChild(p, childNode);
        p.appendChild(childNode);
      }
    } else if (p !== null) {
      while (p.lastChild && this._isWhitespace(p.lastChild)) {
        p.lastChild.remove();
      }
      p = null;
    }
    childNode = nextSibling;
  }

  // Handle special cases for DIV->P conversion
  if (
    this._hasSingleTagInsideElement(node, "P") &&
    this._getLinkDensity(node) < 0.25
  ) {
    var newNode = node.children[0];
    node.parentNode.replaceChild(newNode, node);
    node = newNode;
    elementsToScore.push(node);
  } else if (!this._hasChildBlockElement(node)) {
    node = this._setNodeTag(node, "P");
    elementsToScore.push(node);
  }
}
```

### Phrasing Content Detection

The [`_isPhrasingContent()`](../Readability.js:2041) method determines if a node qualifies as phrasing content:

```javascript
_isPhrasingContent(node) {
  return (
    node.nodeType === this.TEXT_NODE ||
    this.PHRASING_ELEMS.includes(node.tagName) ||
    ((node.tagName === "A" ||
      node.tagName === "DEL" ||
      node.tagName === "INS") &&
      this._everyNode(node.childNodes, this._isPhrasingContent))
  );
}
```

#### Phrasing Elements

The [`PHRASING_ELEMS`](../Readability.js:221) array defines elements considered phrasing content:

```javascript
[
  "ABBR", "AUDIO", "B", "BDO", "BR", "BUTTON", "CITE", "CODE", "DATA",
  "DATALIST", "DFN", "EM", "EMBED", "I", "IMG", "INPUT", "KBD", "LABEL",
  "MARK", "MATH", "METER", "NOSCRIPT", "OBJECT", "OUTPUT", "PROGRESS",
  "Q", "RUBY", "SAMP", "SCRIPT", "SELECT", "SMALL", "SPAN", "STRONG",
  "SUB", "SUP", "TEXTAREA", "TIME", "VAR", "WBR"
]
```

**Note**: Elements like `CANVAS`, `IFRAME`, `SVG`, and `VIDEO` are excluded because they tend to be removed by Readability when put into paragraphs.

### Conversion Process

The conversion happens in three phases:

#### Phase 1: Phrasing Content Grouping
- Iterates through all child nodes of the DIV
- Groups consecutive phrasing content into `<p>` elements
- Preserves non-phrasing content as-is
- Removes trailing whitespace from created paragraphs

#### Phase 2: Single Paragraph DIV Unwrapping
For DIVs that contain only a single `<p>` element with low link density:
- Moves the paragraph's attributes to replace the DIV
- Replaces the DIV with the paragraph
- Adds the new paragraph to the scoring candidates

Conditions:
- Must have exactly one `<p>` child element ([`_hasSingleTagInsideElement()`](../Readability.js:1997))
- Link density must be less than 25% ([`_getLinkDensity()`](../Readability.js:2127))

#### Phase 3: Block-less DIV Conversion
For DIVs that contain no block-level elements:
- Converts the entire DIV to a `<p>` element ([`_setNodeTag()`](../Readability.js:762))
- Preserves all content and attributes
- Adds the new paragraph to scoring candidates

Condition:
- Must not contain any child block elements ([`_hasChildBlockElement()`](../Readability.js:2028))

### Block Element Detection

The [`DIV_TO_P_ELEMS`](../Readability.js:188) set defines elements considered block-level for conversion purposes:

```javascript
new Set(["BLOCKQUOTE", "DL", "DIV", "IMG", "OL", "P", "PRE", "TABLE", "UL"])
```

### Benefits of Conversion

1. **Semantic Consistency**: Ensures inline content is properly wrapped in paragraph tags
2. **Scoring Accuracy**: Paragraphs receive different scoring weights than DIVs
3. **Content Normalization**: Reduces structural variations between different websites
4. **Algorithm Reliability**: Provides consistent structure for subsequent processing steps

## Integration with Main Algorithm

All three cleanup operations work together during the main DOM traversal:

1. **Visibility Check** (line 1076): Removes hidden elements first
2. **Unlikely Candidate Removal** (line 1127): Removes elements with suspicious class names/IDs
3. **DIV to P Conversion** (line 1174): Normalizes remaining structure for scoring

The cleaned and normalized DOM then proceeds to the scoring phase where elements are evaluated for their likelihood of containing article content.
