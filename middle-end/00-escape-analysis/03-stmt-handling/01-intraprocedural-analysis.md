# Intraprocedural Analysis

Here, we will be analyzing a specific subset of program statements. The focus will be on understanding how these statements are represented in the intermediate form and how they influence variable tracking or flow analysis. Each kind of statement will be broken down into its operation type (Op), its syntactic form, and the corresponding handling logic.

When handling assignment statements, we generate a placeholder (hole) for every expression appearing on the left-hand side (LHS). The corresponding right-hand side (RHS) values are then directed into these holes in a one-to-one manner.

For example, consider:
```go
x, y = a, b
```

- We first create individual holes for `x` and `y`.
- Next, we connect the location of `a` to the hole for `x`, and the location of `b` to the hole for `y`.

## Graph Creation

### Flow

Flow is an API that models the creation of an edge between two locations: a hole (dst) and a source (src). The logic works as follows:

- If the hole’s address is already marked as taken, propagate that marking to the source.
- If the hole’s destination has the `attrEscapes` attribute and the operation represents an address-of (`dst = &src`), then propagate escape-related attributes (`attrEscapes`, `attrPersists`, `attrMutates`, `attrCalls`) to the source and stop further processing.
- Otherwise, record the connection by appending an edge from the source to the hole’s destination.

```go
func (b *batch) flow(k hole, src *location) {
	if k.addrtaken {
		src.addrtaken = true
	}

	dst := k.dst
	if dst.hasAttr(attrEscapes) && k.derefs < 0 { // dst = &src
		src.attrs |= attrEscapes | attrPersists | attrMutates | attrCalls
		return
	}
	dst.edges = append(dst.edges, edge{src: src, derefs: k.derefs, notes: k.notes})
}
```

### Hole Creation

For every node appearing on the left-hand side (LHS), we begin by initializing its hole as `heapHole`. After that, the actual hole is determined depending on the operator (`Op`) of the node.

Suppose we are analyzing a node `n` (which is an isolated LHS expression) to determine its hole value:

- **Identifier (`ONAME`)**:  
  If the identifier belongs to the `PEXTERN` class (i.e., it is a global variable), then the hole is simply `heapHole`. Otherwise, the hole corresponds to the memory location of that identifier.  
- **Selector (`ODOT`, written as `X.Sel`)**:  
  The hole is obtained by computing the hole for the base expression `X`. However, this approach is *field-insensitive* because it treats `X.Sel` as equivalent to just `X`, ignoring which specific field is being accessed.  
- **Dereference (`ODEREF`, written as `*X`)**:  
  In this case, the base expression `X` is considered mutable, and the hole is set to `heapHole`. This introduces imprecision, because dereferencing effectively points to an unknown or potentially arbitrary memory location. Without strong alias information, it is conservative to assume the access could target any heap location.  
- **Pointer Selector (`ODOTPTR`, written as `X.Sel` where `X` is a pointer to a struct)**:  
  This is treated the same way as dereference: we take the hole as `heapHole` for the same reason of potential imprecision.  
- **Indexing of an array (`OINDEX`, written as `X[ind]` where `X` is an array)**:  
  The hole is taken as the hole of the underlying array `X`, since accessing an array element is essentially accessing a part of the array.  
- **Indexing of a slice (`OINDEX`, written as `X[ind]` where `X` is a slice)**:  
  This is handled in the same way as dereference, because slices internally behave like pointers to underlying arrays. Hence, the hole defaults to `heapHole`.  

### Location Flow

For each node on the **RHS** of an assignment that has a corresponding **hole** (see *Hole Creation*), we create an edge from the node to its hole.  
The **edge weight** encodes the level of pointer indirection.  

Given an expression node `n` (RHS) and its corresponding hole `k`, the edge creation rules are:

- **Literal / Nil** (`OLITERAL`, `ONIL`): No edge created.  
- **Binary Expressions** (`OADD`, `OSUB`, `OOR`, `OANDAND`, …): No edge created.  
- **Identifier** (`ONAME`): Flow the variable’s location → hole `k`, weight = **0**.  
- **Address-of** (`OADDR`, `&X`): Flow the *address of `X`* → hole `k`, weight = **-1**.  
- **Selector** (`ODOT`, `ODOTMETH`, e.g., `X.Sel`): Flow the location of `X` → hole `k`, weight = **0**.  
- **Pointer Selector** (`ODOTPTR`, e.g., `X.Sel` where `X` is a pointer): Flow the *dereferenced location of `X`* → hole `k`, weight = **1**.  
- **Array Indexing** (`OINDEX`, `X[ind]` where `X` is an array): Flow the array’s location → hole `k`, weight = **0**.  
- **Slice Indexing** (`OINDEX`, `X[ind]` where `X` is a slice): Flow the *dereferenced slice location* → hole `k`, weight = **1**.  
- **Heap Allocation** (`ONEW`, `new(X)`): Create a new location for node `n`, then flow its *address* → hole `k`, weight = **-1**.  


## Variable Declaration

- Op: `ODCL`
- Statement: `var X <type>`

When the analysis encounters a variable declaration, the process is straightforward: we return the location that has already been assigned to the variable. This ensures that the declared variable is properly associated with its storage location in the analysis context.

```go
loc := e.oldLoc(n)
loc.loopDepth = e.loopDepth
return loc.asHole()
```

## Assignment Statements

- Op: `OAS`, `OAS2`, `OAS2DOTTYPE`, `OAS2MAPR`
- Statement: `x = y`, `x, y = a, b`, `v, ok = x.(type)`, `v, ok = m[k]`

When the analysis processes an assignment statement, it first creates holes for the nodes on the left-hand side (LHS). These holes act as placeholders representing the storage locations that will receive new values.
- For each LHS variable, the analysis uses the Flow API to establish dataflow edges from the corresponding right-hand side (RHS) expressions. This models the movement of values:
- In a simple assignment like `x = y`, an edge is created from the node representing `y` to the hole representing `x`.
- In multiple assignments like `x, y = a, b`, separate edges are created from `a → x` and `b → y`.
- For type assertions `(v, ok = x.(type))`, edges are added from the asserted expression `x` to the new variables `v` and `ok`.
- For map lookups (`v, ok = m[k]`), edges are created from the map access result to the target variables.