# Internal Components Deep Dive

## Overview

This document provides a detailed analysis of the internal components that power MD2RazorGenerator. These are the building blocks that handle data representation, parsing, validation, and transformation.

## Component Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Public API Surface                        │
│  ┌─────────────────┐           ┌────────────────────────┐  │
│  │ MD2RazorGenerator│           │ MSBuild Task           │  │
│  │ (Source Gen)     │           │                        │  │
│  └────────┬─────────┘           └──────────┬─────────────┘  │
│           │                                 │                │
│           └────────────┬────────────────────┘                │
│                        │                                     │
└────────────────────────┼─────────────────────────────────────┘
                         │
┌────────────────────────┼─────────────────────────────────────┐
│          Internal Components (Toolbelt.Blazor.MD2Razor.     │
│                              Internals namespace)             │
│                        │                                     │
│  ┌────────────────────┴─────────────────────────────────┐  │
│  │              MD2Razor (Core Engine)                   │  │
│  └───┬─────────────────────────────────────────┬────────┘  │
│      │                                           │           │
│      │ Uses                                      │ Uses      │
│      ↓                                           ↓           │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐ │
│  │ FrontMatter  │    │  Imports     │    │ GlobalOptions│ │
│  │ (Metadata)   │    │ (Cascading)  │    │ (Config)     │ │
│  └──────────────┘    └──────────────┘    └──────────────┘ │
│                                                             │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐ │
│  │ ValidIdentifier   │ PathUtils    │    │ Imports      │ │
│  │ (Sanitization)│    │ (Normalize)  │    │ Collection   │ │
│  └──────────────┘    └──────────────┘    │ Extensions   │ │
│                                           └──────────────┘ │
└───────────────────────────────────────────────────────────┘
```

## FrontMatter.cs - Metadata Representation

### Purpose

Represents YAML front matter metadata from Markdown files as a strongly-typed, immutable data structure.

### Design Decisions

#### Immutability

```csharp
public class FrontMatter
{
    public IEnumerable<string> Pages { get; }
    public IEnumerable<string> Usings { get; }
    public string? Namespace { get; }
    // ... all properties are get-only
    
