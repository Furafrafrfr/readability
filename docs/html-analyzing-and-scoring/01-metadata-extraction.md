# Metadata Extraction Analysis in Readability.js

This document provides a detailed analysis of the metadata extraction process in Readability.js, covering the three main methods responsible for extracting article metadata from HTML documents.

## Overview

Readability.js extracts metadata through three complementary approaches:
1. **JSON-LD structured data** (`_getJSONLD()`) - Modern semantic markup
2. **HTML meta tags** (`_getArticleMetadata()`) - Traditional metadata sources
3. **Document title heuristics** (`_getArticleTitle()`) - Fallback title extraction

The extraction process prioritizes structured data (JSON-LD) over meta tags, with intelligent fallbacks and content validation.

## 1. JSON-LD Extraction (`_getJSONLD()`)

### Purpose
Extracts structured metadata from JSON-LD `<script>` tags that follow Schema.org Article specifications.

### Process Flow

#### 1.1 Script Tag Discovery
```javascript
var scripts = this._getAllNodesWithTag(doc, ["script"]);
```
- Finds all `<script>` elements in the document
- Filters for `type="application/ld+json"` attribute

#### 1.2 Content Processing
```javascript
var content = jsonLdElement.textContent.replace(/^\s*<!\[CDATA\[|\]\]>\s*$/g, "");
var parsed = JSON.parse(content);
```
- Strips CDATA markers (`<![CDATA[...]]>`) if present
- Parses JSON content, with error handling for malformed JSON

#### 1.3 Array Handling
```javascript
if (Array.isArray(parsed)) {
  parsed = parsed.find(it => {
    return it["@type"] && it["@type"].match(this.REGEXPS.jsonLdArticleTypes);
  });
}
```
- Handles JSON-LD arrays by finding the first Article-type object
- Uses regex pattern to match valid article types:
  - `Article`, `AdvertiserContentArticle`, `NewsArticle`
  - `AnalysisNewsArticle`, `AskPublicNewsArticle`, `BackgroundNewsArticle`
  - `OpinionNewsArticle`, `ReportageNewsArticle`, `ReviewNewsArticle`
  - `Report`, `SatiricalArticle`, `ScholarlyArticle`, `MedicalScholarlyArticle`
  - `SocialMediaPosting`, `BlogPosting`, `LiveBlogPosting`
  - `DiscussionForumPosting`, `TechArticle`, `APIReference`

#### 1.4 Schema.org Validation
```javascript
var schemaDotOrgRegex = /^https?\:\/\/schema\.org\/?$/;
var matches = (typeof parsed["@context"] === "string" &&
               parsed["@context"].match(schemaDotOrgRegex)) ||
              (typeof parsed["@context"] === "object" &&
               typeof parsed["@context"]["@vocab"] == "string" &&
               parsed["@context"]["@vocab"].match(schemaDotOrgRegex));
```
- Validates that the JSON-LD uses Schema.org context
- Handles both string and object context formats

#### 1.5 Graph Structure Handling
```javascript
if (!parsed["@type"] && Array.isArray(parsed["@graph"])) {
  parsed = parsed["@graph"].find(it => {
    return (it["@type"] || "").match(this.REGEXPS.jsonLdArticleTypes);
  });
}
```
- Handles JSON-LD Graph structures where data is nested in `@graph` array

#### 1.6 Property Extraction

##### Title Extraction (with Conflict Resolution)
```javascript
if (typeof parsed.name === "string" &&
    typeof parsed.headline === "string" &&
    parsed.name !== parsed.headline) {

  var title = this._getArticleTitle();
  var nameMatches = this._textSimilarity(parsed.name, title) > 0.75;
  var headlineMatches = this._textSimilarity(parsed.headline, title) > 0.75;

  if (headlineMatches && !nameMatches) {
    metadata.title = parsed.headline;
  } else {
    metadata.title = parsed.name;
  }
}
```
- Handles conflicts between `name` and `headline` properties
- Uses text similarity comparison with document title (75% threshold)
- Prefers `headline` if it better matches the document title

##### Author Extraction
```javascript
if (parsed.author) {
  if (typeof parsed.author.name === "string") {
    metadata.byline = parsed.author.name.trim();
  } else if (Array.isArray(parsed.author) && parsed.author[0] &&
             typeof parsed.author[0].name === "string") {
    metadata.byline = parsed.author
      .filter(function (author) {
        return author && typeof author.name === "string";
      })
      .map(function (author) {
        return author.name.trim();
      })
      .join(", ");
  }
}
```
- Handles both single author objects and author arrays
- Joins multiple authors with commas
- Validates author objects have valid name properties

