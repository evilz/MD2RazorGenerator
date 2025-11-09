# MD2RazorGenerator Architecture

## Executive Summary

MD2RazorGenerator is a sophisticated C# source generator that transforms Markdown files into Blazor Razor components at compile time. This document provides a comprehensive architectural overview from a senior developer/architect perspective.

## Architecture Overview

### System Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│                    Blazor Application Project                     │
│                                                                    │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │ *.md Files   │  │ _Imports.razor│  │ MSBuild Props│           │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘           │
│         │                  │                  │                    │
└─────────┼──────────────────┼──────────────────┼────────────────────┘
          │                  │                  │
          │                  │                  │
          ▼                  ▼                  ▼
┌──────────────────────────────────────────────────────────────────┐
│              MD2RazorGenerator (Source Generator)                 │
│                                                                    │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  IIncrementalGenerator Implementation                     │   │
│  │  - Watches for *.md files in AdditionalTextsProvider     │   │
│  │  - Collects _Imports.razor files                         │   │
│  │  - Reads MSBuild properties (namespace, project dir)     │   │
│  └───────────────────────┬──────────────────────────────────┘   │
│                          │                                        │
│                          ▼                                        │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  MD2Razor Core Engine                                     │   │
│  │  ┌────────────────┐  ┌────────────────┐                 │   │
│  │  │ Markdig Parser │  │ YAML Parser    │                 │   │
│  │  │ (Markdown)     │  │ (Front Matter) │                 │   │
│  │  └───────┬────────┘  └───────┬────────┘                 │   │
│  │          │                    │                           │   │
│  │          └────────┬───────────┘                           │   │
│  │                   ▼                                       │   │
│  │          ┌────────────────────┐                          │   │
│  │          │  Code Generator    │                          │   │
│  │          │  - BuildRenderTree │                          │   │
│  │          │  - Attributes      │                          │   │
│  │          │  - Using directives│                          │   │
│  │          └────────┬───────────┘                          │   │
│  └──────────────────────┼──────────────────────────────────┘   │
└─────────────────────────┼────────────────────────────────────────┘
                          │
                          ▼
          ┌───────────────────────────────┐
          │   Generated C# Source Code    │
          │   (*.g.cs files)              │
          │   - Component classes         │
          │   - BuildRenderTree methods   │
          │   - Route attributes          │
          └───────────────┬───────────────┘
                          │
                          ▼
          ┌───────────────────────────────┐
          │   Roslyn Compiler             │
          │   - Compiles generated code   │
          │   - Creates component classes │
          └───────────────────────────────┘
```

## Key Components

### 1. Source Generator Entry Point (`MD2RazorGenerator.cs`)

**Purpose**: Implements Roslyn's `IIncrementalGenerator` interface to hook into the compilation pipeline.

**Key Responsibilities**:
- Registers with Roslyn to receive notifications about file changes
- Filters for Markdown files (*.md) and _Imports.razor files
- Collects global MSBuild properties (namespace, project directory, default base class)
- Orchestrates the generation process for each Markdown file

**Design Pattern**: Observer Pattern - the source generator observes the compilation context and reacts to file additions/changes.

**Performance Considerations**:
- Uses incremental generation to avoid regenerating unchanged files
- Leverages Roslyn's caching mechanisms via `IncrementalValueProvider`
- Parallel processing not implemented at this level (handled by Roslyn)

### 2. Core Transformation Engine (`MD2Razor.cs`)

**Purpose**: The heart of the system - transforms Markdown content into C# Razor component code.

**Architecture Pattern**: Pipeline/Chain of Responsibility

**Processing Pipeline**:

```
Markdown Text
    ↓
1. Parse with Markdig Pipeline
    ↓
2. Extract YAML Front Matter
    ↓
3. Process Imports (_Imports.razor)
    ↓
4. Generate Namespace & Using Directives
    ↓
5. Generate Component Class Declaration
    ↓
6. Process Hyperlinks (add target="_blank" to external links)
    ↓
7. Convert Markdown to HTML
    ↓
8. Generate BuildRenderTree Method
    ↓