    public FrontMatter(IEnumerable<string> pages, /* ... */)
    {
        this.Pages = pages;
        // Properties set once in constructor
    }
}
```

**Benefits**:
- Thread-safe by design
- Safe to cache
- Clear ownership semantics
- No accidental mutations

**Pattern**: Value Object

#### Sequence vs. Scalar Properties

The class handles both YAML formats:

**Sequence Properties** (arrays):
```csharp
public IEnumerable<string> Pages { get; }        // url: [/, /home]
public IEnumerable<string> Usings { get; }       // $using: [System, System.Linq]
public IEnumerable<string> Attributes { get; }   // $attribute: [Authorize, Route("/")]
```

**Scalar Properties** (single values):
```csharp
public string? Namespace { get; }      // $namespace: MyApp.Pages
public string? Layout { get; }         // $layout: MainLayout
public string? Inherit { get; }        // $inherit: CustomBase
public string? PageTitle { get; }      // title: "My Page"
```

**Implementation Strategy**:
```csharp
public FrontMatter(
    IEnumerable<string> pages,
    IEnumerable<string> namespaces,  // Sequence input
    // ...
)
{
    this.Pages = pages;
    this.Namespace = namespaces.FirstOrDefault();  // Take first, ignore rest
    // ...
}
```

**Rationale**: 
- YAML allows sequences for all values
- For some properties (namespace, layout), only first makes sense
- Accept sequence for consistency, use first value
- Flexible input, predictable output

### Empty Default Constructor

```csharp
public FrontMatter() : this([], [], [], [], [], [], [])
{
}
```

**Use Case**: Markdown files without front matter

**Behavior**: All collections empty, all nullable properties null

**Impact on Generation**:
- No `[Route]` attributes
- No `<PageTitle>` component
- Default namespace (computed from path)
- Default base class

### Property Mapping

| Property | YAML Key | Type | Default | Example |
|----------|----------|------|---------|---------|
| `Pages` | `url` | `string[]` | `[]` | `["/home", "/index"]` |
| `PageTitle` | `title` | `string?` | `null` | `"Home Page"` |
| `Usings` | `$using` | `string[]` | `[]` | `["System.Linq", "MyApp.Models"]` |
| `Namespace` | `$namespace` | `string?` | `null` | `"MyApp.Pages"` |
| `Attributes` | `$attribute` | `string[]` | `[]` | `["Authorize", "AllowAnonymous"]` |
| `Layout` | `$layout` | `string?` | `null` | `"MainLayout"` |
| `Inherit` | `$inherit` | `string?` | `null` | `"ComponentBase"` |

### Dollar Sign Convention

**Properties with `$` prefix**: `$using`, `$namespace`, `$attribute`, `$layout`, `$inherit`

**Meaning**: Compile-time directives (borrowed from Razor syntax)

**Contrast**:
- `url`: Runtime-relevant (routing)
- `title`: Runtime-relevant (page title)
- `$namespace`: Compile-time only (C# namespace)
- `$using`: Compile-time only (C# using directives)

**User Mental Model**: 
- Regular keys → affect runtime behavior
- `$` keys → affect code generation

### Extensibility

To add new front matter keys:

1. Add property to `FrontMatter` class:
```csharp
public string? MyNewProperty { get; }
```

2. Add to constructor:
```csharp
public FrontMatter(
    // ... existing params
    IEnumerable<string> myNewProperties
)
{
    // ...
    this.MyNewProperty = myNewProperties.FirstOrDefault();
}
```

3. Parse in `MD2Razor.ParseFrontMatter()`:
```csharp
return new FrontMatter(
    // ... existing args
    myNewProperties: getFrontMatterEntry(yamlNodeMap, "myNewProperty")
);
```

4. Use in `MD2Razor.GenerateCode()`:
```csharp
if (!string.IsNullOrEmpty(frontMatter.MyNewProperty))
{
    // Generate code based on property
}
```

## Imports.cs - Using Directive Management

### Purpose

Represents a single `_Imports.razor` file and extracts `@using` directives from it.

### Key Features

#### Lazy Parsing

```csharp
private string _text;
private IEnumerable<string>? _usings = null;

public IEnumerable<string> GetUsings()
{
    return this._usings ??= this._text.Split(['\n'], StringSplitOptions.RemoveEmptyEntries)
        .Select(line => Regex.Match(line, @"^[ \t]*@using[ \t]+(?<namespace>(static[ \t]+)?[^ \t;]+).*$"))
        .Where(match => match.Success)
        .Select(match => match.Groups["namespace"].Value.Trim())
        .ToArray();
}
```

**Benefits**:
- Only parse when needed
- Parse once, cache result
- Zero overhead if never called

**Pattern**: Lazy Initialization with Null Coalescing

**Thread Safety**: 
- Not thread-safe (multiple parses possible)
- Acceptable trade-off (parsing is idempotent)
- Result is same regardless of thread

#### Regex Pattern Analysis

```regex
^[ \t]*@using[ \t]+(?<namespace>(static[ \t]+)?[^ \t;]+).*$
```

**Breakdown**:

| Part | Meaning | Example Matches |
|------|---------|-----------------|
| `^[ \t]*` | Line start, optional whitespace | `@using`, `  @using`, `\t@using` |
| `@using` | Literal `@using` | Required |
| `[ \t]+` | At least one space or tab | ` `, `  `, `\t` |
| `(static[ \t]+)?` | Optional "static" keyword | `static `, `static  ` |
| `[^ \t;]+` | Namespace (non-whitespace, non-semicolon) | `System`, `System.Linq` |
| `.*` | Rest of line (ignored) | `;`, `; // comment` |
| `$` | End of line | |

**Captured Group**: `namespace` contains the full using directive (including `static` if present)

**Examples**:

```csharp
@using System                        → "System"
@using System.Linq                   → "System.Linq"
@using static System.Math            → "static System.Math"
  @using MyApp.Models                → "MyApp.Models"
@using MyApp.Services;               → "MyApp.Services"
@using MyApp.Components; // comment  → "MyApp.Components"
```

#### Supported Patterns

