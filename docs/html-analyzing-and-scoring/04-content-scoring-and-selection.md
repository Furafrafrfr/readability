# Content Scoring and Candidate Selection Analysis

This document provides a detailed analysis of the content scoring and candidate selection logic within the [`_grabArticle()`](../Readability.js:1041) method in Readability.js. This is the core algorithm that identifies the most likely article content from a web page.

## Overview

The content scoring system operates in several phases:
1. **Candidate Element Initialization** - Identify paragraph-like elements and set base scores
2. **Content Scoring Heuristics** - Calculate scores based on text characteristics
3. **Class/ID Weighting** - Adjust scores based on semantic HTML attributes
4. **Score Propagation** - Distribute scores from child elements to ancestors
5. **Top Candidate Selection** - Rank candidates and select the best one

## 1. Candidate Element Initialization

### Element Identification
The algorithm identifies candidate elements by checking against the [`DEFAULT_TAGS_TO_SCORE`](../Readability.js:128) constant:

```javascript
DEFAULT_TAGS_TO_SCORE: "section,h2,h3,h4,h5,h6,p,td,pre".toUpperCase().split(",")
```

Elements matching these tags are added to the `elementsToScore` array during DOM traversal:

```javascript
if (this.DEFAULT_TAGS_TO_SCORE.includes(node.tagName)) {
  elementsToScore.push(node);
}
```

### Base Score Initialization
Each candidate element receives a base `contentScore` through the [`_initializeNode()`](../Readability.js:903) method:

```javascript
_initializeNode(node) {
  node.readability = { contentScore: 0 };

  switch (node.tagName) {
    case "DIV":
      node.readability.contentScore += 5;
      break;

    case "PRE":
    case "TD":
    case "BLOCKQUOTE":
      node.readability.contentScore += 3;
      break;

    case "ADDRESS":
    case "OL":
    case "UL":
    case "DL":
    case "DD":
    case "DT":
    case "LI":
    case "FORM":
      node.readability.contentScore -= 3;
      break;

    case "H1":
    case "H2":
    case "H3":
    case "H4":
    case "H5":
    case "H6":
    case "TH":
      node.readability.contentScore -= 5;
      break;
  }

  node.readability.contentScore += this._getClassWeight(node);
}
```

**Key Points:**
- Most elements start with a score of 0
- `<div>` elements get a +5 bonus (likely to contain content)
- Block elements like `<pre>`, `<td>`, `<blockquote>` get +3
- List and form elements get -3 (less likely to be main content)
- Heading elements get -5 (headers, not body content)

## 2. Content Scoring Heuristics

The core scoring algorithm analyzes text content characteristics within each candidate element:

```javascript
// Base score: 1 point for any paragraph-like element
contentScore += 1;

// Comma bonus: +1 point per comma found
contentScore += innerText.split(this.REGEXPS.commas).length;

// Length bonus: +1 point per 100 characters, up to 3 points maximum
contentScore += Math.min(Math.floor(innerText.length / 100), 3);
```

### Detailed Scoring Breakdown

#### Base Score (+1 point)
Every paragraph-like element that contains at least 25 characters gets 1 base point. This ensures that even short paragraphs have some positive score.

#### Comma Bonus
The algorithm counts commas using the [`REGEXPS.commas`](../Readability.js:167) pattern:
```javascript
commas: /\u002C|\u060C|\uFE50|\uFE10|\uFE11|\u2E41|\u2E34|\u2E32|\uFF0C/g
```

This pattern matches various comma variants used in different languages:
- `\u002C` - Standard comma (,)
- `\u060C` - Arabic comma (،)
- `\uFE50` - Small comma (﹐)
- `\uFE10` - Vertical comma (︐)
- `\uFE11` - Vertical ideographic comma (︑)
- `\u2E41` - Reversed comma (⹁)
- `\u2E34` - Raised comma (⸴)
- `\u2E32` - Turned comma (⸲)
- `\uFF0C` - Fullwidth comma (，)

The number of commas indicates sentence complexity and is used as a proxy for content richness.

#### Length Bonus
Text length is scored as: `Math.min(Math.floor(innerText.length / 100), 3)`

