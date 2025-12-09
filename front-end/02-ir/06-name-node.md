# Name Node

Name holds Node fields used only by named nodes (ONAME, OTYPE, some OLITERAL). It represents **named entities** in the program: variables, parameters, constants, types, and functions. It embeds the `miniExpr` structure that provides this structure with default fields like `pos`, `type`, `init` etc. This node stores other important fields like:

## Structural Details

**1. `sym`: Symbol Table Entry**

Points to the symbol table entry containing the identifier's name and package information.

Example:
```go
func example() {
    myVar := 10
}

// Name node
Name{
    sym: &Sym{
        Name: "myVar",
        Pkg:  nil,  // Local symbols have no package
    }
}
```

**2. `BuiltinOp`: Builtin Operation Code**

For builtin functions, stores which builtin operation it represents.

```go
// Source: len(slice)
// The 'len' name node:
Name{
    sym:       &Sym{Name: "len"},
    BuiltinOp: OLEN,  // Indicates this is the len() builtin
}

// Source: append(slice, elem)
// The 'append' name node:
Name{
    sym:       &Sym{Name: "append"},
    BuiltinOp: OAPPEND,
}

// Regular user functions have BuiltinOp = 0 (OXXX)
func myFunc() {}
Name{
    sym:       &Sym{Name: "myFunc"},
    BuiltinOp: OXXX,  // Not a builtin
}
```

**3. `Class`: Storage Class**

Describes where and how a variable/function is stored in memory.


Types of storage class:
* `PEXTERN`: Package-level variables, accessible across function boundaries.
* `PAUTO`: Function-local variables that live on the stack.
* `PAUTOHEAP`: Variables that started as local but escape to heap (outlive their function).
* `PPARAM`: Function input parameters.
* `PPARAMOUT`: Function return values (named or unnamed).
* `PFUNC`: Package-level function declarations.

Example:
```go
// Class: PAUTO
func process() {
    x := 10
    y := x + 20
    result := y * 2   
}

// Name nodes
Name{sym: "x", Class: PAUTO, Curfn: *Func{Name: "process"}}
Name{sym: "y", Class: PAUTO, Curfn: *Func{Name: "process"}}

// Class: PPARAM
func compute(a int, b string, c *Data) int {

}

// Name nodes
Name{sym: "a", Class: PPARAM, Curfn: *Func{Name: "compute"}}
Name{sym: "b", Class: PPARAM, Curfn: *Func{Name: "compute"}}

// Class: PPARAMOUT
func calculate(x int) int {
    return x * 2
}

Name{
    sym:   &Sym{Name: "~r0"},  // Synthetic name
    Class: PPARAMOUT,
}

// Class: PFUNC
func helper() { }
func (s *Server) Start() {} // Method

Name{sym: "helper", Class: PFUNC, Func: *Func{...}}
Name{sym: "(*Server).Start", Class: PFUNC, Func: *Func{...}}
```

**4. `DictIndex`: Generic Dictionary Index**

For generic instantiations, points to the dictionary entry containing type information.

**5. `val`: Constant Value**

Stores the compile-time value for constants.

Example:
```go
const Pi = 3.14159

// Name nodes
Name{
    op:  OLITERAL,
    sym: *Sym{name: "Pi", package: "..."},
    val: constant.Value{Kind: Float, Val: 3.14159},
}
```

> For Non-Constants: `val` is `nil`

**6. `Offset_`: Frame Offset**

Byte offset within the stack frame where the variable is stored.

```go
func calculate(a int64, b int32) {
    x := int64(10)  // 8 bytes
    y := int32(20)  // 4 bytes
    z := int64(30)  // 8 bytes
}

// Name nodes with offsets
Name{sym: "a", Class: PPARAM,    Offset_: -16}  // Before frame
Name{sym: "b", Class: PPARAM,    Offset_: -12}
Name{sym: "x", Class: PAUTO,     Offset_: 0}    // First local
Name{sym: "y", Class: PAUTO,     Offset_: 8}
Name{sym: "z", Class: PAUTO,     Offset_: 12}
```

> Note: Offsets are used in assembly generation to access variables.

**7. `Defn`: Definition/Relationship Node**

