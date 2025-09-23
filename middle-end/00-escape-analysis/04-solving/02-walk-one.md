# Walk One

The `walkOne` function is responsible for computing the **minimal number of dereferences** from a given root location to all other locations in the program’s data flow graph. This helps the escape analysis determine whether objects can safely remain on the stack or need to be allocated on the heap.

Because the data flow graph can contain **negative edges** (introduced by addressing operations like `&x`), `walkOne` uses the **Bellman-Ford algorithm** to propagate dereference counts. Negative cycles are not an issue here because the algorithm clamps dereference counts to a lower bound of 0, ensuring the counts remain meaningful.

> Dereference Count: The dereference count from one location (`root`) to other.

Here’s a high-level overview of what `walkOne` does:

1. **Initialization**:  
   The root location is initialized with zero dereferences, and its `walkgen` field is set to track this analysis pass.

2. **Closure Handling**:  
   If the root represents a closure call, the function marks any lost closure results and re-flows from them to correctly track escapes.

3. **Propagation**:  
   Using a queue, `walkOne` iteratively processes each location (BFS traversal):
   - It calculates the effective dereference count to the root.
   - If the location’s address flows to the root, the dereference count is clamped to 0.
   - It determines whether the location **escapes** to the heap, needs to **persist**, or is affected by **mutations/calls** based on the root’s attributes and its own flow.

4. **Function Parameters and Results**:  
   For parameters, the analysis records how values leak or escape, and tags the parameter with mutator and callee information for further function-level analysis.

5. **Edge Updates**:  
   For each outgoing edge from the current location, the function updates dereference counts if a shorter path is found. Locations are then queued for further processing until all reachable locations have been analyzed.

Through this process, `walkOne` identifies **all locations that may escape**, propagating the minimal dereference counts and marking objects that cannot safely stay on the stack. This is essential for precise escape analysis and efficient memory allocation in languages like Go.

