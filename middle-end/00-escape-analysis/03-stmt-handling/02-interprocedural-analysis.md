# Interprocedural Analysis

When analyzing control flow involving function calls, our attention is primarily on the transitions between caller and callee contexts. This requires carefully tracking call statements so that we can model the flow of data between the parameters of the caller and those of the callee. A key construct in this process is the multi-assignment statement of the form `OAS2FUNC` (for example: `x, y = f(args)`), where a function, method, or closure is invoked and its results are unpacked into multiple variables.

To systematically handle such assignments, we adopt the following strategy:
- For each variable or expression that appears on the left-hand side (LHS), we create a placeholder, sometimes referred to as a “hole.” These holes represent the yet-to-be-filled destinations for the results of the function call.
- On the right-hand side (RHS), we evaluate the function expression and identify its return parameters.
- Finally, each return value from the function is connected to a corresponding placeholder on the LHS in a one-to-one mapping, ensuring that the flow of values from callee results to caller variables is explicitly modeled.

## Graph Creation

### Call

In interprocedural analysis, we use a Call API that connects the caller’s locations with the callee’s result parameters, much like how control flow edges are built in intraprocedural analysis. This API takes two inputs:
- a hole (dst) representing the target location for results, and
- a node (call expression) representing the function call.

Internal of Call API:

- The call expression is examined to figure out the actual function being invoked. This is done through the static call graph. For instance:
    ```go
    ...
    f = func a() {
        ...
    }
    ...
    x, y = f()
    ...
    ```
    By traversing the call graph, we can determine that `f` refers to the function `a`, so we use the node of `a` for analysis.
* Handle the case where the callee function is unknown.
    * If the callee can be identified statically (`fn != nil`), we perform a precise mapping between the call arguments and the callee’s parameters/results.
    * If the callee is unknown (`fn == nil`), we conservatively assume that the results may escape to the heap. This acts as a safety fallback for cases such as closures flowing into the call site. [Easter Egg]
* For a statically known callee, we construct `tagHole` nodes for each parameter of the callee. These holes are then connected to the corresponding arguments from the caller’s call expression in a one-to-one mapping.

```go
func (e *escape) call(ks []hole, call ir.Node) {
    // Determine the callee function statically
    var fn *ir.Name
    v := ir.StaticValue(call.Fun)
    fn = ir.StaticCalleeName(v)

    fntype := call.Fun.Type()
    if fn != nil {
        fntype = fn.Type()
    }
    // Evaluate callee function expression.
    calleeK := e.discardHole()
    if fn == nil { // unknown callee
        for _, k := range ks {
            if k.dst != &e.blankLoc {
                // The results flow somewhere, but we don't statically
                // know the callee function. If a closure flows here, we
                // need to conservatively assume its results might flow to
                // the heap.
                calleeK = e.calleeHole().note(call, "callee operand")
                break
            }
        }
    }
    e.expr(calleeK, call.Fun)

    args := call.Args
    for i, param := range fntype.Params() {
        e.expr(e.tagHole(ks, fn, param).note(call, "call parameter"), args[i])
    }
}
```

> Note: When the call appears as a standalone statement in the caller body (e.g., `f(args)` without assigning the results), the analysis only inspects `f` intraprocedurally. No edges are created between the caller’s locations and the callee’s result parameters in such cases.

### Tag Hole Creation

During intraprocedural escape analysis of a standalone function, each formal parameter (function argument) is annotated with a param.Note. This annotation records where the parameter’s value might potentially "leak" or escape during the execution of the function.

The analysis captures several possible escape destinations (also referred to as holes):
- Heap Hole: The parameter is stored in the heap, meaning its lifetime must extend beyond the function call.
- Mutator Hole: The parameter is exposed to a mutator (e.g., written through pointers or modified externally), so its escape status cannot be contained within the function.
- Callee Hole: The parameter is passed further down to another function call (i.e., it escapes via a callee).
- Result Holes: The parameter is returned, either directly or indirectly, as part of the function’s result(s). Each return slot is modeled separately, so the parameter can be associated with one or more result holes.

```go
var tagKs []hole
esc := parseLeaks(param.Note)

if x := esc.Heap(); x >= 0 {
    tagKs = append(tagKs, e.heapHole().shift(x))
}
if x := esc.Mutator(); x >= 0 {
    tagKs = append(tagKs, e.mutatorHole().shift(x))
}
if x := esc.Callee(); x >= 0 {
    tagKs = append(tagKs, e.calleeHole().shift(x))
}
// ks are the holes of the identifiers on the LHS of the `OAS2FUNC`
if ks != nil {
    for i := 0; i < numEscResults; i++ {
        if x := esc.Result(i); x >= 0 {
            tagKs = append(tagKs, ks[i].shift(x))
        }
    }
}
```

- `tagKs` collects the set of holes associated with a parameter.
- `shift(x)` adjusts the position/index of the escape, preserving ordering relative to other flows.
- When a parameter escapes into a result hole, the argument’s abstract location is effectively connected to the caller’s variable hole. This means that the caller now observes the escape, since the returned value can carry the parameter’s data outside the function boundary.

**In short:** param.Note is like a compact summary that tells the escape analyzer where a parameter might flow to heap, to another function, to mutators, or to results. During intraprocedural analysis, these notes are expanded into precise hole connections (`tagKs`), which later get used to propagate escape information interprocedurally.

## Assignment Statement

- Op: `OAS2FUNC`
- Statement: `x, y = f(args)`


Before processing the actual function call, the analysis first walks through any Init nodes associated with the call expression `f(args)`. These nodes represent temporary setup operations that must be executed before the function call itself. Just as in intraprocedural analysis, placeholders (or holes) are created for each variable on the left-hand side `(x, y)`. These holes act as symbolic containers where the results of the function call will later be written. The analysis then executes the Call API, passing two things:

- the hole structure created for the LHS, and
- the function call expression node (f(args)).

This step models the function invocation and connects the call’s return values with the placeholders on the assignment’s left-hand side. Finally, the analysis inspects the holes on the LHS (x, y) to see if the underlying storage locations have been reassigned or altered.

```go
case ir.OAS2FUNC:
		n := n.(*ir.AssignListStmt)
		e.stmts(n.Rhs[0].Init())
		ks := e.addrs(n.Lhs)
		e.call(ks, n.Rhs[0])
		e.reassigned(ks, n)
```