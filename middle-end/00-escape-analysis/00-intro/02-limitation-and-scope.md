# Limitations & Scope

## Pointer Indirections

When analyzing an assignment statement of the form `*X = Y`, the compiler typically treats the left-hand side as a heap hole. This means that if the address of some object flows into `*X`, then that object is conservatively assumed to escape to the heap. for example:

```go
func indirectStore () {
	var c , d int // c should not escape
	var pc* int = &c
	var pd* int = &d
	var ppd** int = &pd
	*ppd = pc
}

func main() {
	indirectStore();
}
```

In this code, `*ppd` is interpreted as a heap hole. Since `pc` (which points to `c`) flows into `*ppd`, the analysis concludes that `pc` must outlive `c`, which causes `c` to be placed on the heap. However, in reality, there is no need for `c` to escape at all.

### Idea

Pointer dereferences are conservatively treated as heap sinks because the analysis cannot always determine what a pointer is referencing (at least not without first building the points-to graph). But we can improve precision by using a reassignment oracle to check whether a pointer variable is assigned only once. Why is it useful?

Since its value never changes, we know with certainty that `ppd` always points to `pd`. By following the right-hand side of this definition, we can resolve the actual target. In turn, instead of flowing the value of `pc` into an abstract “heap hole,” we can redirect it into the precise hole associated with `pd`. This means that the escape of `c` can be avoided, yielding a more accurate and less pessimistic result.

### Limitations

- It works best for simple, single-definition variables. In more complex cases (loops, conditionals, multiple aliases), the analysis may still need to fall back to conservative heap treatment.
- This precision depends on the oracle being sound if reassignments are missed or hidden through aliasing, correctness could be compromised.

## Field Insensitivity

In escape analysis, the compiler does not distinguish between the individual fields of a struct. Instead, it treats all fields as if they were part of the struct object as a whole. This coarse treatment can produce misleading escape results for example:

```go
var sink interface {}
func field () {
    i := 0 // i should not escape
    var x X
    x.p1 = &i
    sink = x.p2
}
```

- The assignment `x.p1 = &i` associates `i`’s address with the struct `x`.
- When `x.p2` is later stored in the global `sink`, the analysis interprets this as if the entire struct `x` (and therefore `&i`) flows to `sink`.
- As a result, the analysis incorrectly concludes that `i` escapes, even though in reality it does not.

### Idea

The root issue is that current analysis does not track flows through individual fields. Instead, it collapses all field flows into the struct itself. A refinement to address the lack of field sensitivity in escape analysis is to attach **synthetic (fake) nodes** to every struct variable in the analysis graph, with each fake node representing a distinct field. Assignments to a particular field would then flow into that field’s corresponding node rather than directly into the struct node, allowing the analysis to distinguish between different fields. To maintain soundness, all field nodes would still connect back to the struct node itself, since if any field escapes the struct as whole has to escape.

### Limitations

- While it helps reduce false positives, it doesn’t completely eliminate them. For example, once a whole struct is copied, all field flows still merge, potentially reintroducing imprecision.
- Adding fake nodes increases the size of the analysis graph which can be an overhead given the returns.
- complexity of one-to-one mapping of fields in copy operations.

> Note: The proposed solutions here are only small adjustments to the existing analysis, in line with Go’s emphasis on keeping compilation straightforward and efficient. Achieving more complex and precise results would require moving towards building full points-to graphs.

## Flow sensitivity

Even with a points-to graph, analyzing only at the IR AST won’t give us flow-sensitive results. For true flow sensitivity, the analysis needs to be performed at the SSA (Static Single Assignment) stage.

**Why SSA enables flow-sensitive results:**
In SSA form, every variable is assigned exactly once, and each new assignment creates a fresh version of that variable. This makes the control flow of values explicit: you can see exactly which definition of a variable reaches each use. Because SSA separates different lifetimes of a variable, it naturally captures changes across program paths, allowing the analysis to respect execution order and produce flow-sensitive information.