Generated C# Code
```

**Key Methods**:

- `GenerateCode()`: Main orchestration method
  - Input: Markdown file path, content, imports, global options
  - Output: Complete C# source code for a Razor component
  - Supports both full generation and declaration-only mode (for MSBuild task)

- `ParseFrontMatter()`: Extracts YAML metadata
  - Uses YamlDotNet for robust YAML parsing
  - Supports scalar and sequence values for flexibility

- `GenerateNamespaceFromPath()`: Computes namespace from file location
  - Mirrors C# project structure conventions
  - Handles special characters and reserved keywords

**Extensibility Points**:
- Markdig pipeline is configurable (currently includes YAML, emoji, and advanced extensions)
- Front matter keys can be extended (see FrontMatter class)

### 3. MSBuild Task (`GenerateRazorClassDeclarationsFromMarkdown.cs`)

**Purpose**: Provides MSBuild integration for design-time builds and IDE support.

**Why It Exists**: 
Source generators run during compilation, but IDEs need early type information for IntelliSense. The MSBuild task generates class declarations (without implementation) during design-time builds.

**Key Features**:
- Generates declaration-only code (class signature, attributes, no BuildRenderTree)
- Tracks global options changes to trigger full regeneration when needed
- Implements incremental generation based on file timestamps
- Cleans up stale generated files

**Caching Strategy**:
```
.globaloptions file stores:
- RootNamespace
- ProjectDir  
- DefaultBaseClass

If these change, all files are regenerated regardless of timestamps.
```

### 4. Internal Components

#### FrontMatter (`FrontMatter.cs`)

**Purpose**: Strongly-typed representation of YAML front matter metadata.

**Supported Metadata**:

| Property | YAML Key | Type | Purpose |
|----------|----------|------|---------|
| Pages | `url` | string[] | Route templates (like @page) |
| PageTitle | `title` | string | Page title (generates `<PageTitle>` component) |
| Usings | `$using` | string[] | Additional using directives |
| Namespace | `$namespace` | string | Override default namespace |
| Attributes | `$attribute` | string[] | Custom attributes |
| Layout | `$layout` | string | Layout class (like @layout) |
| Inherit | `$inherit` | string | Base class (like @inherits) |

**Design Decision**: Properties with `$` prefix indicate compile-time directives (convention borrowed from Razor).

#### Imports (`Imports.cs`)

**Purpose**: Parses and represents using directives from _Imports.razor files.

**Key Features**:
- Lazy parsing using regex: `@using\s+(?<namespace>(static\s+)?[^\s;]+)`
- Supports static using directives
- Implements equality for caching/comparison

**Limitations** (documented in README):
- Simple regex-based parsing
- May not handle complex scenarios (comments, multi-line)
- Trade-off: simplicity vs. full Razor parser

#### GlobalOptions (`GlobalOptions.cs`)

**Purpose**: Encapsulates project-level configuration.

**Properties**:
- `RootNamespace`: From MSBuild property
- `ProjectDir`: Base path for computing relative paths
- `DefaultBaseClass`: Default base class for components (default: `ComponentBase`)

**Design**: Immutable value object with equality implementation for change detection.

#### ValidIdentifier (`ValidIdentifier.cs`)

**Purpose**: Sanitizes file names into valid C# identifiers.

**Algorithm**:
1. Add underscore if name starts with digit
2. Replace invalid characters with underscores
3. Append underscore if name is a C# keyword (including contextual keywords)

**Example Transformations**:
- `2024-Report.md` → `_2024_Report`
- `class.md` → `class_`
- `My-Component!.md` → `My_Component_`

#### PathUtils (`PathUtils.cs`)

**Purpose**: Cross-platform path normalization.

**Key Method**: `NormalizePath(path, separator)`
- Handles both Unix (/) and Windows (\) path separators
- Ensures consistent path handling across platforms
- Removes trailing separators

#### ImportsCollectionExtensions (`ImportsCollectionExtensions.cs`)

**Purpose**: Implements cascading import behavior.

**Algorithm**: Filters imports where the markdown file's directory starts with the import file's directory. This mirrors Razor's behavior where _Imports.razor applies to its directory and all subdirectories.

```
Example:
- /Pages/_Imports.razor applies to /Pages/Home.md
- /Pages/_Imports.razor applies to /Pages/Blog/Post.md
- /Pages/Blog/_Imports.razor also applies to /Pages/Blog/Post.md
```

## Design Patterns Used

### 1. **Incremental Generation Pattern**
- Leverages Roslyn's `IIncrementalGenerator` for efficient recompilation
- Only regenerates files when inputs change
- Reduces build times significantly in large projects

### 2. **Strategy Pattern**
- Different rendering strategies (full vs. declaration-only)
- Controlled by `declarationOnly` parameter in `GenerateCode()`

### 3. **Builder Pattern**
- Uses `StringBuilder` to construct generated code
- Uses `MarkdownPipelineBuilder` to configure Markdown processing

### 4. **Value Object Pattern**
- `FrontMatter`, `GlobalOptions`, `Imports` are immutable value objects
- Implement equality for proper change detection

### 5. **Template Method Pattern**
- `GenerateCode()` method defines the skeleton of the generation algorithm
- Sub-steps can be customized or extended

## Technology Stack

### Core Dependencies

1. **Microsoft.CodeAnalysis.CSharp (v4.8.0)**
   - Roslyn compiler platform
   - Provides source generator infrastructure
   - Type: Build-time dependency (PrivateAssets="all")

2. **Markdig (v0.41.0)**
   - Robust Markdown parser
   - Extensible pipeline architecture
   - Supports CommonMark and GitHub Flavored Markdown
   - Type: Build-time dependency (included in analyzer package)

3. **YamlDotNet (v16.3.0)**
   - YAML parser for front matter
   - Handles complex YAML structures
   - Type: Build-time dependency (included in analyzer package)

4. **System.Memory (v4.6.3)**
   - Required for netstandard2.0 compatibility
   - Provides Span<T> and Memory<T> types
   - Type: Build-time dependency

### Target Framework
- **netstandard2.0**: Ensures broad compatibility with:
  - .NET Framework 4.6.1+
  - .NET Core 2.0+
  - .NET 5.0+
  - Mono, Xamarin, Unity

## Code Generation Strategy

### BuildRenderTree Method Generation

The generator creates a `BuildRenderTree` method that uses Blazor's RenderTreeBuilder API:

```csharp
protected override void BuildRenderTree(RenderTreeBuilder __builder)
{
    // Optional: Add PageTitle component
    __builder.OpenComponent<PageTitle>(0);
    __builder.AddAttribute(1, "ChildContent", (RenderFragment)((__builder2) => {
        __builder2.AddContent(2, @"Page Title");
    }));
    __builder.CloseComponent();
    
    // Add HTML content from Markdown
    __builder.AddMarkupContent(3, @"<h1>Converted HTML</h1>");
}
```

**Key Decisions**:
- Uses `AddMarkupContent` for HTML (already sanitized by Markdig)
- Sequence numbers are auto-incremented for render tree consistency
- HTML is verbatim string literal (@ prefix) with escaped quotes

### Attribute Generation

Attributes are generated based on front matter and global options:

```csharp
// Routes from 'url' front matter
[Route("/page")]
[Route("/alternate-route")]

