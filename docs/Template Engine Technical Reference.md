# Template Engine Technical Reference for Advanced Template Creators

## Introduction

This technical reference provides comprehensive documentation for creating advanced templates in Obsidian Web Clipper. It targets highly technical users who need deep understanding of the template system's capabilities, composition patterns, and deterministic extraction strategies.

The template engine transforms user-defined templates into structured notes by processing variables, filters, and logic constructs in a predictable two-pass compilation pipeline. Understanding these internals enables you to build robust templates that handle diverse web structures reliably.

---

## Template Compilation Lifecycle

### How Templates Are Processed

Every template field (note name, vault path, note content, and property values) undergoes identical compilation:

1. **Logic Pass**: All logic blocks (`{% for ... %}...{% endfor %}`) are expanded first, creating scoped variable contexts for iteration
2. **Variable Pass**: All `{{expression}}` placeholders are resolved and filters are applied sequentially

This two-pass architecture ensures loops expand before variable substitution, allowing loop bodies to reference both global variables and iterator-local variables.

### URL Normalization

Text fragment anchors (`#:~:text=...`) are stripped from URLs at compilation time, ensuring renders remain stable across page scroll states. This normalization happens once at the template compiler boundary.

---

## Core Syntax Elements

### Mustache Expressions

Any `{{expression}}` is intercepted and dispatched based on prefixes or content:

```
{{variable}}              → Simple variable lookup
{{variable.nested.path}}  → Nested property access
{{variable[0].name}}      → Array index access
{{selector:CSS}}          → DOM element extraction
{{selectorHtml:CSS}}      → DOM element HTML extraction
{{schema:@Type:key}}      → Schema.org data access
{{meta:name:description}} → Meta tag content
{{"prompt text"}}         → AI-powered extraction (requires Interpreter)
{{variable|filter}}       → Apply filter to variable
{{variable|filter1|filter2}} → Chain multiple filters
```

**Key behaviors**:
- Undefined variables resolve to empty strings (no errors)
- Objects are JSON-stringified automatically
- Filter chains process left-to-right
- Whitespace inside `{{...}}` is trimmed

### Loop Blocks

```
{% for item in source %}
  {{item.property}}
  {{item.nested[0].value}}
{% endfor %}
```

**Supported sources**:
- Simple variable names: `{% for tag in tags %}`
- Schema variables: `{% for ingredient in schema:@Recipe.ingredients %}`
- Nested paths: `{% for author in book.authors %}`

**Loop behavior**:
- Source must be an array; non-arrays collapse to empty string
- Each iteration clones the variable map and adds the iterator variable
- Nested loops are fully supported
- Loop bodies can contain additional logic blocks (recursively processed)
- Global variables remain accessible inside loops

**Important**: The iterator variable is added **without** curly braces. Reference it as `{{item}}`, not `{{{{item}}}}`.

---

## Variable System

### Preset Variables

Automatically populated from page metadata:

| Variable | Type | Description |
|----------|------|-------------|
| `{{author}}` | string | Page author |
| `{{content}}` | string | Main content in Markdown (article, selection, or highlights) |
| `{{contentHtml}}` | string | Main content in HTML |
| `{{selection}}` | string | Selected text in Markdown |
| `{{selectionHtml}}` | string | Selected text in HTML |
| `{{date}}` | string | Current timestamp (ISO 8601) |
| `{{time}}` | string | Current timestamp (ISO 8601, identical to `date`) |
| `{{description}}` | string | Page description/excerpt |
| `{{domain}}` | string | Domain name |
| `{{favicon}}` | string | Favicon URL |
| `{{fullHtml}}` | string | Complete page HTML (unprocessed) |
| `{{highlights}}` | JSON array | Highlight objects with `text`, `timestamp`, and optional `notes` |
| `{{image}}` | string | Social share image URL |
| `{{noteName}}` | string | Sanitized filename derived from title |
| `{{published}}` | string | Published date |
| `{{site}}` | string | Site name or publisher |
| `{{title}}` | string | Page title |
| `{{url}}` | string | Current page URL (normalized, without text fragments) |
| `{{words}}` | string | Word count |

**Highlights structure**:
```json
[
  {
    "text": "Highlighted text in Markdown",
    "timestamp": "2024-01-15T10:30:00.000Z",
    "notes": ["User note 1", "User note 2"]
  }
]
```

The `notes` property only appears if the user added notes to the highlight.

### Meta Variables

Extract data from HTML `<meta>` tags:

