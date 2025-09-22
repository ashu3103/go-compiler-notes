# Strongly Connected Components

The central idea is to identify the smallest possible sets of functions that are either mutually recursive or standalone non-recursive functions. Each of these sets, also known as **Strongly Connected Components (SCCs)**, can then be analyzed independently. This decomposition ensures that recursive relationships are respected while still allowing modular analysis.

### How SCC Enables Batch Analysis

By organizing the call graph into SCCs, the analysis can be broken down into **batches**. Each SCC forms a batch that is internally consistent (i.e., all functions within the SCC depend on one another), but independent of other SCCs.  
- If a function is non-recursive, it forms a singleton SCC, which can be analyzed alone.  
- If multiple functions are mutually recursive, they form a group SCC, which is analyzed together as one batch.  

This batching prevents redundant re-analysis and guarantees correctness in the presence of recursion.

### Bottom-Up Analysis and SCC Ordering

Since the analysis is applied in a **bottom-up** manner, callees are always analyzed before their callers. The SCC decomposition preserves this ordering:  
- Within each SCC, mutual dependencies are resolved together.  
- Between SCCs, the topological order ensures that when analyzing a caller SCC, all of its callee SCCs have already been analyzed.  

This way, information flows naturally from the leaves of the call graph upward to the root.

### Example Call Graph

Suppose we have the following functions:

- `A` calls `B`  
- `B` calls `A` (mutual recursion with `A`)  
- `C` calls `B`  
- `D` has no recursion and calls no one  
- `E` calls `D`  

Raw call graph looks like this:

```
   +-------+        +-------+
   |   C   |        |   E   |
   +-------+        +-------+
    /      \            |
   /        \           v
 +---+     +---+    +--------+
 | A | <-> | B |    |   D    |
 +---+     +---+    +--------+
```

Generated Constraints:

- `{A, B}` is a batch as they are mutually recursive
- `{C}`, `{D}` and `{E}` are single-ton non-recursive batches
- Batch `{A, B}` will be analysed first since `{C}` is dependent on it
- Batch `{D}` will be analysed first since `{E}` is dependent on it.

### Role of Tarjan’s Algorithm

To achieve this decomposition, **Tarjan’s algorithm** is commonly used. It efficiently computes all SCCs in the call graph in linear time. By identifying cycles (i.e., recursion) and partitioning the graph into SCCs, Tarjan’s algorithm enables the compiler or analyzer to:  
1. Break the global analysis problem into smaller, independent subproblems.  
2. Ensure that recursive cycles are treated as a single unit of analysis.  
3. Establish a topological order between SCCs, which directly supports bottom-up processing.

**In short:** SCCs allow analysis to be organized into batches where recursive functions are grouped together, and non-recursive ones stand alone. Using Tarjan’s algorithm to compute these SCCs ensures that the analysis respects recursion and proceeds in a bottom-up, dependency-aware manner.

> Note: Closures are always considered to be in batch with their enclosing function. Therefore, no closure can be a root of SCCs.