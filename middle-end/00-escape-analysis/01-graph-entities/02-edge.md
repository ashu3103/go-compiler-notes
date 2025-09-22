# Edge

In the context of the Go compiler’s escape analysis, an **edge** represents an assignment relationship between two Go variables. It models how the value of one variable (the source) can flow into another variable (the destination), which is essential for tracking whether a value might "escape" from its original scope (e.g., allocated on the stack vs. needing allocation on the heap).  

The structure of an edge is defined as:

```go
type edge struct {
	src    *location
	derefs int
	notes  *note
}
```

### src

A pointer to the location that acts as the source of the value in this assignment. Each location represents an allocation site or abstract memory location tracked by escape analysis.

### derefs

Indicates the number of pointer indirections that occur in this assignment. For example, a simple assignment like `x = y` has `derefs = 0`, while an assignment involving a pointer dereference, such as `x = *p`, has `derefs = 1`. This helps the compiler understand how values flow through references and whether they need to escape to the heap.

## Intuition

By constructing a graph of such edges, the Go compiler can trace how values flow between variables. If a path exists from a local variable to a global variable, heap, or function return, the compiler concludes that the variable escapes and must be heap allocated. Otherwise, it can safely remain on the stack.