- **100-199 characters**: +1 point
- **200-299 characters**: +2 points
- **300+ characters**: +3 points (maximum)

This scoring rewards substantial text content while preventing extremely long elements from dominating.

### Minimum Content Threshold
Elements with less than 25 characters are excluded from scoring:
```javascript
if (innerText.length < 25) {
  return; // Skip this element
}
```

## 3. Class/ID Weighting

The [`_getClassWeight()`](../Readability.js:2152) method adjusts scores based on semantic HTML attributes:

```javascript
_getClassWeight(e) {
  if (!this._flagIsActive(this.FLAG_WEIGHT_CLASSES)) {
    return 0;
  }

  var weight = 0;

  // Check className
  if (typeof e.className === "string" && e.className !== "") {
    if (this.REGEXPS.negative.test(e.className)) {
      weight -= 25;
    }
    if (this.REGEXPS.positive.test(e.className)) {
      weight += 25;
    }
  }

  // Check id attribute
  if (typeof e.id === "string" && e.id !== "") {
    if (this.REGEXPS.negative.test(e.id)) {
      weight -= 25;
    }
    if (this.REGEXPS.positive.test(e.id)) {
      weight += 25;
    }
  }

  return weight;
}
```

### Regex Patterns

#### Positive Pattern (+25 points)
[`REGEXPS.positive`](../Readability.js:145):
```javascript
/article|body|content|entry|hentry|h-entry|main|page|pagination|post|text|blog|story/i
```

Elements with class names or IDs containing these terms are likely to contain main content.

#### Negative Pattern (-25 points)
[`REGEXPS.negative`](../Readability.js:147):
```javascript
/-ad-|hidden|^hid$| hid$| hid |^hid |banner|combx|comment|com-|contact|footer|gdpr|masthead|media|meta|outbrain|promo|related|scroll|share|shoutbox|sidebar|skyscraper|sponsor|shopping|tags|widget/i
```

Elements with these patterns are likely to be:
- Advertisements (`-ad-`, `banner`, `sponsor`)
- Navigation/UI elements (`sidebar`, `footer`, `masthead`)
- Interactive elements (`comment`, `share`, `scroll`)
- Hidden content (`hidden`, `hid`)

### FLAG_WEIGHT_CLASSES Impact
The class/ID weighting only applies when the [`FLAG_WEIGHT_CLASSES`](../Readability.js:113) flag is active. This flag can be disabled during retry attempts if initial parsing fails.

## 4. Score Propagation

Once an element's content score is calculated, it's propagated up the DOM tree to ancestor elements:

```javascript
this._forEachNode(ancestors, function(ancestor, level) {
  if (typeof ancestor.readability === "undefined") {
    this._initializeNode(ancestor);
    candidates.push(ancestor);
  }

  // Score division by ancestor level:
  // - parent (level 0): scoreDivider = 1 (full score)
  // - grandparent (level 1): scoreDivider = 2 (half score)
  // - great-grandparent+ (level 2+): scoreDivider = level * 3
  if (level === 0) {
    var scoreDivider = 1;
  } else if (level === 1) {
    var scoreDivider = 2;
  } else {
    scoreDivider = level * 3;
  }

  ancestor.readability.contentScore += contentScore / scoreDivider;
});
```

### Propagation Multipliers
- **Direct parent (level 0)**: Receives 100% of child's score (`scoreDivider = 1`)
- **Grandparent (level 1)**: Receives 50% of child's score (`scoreDivider = 2`)
- **Great-grandparent (level 2)**: Receives ~33% of child's score (`scoreDivider = 6`)
- **Great-great-grandparent (level 3)**: Receives ~25% of child's score (`scoreDivider = 9`)

This system ensures that containers with multiple high-scoring children accumulate higher scores while giving diminishing returns for deeper nesting.

### Ancestor Traversal
The algorithm examines up to 5 ancestor levels using [`_getNodeAncestors(elementToScore, 5)`](../Readability.js:1019):