Multi-purpose field that establishes relationships between nodes. Its meaning depends on context.

* Local Variable Initialization: Points to the assignment statement that initializes the variable.

    Example:
    ```go
    func example() {
        x := 42  // Assignment: OAS (OpAssign)
    }

    // Name node for 'x'
    Name{
        sym:   "x",
        Class: PAUTO,
        Defn:  &AssignStmt{  // Points to "x := 42"
            Op:  OAS,
            Lhs: Name{sym: "x"},
            Rhs: BasicLit{val: 42},
        },
    }
    ```

* Closure Variable: Points to the original (outermost) captured variable.

    Example:
    ```go
    func outer() {
        x := 10
        
        middle := func() {
            inner := func() {
                println(x)  // References original 'x'
            }
            inner()
        }
        middle()
    }

    // In outer()
    Name{sym: "x", Class: PAUTO}

    // In middle()
    Name{
        sym:   "x",
        Class: PAUTOHEAP,
        Defn:  Name{sym: "x", Class: PAUTO},
        Outer: Name{sym: "x", Class: PAUTO},
    }

    // In inner()
    Name{
        sym:   "x",
        Class: PAUTOHEAP,
        Defn:  Name{sym: "x", Class: PAUTO},
        Outer: Name{sym: "x", Class: PAUTOHEAP},
    }
    ```

* Type Switch Case Variable: Points to the type switch guard (`OTYPESW` node).

    Example:

    ```go
    func process(val interface{}) {
        switch v := val.(type) {
        case int:
            // 'v' here is case-local variable
        case string:
            // 'v' here is different case-local variable
        }
    }

    // Name node for 'v' in each case
    Name{
        sym:   "v",
        Class: PAUTO,
        Defn:  &TypeSwitchGuard{  // OTYPESW
            Tag:  Ident{sym: "v"},
            X:    Name{sym: "val"},
        },
    }
    ```

* Range Variable: Points to the `ORANGE` (range) statement.

    Example:
    ```go
    func iterate(items []int) {
        for i, val := range items {
            println(i, val)
        }
    }

    // Name nodes
    Name{
        sym:   "i",
        Class: PAUTO,
        Defn:  &RangeStmt{  // ORANGE
            Key:   Name{sym: "i"},
            Value: Name{sym: "val"},
            X:     Name{sym: "items"},
        },
    }

    Name{
        sym:   "val",
        Class: PAUTO,
        Defn:  &RangeStmt{...},  // Same ORANGE node
    }
    ```

* Select Receive Variable: Points to the receive assignment (`OSELRECV2`).

    Example:

    ```go
    func example(ch <-chan int) {
        select {
        case val, ok := <-ch:
            println(val, ok)
        }
    }

    // Name nodes
    Name{
        sym:   "val",
        Class: PAUTO,
        Defn:  &AssignListStmt{  // OSELRECV2
            Op:   OSELRECV2,
            Lhs:  []Node{Name{sym: "val"}, Name{sym: "ok"}},
            Rhs:  []Node{&UnaryExpr{Op: ORECV, X: Name{sym: "ch"}}},
        },
    }
    ```

* Function Name: Points to the corresponding `Func` node.

    Example:

    ```go
    func calculate(x int) int {
        return x * 2
    }

    // Name node for function 'calculate'
    Name{
        sym:   "calculate",
        Class: PFUNC,
        Func:  &Func{  // The actual function IR
            Nname: Name{sym: "calculate"},  // Back-reference
            Body:  [...],
            Dcl:   [...],
        },
        Defn: &Func{...},  // Also points to same Func
    }
    ```

**8. `Curfn` Current Enclosing Function**

Points to the function/method/closure in which this variable is declared.

Example:

```go
func process(input int) {
    temp := input * 2
    result := temp + 1
    println(result)
}

// All local variables point to 'process'
Name{sym: "input",  Class: PPARAM, Curfn: &Func{Name: "process"}}
Name{sym: "temp",   Class: PAUTO,  Curfn: &Func{Name: "process"}}
Name{sym: "result", Class: PAUTO,  Curfn: &Func{Name: "process"}}
```

**9. `Heapaddr`: Heap Address Temporary**

When a parameter escapes to heap, `Heapaddr` holds a temporary variable containing the heap address.

