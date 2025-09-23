# Walk All

After constructing the escape graph for a set of functions, the next step is to perform the analysis that determines which objects can safely be allocated on the stack. The **walk all** procedure calculates the **minimal dereference** required between every pair of locations in the program.  

The concept of **minimal dereference** is important because it identifies the shortest chain of pointer dereferences needed to reach a particular object. By finding this minimum, the analysis can precisely determine the points in the program where an object might escape or remain confined to the stack. This ensures that stack allocation is done safely without unnecessary memory heap allocations, which can improve both performance and memory usage.

## Working

- The algorithm maintains a **queue**, called `walkAllQueue`, which holds all the locations that need to be processed. The process continues iteratively until a **fixed point** is reached meaning that further walking no longer updates the minimal dereference information.  
- Visiting a location involves performing a procedure called `walkOne`. This procedure is based on the **Bellman-Ford algorithm**, and it calculates the **minimal number of pointer dereferences** required to reach every other location in the program starting from the current location, which acts as the root.  

**Example:**  
Suppose we have three objects `A`, `B`, and `C` in memory, where `A` points to `B` and `B` points to `C`.  
- If we start a `walkOne` from `A`, the minimal dereference counts to reach each object would be:  
  - `A` → `A`: 0 dereferences  
  - `A` → `B`: 1 dereference  
  - `A` → `C`: 2 dereferences  

By performing `walkOne` for each location in `walkAllQueue` and propagating the minimal dereference information, the algorithm ensures that we know the **shortest pointer path** between all objects. This is essential for determining which objects can safely remain on the stack without escaping. How?

Let's take it to `walkOne`.
