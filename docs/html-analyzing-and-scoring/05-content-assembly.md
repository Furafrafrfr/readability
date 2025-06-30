# Content Assembly Phase

This document analyzes the content assembly logic within the [`_grabArticle()`](../Readability.js:1041) method, which occurs after the `topCandidate` has been selected. This phase is responsible for gathering related content from sibling elements and assembling the final article content.

## 1. Article Container Creation

The content assembly begins by creating a new `<div>` element to serve as the article container:

```javascript
var articleContent = doc.createElement("DIV");
if (isPaging) {
  articleContent.id = "readability-content";
}
```

**Key Points:**
- A new `<div>` element is created as the root container for the assembled article content
- If the extraction is part of a paging operation, the container receives the ID `"readability-content"`
- This container will ultimately hold the `topCandidate` and any qualifying sibling elements

## 2. Sibling Score Threshold Calculation

Before processing siblings, a threshold score is calculated to determine which siblings qualify for inclusion:

```javascript
var siblingScoreThreshold = Math.max(
  10,
  topCandidate.readability.contentScore * 0.2
);
```

**Threshold Logic:**
- The threshold is set to **20% of the top candidate's score**
- A **minimum threshold of 10** is enforced to prevent extremely low-scoring content from being included
- This ensures that only reasonably high-quality sibling content is considered for inclusion

## 3. Sibling Processing Loop

The algorithm iterates through all children of the `topCandidate`'s parent element:

```javascript
parentOfTopCandidate = topCandidate.parentNode;
var siblings = parentOfTopCandidate.children;

for (var s = 0, sl = siblings.length; s < sl; s++) {
  var sibling = siblings[s];
  var append = false;
  // ... inclusion logic
}
```

**Important Implementation Details:**
- The loop processes siblings in document order
- The sibling collection is refreshed after each append operation to handle DOM changes
- Array indices are adjusted when elements are moved to account for the shifting collection

## 4. Sibling Inclusion Logic

The algorithm uses a multi-tiered approach to determine whether each sibling should be included:

### 4.1. Top Candidate Inclusion
```javascript
if (sibling === topCandidate) {
  append = true;
}
```
- The `topCandidate` itself is **always included** in the final content

### 4.2. Score-Based Inclusion
```javascript
var contentBonus = 0;

// Give a bonus if sibling nodes and top candidates have the same classname
if (
  sibling.className === topCandidate.className &&
  topCandidate.className !== ""
) {
  contentBonus += topCandidate.readability.contentScore * 0.2;
}

if (
  sibling.readability &&
  sibling.readability.contentScore + contentBonus >= siblingScoreThreshold
) {
  append = true;
}
```

**Score-Based Criteria:**
- Siblings with readability scores above the threshold are included
- **Class Name Bonus**: Siblings sharing the same non-empty class name as the `topCandidate` receive a bonus equal to 20% of the top candidate's score
- This bonus helps include content that may have been split across multiple containers with the same styling

### 4.3. Special Handling for `<p>` Tags

Paragraph elements receive special consideration even if they don't meet the score threshold:

```javascript
else if (sibling.nodeName === "P") {
  var linkDensity = this._getLinkDensity(sibling);
  var nodeContent = this._getInnerText(sibling);
  var nodeLength = nodeContent.length;

  if (nodeLength > 80 && linkDensity < 0.25) {
    append = true;
  } else if (
    nodeLength < 80 &&
    nodeLength > 0 &&
    linkDensity === 0 &&
    nodeContent.search(/\.( |$)/) !== -1
  ) {
    append = true;
  }
}
```

**Paragraph Inclusion Rules:**

1. **Long Paragraphs with Low Link Density:**
   - Content length **> 80 characters**
   - Link density **< 25%**
   - These are considered substantial content paragraphs

2. **Short Paragraphs with Complete Sentences:**
   - Content length **< 80 characters** but **> 0**
   - **Zero link density** (no links at all)
   - Must contain at least one period followed by a space or end of string (`/\.( |$)/`)
   - These are likely to be complete, standalone sentences

## 5. Element Normalization

When a sibling qualifies for inclusion, it may be normalized before being added to the article content:

```javascript
if (append) {
  if (!this.ALTER_TO_DIV_EXCEPTIONS.includes(sibling.nodeName)) {
    sibling = this._setNodeTag(sibling, "DIV");
  }
  articleContent.appendChild(sibling);
}
```

**Normalization Logic:**
- Elements **not** in the `ALTER_TO_DIV_EXCEPTIONS` list are converted to `<div>` tags
- The exceptions list includes: `["DIV", "ARTICLE", "SECTION", "P", "OL", "UL"]`
- This prevents uncommon block-level elements (like `<form>`, `<td>`) from being filtered out later
- The element is then appended to the article content container

## 6. Notable Absence of Specific Tag Handling

**Important Observation:** The sibling processing logic does **not** include special handling for:

- **Headings** (`<h1>`, `<h2>`, etc.): These are evaluated purely on their readability scores
- **Lists** (`<ol>`, `<ul>`, `<dl>`): These follow the same score-based criteria as other elements
- **Specific class names** like `"page"` or `"author"`: Only the class name matching bonus applies

**Implications:**
- Headings and lists must either:
  1. Have sufficient readability scores to meet the threshold, or
  2. Be direct children of the `topCandidate` (not siblings)
- Content with special semantic meaning (like author information) is not given preferential treatment during sibling processing

## 7. Index Management

The algorithm carefully manages array indices during the append process:

```javascript
articleContent.appendChild(sibling);
siblings = parentOfTopCandidate.children;
s -= 1;
sl -= 1;
```

**Why This Is Necessary:**
- `appendChild()` removes the element from its current position in the DOM
- This causes the `siblings` collection to shift, changing the indices
- The loop counter and length are decremented to account for this shift
- The siblings collection is refreshed to maintain compatibility with DOM parsers

## 8. Summary

The content assembly phase employs a sophisticated multi-criteria approach to gather related content:

1. **Guaranteed Inclusion**: The top candidate is always included
2. **Score-Based Inclusion**: High-scoring siblings with optional class name bonuses
3. **Special Paragraph Handling**: Content-based inclusion for paragraphs that might have low scores
4. **Element Normalization**: Conversion of uncommon elements to divs for consistency
5. **Careful DOM Management**: Proper handling of live collections during element movement

This approach ensures that the final article content includes not only the highest-scoring content but also related paragraphs and content that might have been split across multiple containers or affected by advertisements and other interruptions.