✅ **Supported**:
- Standard using: `@using System`
- Static using: `@using static System.Math`
- With semicolon: `@using System;`
- With comment: `@using System // comment`
- Indented: `  @using System`

❌ **Not Supported** (documented limitations):
- Inline comment: `@using /* comment */ System`
- Multi-line: 
  ```csharp
  @using
      System
  ```

### Equality Implementation

```csharp
public bool Equals(Imports? other)
{
    return other is not null &&
           this.Path == other.Path &&
           this._text == other._text;
}
```

**Why This Matters**:
- Used for caching by Roslyn
- Two `Imports` instances are equal if same file with same content
- Path alone insufficient (file might change)

**Hash Code**:
```csharp
public override int GetHashCode()
{
    var hashCode = 833777843;  // Arbitrary prime seed
    hashCode = hashCode * -1521134295 + EqualityComparer<string>.Default.GetHashCode(this.Path);
    hashCode = hashCode * -1521134295 + EqualityComparer<string>.Default.GetHashCode(this._text);
    return hashCode;
}
```

**Algorithm**: Multiplicative hash with prime numbers
- Standard technique for combining hashes
- Good distribution of hash values
- Consistent with `Equals()` implementation

## ImportsCollectionExtensions.cs - Cascading Import Logic

### Purpose

Implements Razor's cascading `_Imports.razor` behavior: imports apply to their directory and all subdirectories.

### Core Method

```csharp
public static IEnumerable<Imports> GetApplicableImports(
    this IEnumerable<Imports> importsCollection, 
    string path)
{
    var baseDir = Path.GetDirectoryName(PathUtils.NormalizePath(path)) 
        ?? throw new ArgumentException($"Invalid path: {path}");
    
    return importsCollection
        .Where(i => baseDir.StartsWith(
            Path.GetDirectoryName(PathUtils.NormalizePath(i.Path)), 
            StringComparison.InvariantCultureIgnoreCase));
}
```

### Algorithm Explained

**Step 1: Extract Directory**
```csharp
var baseDir = Path.GetDirectoryName(PathUtils.NormalizePath(path));
```

**Example**:
```
Input: /project/Pages/Blog/Post.md
Output: /project/Pages/Blog
```

**Step 2: Filter Imports**

For each import file, check if the Markdown file's directory starts with the import file's directory.

**Example Scenario**:

```
Project structure:
/project/
  ├── _Imports.razor              (Import A)
  ├── Pages/
  │   ├── _Imports.razor          (Import B)
  │   ├── Index.md                (File 1)
  │   └── Blog/
  │       ├── _Imports.razor      (Import C)
  │       └── Post.md             (File 2)
  └── Components/
      └── _Imports.razor          (Import D)
```

**For File 1** (`/project/Pages/Index.md`):
- `baseDir = /project/Pages`
- Import A: `/project/Pages` starts with `/project` ✓ **Applies**
- Import B: `/project/Pages` starts with `/project/Pages` ✓ **Applies**
- Import C: `/project/Pages` starts with `/project/Pages/Blog` ✗ Doesn't apply
- Import D: `/project/Pages` starts with `/project/Components` ✗ Doesn't apply

**Result**: Imports A, B apply

**For File 2** (`/project/Pages/Blog/Post.md`):
- `baseDir = /project/Pages/Blog`
- Import A: `/project/Pages/Blog` starts with `/project` ✓ **Applies**
- Import B: `/project/Pages/Blog` starts with `/project/Pages` ✓ **Applies**
- Import C: `/project/Pages/Blog` starts with `/project/Pages/Blog` ✓ **Applies**
- Import D: `/project/Pages/Blog` starts with `/project/Components` ✗ Doesn't apply

**Result**: Imports A, B, C apply

### Case Sensitivity

```csharp
StringComparison.InvariantCultureIgnoreCase
```

**Reasoning**: Windows file systems are case-insensitive
- `/Project/Pages` == `/project/pages` on Windows
- Must match behavior for cross-platform consistency

### Path Normalization

```csharp
PathUtils.NormalizePath(path)
```

