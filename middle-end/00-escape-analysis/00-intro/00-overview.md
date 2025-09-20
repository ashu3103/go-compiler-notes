# Escape Analysis

In Go, **escape analysis** is a static analysis performed by the compiler to decide where each variable should be allocated — either on the **stack** or on the **heap**.  
The variables considered here include:  
- **Explicit variables**, declared directly using `var` or `:=`.  
- **Implicit variables**, introduced by operations like `new`, `make`, or composite literals.  

## Invariants

Go’s memory model is built on two crucial invariants that escape analysis must enforce:

1. **No heap pointers to stack objects**  
   A stack object’s address must not be stored in the heap, otherwise it could become invalid after the function returns.

2. **No dangling pointers to expired stack objects**  
   A stack object cannot outlive the function (or scope) that owns it, because once the function returns or a loop iteration ends, its stack frame may be destroyed or reused.

Violating either of these would lead to memory unsafety. Escape analysis ensures these invariants are preserved while still maximizing the use of stack allocation for performance.

## Glimpse of the Analysis

Go’s escape analysis is implemented as a **static data-flow analysis** over the compiler’s **IR AST** (intermediate representation).  
> Note: This IR AST is distinct from the syntactic AST. It is a simplified, canonical form of the source program used for later compiler phases.

The core idea is to model variable allocations and their relationships as a **directed weighted graph**:

- **Vertices** represent *locations* (allocation sites of variables).  
- **Edges** represent *assignments* between locations.  
- **Weights** encode how many dereferences are applied during the assignment, calculated as:

`weight = (# of dereferences) - (# of references)`

For example:

```go
p = &q    // -1   (reference)
p = q     //  0   (direct copy)
p = *q    //  1   (dereference once)
p = **q   //  2   (dereference twice)
```

Each assignment adds an edge to the graph, and collectively these edges describe how data may flow through the program.

### Analysis Pipeline

Escape analysis is performed across all functions in the program. The process is broken into batches of functions, where each batch corresponds to a set of functions that may reference each other (e.g., via recursion). The pipeline looks like this:

- **Batching functions:** The call graph is decomposed into strongly connected components (SCCs). Each SCC forms a batch that is analyzed together.

- **Building locations:** The first pass over a batch records all variable declarations and definitions. Each variable is represented as a unique location in the escape graph.

- **Adding edges:** Subsequent passes add edges between locations by analyzing assignments and other dataflow expressions.

- **Solving the graph:** Finally, the graph solver determines how values flow between locations, whether they reach the heap, and if any stack values must “escape” to heap allocation.

### Intraprocedural Analysis

Within a single function, escape analysis inspects the body to identify how variables are used and how their values move around.

- Each assignment (`x = y`, `x = &y`, etc.) creates edges between the corresponding locations.
- Dereferences (`*p`) and references (`&x`) modify the edge weight, influencing whether a variable’s address or value flows to another location.
- Control-flow constructs like loops and conditionals are handled by treating the graph edges as conservative approximations of all possible paths.

### Interprocedural Analysis

Since Go programs are composed of many functions calling one another, escape analysis must also track how data flows across function boundaries.

* For each function, the analysis records how its parameters may flow:
  * To the heap (forcing the argument to escape).
  * To the function’s results (meaning the caller may also see this escape).
* This information is summarized into parameter tags attached to each function.
These tags describe, for example:
  * “param x flows directly to the heap,”
  * “param y may flow into result #0.”
* At call sites, these tags are used to simulate the callee’s behavior without re-analyzing it each time. If the callee is statically known, tags are applied directly; otherwise, a conservative assumption is made (arguments may escape)

## Summary

Escape analysis in Go is a static data-flow analysis used to decide whether variables are allocated on the stack or must escape to the heap.  
The compiler models variables as **locations** and assignments as **edges** in a weighted graph that tracks how values and addresses flow. 

The outcome is a (not so) precise allocation decision: stack allocation whenever safe, and heap allocation only when necessary, while preserving Go’s safety invariants.