##### Other Properties
- **Description**: `parsed.description` → `metadata.excerpt`
- **Publisher**: `parsed.publisher.name` → `metadata.siteName`
- **Date Published**: `parsed.datePublished` → `metadata.datePublished`

## 2. HTML Meta Tag Extraction (`_getArticleMetadata()`)

### Purpose
Extracts metadata from HTML `<meta>` tags including Open Graph, Dublin Core, Twitter Cards, and standard metadata.

### Metadata Source Patterns

#### 2.1 Property Pattern (Space-separated values)
```javascript
var propertyPattern = /\s*(article|dc|dcterm|og|twitter)\s*:\s*(author|creator|description|published_time|title|site_name)\s*/gi;
```
Matches meta tags with `property` attribute containing:
- **Open Graph**: `og:title`, `og:description`, `og:site_name`
- **Dublin Core**: `dc:title`, `dc:creator`, `dc:description`
- **Dublin Core Terms**: `dcterm:title`, `dcterm:creator`, `dcterm:description`
- **Article**: `article:author`, `article:published_time`
- **Twitter**: `twitter:title`, `twitter:description`

#### 2.2 Name Pattern (Single values)
```javascript
var namePattern = /^\s*(?:(dc|dcterm|og|twitter|parsely|weibo:(article|webpage))\s*[-\.:]\s*)?(author|creator|pub-date|description|title|site_name)\s*$/i;
```
Matches meta tags with `name` attribute containing:
- Standard names: `author`, `description`, `title`
- Prefixed names: `parsely-author`, `parsely-title`, `parsely-pub-date`
- Weibo formats: `weibo:article:title`, `weibo:webpage:title`

### Extraction Process

#### 2.3 Meta Tag Processing
```javascript
this._forEachNode(metaElements, function (element) {
  var elementName = element.getAttribute("name");
  var elementProperty = element.getAttribute("property");
  var content = element.getAttribute("content");

  if (!content) return;

  // Process property attribute
  if (elementProperty) {
    matches = elementProperty.match(propertyPattern);
    if (matches) {
      name = matches[0].toLowerCase().replace(/\s/g, "");
      values[name] = content.trim();
    }
  }

  // Process name attribute
  if (!matches && elementName && namePattern.test(elementName)) {
    name = elementName.toLowerCase().replace(/\s/g, "").replace(/\./g, ":");
    values[name] = content.trim();
  }
});
```

#### 2.4 Priority-based Property Selection

##### Title Priority
```javascript
metadata.title = jsonld.title ||
  values["dc:title"] ||
  values["dcterm:title"] ||
  values["og:title"] ||
  values["weibo:article:title"] ||
  values["weibo:webpage:title"] ||
  values.title ||
  values["twitter:title"] ||
  values["parsely-title"];
```

##### Author Priority
```javascript
metadata.byline = jsonld.byline ||
  values["dc:creator"] ||
  values["dcterm:creator"] ||
  values.author ||
  values["parsely-author"] ||
  articleAuthor; // article:author (if not URL)
```

##### Description Priority
```javascript
metadata.excerpt = jsonld.excerpt ||
  values["dc:description"] ||
  values["dcterm:description"] ||
  values["og:description"] ||
  values["weibo:article:description"] ||
  values["weibo:webpage:description"] ||
  values.description ||
  values["twitter:description"];
```

#### 2.5 Special Handling

##### Article Author Validation
```javascript
const articleAuthor = typeof values["article:author"] === "string" &&
                     !this._isUrl(values["article:author"])
                     ? values["article:author"]
                     : undefined;
```
- Validates that `article:author` is not a URL before using

#### 2.6 HTML Entity Unescaping
```javascript
metadata.title = this._unescapeHtmlEntities(metadata.title);
metadata.byline = this._unescapeHtmlEntities(metadata.byline);
metadata.excerpt = this._unescapeHtmlEntities(metadata.excerpt);
metadata.siteName = this._unescapeHtmlEntities(metadata.siteName);
metadata.publishedTime = this._unescapeHtmlEntities(metadata.publishedTime);
```
Handles common HTML entities: `&quot;`, `&amp;`, `&apos;`, `&lt;`, `&gt;`, and numeric character references.