**Purpose**: Handle both `/` and `\` separators

**Example**:
```
Input (Windows): C:\project\Pages\Index.md
Input (Unix):    /project/Pages/Index.md
Both normalize to same format for comparison
```

### Edge Cases

#### Empty Import Collection

```csharp
importsCollection = []
GetApplicableImports("/project/Page.md")
→ returns []
```

**Behavior**: No imports apply (graceful)

#### Root-Level Markdown File

```csharp
path = "/project/Page.md"
baseDir = "/project"

_Imports.razor at /project/_Imports.razor
importDir = "/project"

"/project".StartsWith("/project") → true ✓
```

**Behavior**: Root imports apply to root files

#### Invalid Path

```csharp
path = ""
Path.GetDirectoryName("") → null
→ throw new ArgumentException("Invalid path: ")
```

**Behavior**: Fail fast with clear error

## GlobalOptions.cs - Project Configuration

### Purpose

Encapsulates project-wide configuration that affects all generated code.

### Properties

```csharp
public string RootNamespace { get; }      // From build_property.RootNamespace
public string ProjectDir { get; }         // From build_property.ProjectDir
public string DefaultBaseClass { get; }   // From build_property.MD2RazorDefaultBaseClass
```

### Default Base Class Logic

```csharp
public GlobalOptions(string rootNamespace, string projectDir, string defaultBaseClass)
{
    this.RootNamespace = rootNamespace;
    this.ProjectDir = projectDir;
    this.DefaultBaseClass = string.IsNullOrWhiteSpace(defaultBaseClass) 
        ? "global::Microsoft.AspNetCore.Components.ComponentBase" 
        : defaultBaseClass;
}
```

**Default**: `global::Microsoft.AspNetCore.Components.ComponentBase`

**Why `global::`?**
- Fully qualified type reference
- Avoids ambiguity with user-defined types
- Ensures correct type resolution regardless of using directives

**Example**:
```csharp
// User might have:
namespace MyApp.Components;
public class ComponentBase { }  // Collision!

// Generated code with global:: prefix:
public partial class MyComponent : global::Microsoft.AspNetCore.Components.ComponentBase
{
    // Correctly references Blazor's ComponentBase, not user's
}
```

### Equality Implementation

```csharp
public bool Equals(GlobalOptions? other)
{
    return other is not null &&
        this.RootNamespace == other.RootNamespace &&
        this.ProjectDir == other.ProjectDir &&
        this.DefaultBaseClass == other.DefaultBaseClass;
}
```

**Use Case**: Detect when global options change

**Impact**: MSBuild task regenerates all files when global options change

**Scenario**:
```
Build 1: RootNamespace = "MyApp"
  → Generate all files with namespace MyApp.*

User edits .csproj: <RootNamespace>MyApp.Web</RootNamespace>

Build 2: RootNamespace = "MyApp.Web"
  → GlobalOptions not equal to previous
  → Regenerate all files with namespace MyApp.Web.*
