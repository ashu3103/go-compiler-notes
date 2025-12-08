# Package

## Overview

The `Package` struct holds comprehensive information about the package being compiled. It serves as the central repository for all top-level declarations, functions, imports, and special directives that the compiler needs to process. This struct is typically accessed globally via `typecheck.Target`.

## Structure Definition

```go
type Package struct {
    Imports       []*types.Pkg
    Inits         []*Func
    Funcs         []*Func
    Externs       []*Name
    AsmHdrDecls   []*Name
    CgoPragmas    [][]string
    Embeds        []*Name
    PluginExports []*Name
}
```

### 1. `Imports []*types.Pkg`

**Purpose**: Lists all packages imported by the current package, in source order.

**Possible Values**:
```go
[]*types.Pkg{
    &types.Pkg{Path: "fmt",     Name: "fmt"},
    &types.Pkg{Path: "os",      Name: "os"},
    &types.Pkg{Path: "io",      Name: "io"},
    &types.Pkg{Path: "net/http", Name: "http"},
    &types.Pkg{Path: "encoding/json", Name: "json"},
}
```

**Example Source Code**:
```go
package main

import (
    "fmt"         // → Imports[0]
    "os"          // → Imports[1]
    "io"          // → Imports[2]
    "net/http"    // → Imports[3]
    "encoding/json" // → Imports[4]
)
```

**Use Case**: 
- Dependency tracking
- Import cycle detection
- Linking imported symbols


### 2. `Inits []*Func`

**Purpose**: Contains all package initialization functions (`init`), in source order.

**Possible Values**:
```go
[]*Func{
    &Func{Nname: Name{Sym: "init.0"}, Body: [...]},
    &Func{Nname: Name{Sym: "init.1"}, Body: [...]},
    &Func{Nname: Name{Sym: "init.2"}, Body: [...]},
}
```

**Example Source Code**:
```go
package main

var globalDB *Database

func init() {  // → Inits[0] (init.0)
    log.Println("First init")
    globalConfig = loadConfig()
}

func init() {  // → Inits[1] (init.1)
    log.Println("Second init")
    globalDB = connectDB()
}

func init() {  // → Inits[2] (init.2)
    log.Println("Third init")
    registerHandlers()
}
```

**Use Case**:
- Package initialization order
- Global variable initialization
- Side-effect setup (registrations, connections)

**Note**: Multiple `init()` functions are numbered sequentially (init.0, init.1, etc.)

### 3. `Funcs []*Func`

**Purpose**: Contains **all** functions to be compiled, including:
- Top-level functions
- Methods (including generic instantiations)
- Function literals / closures
- Instantiated generic functions

**Possible Values**:
```go
[]*Func{
    // Top-level function
    &Func{Nname: Name{Sym: "main"},    Body: [...]},
    
    // Method
    &Func{Nname: Name{Sym: "(*Server).HandleRequest"}, Body: [...]},
    
    // Generic function instantiation
    &Func{Nname: Name{Sym: "Process[int]"}, Body: [...]},
    &Func{Nname: Name{Sym: "Process[string]"}, Body: [...]},
    
    // Closure
    &Func{Nname: Name{Sym: "main.func1"}, Body: [...], ClosureVars: [...]},
    &Func{Nname: Name{Sym: "main.func2"}, Body: [...], ClosureVars: [...]},
}
```

**Example Source Code**:
```go
package main

// Top-level function → Funcs[0]
func main() {
    srv := &Server{}
    srv.HandleRequest()
    
    // Closure → Funcs[4]
    handler := func(x int) int {
        return x * 2
    }
    
    Process[int](42)     // → Funcs[2] (instantiation)
    Process[string]("hi") // → Funcs[3] (instantiation)
}

// Method → Funcs[1]
func (s *Server) HandleRequest() {
    // ...
}

// Generic function (generates multiple Funcs entries)
func Process[T any](val T) {
    fmt.Println(val)
}
```

**Use Case**:
- Code generation
- Inlining decisions
- Escape analysis
- SSA generation

**Important**: Closures are added to `Funcs` dynamically during compilation, not just from initial parsing.


### 4. `Externs []*Name`

**Purpose**: Holds **package-scope** declarations:
- Constants
- Non-generic types
- Variables (including `var` blocks)

