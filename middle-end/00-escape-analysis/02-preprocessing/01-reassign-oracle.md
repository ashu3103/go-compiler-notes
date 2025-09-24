# Reassign Oracle

The **Reassign Oracle** is a utility that helps determine whether a local variable within a function is ever reassigned after its initial definition.  
It does this by analyzing function parameters (`PPARAM`) and local variables (`PAUTO`) that meet the following conditions:  

- Their address is **not** taken (otherwise they could be indirectly modified).  
- They are **not** reassigned after the declaration.  

In essence, this oracle identifies variables that are assigned **exactly once** within the function body.

```go
type ReassignOracle struct {
    fn *Func
    // Maps a candidate variable (param/local) to its defining
    // assignment node, or for params, to the defining function.
    singleDef map[*Name]Node
}
```

### fn

Represents the function for which this Reassign Oracle instance is constructed. This provides the scope in which variable assignments are analyzed.

### singleDef

This map stores the definition node for each candidate variable (where a candidate is a parameter or a local variable that is only assigned once at declaration).
- For locals, the map points to the assignment node (`OAS` or `OAS2`) created when the variable was declared.
- For params, the map points directly to the defining function node.

Think of this as a simplified def-use chain, but specialized for variables that are single-assignment only. The oracle effectively captures the initial definition site of such variables.

## Construction of Reassign Oracles

The process begins by scanning through the function body to identify candidate variables that could potentially be reassigned. These candidates include **function parameters** and **local variables**.

### Step 1: Collecting Candidates  

First, we gather all function parameters and local variables:

```go
// collect all the function params
for _, param := range fn.Dcl[:numParams] {
    if IsBlank(param) {
        continue
    }
    // For params, use func itself as defining node.
    ro.singleDef[param] = fn
}
// collect all the local variables
// Walk the function body to discover any locals assigned
// via ":=" syntax (e.g. "a := <expr>").
func(n Node) bool {
    if nn, ok := n.(*Name); ok {
        if nn.Defn != nil && !nn.Addrtaken() && nn.Class == PAUTO { 
            ro.singleDef[nn] = nn.Defn
        }
    }
    return false
}
```

* Parameters (`PPARAM`): Every parameter is considered a candidate, except blank identifiers (_).
Instead of pointing to an assignment statement, parameters are mapped to the function itself as their defining node, since their “definition” happens at the function boundary.
* Locals (`PAUTO`): When walking the function body, any variable declared through `:=` is marked as a candidate.
   * Its Defn field points to the initializing assignment node (the `OAS` or `OAS2` node created during compilation).
   * Variables whose address is taken (Addrtaken) are excluded since they might escape the function scope.

### Pruning Reassigned Variables

Not all collected candidates qualify. If a variable is ever reassigned, it is no longer a single-definition variable and must be removed from consideration.

To detect this, we check if the variable appears on the left-hand side (LHS) of an assignment.

```go
...
case OAS:
    asn := n.(*AssignStmt)
    pruneIfNeeded(asn.X, n)
case OAS2, OAS2FUNC, OAS2MAPR, OAS2DOTTYPE, OAS2RECV, OSELRECV2:
    asn := n.(*AssignListStmt)
    for _, p := range asn.Lhs {
        pruneIfNeeded(p, n)
    }
...
```

- Simple asignments (`OAS`): A variable directly assigned (`x = ...`) is pruned.
- Multiple assignments (`OAS2`, `OAS2FUNC`, etc.): If the variable shows up in any of the left-hand expressions, it’s pruned as well.

## Use Cases

This setup is primarily used to determine the static value of a variable during compilation.  
The **Reassign Oracle** becomes particularly valuable in scenarios where it is important to reason about variable immutability, predictability, and optimizations.  

Some common use cases include:

- **Constant Propagation**: Detecting when a variable always evaluates to the same value, enabling the compiler to replace the variable with that constant directly.  
- **Dead Code Elimination**: Identifying assignments that do not affect the final outcome of the program, allowing the compiler to remove redundant operations.  
- **Escape Analysis Support**: Ensuring that reassignments are properly tracked, which helps in determining whether an object must live on the heap or can remain on the stack. Helps to track and report mutating site