```
{{meta:name:description}}           → <meta name="description" content="...">
{{meta:name:keywords}}              → <meta name="keywords" content="...">
{{meta:property:og:title}}          → <meta property="og:title" content="...">
{{meta:property:og:image}}          → <meta property="og:image" content="...">
{{meta:property:article:author}}    → <meta property="article:author" content="...">
```

**Syntax**:
- `{{meta:name:NAME}}` for `name` attribute
- `{{meta:property:PROPERTY}}` for `property` attribute (common in Open Graph)

### Selector Variables

Extract content from the live DOM using CSS selectors:

```
{{selector:h1}}                     → Text of first <h1>
{{selector:.author-name}}           → Text of first .author-name
{{selector:a.download-link?href}}   → href attribute of first matching link
{{selector:img.hero?src}}           → src attribute of first .hero image
{{selectorHtml:article}}            → HTML of first <article> element
{{selectorHtml:.content}}           → HTML of first .content element
```

**Attribute extraction**:
Use `?attribute` to extract a specific attribute instead of text content:
- `?href`, `?src`, `?alt`, `?title`, `?data-id`, etc.

**Multiple matches**:
When multiple elements match, the selector returns a JSON array:
```
{{selector:.tag}}  →  ["Design", "Engineering", "Product"]
```

You can then process this array with filters:
```
{{selector:.tag|join:", "}}         → "Design, Engineering, Product"
{{selector:.tag|map:item => item|list}} → Markdown bullet list
{{selector:article p|html_to_json|map:item => item.content|join:"\n\n"}}
```

**HTML selectors**:
`{{selectorHtml:...}}` returns the full HTML (`outerHTML`), not just text. Pipe to `markdown` filter to convert:
```
{{selectorHtml:main|markdown}}
```

**Complex selectors**:
All CSS selector syntax is supported:
```
{{selector:div.post > p:first-child}}
{{selector:article[data-type="news"]}}
{{selector:nav ul > li:nth-child(2) a?href}}
```

### Schema.org Variables

Access structured data from JSON-LD:

**Full notation**:
```
{{schema:@Recipe:name}}              → Recipe name
{{schema:@Recipe:description}}       → Recipe description
{{schema:@Recipe:author.name}}       → Nested property access
{{schema:@Recipe:ingredients[0]}}    → First ingredient
{{schema:@Recipe:ingredients[2].name}} → Name of third ingredient
{{schema:@Recipe:ingredients[*].name}} → All ingredient names (array)
```

**Shorthand notation** (auto-resolves `@Type`):
```
{{schema:name}}                      → Finds first "name" in any schema type
{{schema:author}}                    → Finds first "author"
{{schema:author.name}}               → Nested access with auto-resolution
{{schema:ingredients[0].name}}       → Array access with auto-resolution
```

**Array splat syntax**:
The `[*]` operator extracts a property from every array element:
```
{{schema:@Recipe:ingredients[*].name}}
→ Returns JSON array: ["flour", "sugar", "eggs"]
```

This array can be immediately filtered:
```
{{schema:@Recipe:ingredients[*].name|list}}
→ - flour
→ - sugar
→ - eggs
```

**Data types**:
- Scalar values (strings, numbers, booleans) are returned as strings
- Arrays are returned as JSON
- Objects are returned as JSON
- Nested structures preserve their hierarchy

**List coercion**:
If schema data is stored as a plain-text list (numbered or bulleted), the engine automatically splits it into a JSON array:
```
1. Item one
2. Item two
→ Becomes: ["Item one", "Item two"]
```

### Prompt Variables

Use natural language to extract data via AI models (requires Interpreter to be enabled):

```
{{"a summary of the page"}}
{{"extract the author's name"}}
{{"list of tags as JSON array"}}
{{"three bullet points about the article"|blockquote}}
{{"return JSON with keys: title, date, author"}}
```

**Key characteristics**:
- Double quotes are mandatory: `{{"prompt"}}`, not `{{prompt}}`
- Processed only when Interpreter runs (async, after initial render)
- Can be filtered like any variable: `{{"summary"|trim|capitalize}}`
- Can output JSON for downstream filter processing
- Alternative syntax: `{{prompt:"text"}}` (identical behavior)

**When to use**:
- Content varies structurally across sites (no consistent DOM or schema)
- Fuzzy extraction requirements (e.g., "main idea", "sentiment")
- Translation or transformation tasks

**When NOT to use**:
- Deterministic extraction is possible via selectors or schema
- Performance is critical (prompts add latency)
- Privacy is a concern (data sent to external LLM)