Example:

```go
func makeCounter(start int) func() int {
    return func() int {
        start++
        return start
    }
}

// Initial state: 'start' is on stack (PPARAM)
Name{
    sym:   "start",
    Class: PPARAM,
    Offset_: 0,
}

// After escape analysis:
// 1. Heap allocation created
Name_heap := Name{
    sym:      ".autotmp_1",  // Compiler-generated temp
    Class:    PAUTO,
    typ:      *int,          // Pointer to int
    // This holds the heap address
}

// 2. Original 'start' updated
Name{
    sym:      "start",
    Class:    PPARAM,        // Still a parameter
    Heapaddr: &Name_heap,    // Points to heap address temp
}

// 3. Compiler inserts code at function entry:
//    Name_heap = new(int)
//    *Name_heap = start  // Copy param value to heap
```

**10. `Outer`: Closure Variable Chain**

Forms a chain linking closure variables to their immediate parent scope.

Example:

```go
func level1() {
    x := 100  // Original
    
    level2 := func() {
        y := x + 10  // Captures x
        
        level3 := func() {
            z := x + y  // Captures both x and y
            println(z)
        }
        level3()
    }
    level2()
}

// In level1()
x_level1 := Name{
    sym:   "x",
    Class: PAUTO,
    Curfn: &Func{Name: "level1"},
    Defn:  nil,    // Original variable
    Outer: nil,    // No parent
}

// In level2()
x_level2 := Name{
    sym:   "x",
    Class: PAUTOHEAP,
    Curfn: &Func{Name: "level2"},
    Defn:  x_level1,     // Points to original in level1
    Outer: x_level1,     // Immediate parent is also level1
}

y_level2 := Name{
    sym:   "y",
    Class: PAUTO,
    Curfn: &Func{Name: "level2"},
    Defn:  &AssignStmt{...},
    Outer: nil,  // Not a captured variable
}

// In level3()
x_level3 := Name{
    sym:   "x",
    Class: PAUTOHEAP,
    Curfn: &Func{Name: "level3"},
    Defn:  x_level1,     // Still points to original!
    Outer: x_level2,     // Immediate parent is level2's copy
}

y_level3 := Name{
    sym:   "y",
    Class: PAUTOHEAP,
    Curfn: &Func{Name: "level3"},
    Defn:  y_level2,     // Original is in level2
    Outer: y_level2,     // Immediate parent is level2
}
```


## Name Node Constructors

**1. `NewNameAt`: General Purpose Name Node**

Creates a general-purpose `ONAME` node for variables, with optional type information.

Parameters
- `pos` — Source position
- `sym` — Symbol (identifier name)
- `typ` — Type (can be nil; if provided, node is marked as typechecked)

Returns
A `Name` node with `Op = ONAME`

Responsibility
**Caller must set `Curfn`** (the enclosing function) manually.

Example:
```go
// Source: var x int
sym := types.NewSym("x", pkg)
typ := types.Types[TINT]
pos := src.XPos{...}

nameNode := NewNameAt(pos, sym, typ)
// Result:
// Name{
//     pos:   pos,
//     op:    ONAME,
//     sym:   "x",
//     typ:   int,
//     Class: 0,  // Caller must set to PAUTO, PEXTERN, etc.
//     Curfn: nil // Caller must set
// }

// Caller must complete setup:
nameNode.Class = PAUTO
nameNode.Curfn = currentFunc
```

**2. `NewLocal`: Function-Local Variables**

Creates a new **local variable** within a function. This is the primary way to create variables inside function bodies.

