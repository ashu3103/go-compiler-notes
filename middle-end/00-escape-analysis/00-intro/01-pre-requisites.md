# Pre-requisites

From the [overview](https://github.com/ashu3103/go-compiler-notes/blob/main/middle-end/00-escape-analysis/00-intro/00-overview.md), it is already clear that escape analysis is performed on the IR (Intermediate Representation) abstract syntax tree (AST).  

At the lowest level, every IR node can be reduced to one of two fundamental categories:  
- a **literal node** (`OLITERAL`)  
- an **identifier node** (`ONAME`)  

However, since escape analysis is primarily concerned with tracking whether variables (identifiers) escape to the heap or not, we will be dealing almost exclusively with `ONAME` nodes. For this reason, you can safely ignore `OLITERAL` nodes in the context of escape analysis. These `ONAME` and `OLITERAL` are called the ops of a node.

We will also look closely how a function is modelled to able to connect dots later in interprocedural analysis.

## What exactly are ops?

Every IR node has an associated `Op` (ops) field. This field uniquely identifies the type of the node and determines how the compiler should treat it (for example, during traversals, cloning, or transformations of the AST).  

The values of `Op` are constants, conventionally prefixed with a capital **O**. Some common ones include:  
- `ONAME`: identifier nodes (like variables `a`, `b`, `c`)  
- `OLITERAL`: literal values (like numbers `42`, strings `"hello"`)  
- `OAS`: assignment statements  
- `OADD`: addition expressions

To illustrate, consider the simple Go statement:

```go
a = b + c
```

The compiler lowers this into an IR tree which is represented as:

```
        OAS
       /   \
   ONAME    OADD
    (a)    /    \
         ONAME  ONAME
          (b)    (c)
```

So, the leaf node of an IR will always be either `ONAME`, `OLITERAL` or maybe `ONIL`.

## Identifier Node

In the Intermediate Representation (IR), an identifier is modeled using the following structure:

```go
type Name struct {
    BuiltinOp Op              // uint8
    Class     Class           // uint8
    val       constant.Value  // string
    Opt       interface{}
    Defn      Node
    Curfn     *Func
    Heapaddr  *Name
    ...
}
```

The actual `Name` struct contains many more fields, but for escape analysis, these are the most relevant. Let’s focus on understanding them.

### BuiltinOp

Interestingly, not only identifiers, but also literals and types, share this same Name structure.
The BuiltinOp field is what differentiates these different node kinds.
- For escape analysis, the nodes of interest are those with BuiltinOp set to `ONAME`, which indicates they are identifier nodes.
- Other values of BuiltinOp can mark literals, built-in operators, or type nodes, but they aren’t our concern in this particular analysis.

### Class

The Class field captures how an identifier is being used.
Since identifiers can play multiple roles in a program, this field distinguishes their category. Common values include:

- `PAUTO` – local variables
- `PEXTERN` – global variables
- `PAUTOHEAP` – local variables or parameters that have been moved to the heap
- `PPARAM` – input parameters to a function
- `PPARAMOUT` – output parameters (results) of a function
- `PFUNC` – global functions

This classification is crucial because it guides escape analysis in determining where variables live (stack vs heap) and how they interact with function boundaries.

For example, in IR, every identifier used in a return statement is internally represented as an output parameter (PPARAMOUT).

```go
func add(x, y int) int {
    z := x + y
    return z
}
```

The higher representation of the above code will be:

```go
func add(x, y int, ~ret int) {
    var z int
    z = x + y
    ~ret = z
    return
}
```

Here `x` and `y` are `PPARAM` and `~ret` is `PPARAMOUT` while `z` is `PAUTO` (local variable).

> Easter Egg: The global variables are always allocated to heap, have a look at the code below

```go
func (n *Name) OnStack() bool {
	if n.Op() == ONAME {
		switch n.Class {
		case PPARAM, PPARAMOUT, PAUTO:
			return n.Esc() != EscHeap
		case PEXTERN, PAUTOHEAP:
			return false
		}
	}
}
```
### Val

This field holds the **name value** of the identifier, usually stored as a string.  
For example, in the declaration:

```go
var go_is_easy bool
```

the identifier node would carry `"go_is_easy"` as its value in `val`

### Opt

The Opt field refers to the abstract [location](https://github.com/ashu3103/go-compiler-notes/blob/main/middle-end/00-escape-analysis/01-entities/00-location.md) used in escape analysis. It represents the allocation site of the variable — that is, where in the program the variable is created or initialized.

This abstraction helps the compiler reason about whether a value can stay on the stack or must be moved to the heap.

### Defn

The `Defn` field points to the node where the identifier is originally declared. Its value depends on the type of variable or entity:

- Local or global variables: Points to the initializing assignment node (`OAS`, `OAS2`).
- Closure-captured variables: Points to the ONAME node of the outermost variable being captured.
- Function names: Points to the corresponding Func node.

### Curfn

The function, method, or closure in which local variable or param is declared. Typically, this is set to the function/method/closure node that owns the declaration, providing a link between identifiers and their enclosing scope.

## Function Node

In the Intermediate Representation (IR), an identifier is modeled using the following structure:

```go
type Func struct {
	Body Nodes
	Nname    *Name        // ONAME node
	OClosure *ClosureExpr // OCLOSURE node
	Dcl []*Name
	ClosureVars []*Name
	Closures []*Func
    ...
}
```

### Nname

This field refers to the `ONAME` node that represents the identifier of the function or method itself.  
In other words, it stores the function’s own name as an `ONAME` node. 

### OClosure

This field holds the `OCLOSURE` node, which is created when the function is actually a closure.
A closure is a function literal that “captures” variables from its surrounding scope.

```go
func main() {
    x := 42
    f := func(y int) int {
        return x + y
    }
    _ = f(10)
}
```

Here, the function literal `func(y int) int { return x + y }` is represented by an `OCLOSURE` node, and the `OClosure` field of the enclosing node points to it.

### Dcl

This field stores a list of all `ONAME` nodes corresponding to the parameters and local variables declared within the function or closure.

The order of names is significant:

- `PPARAM` (function parameters)
- `PPARAMOUT` (return values)
- `PAUTO` (local variables declared inside the function)

The ordering of PPARAM and PPARAMOUT must match the order of the function’s signature.

Additionally, the Go compiler internally generates names for blank identifiers:
- parameters as `~pNN`
- results as `~rNN`

```go
func divide(a int, b int) (c int, error) {
    quotient := a / b
    return quotient, nil
}
```

For this function, this `Dcl` should look like this:

```
Dcl = [ONAME(a), ONAME(b), ONAME(c), ONAME(~r0), ONAME(quotient)]
       <--- PPRAM ------>  <-----PPARAMOUT---->  <----PAUTO---->
```

This field becomes crucial in escape analysis, since it is used to initialize new locations while analyzing the function body.

### ClosureVars

This field lists the free variables used inside a closure (function literal) that are formally declared in an outer function.

In other words, it represents the closure’s captured variables.
The closure gets its own copy of these variables, which are then referenced within its body.

```go
func outer() func() int {
    x := 10
    return func() int {
        return x + 5
    }
}
```

The IR representation of the above code:

```
Func (outer)
│
├── Nname: ONAME("outer")
├── OClosure: nil
├── Dcl:
│   ├── PPARAM:     []
│   ├── PPARAMOUT:  [~r0: func() int]
│   └── PAUTO:      [x: int]
│
├── ClosureVars: []
│
└── Closures:
    └── Func (closure func() int)
        │
        ├── Nname: ONAME("outer.func1")  // compiler genrated name whenever it encounters a 'func' literal
        ├── OClosure: OCLOSURE node
        ├── Dcl:
        │   ├── PPARAM:    []
        │   ├── PPARAMOUT: [~r0: int]
        │   └── PAUTO:    []
        │
        ├── ClosureVars:
        │   └── [x]       // captured from outer, separate node for closure copy
        │
        └── Closures: []
```

## Init Nodes

Certain statements in Go require additional setup before their main IR (Intermediate Representation) node can be constructed. This setup is expressed using Init nodes, which are extra nodes automatically generated by the compiler.

**Short Variable Definition (`:=`):** A short declaration introduces a new variable on the left-hand side (LHS) and assigns it a value from the right-hand side (RHS). In the IR, the assignment itself is represented as `OAS`. To capture the declaration part, the compiler inserts an extra initialization node (`ODCL`) for the new variable. 

**Function calls (`f(args)`):** represented by IR nodes such as `OCALLFUNC`,`OCALLMETH`, and `OCALLINTER`. All share a common structure. Since Go supports functions as first-class values, a function call’s arguments may themselves be complex expressions or function calls. To handle this safely, the compiler rewrites such calls during the walk phase:
- Complex arguments are first evaluated into compiler-generated temporary variables.
- These evaluations are placed inside the Init list.
- The function’s argument list (Args) is updated to use the temporaries.

```go
f(x[i], g(b))
```

**Before Walk IR:**
```go
OCALLFUNC
   X: f
   Args: [ x[i], g(b) ]
   Init: []
```

**After Walk IR:**
```go
OCALLFUNC
   X: f
   Init: [
       T0 := x[i]  // node
       T1 := g(b)  // node
   ]
   Args: [ T0, T1 ]
```

- At runtime, nodes in Init are executed before the main statement.
- Their results are stored in compiler-generated temporaries.
- The primary IR node (e.g., function call) then operates on these stable temporaries.

In short, Init nodes guarantee that any required side computations are evaluated first in a well-defined order, ensuring correctness and simplifying the IR.