**Does NOT include**: Generic types (they're handled separately)

**Possible Values**:
```go
[]*Name{
    // Constants
    &Name{Sym: "MaxRetries",   Class: PEXTERN, Defn: &ConstExpr{Val: 3}},
    &Name{Sym: "DefaultPort",  Class: PEXTERN, Defn: &ConstExpr{Val: 8080}},
    
    // Type declarations
    &Name{Sym: "Server",       Class: PEXTERN, Defn: &TypeNode{...}},
    &Name{Sym: "Config",       Class: PEXTERN, Defn: &TypeNode{...}},
    
    // Variables
    &Name{Sym: "globalConfig", Class: PEXTERN, Defn: &AssignStmt{...}},
    &Name{Sym: "logger",       Class: PEXTERN, Defn: &CallExpr{...}},
}
```

**Example Source Code**:
```go
package main

// Constants → Externs[0], Externs[1]
const (
    MaxRetries  = 3
    DefaultPort = 8080
)

// Type declarations → Externs[2], Externs[3]
type Server struct {
    Port int
}

type Config struct {
    Debug bool
}

// Package-level variables → Externs[4], Externs[5]
var (
    globalConfig = &Config{Debug: true}
    logger       = log.New(os.Stdout, "", 0)
)
```

**Use Case**:
- Symbol table construction
- Global variable initialization
- Type checking
- ASM header generation (subset)

### 5. `AsmHdrDecls []*Name`

**Purpose**: Declarations to include in `-asmhdr` output (Go → Assembly interface).

**Only Populated When**: `-asmhdr` compiler flag is set

**Possible Values**:
```go
[]*Name{
    // Exported constants used in assembly
    &Name{Sym: "PageSize",     Defn: &ConstExpr{Val: 4096}},
    &Name{Sym: "CacheLineSize", Defn: &ConstExpr{Val: 64}},
    
    // Struct types with field offsets for assembly
    &Name{Sym: "Context",      Defn: &TypeNode{/* struct layout */}},
    &Name{Sym: "GoroutineInfo", Defn: &TypeNode{/* struct layout */}},
}
```

**Example Source Code**:
```go
package runtime

// Constants for assembly → AsmHdrDecls[0], AsmHdrDecls[1]
const (
    PageSize      = 4096
    CacheLineSize = 64
)

// Struct for assembly offset calculation → AsmHdrDecls[2]
type Context struct {
    PC   uintptr  // offset 0
    SP   uintptr  // offset 8
    Regs [16]uint64 // offset 16
}
```

**Generated ASM Header** (example):
```c
#define PageSize 4096
#define CacheLineSize 64
#define Context__PC 0
#define Context__SP 8
#define Context__Regs 16
```

**Use Case**:
- Hand-written assembly code interfacing with Go
- Low-level runtime operations
- Architecture-specific optimizations

### 6. `CgoPragmas [][]string`

**Purpose**: Stores CGO directives parsed from comments.

**Possible Values**:
```go
[][]string{
    {"#cgo", "CFLAGS:", "-I/usr/local/include"},
    {"#cgo", "LDFLAGS:", "-L/usr/local/lib", "-lssl", "-lcrypto"},
    {"#cgo", "linux", "CFLAGS:", "-DLINUX"},
    {"#cgo", "darwin", "LDFLAGS:", "-framework", "CoreFoundation"},
    {"#cgo", "pkg-config:", "gtk+-3.0"},
}
```

**Example Source Code**:
```go
package main

/*
#cgo CFLAGS: -I/usr/local/include            → CgoPragmas[0]
#cgo LDFLAGS: -L/usr/local/lib -lssl -lcrypto → CgoPragmas[1]
#cgo linux CFLAGS: -DLINUX                    → CgoPragmas[2]
#cgo darwin LDFLAGS: -framework CoreFoundation → CgoPragmas[3]
#cgo pkg-config: gtk+-3.0                     → CgoPragmas[4]

#include <stdlib.h>
#include <openssl/ssl.h>
*/
import "C"

func main() {
    C.SSL_library_init()
}
```

**Use Case**:
- Passing flags to C compiler
- Linking external libraries
- Platform-specific compilation
- pkg-config integration

### 7. `Embeds []*Name`

**Purpose**: Variables with `//go:embed` directives for embedding files into the binary.

**Possible Values**:
```go
[]*Name{
    &Name{
        Sym:   "templateHTML",
        Class: PEXTERN,
        Embed: &EmbedInfo{
            Patterns: []string{"templates/*.html"},
            Files:    []string{"templates/index.html", "templates/about.html"},
        },
    },
    &Name{
        Sym:   "staticFS",
        Class: PEXTERN,
        Embed: &EmbedInfo{
            Patterns: []string{"static/*"},
            Files:    []string{"static/style.css", "static/app.js", "static/logo.png"},
        },
    },
}
```

**Example Source Code**:
```go
package main

import _ "embed"

//go:embed templates/*.html
var templateHTML string  // → Embeds[0]

//go:embed static/*
var staticFS embed.FS    // → Embeds[1]

//go:embed config.json
var configJSON []byte    // → Embeds[2]

//go:embed VERSION
var version string       // → Embeds[3]
```

**Use Case**:
- Embedding static assets
- Configuration files in binary
- Templates and resources
- Version information

**Supported Types**:
- `string` — single file as string
- `[]byte` — single file as bytes
- `embed.FS` — multiple files as filesystem

### 8. `PluginExports []*Name`

**Purpose**: Exported symbols accessible through the plugin API.

**Only Populated When**: 
- `-buildmode=plugin` is set
- Compiling `package main`
- `-dynlink` flag is enabled

**Possible Values**:
```go
[]*Name{
    // Exported functions
    &Name{Sym: "ProcessData",   Class: PFUNC,   Export: true},
    &Name{Sym: "GetVersion",    Class: PFUNC,   Export: true},
    
    // Exported variables
    &Name{Sym: "PluginConfig",  Class: PEXTERN, Export: true},
    &Name{Sym: "PluginMetadata", Class: PEXTERN, Export: true},
}
```

**Example Source Code** (plugin):
```go
package main

// Exported function → PluginExports[0]
func ProcessData(input string) string {
    return strings.ToUpper(input)
}

// Exported function → PluginExports[1]
func GetVersion() string {
    return "1.0.0"
}

// Exported variable → PluginExports[2]
var PluginConfig = map[string]interface{}{
    "name": "MyPlugin",
    "author": "Developer",
}

// Not exported (lowercase) → NOT in PluginExports
func internalHelper() {
    // ...
}
```

**Example Usage** (plugin loader):
```go
package main

import "plugin"

func main() {
    p, _ := plugin.Open("myplugin.so")
    
    // Look up exported function
    processData, _ := p.Lookup("ProcessData")
    fn := processData.(func(string) string)
    result := fn("hello")  // → "HELLO"
    
    // Look up exported variable
    config, _ := p.Lookup("PluginConfig")
    cfg := config.(*map[string]interface{})
}
```

**Use Case**:
- Dynamic plugin loading
- Hot-swappable modules
- Extension systems
- Plugin architectures

## Complete Example Illustration

Given this source file:

```go
package myapp

import (
    "fmt"
    "net/http"
)

const Version = "1.0.0"

type Server struct {
    Port int
}

var globalServer *Server

func init() {
    globalServer = &Server{Port: 8080}
}

func (s *Server) Start() error {
    return http.ListenAndServe(fmt.Sprintf(":%d", s.Port), nil)
}

func main() {
    handler := func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "Version: %s", Version)
    }
    http.HandleFunc("/", handler)
    globalServer.Start()
}
```

The resulting `Package` struct would be populated as:

```go
&ir.Package{
    Imports: []*types.Pkg{
        {Path: "fmt",      Name: "fmt"},
        {Path: "net/http", Name: "http"},
    },
    
    Inits: []*Func{
        {Nname: {Sym: "init.0"}, Body: [...]},  // init function
    },
    
    Funcs: []*Func{
        {Nname: {Sym: "main"},              Body: [...]},
        {Nname: {Sym: "(*Server).Start"},   Body: [...]},
        {Nname: {Sym: "main.func1"},        Body: [...]},  // closure
    },
    
    Externs: []*Name{
        {Sym: "Version",       Defn: &ConstExpr{Val: "1.0.0"}},
        {Sym: "Server",        Defn: &TypeNode{...}},
        {Sym: "globalServer",  Defn: nil},  // initialized in init
    },
    
    AsmHdrDecls:   nil,  // not using -asmhdr
    CgoPragmas:    nil,  // no cgo
    Embeds:        nil,  // no //go:embed
    PluginExports: nil,  // not a plugin
}
```

## Access Pattern

The `Package` struct is typically accessed through a global singleton:

```go
// In cmd/compile/internal/typecheck
var Target *ir.Package

// Usage throughout compiler
for _, fn := range typecheck.Target.Funcs {
    // Process each function
}

for _, imp := range typecheck.Target.Imports {
    // Process each import
}
```

## Compilation Phase Usage

| Phase | Fields Used | Purpose |
|-------|-------------|---------|
| **Parsing** | All fields populated | Initial AST construction |
| **Type Checking** | `Externs`, `Funcs`, `Imports` | Symbol resolution, type inference |
| **Inlining** | `Funcs` | Inline candidate selection |
| **Escape Analysis** | `Funcs`, `Externs` | Determine heap vs stack allocation |
| **Walk** | `Funcs`, `Inits` | Lower high-level IR to low-level |
| **SSA Generation** | `Funcs` | Generate SSA form for optimization |
| **Code Gen** | `Funcs`, `Externs`, `AsmHdrDecls` | Emit machine code |
| **Linking** | `Imports`, `PluginExports` | Resolve external symbols |


## Summary

The `Package` struct is the **central hub** for all package-level information during compilation:

- **Navigation**: Organized by declaration type (functions, externs, imports, etc.)
- **Order Preservation**: `Imports` and `Inits` maintain source order for correctness
- **Completeness**: Contains everything needed from parsing through code generation
- **Extensibility**: Special fields for CGO, embeds, plugins, and ASM headers
- **Global Access**: Typically accessed via `typecheck.Target` singleton

Each field serves a specific purpose in the compilation pipeline, and together they provide a complete representation of the package being compiled.
