# Attributes

In the Go compiler’s escape analysis, **Attributes** describe properties of a variable (or more precisely, a memory location). Think of them as metadata that guides the compiler’s decisions about whether a variable should live on the stack, be promoted to the heap, or whether certain optimizations are safe. These attributes collectively capture the "lifetime" and "safety properties" of variables in a program.

## Types of Attributes

### attrEscapes

`attrEscapes` specifies whether the address of a variable "escapes" its declaring function.  
If a variable’s address escapes, it means the variable cannot safely live on the stack and must instead be heap-allocated.

**Example:**  
```go
func f() *int {
    x := 10
    return &x // x escapes because its address is returned
}
```

Here, `x` is marked with `attrEscapes` because its lifetime extends beyond the function.

### attrPersists

`attrPersists` specifies whether the address of an expression survives beyond the statement in which it is created. In such cases, the compiler cannot immediately reuse that memory.

```go
func g() {
    s := "hello"
    p := &s[0] // address of s[0] persists beyond the statement
    _ = p
}
```

The pointer `p` holds onto the memory of `s`, meaning the storage cannot be discarded right after the statement.

### attrMutates

`attrMutates` indicates whether pointers reachable from a variable’s location may be used to mutate the underlying memory. This attribute is crucial for detecting unsafe conversions, such as when immutable strings are converted into mutable byte slices.

```go
func h(s string) []byte {
    b := []byte(s) // mutation possible if optimization is not safe
    b[0] = 'H'
    return b
}
```

Here, `attrMutates` ensures the compiler accounts for potential mutations of the memory originally referenced by `s`.

This is actually used in conversion optimisations as it would tell the compiler not to make a separate copy of an object while converion (example: string->[]byte), because the object will not be mutated.

### attrCalls

`attrCalls` specifies whether closures reachable from a given variable might be called. This allows the compiler to better optimize scenarios involving indirect closure calls, without fully tracking their return values. In simple words, having this attribute just suggests that the location (that's been invoked) may hold a closure, so the compiler can handle its side-effects.

```go
func i() {
    f := func() int { return 42 }
    g := f // closure captured
    g()    // closure may be invoked later
}
```

The variable `g` would be marked with `attrCalls` since the closure stored in `f` can be indirectly invoked.

## Closure Mutation

The attributes `attrCalls` and `attrMutates` together helps escape analysis move desired objects to heap.
Just recall:

- attrMutates → “Pointers reachable from this location can mutate memory.”
- attrCalls → “This location may hold a closure that can be invoked.”

Example:

```go
func counter() func() int {
    x := 0
    return func() int {
        x++        // (1) closure mutates x
        return x
    }
}
```

- `x` is declared on the stack inside counter.
- The returned closure captures `x`.
- Since the closure is returned and called later, `x` must live longer than counter’s stack frame, `x` escapes to heap.

So, for x:
- `attrMutates` is set because `x++` mutates `x`.
- Escape analysis must ensure that `x` isn’t placed in stack memory that could be reused after counter returns.

For the closure location (the return value):
- `attrCalls` is set because the closure may be invoked outside counter.
- Escape analysis must track that the closure’s environment (including `x`) must remain valid when invoked.

So both of these attributes are used in associativity with each other to make precise judgements.