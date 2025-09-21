# Location

A location is an abstract location that stores a Go variable, the location is in fact the vertex of our escape analysis graph.

```go
type location struct {
	n         ir.Node  // represented variable or expression, if any
	curfn     *ir.Func // enclosing function, the function to which the variable belong
	edges     []edge   // incoming edges
	loopDepth int      // loopDepth at declaration

	// resultIndex records the tuple index (starting at 1) for
	// PPARAMOUT variables within their function's result type.
	// For non-PPARAMOUT variables it's 0.
	resultIndex int

	// derefs and walkgen are used during walkOne to track the
	// minimal dereferences from the walk root.
	derefs  int // >= -1
	walkgen uint32

	// dst and dstEdgeindex track the next immediate assignment
	// destination location during walkone, along with the index
	// of the edge pointing back to this location.
	dst        *location
	dstEdgeIdx int

	// queuedWalkAll is used by walkAll to track whether this location is
	// in its work queue.
	queuedWalkAll bool

	// queuedWalkOne is used by walkOne to track whether this location is
	// in its work queue. The value is the walkgen when this location was
	// last queued for walkOne, or 0 if it's not currently queued.
	queuedWalkOne uint32

	// attrs is a bitset of location attributes.
	attrs locAttr

	// paramEsc records the represented parameter's leak set.
	paramEsc leaks

	captured   bool // has a closure captured this variable?
	reassigned bool // has this variable been reassigned?
	addrtaken  bool // has this variable's address been taken?
	param      bool // is this variable a parameter (ONAME of class ir.PPARAM)?
	paramOut   bool // is this variable an out parameter (ONAME of class ir.PPARAMOUT)?
}
```

Time to discuss some of the important fields of a `location` that are not very straightforward.

### edges

Think of [**edges**](https://github.com/ashu3103/go-compiler-notes/blob/main/middle-end/00-escape-analysis/01-entities/02-edge.md) as arrows showing "who is feeding values into whom."  
Each edge captures an assignment where the variable at the current location is updated from another location. Along with the source, it records how many `&` or `*` indirections are involved (the *weight* of the edge).

For example:

```go
func demo() {
	var x *int
	var y, z int

	x = &y       // x now points to y
	if someCond {
		x = &z   // x may also point to z
	}
}
```

- Edge from `y → x` (via address-of `&y`)
- Edge from `z → x` (via address-of `&z`)

### loopDepth

Now imagine loopDepth as a nesting counter that tracks how deep you are inside looping constructs.
- Enters a for loop: loopDepth++
- Hits a backward goto (unstructured loop): loopDepth++
- Exits any loops: loopDepth--

This depth matters in escape analysis because objects allocated inside a loop can easily “outlive” the loop body if their reference is stored outside.

```go
var l *int   // loopDepth = 0
for {
	l = new(int)  // loopDepth = 1
}
```

- Each new(int) creates a fresh object.
- The pointer l survives every iteration since it’s declared outside the loop.
- Escape analysis realizes these objects cannot stay on the stack because their lifetime exceeds the loop body.

### attrs  

A bitmap used to capture the state of a location. Each bit represents one of the predefined [**attributes**](https://github.com/ashu3103/go-compiler-notes/blob/main/middle-end/00-escape-analysis/01-entities/00-attributes.md) (such as whether the location is on the stack, escapes to the heap, is read-only, etc.). This compact representation makes it efficient to check or update the state of a location during analysis.  

### paramEsc  

This field plays a key role in **interprocedural escape analysis**. It encodes how a function parameter is treated with respect to memory escape. Specifically, it describes whether a parameter:  

- Escapes to the **heap** (e.g., stored in a global or returned by reference).  
- Flows to a **mutator** (i.e., it may be modified by the function).  
- Propagates to a **callee** (passed further down to another function).  
- Is forwarded to one of the function’s **result parameters**.  

By tracking this, the compiler can reason about whether a function parameter stays safely on the stack or must be allocated on the heap. This allows optimizations like avoiding unnecessary heap allocations when a parameter doesn’t actually escape.  

#### Example  

Consider the following Go function:  

```go
func storeGlobally(p *int) {
    global = p
}
```

- Here, the parameter `p` escapes because it is assigned to a global variable.
- Escape analysis will set `paramEsc` for `p` to indicate flow-to-heap.

another example:

```go
func addOne(x *int) *int {
    *x = *x + 1
    return x
}
```

- In this case, the parameter `x` does not escape to the heap.
- Instead, it flows directly to the result parameter (the return value).
- `paramEsc` records this flow-to-result relationship.

This tracking is what makes interprocedural escape analysis compositional: the compiler knows how parameters behave across function boundaries, so it can reuse this information when analyzing call sites.

## New Location Creation

```go
func (e *escape) newLoc(n ir.Node, persists bool) *location {
	loc := &location{
		n:         n,
		curfn:     e.curfn,
		loopDepth: e.loopDepth,
	}
	if loc.isName(ir.PPARAM) {
		loc.param = true
	} else if loc.isName(ir.PPARAMOUT) {
		loc.paramOut = true
	}

	if persists {
		loc.attrs |= attrPersists
	}
	e.allLocs = append(e.allLocs, loc)
	if n != nil {
		if n.Op() == ir.ONAME {
			n := n.(*ir.Name)
			if n.Opt != nil {
				base.FatalfAt(n.Pos(), "%v already has a location", n)
			}
			n.Opt = loc
		}
	}
	return loc
}
```

Steps of creating a new location:

- For every variable node (`ONAME`), create a corresponding **location object**.  
- If the variable is a **parameter**, set its `param` field to `true`.  
  If it is a **result parameter (function output)**, set the `paramOut` field to `true`.  
- Add this newly created location to the `allLocs` slice.  
  This slice is initialized per function and is used to batch together all variable declarations collected during function setup.  
- Finally, establish a **back pointer** from the node to this location (if the node does not already have one).
