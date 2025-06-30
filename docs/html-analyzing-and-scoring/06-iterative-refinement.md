# Iterative Refinement and Retry Logic

This document analyzes the iterative refinement and retry logic within the [`_grabArticle()`](../../Readability.js:1041) method in `Readability.js`. This sophisticated retry mechanism ensures that content extraction succeeds even when the initial attempt yields insufficient content.

## Overview

The [`_grabArticle()`](../../Readability.js:1041) method implements a progressive relaxation strategy. If the first attempt at content extraction produces text that is too short, the algorithm systematically disables various content filtering flags and retries the extraction process. This approach increases the likelihood of finding sufficient content while maintaining a preference for higher-quality extraction when possible.

## The Trigger Condition

### Length Check Implementation

The retry mechanism is triggered by checking the length of the extracted content against a configurable threshold:

```javascript
// Line 1555-1556
var textLength = this._getInnerText(articleContent, true).length;
if (textLength < this._charThreshold) {
    parseSuccessful = false;
    // ... retry logic
}
```

### Key Components

1. **Text Length Calculation**: Uses [`_getInnerText(articleContent, true)`](../../Readability.js:2068) to get normalized text content
2. **Threshold Comparison**: Compares against [`this._charThreshold`](../../Readability.js:54) (default: 500 characters)
3. **Retry Trigger**: When content is insufficient, `parseSuccessful` is set to `false`, triggering the retry mechanism

## Flag-Based Relaxation Strategy

The retry logic systematically disables three content filtering flags in a specific order, each retry attempt becoming more permissive:

### 1. FLAG_STRIP_UNLIKELYS

**Purpose**: Controls removal of elements with suspicious class names and IDs.

**When Active** (lines 1127-1139):
- Elements matching [`REGEXPS.unlikelyCandidates`](../../Readability.js:140) are removed
- Pattern includes: `-ad-|banner|comment|sidebar|footer|menu|related|sponsor|popup`
- Exception: Elements also matching [`REGEXPS.okMaybeItsACandidate`](../../Readability.js:142) are preserved
- Exception pattern: `and|article|body|column|content|main|shadow`

**When Disabled** (retry attempt):
- Previously discarded elements are now considered for content extraction
- Elements with class names like `comment`, `sidebar`, `related` become eligible
- Significantly increases the pool of candidate elements

**Impact**: Allows inclusion of content that might be legitimate but has misleading class names (e.g., user comments, related articles, sidebar content that contains main article text).

### 2. FLAG_WEIGHT_CLASSES

**Purpose**: Controls class and ID-based scoring adjustments.

**When Active** (lines 2153-2181 in [`_getClassWeight()`](../../Readability.js:2152)):
```javascript
// Negative scoring for suspicious classes/IDs
if (this.REGEXPS.negative.test(e.className)) {
    weight -= 25;
}
// Positive scoring for content-indicative classes/IDs
if (this.REGEXPS.positive.test(e.className)) {
    weight += 25;
}
```

**When Disabled** (retry attempt):
- Class and ID names no longer influence content scoring
- [`_getClassWeight()`](../../Readability.js:2152) returns 0 for all elements
- Scoring becomes purely based on element type and content characteristics

**Impact**: Neutralizes bias against elements with negative class names (like `ad-container` that might contain legitimate content) and eliminates preference for elements with positive class names.

### 3. FLAG_CLEAN_CONDITIONALLY

**Purpose**: Controls aggressive post-processing cleanup of extracted content.

**When Active** (lines 2444-2641 in [`_cleanConditionally()`](../../Readability.js:2444)):
- Removes elements based on complex heuristics:
  - Link density ratios
  - Image-to-paragraph ratios
  - Content length thresholds
  - Embed counts
  - Text density calculations

**When Disabled** (retry attempt):
- [`_cleanConditionally()`](../../Readability.js:2444) exits early without processing
- Elements that would normally be removed as "low quality" are preserved
- More permissive approach to content inclusion

**Impact**: Prevents removal of elements that might be legitimate content but fail the quality heuristics (e.g., short paragraphs, content with higher link density, elements with unusual structure).

## The Retry Process

### DOM State Management

```javascript
// Line 1053: Cache original DOM state
var pageCacheHtml = page.innerHTML;

// Line 1559: Reset DOM for retry
page.innerHTML = pageCacheHtml;
```

### Attempt Tracking

```javascript
// Lines 1561-1564: Store each attempt
this._attempts.push({
    articleContent,
    textLength,
});
```

### Progressive Flag Removal

```javascript
// Lines 1566-1585: Sequential flag removal
if (this._flagIsActive(this.FLAG_STRIP_UNLIKELYS)) {
    this._removeFlag(this.FLAG_STRIP_UNLIKELYS);
} else if (this._flagIsActive(this.FLAG_WEIGHT_CLASSES)) {
    this._removeFlag(this.FLAG_WEIGHT_CLASSES);
} else if (this._flagIsActive(this.FLAG_CLEAN_CONDITIONALLY)) {
    this._removeFlag(this.FLAG_CLEAN_CONDITIONALLY);
} else {
    // All flags exhausted - return best attempt
    this._attempts.sort(function (a, b) {
        return b.textLength - a.textLength;
    });
    articleContent = this._attempts[0].articleContent;
    parseSuccessful = true;
}
```

### Retry Sequence

1. **Initial Attempt**: All flags active (most restrictive)
2. **Retry 1**: Disable `FLAG_STRIP_UNLIKELYS` (include previously "unlikely" elements)
3. **Retry 2**: Disable `FLAG_WEIGHT_CLASSES` (ignore class-based scoring)
4. **Retry 3**: Disable `FLAG_CLEAN_CONDITIONALLY` (skip aggressive cleanup)
5. **Final Fallback**: Return the longest content found across all attempts

## Implementation Details

### Flag State Management

```javascript
// Initial state (line 69-72)
this._flags = this.FLAG_STRIP_UNLIKELYS |
              this.FLAG_WEIGHT_CLASSES |
              this.FLAG_CLEAN_CONDITIONALLY;

// Flag checking
_flagIsActive(flag) {
    return (this._flags & flag) > 0;
}

// Flag removal
_removeFlag(flag) {
    this._flags = this._flags & ~flag;
}
```

### Loop Structure

The entire retry mechanism is wrapped in a `while (true)` loop (line 1055) that continues until either:
- Sufficient content is found (`parseSuccessful = true`)
- All retry attempts are exhausted

This ensures the algorithm will always return some result, even if suboptimal.

## Strategic Rationale

The progressive relaxation strategy embodies several key principles:

1. **Quality Preference**: Attempts to extract high-quality content first
2. **Graceful Degradation**: Systematically reduces restrictions when high-quality extraction fails
3. **Comprehensive Coverage**: Ensures some content is always returned
4. **Minimal Impact**: Each retry only disables one restriction at a time

This approach is particularly effective for handling edge cases like:
- Articles with unconventional DOM structure
- Content management systems with misleading class names
- Sites where legitimate content is mixed with elements that appear to be advertisements or navigation
- Documents with minimal text content that still contains the main article

The retry logic transforms a potentially brittle extraction process into a robust system that can handle a wide variety of web page structures while maintaining content quality when possible.
