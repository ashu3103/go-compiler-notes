# Walk One

The `walkOne` function is responsible for computing the **minimal number of dereferences** from a given root location to all other locations in the program’s data flow graph. This helps the escape analysis determine whether objects can safely remain on the stack or need to be allocated on the heap.

Because the data flow graph can contain **negative edges** (introduced by addressing operations like `&x`), `walkOne` uses the **Bellman-Ford algorithm** to propagate dereference counts. Negative cycles are not an issue here because the algorithm clamps dereference counts to a lower bound of 0, ensuring the counts remain meaningful.

> Dereference Count: The dereference count from one location (`root`) to other.

## Bellman Ford Style Walk

Starting from a root location in a weighted graph (where vertices are locations and edges carry dereference costs), the goal is to compute the minimal dereference count (analogous to shortest path cost) from the root to every other location.

The process works as follows:

* Initialization:
    * Set `root.walkgen` to the current traversal identifier.
    * Assign `root.derefs = 0` (since the root has no cost to reach itself).
    * Clear `root.dst`.
    * Initialize a work queue todo and push root into it.
* Iterative Walk:
  While the queue is not empty:
    * Remove a location `l` from the queue.
    * Mark `l` as no longer pending for processing.
    * Record its current dereference count (`derefs`) and compute any new attributes that will be propagated further.
* Neighbor Relaxation:
  For each outgoing edge of `l`:
    * Skip the edge if its source already has the `attrEscapes` attribute.
    * Otherwise, compute the tentative dereference value `d = l.derefs + edge.derefs`.
    * If the edge’s source location (`edge.src`) has not been visited in this traversal (`walkgen`) or can now be reached with fewer dereferences (`edge.src.derefs > d`):
        * Update its walkgen to the current traversal.
        * Set its derefs to `d`.
        * Record `l` as the predecessor (`dst`) along with the index of the edge used.
        * If not already queued, enqueue the source location for further exploration.

```go
root.walkgen = walkgen
root.derefs = 0
root.dst = nil

todo := newQueue(1)
todo.pushFront(root)

for todo.len() > 0 {
    l := todo.popFront()
    l.queuedWalkOne = 0 // no longer queued for walkOne

    derefs := l.derefs
    var newAttrs locAttr

    analysis() // in next section

    for i, edge := range l.edges {
        if edge.src.hasAttr(attrEscapes) {
            continue
        }
        d := derefs + edge.derefs
        if edge.src.walkgen != walkgen || edge.src.derefs > d {
            edge.src.walkgen = walkgen
            edge.src.derefs = d
            edge.src.dst = l
            edge.src.dstEdgeIdx = i
            if edge.src.queuedWalkOne != walkgen {
                edge.src.queuedWalkOne = walkgen
                todo.pushBack(edge.src)
            }
        }
    }
}
```

## Attribute Propagation

- Address flow check: If the address of a location (`l`) flows up to the `root` and the `root` outlives `l`, then `l` must be placed on the heap. In that case, we copy all important attributes (escapes, persists, mutates, calls) to it.
- Value flow tracking: The value of `l` always flows to the `root`. If `l` is a function parameter and the `root` is the heap or a result parameter, we record this flow so the function can be tagged later. [Easter Egg]
- Stop if escaping: If `l` is marked as escaping (`attrEscapes`), we don’t analyze its children further and just move on to the next item in the queue.


Through this process, `walkOne` identifies **all locations that may escape**, propagating the minimal dereference counts and marking objects that cannot safely stay on the stack. This is essential for precise escape analysis and efficient memory allocation in languages like Go.