---

## Filter System

### Filter Application

Filters modify variable output using the pipe operator:
```
{{variable|filter}}
{{variable|filter:param}}
{{variable|filter:("param1", "param2")}}
{{variable|filter1|filter2|filter3}}
```

**Pipeline behavior**:
- Filters execute sequentially left-to-right
- Each filter receives the previous output as input
- If output is JSON-parsable, it's automatically parsed for the next filter
- Final output is always stringified

**Automatic JSON propagation**:
Filters that produce structured data (arrays, objects) emit JSON strings. The next filter in the chain auto-parses these, allowing seamless composition:
```
{{selector:.tag|split:","|map:item => item|join:" · "}}
     ↓ JSON array     ↓ array      ↓ array    ↓ string
```

### Context Injection

Certain filters receive the current URL automatically:
- `markdown` (for resolving relative URLs in links/images)
- `fragment_link` (for generating text fragment links)

You don't need to pass the URL explicitly unless overriding.

### Comprehensive Filter Reference

#### Date and Time

**`date`**: Format timestamps
```
{{date|date:"YYYY-MM-DD"}}
{{published|date:"MMM D, YYYY"}}
"2024-01-15"|date:("YYYY-MM-DD", "MM/DD/YYYY")  → Parse with input format
```
[Day.js format reference](https://day.js.org/docs/en/display/format)

**`date_modify`**: Add/subtract time
```
{{date|date_modify:"+1 year"}}
{{published|date_modify:"-2 months"}}
{{date|date_modify:"+3 days"|date:"YYYY-MM-DD"}}
```

**`duration`**: Format ISO 8601 durations or seconds
```
"PT1H30M"|duration:"HH:mm:ss"           → "01:30:00"
"3665"|duration:"H:mm:ss"               → "1:01:05"
"PT6702S"|duration                      → "1:51:42" (auto-format)
```
Tokens: `HH` (padded hours), `H` (hours), `mm` (padded minutes), `m`, `ss`, `s`

#### Text Transformation

**`camel`**: Convert to camelCase
```
"hello world"|camel  → "helloWorld"
```

**`capitalize`**: Capitalize first character, lowercase rest
```
"hELLO wORLD"|capitalize  → "Hello world"
```

**`kebab`**: Convert to kebab-case
```
"Hello World"|kebab  → "hello-world"
```

**`lower`**: Lowercase
```
"HELLO"|lower  → "hello"
```

**`pascal`**: Convert to PascalCase
```
"hello world"|pascal  → "HelloWorld"
```

**`snake`**: Convert to snake_case
```
"Hello World"|snake  → "hello_world"
```

**`title`**: Title Case
```
"hello world"|title  → "Hello World"
```

**`uncamel`**: Convert camelCase to spaces
```
"camelCase"|uncamel         → "camel case"
"PascalCase"|uncamel|title  → "Pascal Case"
```

**`upper`**: Uppercase
```
"hello"|upper  → "HELLO"
```

**`trim`**: Remove leading/trailing whitespace
```
"  hello  "|trim  → "hello"
```

**`replace`**: Text replacement (supports regex)
```
"hello!"|replace:",":""                         → Remove commas
"hello world"|replace:("e":"a","o":"0")         → Multiple replacements: "hall0 w0rld"
"hello world"|replace:"/[aeiou]/g":"*"          → Regex: "h*ll* w*rld"
"HELLO world"|replace:"/hello/i":"hi"           → Case-insensitive: "hi world"
"hello world"|replace:("/[aeiou]/g":"*","/\s+/":"-")  → Chain: "h*ll*-w*rld"
```

Escape special chars (`:`, `|`, `{`, `}`, `(`, `)`, `'`, `"`) with backslash in search terms.

**`safe_name`**: Convert to safe filename
```
"My Note: Draft #1"|safe_name              → "My Note Draft 1"
"File/Name"|safe_name:windows              → OS-specific rules
```
OS options: `windows`, `mac`, `linux`

**`unescape`**: Decode escape sequences
```
"Hello\\nWorld"|unescape  → "Hello\nWorld" (literal newline)
```

#### Markdown Formatting

**`blockquote`**: Add `> ` prefix
```
"Line 1\nLine 2"|blockquote
→ > Line 1
→ > Line 2
```

**`callout`**: Create Obsidian callout
```
"Important note"|callout
"Warning text"|callout:("warning", "Watch out!", true)
  → type: "warning", title: "Watch out!", folded: true
```

**`footnote`**: Generate footnotes
```
["First note", "Second note"]|footnote
→ [^1]: First note
→ [^2]: Second note

{"Key One": "Content", "Key Two": "More"}|footnote
→ [^key-one]: Content
→ [^key-two]: More
```

**`list`**: Create Markdown lists
```
["Item 1", "Item 2"]|list             → Bullet list
["Task A", "Task B"]|list:task        → Task list with checkboxes
["First", "Second"]|list:numbered     → Numbered list
["Do this", "Do that"]|list:numbered-task
```

**`table`**: Generate Markdown tables
```
[{"name": "Alice", "age": 30}, {"name": "Bob", "age": 25}]|table
["a", "b", "c", "d", "e", "f"]|table:("Col1", "Col2", "Col3")  → 3-column table
[["a", "b"], ["c", "d"]]|table:("X", "Y")                     → Custom headers
```

**`link`**: Create Markdown links
```
"https://example.com"|link:"Example"     → [Example](https://example.com)
["url1", "url2"]|link:"Link"             → Array of links with same text
{"url1": "Text1", "url2": "Text2"}|link  → Custom text per link
```

**`wikilink`**: Create Obsidian wikilinks
```
"Note Name"|wikilink                     → [[Note Name]]
"Note Name"|wikilink:"Alias"             → [[Note Name|Alias]]
["Page1", "Page2"]|wikilink              → [[[Page1]], [[Page2]]]
{"Page1": "Alias1", "Page2": "Alias2"}|wikilink
```

**`image`**: Create image Markdown
```
"image.jpg"|image:"Alt text"                    → ![Alt text](image.jpg)
["img1.jpg", "img2.jpg"]|image:"Alt"            → Array of images
{"img1.jpg": "Alt1", "img2.jpg": "Alt2"}|image
```

**`fragment_link`**: Create text fragment links
```
{{highlights|fragment_link}}
{{highlights|fragment_link:"jump"}}
```
Creates `#:~:text=...` links that scroll to and highlight the text.

#### HTML Processing

**`markdown`**: Convert HTML to Markdown
```
{{contentHtml|markdown}}
{{selectorHtml:.article|markdown}}
```
Automatically resolves relative URLs using page context.

**`strip_tags`**: Remove all HTML tags (keep content)
```
"<p>Hello <b>world</b>!</p>"|strip_tags              → "Hello world!"
"<p>Hello <b>world</b>!</p>"|strip_tags:("b")        → "Hello <b>world</b>!" (keep <b>)
```

**`remove_tags`**: Remove specific tags (keep content)
```
"<p>Hello <b>world</b>!</p>"|remove_tags:"b"         → "<p>Hello world!</p>"
"<p>text <em>italic</em></p>"|remove_tags:("em","strong")
```

**`replace_tags`**: Replace tag types
```
{{contentHtml|replace_tags:"strong":"h2"}}  → Replace <strong> with <h2>
```

**`strip_attr`**: Remove all attributes
```
"<div class='test' id='x'>Hi</div>"|strip_attr           → "<div>Hi</div>"
"<div class='test' id='x'>Hi</div>"|strip_attr:("id")    → "<div class='test'>Hi</div>" (keep id)
```

**`remove_attr`**: Remove specific attributes
```
"<div class='test' id='x'>Hi</div>"|remove_attr:"class"  → "<div id='x'>Hi</div>"
"<div class='test' id='x'>Hi</div>"|remove_attr:("class","id")
```

**`remove_html`**: Remove elements and their content
```
{{contentHtml|remove_html:"img"}}                    → Remove all images
{{contentHtml|remove_html:(".ad-banner","#popup")}}  → Remove by class/ID
```

**`strip_md`**: Remove Markdown formatting
```
"**bold** and *italic*"|strip_md  → "bold and italic"
```

**`html_to_json`**: Parse HTML into structured JSON
```
"<p>Hello</p><p>World</p>"|html_to_json
→ [
    {"type": "element", "tag": "p", "children": [{"type": "text", "content": "Hello"}]},
    {"type": "element", "tag": "p", "children": [{"type": "text", "content": "World"}]}
  ]
```

JSON structure:
- Text nodes: `{"type": "text", "content": "..."}`
- Elements: `{"type": "element", "tag": "...", "attributes": {...}, "children": [...]}`

#### Array and Object Manipulation

**`first`**: First element
```
["a", "b", "c"]|first  → "a"
```

**`last`**: Last element
```
["a", "b", "c"]|last  → "c"
```

**`nth`**: Extract nth elements (CSS nth-child syntax)
```
array|nth:3              → 3rd element only
array|nth:3n             → Every 3rd element (3, 6, 9, ...)
array|nth:n+3            → From 3rd onward
array|nth:1,2,3:5        → Positions 1,2,3 from each group of 5
```

Example:
```
[1,2,3,4,5,6,7,8,9,10]|nth:1,2,3:5  → [1,2,3,6,7,8]
```

**`reverse`**: Reverse order
```
[1,2,3]|reverse           → [3,2,1]
"hello"|reverse           → "olleh"
{"a":1,"b":2}|reverse     → {"b":2,"a":1}
```

**`slice`**: Extract portion
```
"hello"|slice:1,4         → "ell" (indices 1-3)
["a","b","c","d"]|slice:1,3  → ["b","c"]
"hello"|slice:2           → "llo" (from index 2 to end)
"hello"|slice:-3          → "llo" (last 3 characters)
"hello"|slice:0,-2        → "hel" (exclude last 2)
```

**`split`**: Split string into array
```
"a,b,c"|split:","         → ["a","b","c"]
"hello world"|split:" "   → ["hello","world"]
"hello"|split             → ["h","e","l","l","o"] (every character)
"a1b2c3"|split:/[0-9]/    → ["a","b","c"] (regex)
```

**`join`**: Join array into string
```
["a","b","c"]|join        → "a,b,c" (default comma)
["a","b","c"]|join:" "    → "a b c"
["a","b","c"]|join:"\n"   → "a\nb\nc" (line breaks)
```

**`unique`**: Remove duplicates
```
[1,2,2,3,3]|unique                    → [1,2,3]
[{"a":1},{"b":2},{"a":1}]|unique      → [{"a":1},{"b":2}]
```

**`merge`**: Add values to array
```
["a","b"]|merge:("c","d")    → ["a","b","c","d"]
["a","b"]|merge:"c"          → ["a","b","c"]
"a"|merge:("b","c")          → ["a","b","c"] (creates array)
["a"]|merge:('b,"c,d",e')    → ["a","b","c,d","e"] (quoted values)
```

**`map`**: Transform array elements
```
[{"name": "Alice"}, {"name": "Bob"}]|map:item => item.name
→ ["Alice", "Bob"]

array|map:item => ({key: item.value, other: item.nested.prop})
→ Array of transformed objects

["rock", "pop"]|map:item => "genres/${item}"
→ [{"str": "genres/rock"}, {"str": "genres/pop"}]
```

**Arrow function syntax**:
- `item => item.property` for simple property access
- `item => item.nested.deep.property` for nested paths
- `item => item.array[0].property` for array indexing
- `item => ({key: value, another: value})` for object literals (requires parentheses)
- `item => "text ${item}"` for string literals (produces `{str: "..."}` objects)

**Object literals**:
```
map:item => ({title: item.headline, url: item.link})
```
Produces:
```json
[{"title": "Headline 1", "url": "http://..."}, ...]
```

**String literals**:
```
map:item => "- ${item.name}"
```
Produces:
```json
[{"str": "- Item 1"}, {"str": "- Item 2"}]
```

Then use `template` to unwrap:
```
map:item => "- ${item.name}"|template:"${str}"
→ - Item 1
→ - Item 2
```

**`template`**: Apply template to objects
```
{"name": "Alice", "age": 30}|template:"${name} is ${age} years old"
→ "Alice is 30 years old"

[{"name": "Alice"}, {"name": "Bob"}]|template:"- ${name}\n"
→ - Alice
→ - Bob

array|template:"**${title}**\n${description}\n"
```

**Nested property access**:
```
{"person": {"name": "Alice"}}|template:"${person.name}"
```

**String literal unwrapping**:
```
map:item => "genres/${item}"|template:"${str}"
```

**`object`**: Object manipulation
```
{"a":1,"b":2}|object:keys      → ["a","b"]
{"a":1,"b":2}|object:values    → [1,2]
{"a":1,"b":2}|object:array     → [["a",1],["b",2]]
```

#### Numeric

**`length`**: Get length
```
"hello"|length                 → 5
["a","b","c"]|length           → 3
{"a":1,"b":2}|length           → 2
```

**`calc`**: Arithmetic
```
5|calc:"+10"       → 15
2|calc:"**3"       → 8 (exponentiation)
10|calc:"/2"       → 5
3|calc:"*4+2"      → 14
```
Operators: `+`, `-`, `*`, `/`, `**` or `^` (exponentiation)

**`round`**: Round numbers
```
3.7|round          → 4
3.14159|round:2    → 3.14
```

**`number_format`**: Format numbers with separators
```
"1234567.89"|number_format:(2, ".", ",")     → "1,234,567.89"
"1000"|number_format:(0, ".", " ")           → "1 000"
[1234.5, 6789.1]|number_format:(2, ",", " ")
```
Parameters: `(decimals, decimal_point, thousands_separator)`

---

## Advanced Composition Patterns

### Chaining Selectors and Filters

Extract complex structures from unpredictable DOM:

```
{{selectorHtml:article|html_to_json|map:item => item.children|template:"${str}"}}
```
Flow:
1. Extract article HTML
2. Parse into JSON tree
3. Map over top-level elements to extract children
4. Format children as strings

### Schema to Formatted Lists

```
{{schema:@Recipe:ingredients[*].name|map:item => ({name: item})|template:"- [ ] ${name}\n"}}
```
Flow:
1. Extract all ingredient names (array)
2. Wrap each in an object with `name` key
3. Apply template to create task list

### Highlights with Fragment Links

```
{{highlights|map:item => ({text: item.text, time: item.timestamp|date:"YYYY-MM-DD"})|template:"**${time}**: ${text}\n"|fragment_link}}
```
Flow:
1. Map highlights to include formatted timestamp
2. Apply template to format each highlight
3. Add text fragment links

**Note**: Filters inside `map` are not yet supported. Apply date formatting before or after the map:
```
{{highlights|map:item => ({text: item.text, time: item.timestamp})|template:"**${time|date:\"YYYY-MM-DD\"}**: ${text}\n"}}
```
(This won't work. Instead, format after mapping or use separate variables.)

**Working pattern**:
```
{{highlights|fragment_link|map:item => item.text|join:"\n\n"}}
```

### Multi-Level Schema Extraction

```
{{schema:@NewsArticle:author[*].name|join:", "}}
→ "Alice Smith, Bob Jones"

{{schema:@Product:offers[*].price|map:item => item|template:"$${str}"|join:" / "}}
→ "$29.99 / $39.99"
```

### Conditional Content via Filters

Simulate conditionals using array filters:
```
{{schema:@Recipe:ingredients[*].name|slice:0,5|list}}
```
Only shows first 5 ingredients (rest ignored).

```
{{selector:.tag|nth:n+3|join:" · "}}
```
Skip first 2 tags, show rest.

### HTML Transformation Pipelines

```
{{selectorHtml:.content|remove_html:("script", "style", ".ad")|markdown|trim}}
```
Flow:
1. Extract content HTML
2. Remove scripts, styles, and ads
3. Convert clean HTML to Markdown
4. Trim whitespace

### Nested Map and Template

```
{{schema:@ItemList:itemListElement[*]|map:item => ({name: item.name, url: item.url})|template:"- [${name}](${url})\n"}}
```

### Building Tables from Schema

```
{{schema:@Table:rows[*]|map:row => ({col1: row.cells[0], col2: row.cells[1]})|table}}
```

### Regex-Based Text Extraction

```
{{content|split:/\n\n/|nth:1|trim}}
```
Extract first paragraph (split on double newlines, take 2nd element).

### Complex List Processing

```
{{selector:.item|html_to_json|map:item => item.children[0].content|unique|sort|join:" | "}}
```
(Note: `sort` filter doesn't exist, but you can combine with external sorting if needed.)

### Combining Prompts and Filters

```
{{"extract all mentioned tools as JSON array"|map:item => item|wikilink|join:", "}}
```
Flow:
1. Prompt returns JSON array of tool names
2. Map converts each to wikilink
3. Join with commas

### Dynamic Property Access

```
{{schema:@Event:location.address.streetAddress}}
{{schema:@Event:performer[0].name}}
{{schema:@Event:performer[*].name|join:" & "}}
```

---

## Deterministic Extraction Strategies

### Principle: Prefer Specificity Over Generality

Always choose the most deterministic extraction method:
1. **Schema.org** (most reliable, structured data)
2. **Meta tags** (reliable, standardized)
3. **CSS selectors** (site-specific, fragile to DOM changes)
4. **Prompt variables** (flexible but non-deterministic)

### Strategy: Layered Fallbacks

Use multiple variables with increasing generality:
```
**Author**: {{schema:@Article:author.name}}{{meta:property:article:author}}{{selector:.author-name}}{{author}}
```
First non-empty value renders. Empty variables produce no output.

**Better approach**: Use template logic to show only first match
```
{{schema:@Article:author.name}}{% if not schema:@Article:author.name %}{{meta:property:article:author}}{% endif %}
```
(Note: Conditionals beyond `for` are not currently supported. Use concatenation with empty string fallback.)

### Strategy: Normalize Before Formatting

```
{{selector:.tag|split:","|map:item => item|trim|unique|sort|join:", "}}
```
Order of operations:
1. Extract (may include commas, spaces, duplicates)
2. Split into array
3. Trim whitespace from each
4. Remove duplicates
5. Sort (ensures stable output)
6. Join into final string

### Strategy: Validate and Sanitize

```
{{title|safe_name}}
{{date|date:"YYYY-MM-DD"}}
{{url|split:"?"|first}}
```
Always sanitize user-facing or filename outputs.

### Strategy: Capture Full Context for Selectors

Store snapshots for offline processing:
```
fullContent: {{selectorHtml:article}}
```
Then apply filters locally:
```
{{fullContent|markdown|split:"\n\n"|slice:0,3|join:"\n\n"}}
```

### Strategy: Use Schema Splats for Iteration

Instead of hardcoding indices:
```
{% for ingredient in schema:@Recipe.ingredients %}
- {{ingredient.name}}: {{ingredient.amount}}
{% endfor %}
```

Or use splat + template:
```
{{schema:@Recipe:ingredients[*]|map:item => ({name: item.name, amt: item.amount})|template:"- ${name}: ${amt}\n"}}
```

### Strategy: Graceful Degradation

Templates should never break:
- Undefined variables → empty string
- Invalid selector → empty string
- Non-array in loop → loop skipped
- Filter errors → original value returned

Design templates to render **something** even on partial failures.

### Strategy: Debugging Variables

Use the popup's variable inspector (`:` menu) to see all available variables and their values before building complex templates.

### Strategy: Test on Multiple Pages

Build templates on one page, test on 5+ others with varying structures. Adjust selectors and schema paths based on real variance.

---

## Performance and Optimization

### Filter Order Matters

Place cheap filters early, expensive filters late:
```
{{content|trim|slice:0,500|markdown}}
```
Trim first (cheap), slice to reduce input size, then convert to Markdown (expensive).

### Avoid Redundant Operations

Bad:
```
{{content|markdown|strip_md}}
```
Why convert to Markdown just to strip it?

Good:
```
{{contentHtml|strip_tags}}
```

### Minimize Selector Calls

Selectors trigger content script communication. Batch extractions:
```
{{selectorHtml:article|markdown}}
```
Instead of multiple `{{selector:article h1}}`, `{{selector:article p}}`, etc.

### Memoization

The template compiler memoizes compilation results. Identical templates on the same page won't recompile.

### JSON Parsing Overhead

Filters auto-parse JSON strings. For very large arrays, this adds overhead. If you don't need structure, keep it as a string:
```
{{highlights|length}}
```
This parses JSON. If you just want to check existence:
```
{{highlights}}
```
Returns JSON string or empty.

---

## Troubleshooting and Debugging

### Variable Not Resolving

1. Check spelling and case (case-sensitive)
2. Verify variable exists in popup inspector
3. Check for nested property typos: `{{item.naem}}` vs `{{item.name}}`
4. Ensure objects are accessed via properties, not as strings

### Selector Returns Empty

1. Open DevTools, run `document.querySelector("selector")` to verify
2. Check if content loads dynamically (after page load)
3. Verify selector syntax (escape special characters if needed)
4. Use `{{selectorHtml:...}}` to see raw HTML output
5. Ensure no typos: `.class` vs `#id`

### Loop Not Rendering

1. Verify source is an array: `{{arrayVar}}` should output JSON array
2. Check loop syntax: `{% for item in source %}`, not `{% for item of source %}`
3. Ensure iterator variable is referenced correctly: `{{item}}`, not `{{{{item}}}}`
4. Check for nested property access errors inside loop

### Filter Not Working

1. Verify filter name (case-sensitive, check docs)
2. Check parameter syntax: `filter:param`, `filter:("p1","p2")`
3. Escape special characters in parameters: `replace:"\:":"."` to find `:`
4. Check filter order (some filters expect specific input types)

### Schema Variable Empty

1. Verify page has JSON-LD: View source, search for `application/ld+json`
2. Check schema type: `{{schema:@Recipe:...}}` won't work if type is `@Article`
3. Try shorthand: `{{schema:name}}` instead of full path
4. Use popup inspector to see exact schema keys available

### Prompt Not Replacing

1. Ensure Interpreter is enabled in settings
2. Check that Interpreter has run (click "Interpret" button)
3. Verify prompt syntax: `{{"prompt text"}}` with double quotes
4. Check filters after prompt: `{{"prompt"|filter}}` processes the response

### Template Renders Blank

1. Check if all variables are undefined (use inspector)
2. Verify loops have valid sources
3. Look for syntax errors in logic blocks
4. Test with simple variable: `{{title}}` to confirm compilation works

### Escaped Characters Not Working

Some contexts require double-escaping:
```
{{content|replace:"\\n":"\n"}}
```
Escape backslash in the search term.

### JSON Not Parsing

If a filter expects JSON but gets a string:
```
{{variable|split:","}}
```
Might fail if `variable` is already an array. Check variable type first.

---

## Extensibility Notes

While end users cannot directly extend the template engine, understanding its architecture helps with feature requests and understanding limitations.

### Current Extensibility Points

**Filters**: New filters can be added by developers in `src/utils/filters/`. Each filter is a simple function:
```typescript
export const filterName = (input: string, param?: string): string => {
  // Transform input
  return output;
};
```

**Logic handlers**: New logic constructs (e.g., conditionals) can be added to `logicHandlers` array in `template-compiler.ts`.

**Variable sources**: New data sources can be injected into `initializePageContent` in `content-extractor.ts`.

**Selector channels**: The content script can be extended to support new extraction modes beyond text/HTML/attribute.

### Requesting Features

If you need functionality not covered here:
1. Check if composition of existing filters can achieve it
2. Prototype the desired syntax
3. File a feature request with use cases

---

## Summary of Key Principles

1. **Two-pass compilation**: Logic blocks expand first, then variables resolve
2. **Fail-safe design**: Undefined variables and errors never break renders
3. **JSON propagation**: Filters auto-parse JSON outputs for seamless chaining
4. **Deterministic first**: Prefer schema > meta > selectors > prompts
5. **Filter composition**: Complex transformations come from chaining simple filters
6. **Test broadly**: Templates should work across varied page structures

---

## Quick Reference

### Syntax Cheat Sheet

```
{{variable}}                      Simple variable
{{variable.nested.path}}          Nested access
{{variable[0].item}}              Array index
{{variable|filter}}               Single filter
{{variable|filter1|filter2}}      Chained filters
{{selector:CSS}}                  DOM text extraction
{{selectorHtml:CSS}}              DOM HTML extraction
{{selector:CSS?attr}}             Attribute extraction
{{schema:@Type:key}}              Schema full notation
{{schema:key}}                    Schema shorthand
{{schema:array[*].prop}}          Schema array splat
{{meta:name:description}}         Meta tag (name attr)
{{meta:property:og:title}}        Meta tag (property attr)
{{"prompt text"}}                 AI extraction
{% for item in array %}...{% endfor %}  Loop block
```

### Common Patterns

**Extract and format list**:
```
{{selector:.item|list}}
```

**Schema array to Markdown**:
```
{{schema:@Recipe:ingredients[*].name|list}}
```

**HTML to clean Markdown**:
```
{{selectorHtml:.content|remove_html:"script"|markdown}}
```

**Highlights with timestamps**:
```
{{highlights|map:item => ({text: item.text, time: item.timestamp})|template:"**${time}**: ${text}\n"}}
```

**Conditional rendering** (via concatenation):
```
{{schema:author}}{{selector:.author}}{{author}}
```
First non-empty value renders.

**Multi-column table**:
```
{{array|table:("Column 1", "Column 2", "Column 3")}}
```

**Filter complex HTML**:
```
{{fullHtml|remove_html:(".ad", "#popup")|markdown|slice:0,1000}}
```

---

## Additional Resources

- [Official Templates Documentation](https://help.obsidian.md/web-clipper/templates)
- [Official Variables Reference](https://help.obsidian.md/web-clipper/variables)
- [Official Filters Reference](https://help.obsidian.md/web-clipper/filters)
- [Example Templates Repository](https://github.com/kepano/clipper-templates)
- [Day.js Format Reference](https://day.js.org/docs/en/display/format)
- [CSS Selectors Reference](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_selectors)
- [Schema.org Documentation](https://schema.org/)

---

**Document Version**: 1.0
**Last Updated**: 2025-10-25
**Target Audience**: Advanced template creators with technical background
