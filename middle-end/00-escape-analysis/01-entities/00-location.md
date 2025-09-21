# Location

A location is an abstract location that stores a Go variable, the location is in fact the vertex of our escape analysis graph.

```go
type location struct {
	n         ir.Node  // represented variable or expression, if any
	curfn     *ir.Func // enclosing function, the function to which the variable belong
	edges     []edge   // incoming edges, includes details about the other places whose values are transferred to this one.
	loopDepth int      // loopDepth at declaration, generally used to model the scope of a variable

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

// Talk about the Function IR Node and the ONAME

Now since we are talking about variables we have to have the idea of how identifiers are modelled in IR AST. The 

## Pre-requisites

Now, we know that escape analysis is performed on a 