```

### Hash Code Implementation

```csharp
public override int GetHashCode()
{
    var hashCode = -847805451;  // Seed
    hashCode = hashCode * -1521134295 + EqualityComparer<string>.Default.GetHashCode(this.RootNamespace);
    hashCode = hashCode * -1521134295 + EqualityComparer<string>.Default.GetHashCode(this.ProjectDir);
    hashCode = hashCode * -1521134295 + EqualityComparer<string>.Default.GetHashCode(this.DefaultBaseClass);
    return hashCode;
}
```

**Pattern**: Polynomial rolling hash
- Combines multiple string hashes
- Uses prime multiplier for good distribution
- Consistent with equality implementation

## ValidIdentifier.cs - Identifier Sanitization

### Purpose

Converts arbitrary file names into valid C# identifiers by sanitizing characters and handling keywords.

### Reserved Keywords

```csharp
private static readonly HashSet<string> _reservedWords = new([
    "add", "field", "nint", "scoped",
    "allows", "file", "not", "select",
    // ... comprehensive list of C# keywords
]);
```

**Includes**:
- Core keywords: `class`, `namespace`, `public`
- Contextual keywords: `async`, `await`, `var`, `dynamic`
- Newer keywords: `record`, `init`, `required`, `scoped`
- LINQ keywords: `from`, `where`, `select`, `group`

**Strategy**: Conservative inclusion
- Include all keywords (even contextual)
- Better safe than sorry
- Small performance cost (hash lookup)

### Sanitization Algorithm

```csharp
public static string Create(string? name)
{
    if (name is null) throw new ArgumentNullException(nameof(name));
    if (name.Length == 0) return name;

    // Step 1: Prepend underscore if starts with digit
    if (Regex.IsMatch(name[0].ToString(), @"^\p{Nd}")) 
        name = "_" + name;
    
    // Step 2: Replace invalid leading characters
    name = Regex.Replace(name, @"^[^\p{L}\p{Nl}_]", "_");
    
    // Step 3: Replace invalid characters in body
    name = Regex.Replace(name, @"[^\p{L}\p{Mn}\p{Mc}\p{Nd}\p{Nl}\p{Pc}_]", "_");

    // Step 4: Append underscore if reserved word
    if (_reservedWords.Contains(name)) 
        name += "_";

    return name;
}
```

### Unicode Category Reference

C# identifier rules use Unicode categories:

| Category | Code | Meaning | Examples |
|----------|------|---------|----------|
| Letter | `\p{L}` | Any letter | `a`, `Z`, `ñ`, `中` |
| Letter Number | `\p{Nl}` | Letter-like number | `Ⅳ` (Roman numeral) |
| Non-spacing Mark | `\p{Mn}` | Accent marks | `́` (combining acute) |
| Spacing Mark | `\p{Mc}` | Spacing accent | `ः` (visarga) |
| Decimal Number | `\p{Nd}` | Digit | `0-9`, `٠-٩` (Arabic) |
| Connector Punctuation | `\p{Pc}` | Connector | `_` (underscore) |

**C# Identifier Rules**:
- **First character**: Letter, letter number, or underscore
- **Subsequent characters**: Letter, letter number, decimal number, spacing mark, non-spacing mark, connector punctuation

### Transformation Examples

| Input | Output | Reason |
|-------|--------|--------|
| `MyPage` | `MyPage` | Valid identifier |
| `My-Page` | `My_Page` | Dash not allowed |
| `My Page` | `My_Page` | Space not allowed |
| `2023-Report` | `_2023_Report` | Can't start with digit |
| `class` | `class_` | Reserved keyword |
| `async` | `async_` | Contextual keyword |
| `HelloWorld!` | `HelloWorld_` | Exclamation not allowed |
| `café` | `café` | Valid (accented letters allowed) |
| `文件` | `文件` | Valid (Unicode letters) |

### Edge Cases

#### Empty String

```csharp
if (name.Length == 0) return name;
```

**Behavior**: Return empty string unchanged
**Use Case**: Shouldn't happen, but safe fallback

#### Single Character

```csharp
name = "3"
→ Starts with digit
→ Prepend "_"
→ Result: "_3"
```

#### All Invalid Characters

```csharp
name = "---"
→ No digits
→ Replace leading invalid: "_--"
→ Replace body invalid: "___"
→ Not a keyword
→ Result: "___"
```

#### Multiple Transformations

```csharp
name = "2023-year-end-report.final"
→ "_2023-year-end-report.final" (prepend _)
→ "_2023_year_end_report_final" (replace invalid)
→ Not a keyword
→ Result: "_2023_year_end_report_final"
```

### Performance Characteristics

**Time Complexity**: O(n) where n = length of input string
- Three regex replacements: each O(n)
- Hash set lookup: O(1)
- Overall: O(n)

**Space Complexity**: O(n)
- Creates new strings for each replacement
- Final result length ≤ original + 1 (underscore prefix/suffix)

## PathUtils.cs - Cross-Platform Path Handling

### Purpose

Normalize file paths to handle both Windows (`\`) and Unix (`/`) path separators consistently.

### Primary Method

```csharp
public static string NormalizePath(string path, char pathSeparator)
{
    return string.Join(pathSeparator.ToString(), path.Split('\\', '/', ':'))
        .TrimEnd(pathSeparator);
}
```

### Algorithm Breakdown

**Step 1: Split on all separators**
```csharp
path.Split('\\', '/', ':')
```

**Separators**:
- `\` - Windows path separator
- `/` - Unix path separator
- `:` - Drive letter separator (Windows: `C:`)

**Example**:
```csharp
Input: "C:\project\Pages/Blog\Post.md"
Split: ["C", "project", "Pages", "Blog", "Post.md"]
```

**Step 2: Join with target separator**
```csharp
string.Join(pathSeparator.ToString(), parts)
```

**Example**:
```csharp
Parts: ["C", "project", "Pages", "Blog", "Post.md"]
Join with '/': "C/project/Pages/Blog/Post.md"
```

**Step 3: Trim trailing separator**
```csharp
.TrimEnd(pathSeparator)
```

**Example**:
```csharp
Input: "C/project/Pages/"
Output: "C/project/Pages"
```

**Why**: Consistent behavior for directory paths

### Convenience Overload

```csharp
public static string NormalizePath(string path) 
    => NormalizePath(path, Path.DirectorySeparatorChar);