Key Features
- Automatically sets `Class = PAUTO`
- Automatically sets `Curfn = fn`
- Automatically adds to `fn.Dcl` (function's declaration list)

Parameters
- `fn` — The function containing this local
- `pos` — Source position
- `sym` — Symbol name
- `typ` — Variable type

Example
```go
func process() {
    x := 10  // Local variable
}

// Compiler creates:
fn := &Func{Nname: Name{sym: "process"}}
xLocal := fn.NewLocal(pos, types.NewSym("x", nil), types.Types[TINT])

// Result:
// Name{
//     sym:   "x",
//     Class: PAUTO,      // Set automatically
//     Curfn: fn,         // Set automatically
//     typ:   int,
// }
// fn.Dcl = []*Name{xLocal}  // Added automatically
```

**3. `NewDeclNameAt`: Generic Declaration Node**

Creates a `Name` node for declarations with a specific opcode. More flexible than `NewNameAt`.

Parameters
- `pos` — Source position
- `op` — Must be one of: `ONAME`, `OTYPE`, `OLITERAL`
- `sym` — Symbol name

Valid Opcodes
- `ONAME` — Variable declaration
- `OTYPE` — Type declaration
- `OLITERAL` — Constant declaration

Example
```go
// Source: type MyInt int
typeSym := types.NewSym("MyInt", pkg)
typeNode := NewDeclNameAt(pos, OTYPE, typeSym)

// Result:
// Name{
//     op:  OTYPE,
//     sym: "MyInt",
// }
```

**4. `NewConstAt`: Constant Literal**

Creates a named constant with a compile-time value.

Parameters
- `pos` — Source position
- `sym` — Symbol name
- `typ` — Constant type
- `val` — Constant value (from `go/constant` package)

Characteristics
- Always has `Op = OLITERAL`
- Marked as typechecked automatically
- Stores the actual constant value

Example
```go
// Source: const MaxRetries = 5
sym := types.NewSym("MaxRetries", pkg)
val := constant.MakeInt64(5)
constNode := NewConstAt(pos, sym, types.Types[TINT], val)

// Result:
// Name{
//     op:  OLITERAL,
//     sym: "MaxRetries",
//     typ: int,
//     val: 5,
//     Typecheck: 1,
// }
```

**5. `NewBuiltin`: Builtin Functions**

Creates a `Name` node for builtin functions like `len`, `make`, `append`, etc.

Parameters
- `sym` — Symbol name (e.g., "len", "make")
- `op` — Builtin operation code (e.g., `OLEN`, `OMAKE`)

Characteristics
- Has `BuiltinOp` set to the operation
- No position (uses `src.NoXPos`)
- Automatically registers in symbol table

Example
```go
lenSym := types.NewSym("len", types.BuiltinPkg)
lenNode := NewBuiltin(lenSym, OLEN)

// Result:
// Name{
//     op:        ONAME,
//     sym:       "len",
//     BuiltinOp: OLEN,
//     Typecheck: 1,
// }

// Usage in code:
// arr := [5]int{1,2,3,4,5}
// n := len(arr)  // References this Name node
```

**6. `NewClosureVar`: Closure Captured Variables**

Creates a **closure variable** that captures an outer variable. This is used when a nested function (closure) references a variable from an enclosing scope.

Parameters
- `pos` — Source position where capture occurs
- `fn` — The closure/nested function doing the capturing
- `n` — The outer variable being captured

Restrictions
Can only capture:
- `PAUTO` — Local variables
- `PPARAM` — Parameters
- `PPARAMOUT` — Return values
- `PAUTOHEAP` — Already-captured variables

Cannot capture:
- `PEXTERN` — Global variables (no need to capture)

Example
```go
// Source
func outer() {
    x := 10  // Original variable
    
    closure := func() {
        println(x)  // Captures 'x'
    }
    closure()
}

// Compiler creates:
// In outer():
xOuter := fn_outer.NewLocal(pos, sym_x, types.Types[TINT])
// Name{sym: "x", Class: PAUTO, Curfn: fn_outer}

// In closure:
xCaptured := NewClosureVar(pos, fn_closure, xOuter)
// Result:
// Name{
//     sym:   "x",
//     Class: PAUTOHEAP,        // Moved to heap
//     Curfn: fn_closure,       // Belongs to closure
//     Defn:  xOuter,           // Points to original
//     Outer: xOuter,           // Immediate parent
//     IsClosureVar: true,
// }

// fn_closure.ClosureVars = [xCaptured]
```

**7. `NewHiddenParam`: Hidden Parameters**

Creates a **hidden parameter** for passing runtime context or type dictionaries. These are parameters that don't appear in the source code but are added by the compiler.

What Are Hidden Parameters?

Hidden parameters are compiler-generated parameters used for:

1. **Closure Context** — Pointer to captured variables
2. **Method Receivers** — For method calls (though usually explicit)
3. **Type Dictionaries** — For generic function instantiations
4. **Interface Method Context** — For interface method dispatch

Restrictions
- **Cannot be used on closures** — Closures capture via `ClosureVars`, not hidden params
- Only for "real" functions

Example

```go
// Source
func makeCounter() func() int {
    count := 0
    return func() int {
        count++
        return count
    }
}

// The closure needs access to 'count'
// Compiler creates hidden parameter for context:

closureFn := /* the closure function */
ctxtSym := types.NewSym(".ctxt", nil)
ctxtType := types.NewPtr(closureContextType)

hiddenCtxt := NewHiddenParam(pos, closureFn, ctxtSym, ctxtType)

// Result:
// Name{
//     sym:   ".ctxt",
//     Class: PAUTOHEAP,
//     typ:   *closureContext,
//     Curfn: closureFn,
//     Byval: true,
// }

// Closure function signature (conceptually):
// func closure(ctxt *closureContext) int {
//     count := &ctxt.count
//     (*count)++
//     return *count
// }
```





## Utility Functions

**Canonical**

Returns the **original (outermost) declaration** of a variable. For closure variables, it follows the chain back to the original declaration. For regular variables, it returns itself.

Use Cases:
1. **Finding the original variable** in nested closures
2. **Setting properties** that must be on the original
3. **Escape analysis** to track the root variable

Example:
```go
func example() {
    x := 10
}

xLocal := /* Name node for x */

canonical := xLocal.Canonical()
// Returns xLocal itself (not a closure var)
// canonical == xLocal  // true
```

**SameSource**

Checks if two nodes refer to the same source element (variable, type, constant). Future-proof for `IdentExpr` migration.


Example 1: Same Variable Reference
```go
func example() {
    x := 10
    y := x + x  // Both 'x' refer to same Name node
}

leftX := /* left operand */
rightX := /* right operand */

SameSource(leftX, rightX)  // true - same variable
```

Example 2: Different Variables
```go
func example() {
    x := 10
    y := 20
    z := x + y
}

xNode := /* 'x' Name */
yNode := /* 'y' Name */

SameSource(xNode, yNode)  // false - different variables
```

**Uses**

Checks if expression `x` is a **direct use** of variable `v`.

Example:

```go
func process(data []int) {
    n := len(data)  // Uses 'data'
}

dataParam := /* Name for 'data' */
lenArg := /* argument to len() */

Uses(lenArg, dataParam)  // true - direct use
```

**DeclaredBy**

Checks if expression `x` refers to a variable that was declared by statement `stmt`.

Use Cases
- **Dead code elimination**: Check if variable is used
- **Initialization tracking**: Find where variable is defined
- **SSA construction**: Link uses to definitions

Example:

```go
func example() {
    x := 42     // Declaration statement
    y := x + 1  // Uses x
}

declStmt := /* x := 42 (OAS) */
xInUse := /* 'x' in 'x + 1' */

DeclaredBy(xInUse, declStmt)  // true
```

## Flags

**nameReadonly**

Marks that a variable contains **read-only data** (cannot be modified at runtime).

When Set:
- String literals
- Constant data embedded in binary
- Read-only global variables

Example:
```go
const greeting = "Hello, World!"

// Compiler marks as readonly:
greetingNode := NewConstAt(pos, sym, stringType, val)
greetingNode.MarkReadonly()

// Result:
// - Stored in read-only data section
// - Shared across all uses
// - Write attempt causes segmentation fault
```

**nameByval**

Controls whether a closure variable is captured **by value** (copy) or **by reference** (pointer).

When Set
- Variable is never modified in closure - by value
- Variable is modified in closure - by reference (false)

Example:

```go
func makeAdder(x int) func(int) int {
    return func(y int) int {
        return x + y  // x never modified
    }
}

// x captured by value:
xParam.Canonical().SetByval(true)

// Closure struct:
// struct {
//     x int  // Copy of x
// }
```

**nameNeedzero**

Indicates variable **must be zeroed** on function entry (contains pointers, interfaces, or other types requiring initialization).

When Set:
- Variable type contains pointers
- Variable type has interface fields
- Variable type requires zero-initialization

Example:

```go
func process() {
    var p *int  // Must be zeroed
}

// Compiler sets:
pVar.SetNeedzero(true)

// Generated code:
// func process() {
//     var p *int
//     p = nil  // Zero initialization
// }
```

**nameAutoTemp**

Marks variable as **compiler-generated temporary** (not in source code).

Effects
- Omitted from debug info (DWARF)
- Different naming conventions
- Reset if variable escapes to heap

When Set
- Expression decomposition
- Intermediate results
- Lowering complex expressions

Example 1:
```go
// Source
result := expensive1() + expensive2() + expensive3()

// Compiler creates temporaries:
tmp1 := fn.NewLocal(pos, types.NewSym(".autotmp_1", nil), retType1)
tmp2 := fn.NewLocal(pos, types.NewSym(".autotmp_2", nil), retType2)
tmp1.SetAutoTemp(true)
tmp2.SetAutoTemp(true)

// Becomes:
// tmp1 = expensive1()
// tmp2 = expensive2()
// tmp3 = tmp1 + tmp2
// result = tmp3 + expensive3()
```

Example 2:
```go
// Source
for i, v := range slice {
    process(i, v)
}

// Compiler creates:
iterVar := fn.NewLocal(pos, types.NewSym(".iter", nil), intType)
iterVar.SetAutoTemp(true)

// Becomes:
// for .iter = 0; .iter < len(slice); .iter++ {
//     i = .iter
//     v = slice[.iter]
//     process(i, v)
// }
```

**nameUsed**

Tracks whether a variable is **actually used** in the code (for "declared but not used" errors).

When Set
- Variable is read
- Variable is assigned (after declaration)
- Variable's address is taken

Example:
```go
func example() {
    x := 10  // Declared but never used
}

// Compiler:
xVar := fn.NewLocal(...)
// ... scan function body ...
// xVar.Used() == false

// Error: "x declared but not used"
```

**nameIsClosureVar**

Marks that this variable is a **closure variable** (captured from outer scope).

Example:
```go
func outer() {
    x := 10
    
    inner := func() {
        println(x)
    }
}

// In inner():
xCaptured := NewClosureVar(pos, fn_inner, xOuter)
// xCaptured.IsClosureVar() == true
// xCaptured.Class == PAUTOHEAP
```

**nameIsOutputParamHeapAddr**

Marks temporary variable holding **heap address of escaped return parameter**.

When Used:
- Named return values that escape to heap need special handling.

Example:

```go
func create() (result *int) {
    return &someGlobal  // result escapes
}

// Compiler creates temporary:
heapAddr := fn.NewLocal(pos, types.NewSym(".heapaddr", nil), ptrType)
heapAddr.SetIsOutputParamHeapAddr(true)

// Generated:
// func create() (result *int) {
//     .heapaddr = &heap_allocated_result
//     result = .heapaddr
//     return
// }
```

**nameIsOutputParamInRegisters**

Marks that output parameter is **returned in registers**

When Set
- Function uses register-based calling convention
- Return value fits in registers

Example:
```go
func calculate() (int, int) {
    return 42, 100
}

// Both returns fit in registers:
ret0.SetIsOutputParamInRegisters(true)
ret1.SetIsOutputParamInRegisters(true)
```

**nameAddrtaken**

Marks that **variable's address is taken** 

Effects:
- Prevents certain optimizations
- May force heap allocation
- Affects escape analysis

Example:

```go
func example() {
    x := 10
    p := &x  // Address taken
}

// Compiler sets:
xVar.SetAddrtaken(true)
// x might escape to heap
```

**nameAlias**

Marks type name as **type alias** (not a new definition)

Example:

```go
type MyInt = int  // Alias

// Compiler marks:
myIntNode.SetAlias(true)
// MyInt and int are the same type
```

**nameNonMergeable**

Prevents **stack slot merging** optimization.

When Set
- Variable lifetimes overlap in complex ways that prevent safe merging.

Example:

```go
func complex() {
    if condition {
        x := expensive1()
        use(x)
    }
    if otherCondition {
        y := expensive2()
        use(y)
    }
}

// Normally x and y could share stack slot
// But if lifetimes are complex:
yVar.SetNonMergeable(true)
```