// Layout from '$layout' front matter
[LayoutAttribute(typeof(MyLayout))]

// Custom attributes from '$attribute' front matter
[Authorize]
[StreamRendering]

public partial class PageComponent : ComponentBase
{
    // ...
}
```

### Namespace Resolution

Namespace determination follows this priority:

1. Explicit `$namespace` in front matter
2. Computed from file path relative to project directory
3. Combined with `RootNamespace` from MSBuild

**Example**:
```
Project: MyApp
RootNamespace: MyApp
File: /Pages/Blog/Post.md

Result: MyApp.Pages.Blog
```

## Performance Characteristics

### Compilation Impact

**First Build**:
- Parses all Markdown files
- Generates complete C# source for each file
- Typical: 10-50ms per file (depends on file size)

**Incremental Build**:
- Only processes changed files
- Roslyn caches unchanged generators
- Near-zero impact if no Markdown files changed

### Runtime Impact

**Zero Runtime Overhead**:
- All conversion happens at compile time
- No runtime Markdown parsing
- No additional memory allocation for conversion
- Generated code is identical to hand-written Razor

### Memory Usage

**Build Time**:
- Proportional to number and size of Markdown files
- Markdig and YamlDotNet are efficient parsers
- Generator itself is stateless between invocations

**Runtime**:
- No additional memory beyond standard Razor components
- HTML is compiled into the assembly
- No dynamic allocation for Markdown rendering

## Security Considerations

### HTML Sanitization

**Approach**: Relies on Markdig's built-in HTML sanitization
- Markdig follows CommonMark spec for HTML handling
- Potentially unsafe HTML is escaped by default
- Advanced extensions are carefully selected

**Trade-offs**:
- Trusts Markdown content at compile time
- Appropriate for developer-authored content
- Not suitable for user-generated content at runtime (but that's not the use case)

### External Link Handling

**Strategy**: Automatically adds `target="_blank"` to external links
- Prevents navigation away from the SPA
- Improves user experience
- Implemented via URL pattern matching: `^[a-z]+://`

### Code Injection Prevention

**Built-in Protection**:
- YAML front matter is parsed, not executed
- Attribute values are used as-is but validated by Roslyn compiler
- No dynamic code execution or eval