```

**Use Case**: Normalize to current platform's separator
- Windows: `\`
- Unix/Linux/macOS: `/`

### Use Cases

#### 1. Path Comparison

```csharp
// Before normalization:
"/project/Pages" == "\\project\\Pages"  // false (different strings)

// After normalization:
NormalizePath("/project/Pages", '/') == NormalizePath("\\project\\Pages", '/')  // true
```

#### 2. Relative Path Computation

```csharp
var fullPath = "C:\\project\\Pages\\Home.md";
var basePath = "C:/project/";

// Normalize both before substring operations
fullPath = NormalizePath(fullPath, '/');  // "C/project/Pages/Home.md"
basePath = NormalizePath(basePath, '/');  // "C/project"

var relative = fullPath.Substring(basePath.Length);  // "/Pages/Home.md"
```

#### 3. Dot-Separated Path Generation

```csharp
var path = "C:\\project\\Pages\\Blog\\Post.md";
var basePath = "C:\\project\\";

path = NormalizePath(path, '.');       // "C.project.Pages.Blog.Post.md"
basePath = NormalizePath(basePath, '.');  // "C.project"

// Use in hint name generation
var hintName = path.Substring(basePath.Length);  // ".Pages.Blog.Post.md"
```

### Edge Cases

#### Empty Path

```csharp
NormalizePath("", '/')
→ Split: [""]
→ Join: ""
→ Trim: ""
→ Result: ""
```

#### Root Path

```csharp
NormalizePath("/", '/')
→ Split: ["", ""]
→ Join: "/"
→ Trim: ""
→ Result: ""  // Root becomes empty
```

**Implication**: Root directory handling needs special care

#### Drive Letter (Windows)

```csharp
NormalizePath("C:", '/')
→ Split: ["C", ""]
→ Join: "C/"
→ Trim: "C"
→ Result: "C"
```

#### Multiple Consecutive Separators

```csharp
NormalizePath("C://project//Pages", '/')
→ Split: ["C", "", "project", "", "Pages"]
→ Join: "C//project//Pages"  // Preserves empty parts
→ Result: "C//project//Pages"
```

**Note**: Does not deduplicate separators (by design)

### Performance

**Time Complexity**: O(n) where n = length of path
- Split: O(n)
- Join: O(n)
- TrimEnd: O(1) typically

**Space Complexity**: O(n)
- Split creates array of parts
- Join creates new string
- Minimal overhead

### Design Trade-offs

**Simplicity vs. Robustness**:
- ✓ Simple implementation (one line + trim)
- ✓ Handles most cases
- ✗ Doesn't validate path correctness
- ✗ Doesn't resolve `.` or `..`
- ✗ Doesn't deduplicate separators

**Chosen Approach**: Simple normalization sufficient for this use case
- Paths come from MSBuild (already valid)
- Used for comparison and transformation, not validation
- Adding full path validation would complicate unnecessarily

## Integration and Data Flow

### Markdown File Processing Flow

```
1. Input: Markdown file path and content
   ↓
2. PathUtils.NormalizePath() → Consistent path format
   ↓
3. ImportsCollectionExtensions.GetApplicableImports() → Find relevant imports
   ↓