## 3. Article Title Extraction (`_getArticleTitle()`)

### Purpose
Determines the best article title using document title, hierarchical separators, and heading tags as fallbacks.

### Process Flow

#### 3.1 Initial Title Extraction
```javascript
try {
  curTitle = origTitle = doc.title.trim();

  if (typeof curTitle !== "string") {
    curTitle = origTitle = this._getInnerText(doc.getElementsByTagName("title")[0]);
  }
} catch (e) {
  /* ignore exceptions */
}
```

#### 3.2 Hierarchical Separator Handling
```javascript
const titleSeparators = /\|\-–—\\\/>»/.source;
if (new RegExp(`\\s[${titleSeparators}]\\s`).test(curTitle)) {
  titleHadHierarchicalSeparators = /\s[\\\/>»]\s/.test(curTitle);
  let allSeparators = Array.from(origTitle.matchAll(new RegExp(`\\s[${titleSeparators}]\\s`, "gi")));
  curTitle = origTitle.substring(0, allSeparators.pop().index);

  // If resulting title too short, remove first part instead
  if (wordCount(curTitle) < 3) {
    curTitle = origTitle.replace(new RegExp(`^[^${titleSeparators}]*[${titleSeparators}]`, "gi"), "");
  }
}
```

**Supported Separators**: `|`, `-`, `–`, `—`, `\`, `/`, `>`, `»`

**Logic**:
- Removes the last separator and everything after it (typically site name)
- If result has fewer than 3 words, removes first part instead (typically category)
- Tracks whether "hierarchical" separators (`\`, `/`, `>`, `»`) were found

#### 3.3 Colon Separator Handling
```javascript
else if (curTitle.includes(": ")) {
  var headings = this._getAllNodesWithTag(doc, ["h1", "h2"]);
  var trimmedTitle = curTitle.trim();
  var match = this._someNode(headings, function (heading) {
    return heading.textContent.trim() === trimmedTitle;
  });

  if (!match) {
    curTitle = origTitle.substring(origTitle.lastIndexOf(":") + 1);

    if (wordCount(curTitle) < 3) {
      curTitle = origTitle.substring(origTitle.indexOf(":") + 1);
    } else if (wordCount(origTitle.substr(0, origTitle.indexOf(":"))) > 5) {
      curTitle = origTitle; // Too many words before colon, use original
    }
  }
}
```

**Process**:
- Checks if any H1/H2 heading matches the full title
- If no match, removes everything before the last colon
- If result too short, uses content after first colon instead
- If too many words before first colon (>5), keeps original title

#### 3.4 Length-based Fallback
```javascript
else if (curTitle.length > 150 || curTitle.length < 15) {
  var hOnes = doc.getElementsByTagName("h1");

  if (hOnes.length === 1) {
    curTitle = this._getInnerText(hOnes[0]);
  }
}
```

Falls back to single H1 tag if title is too long (>150 chars) or too short (<15 chars).

#### 3.5 Final Validation
```javascript
curTitle = curTitle.trim().replace(this.REGEXPS.normalize, " ");

var curTitleWordCount = wordCount(curTitle);
if (curTitleWordCount <= 4 &&
    (!titleHadHierarchicalSeparators ||
     curTitleWordCount != wordCount(origTitle.replace(new RegExp(`\\s[${titleSeparators}]\\s`, "g"), "")) - 1)) {
  curTitle = origTitle;
}
```

**Final Check**:
- Normalizes whitespace
- If result has ≤4 words AND either:
  - No hierarchical separators were found, OR
  - Word count reduction is more than 1 word
- Then reverts to original title

## Integration and Priority

The three methods work together with this priority system:

1. **JSON-LD metadata** (highest priority)
2. **HTML meta tags** (medium priority)
3. **Document title heuristics** (fallback)

The `parse()` method orchestrates this process:

```javascript
// Extract JSON-LD first
var jsonLd = this._disableJSONLD ? {} : this._getJSONLD(this._doc);

// Then extract meta tags, passing JSON-LD as priority source
var metadata = this._getArticleMetadata(jsonLd);

// Set article title from metadata or fallback to heuristics
this._articleTitle = metadata.title;
```

This layered approach ensures robust metadata extraction across different website implementations and content management systems.