## Extensibility Points

### Custom Front Matter Keys

To add new front matter keys:
1. Extend `FrontMatter` class with new property
2. Update parsing logic in `ParseFrontMatter()`
3. Implement code generation in `GenerateCode()`

### Custom Markdown Extensions

To add Markdig extensions:
1. Modify `MarkdownPipelineBuilder` in MD2Razor constructor
2. Add new extension via `.UseXxx()` method
3. Consider package size impact (extensions are bundled)

### Custom Base Classes

Users can:
- Set per-file base class via `$inherit` front matter
- Set project-wide default via `MD2RazorDefaultBaseClass` MSBuild property

**Common Use Case**: Custom base class for code syntax highlighting (Prism.js integration)

## Testing Strategy

The project includes comprehensive tests:

1. **GeneratedPageTests**: Validates generated component structure
   - Attributes are correctly applied
   - Namespaces are correctly computed
   - Inheritance works as expected

2. **ImportsTests**: Validates _Imports.razor processing
   - Cascading behavior
   - Using directive extraction

3. **MD2RazorTests**: Unit tests for core transformation logic

4. **BuildTests**: Integration tests for MSBuild task

## Known Limitations

### 1. _Imports.razor Parsing

**Limitation**: Simple regex-based parsing
**Impact**: May not handle:
- Comments on same line as @using
- Multi-line directives
- Complex Razor syntax

**Rationale**: Trade-off between simplicity and full Razor parser complexity

**Mitigation**: Documented in README with recommended practices

### 2. Source Generator Debugging

**Challenge**: Source generators run in separate process
**Mitigation**: 
- Use `declarationOnly` mode for testing
- Extensive unit tests
- MSBuild task can be debugged normally

### 3. Hot Reload Support

**Status**: Fully supported by design
- Incremental generation enables hot reload
- File watcher triggers regeneration
- Works with all Blazor hosting models

## NuGet Package Structure

```
MD2RazorGenerator.nupkg
├── analyzers/dotnet/cs/
│   ├── MD2RazorGenerator.dll
│   ├── Markdig.dll
│   ├── YamlDotNet.dll
│   └── System.Memory.dll
├── tools/
│   └── MD2RazorGenerator.MSBuild.Task.dll
├── build/
│   └── MD2RazorGenerator.props/targets
└── README.md
```

**Key Points**:
- Source generator assemblies in `analyzers/` directory (Roslyn convention)
- MSBuild task in `tools/` directory
- All dependencies are private (not transitive to consuming project)
- Marked as `DevelopmentDependency=true`

## Comparison with Alternative Approaches

### Runtime Markdown Libraries

**Examples**: Radzen, Blazorise, MudBlazor Markdown components

**MD2RazorGenerator Advantages**:
1. **Zero Runtime Overhead**: No parsing at runtime
2. **Smaller App Size**: No Markdown library in published output
3. **Better Performance**: Pre-converted HTML
4. **Native Routing**: Each file becomes a routable page
5. **Better IDE Support**: Full IntelliSense via MSBuild task
6. **Markdown Editor Integration**: Work in .md files with full tooling

**Trade-offs**:
- Not suitable for dynamic/user-generated Markdown
- Requires rebuild to see Markdown changes (mitigated by hot reload)

## Future Enhancement Opportunities

### Potential Improvements

1. **Enhanced _Imports.razor Parsing**
   - Use Roslyn's Razor parser
   - Full support for complex directives

2. **Source Link Support**
   - Map generated code back to Markdown files
   - Better debugging experience

3. **Custom Markdig Extensions Configuration**
   - Allow per-project Markdig configuration
   - MSBuild properties for extension selection

4. **Metadata Extraction**
   - Generate manifest of all pages
   - Support for building navigation/TOC

5. **Validation and Diagnostics**
   - Warn about invalid front matter
   - Suggest corrections for common mistakes

## Conclusion

MD2RazorGenerator demonstrates advanced source generator capabilities while maintaining simplicity and performance. The architecture balances compile-time transformation with runtime efficiency, making it ideal for documentation sites, blogs, and content-heavy Blazor applications.

The design prioritizes:
- **Developer Experience**: Natural Markdown editing with full tooling support
- **Performance**: Zero runtime overhead through compile-time generation
- **Flexibility**: Extensive front matter options and base class customization
- **Maintainability**: Clean architecture with well-separated concerns
- **Compatibility**: Broad framework support via netstandard2.0

This architectural approach positions MD2RazorGenerator as a production-ready solution for Markdown-driven Blazor applications.