4. Imports.GetUsings() → Extract using directives
   ↓
5. MD2Razor.ParseFrontMatter() → Parse YAML → FrontMatter object
   ↓
6. ValidIdentifier.Create() → Sanitize class name
   ↓
7. MD2Razor.GenerateNamespaceFromPath() → Compute namespace
   ↓
8. MD2Razor.GenerateCode() → Combine everything → C# source code
```

### Component Interactions

```
┌─────────────┐
│ MD2Razor    │────────uses────────→ ValidIdentifier.Create()
│             │                      (sanitize class name)
│             │────────uses────────→ PathUtils.NormalizePath()
│             │                      (normalize paths)
│             │────────uses────────→ GlobalOptions
│             │                      (root namespace, project dir)
│             │────────creates─────→ FrontMatter
│             │                      (from YAML parsing)
└─────┬───────┘
      │
      │ uses
      ↓
┌─────────────────────────┐
│ ImportsCollection       │
│ Extensions              │────uses────→ Imports.GetUsings()
│                         │              (extract using directives)
│                         │────uses────→ PathUtils.NormalizePath()
│                         │              (compare paths)
└─────────────────────────┘
```

## Testing Strategy

### Unit Testing Internal Components

Each internal component should have focused unit tests:

#### FrontMatter Tests

```csharp
[Test]
public void FrontMatter_WithSequenceUrl_ReturnsAllUrls()
{
    var fm = new FrontMatter(
        pages: ["/", "/home"], 
        usings: [], 
        namespaces: [], 
        attributes: [], 
        layouts: [], 
        inherits: [], 
        pageTitles: []);
    
    fm.Pages.Should().HaveCount(2);
    fm.Pages.Should().Contain(new[] { "/", "/home" });
}
```

#### ValidIdentifier Tests

```csharp
[Test]
public void Create_WithReservedKeyword_AppendsUnderscore()
{
    ValidIdentifier.Create("class").Should().Be("class_");
    ValidIdentifier.Create("async").Should().Be("async_");
}

[Test]
public void Create_WithLeadingDigit_PrependsUnderscore()
{
    ValidIdentifier.Create("2023Report").Should().Be("_2023Report");
}
```

#### PathUtils Tests

```csharp
[Test]
public void NormalizePath_WithMixedSeparators_Normalizes()
{
    PathUtils.NormalizePath("C:\\project/Pages\\Blog", '/')
        .Should().Be("C/project/Pages/Blog");
}
```

## Best Practices for Internal Components

### 1. Immutability

✓ Make classes immutable when possible  
✓ Use `{ get; }` properties (no setters)  
✓ Initialize in constructor  
✓ Benefits: Thread-safety, caching, clarity  

### 2. Null Safety

✓ Use nullable reference types (`string?`)  
✓ Validate inputs early  
✓ Throw meaningful exceptions  
✓ Use null-conditional operators (`?.`, `??`)  

### 3. Performance

✓ Use lazy initialization for expensive operations  
✓ Cache computed results  
✓ Avoid unnecessary allocations  
✓ Use efficient algorithms (O(n) or better)  

### 4. Testability

✓ Keep methods pure (same input → same output)  
✓ Avoid static mutable state  
✓ Don't depend on file system in core logic  
✓ Use dependency injection where appropriate  

### 5. Documentation

✓ XML comments on public APIs  
✓ Explain "why" not just "what"  
✓ Document assumptions and limitations  
✓ Provide usage examples  

## Conclusion

The internal components of MD2RazorGenerator demonstrate solid software engineering practices:

✓ **Single Responsibility**: Each component has one clear purpose  
✓ **Immutability**: Value objects are immutable for safety  
✓ **Testability**: Pure functions, minimal dependencies  
✓ **Performance**: Lazy evaluation, efficient algorithms  
✓ **Robustness**: Input validation, error handling  

These components form a solid foundation that enables the source generator and MSBuild task to work reliably and efficiently. Understanding their design helps when:

- Contributing new features
- Debugging issues
- Optimizing performance
- Extending functionality

The architecture prioritizes correctness and performance while maintaining simplicity and readability.