```javascript
_getNodeAncestors(node, maxDepth) {
  maxDepth = maxDepth || 0;
  var i = 0, ancestors = [];
  while (node.parentNode) {
    ancestors.push(node.parentNode);
    if (maxDepth && ++i === maxDepth) {
      break;
    }
    node = node.parentNode;
  }
  return ancestors;
}
```

## 5. Top Candidate Selection

After all elements are scored, the algorithm selects the best candidate through a multi-step process:

### Link Density Adjustment
Each candidate's score is adjusted based on its link density:

```javascript
var candidateScore = candidate.readability.contentScore * (1 - this._getLinkDensity(candidate));
```

The [`_getLinkDensity()`](../Readability.js:2127) method calculates the ratio of link text to total text:

```javascript
_getLinkDensity(element) {
  var textLength = this._getInnerText(element).length;
  if (textLength === 0) {
    return 0;
  }

  var linkLength = 0;
  this._forEachNode(element.getElementsByTagName("a"), function(linkNode) {
    var href = linkNode.getAttribute("href");
    // Hash links get reduced weight (0.3x multiplier)
    var coefficient = href && this.REGEXPS.hashUrl.test(href) ? 0.3 : 1;
    linkLength += this._getInnerText(linkNode).length * coefficient;
  });

  return linkLength / textLength;
}
```

**Key behaviors:**
- **Hash links** (e.g., `#section1`) get reduced weight (30% instead of 100%)
- **High link density** (lots of links) reduces the element's final score
- **Low link density** (mostly text) preserves the element's score

### Candidate Ranking
Candidates are ranked by their adjusted scores and stored in a [`topCandidates`](../Readability.js:1288) array:

```javascript
for (var t = 0; t < this._nbTopCandidates; t++) {
  var aTopCandidate = topCandidates[t];

  if (!aTopCandidate || candidateScore > aTopCandidate.readability.contentScore) {
    topCandidates.splice(t, 0, candidate);
    if (topCandidates.length > this._nbTopCandidates) {
      topCandidates.pop();
    }
    break;
  }
}
```

The algorithm maintains the top 5 candidates by default ([`DEFAULT_N_TOP_CANDIDATES`](../Readability.js:125)).

### Final Selection
The highest-scoring candidate becomes the [`topCandidate`](../Readability.js:1318):

```javascript
var topCandidate = topCandidates[0] || null;
```

If no suitable candidate is found, the algorithm falls back to creating a new container with the entire page body.

## Edge Cases and Fallbacks

### Alternative Candidate Detection
The algorithm looks for alternative candidates that might be better containers:

```javascript
// Find alternative candidates with scores >= 75% of top candidate
if (topCandidates[i].readability.contentScore / topCandidate.readability.contentScore >= 0.75) {
  alternativeCandidateAncestors.push(this._getNodeAncestors(topCandidates[i]));
}
```

If 3 or more alternatives share a common ancestor, that ancestor may be promoted to top candidate.

### Parent Score Climbing
The algorithm checks if a parent element has a higher score than the current top candidate:

```javascript
var parentScore = parentOfTopCandidate.readability.contentScore;
if (parentScore > lastScore) {
  topCandidate = parentOfTopCandidate; // Promote parent
}
```

This helps capture content that might be split across multiple containers.

### Single Child Optimization
If the top candidate is the only child of its parent, the parent becomes the new top candidate:

```javascript
while (parentOfTopCandidate.tagName != "BODY" && parentOfTopCandidate.children.length == 1) {
  topCandidate = parentOfTopCandidate;
  parentOfTopCandidate = topCandidate.parentNode;
}
```

This prevents overly narrow content selection.

## Summary

The content scoring and candidate selection system uses a sophisticated multi-factor approach:

1. **Base scoring** rewards paragraph-like elements with meaningful content
2. **Heuristic scoring** analyzes text characteristics (length, punctuation)
3. **Semantic scoring** leverages HTML class/ID attributes
4. **Hierarchical scoring** propagates scores through the DOM tree
5. **Link density adjustment** penalizes navigation-heavy content
6. **Competitive ranking** selects the best candidate from multiple options

This comprehensive approach enables Readability to identify main article content across a wide variety of website layouts and content management systems.
