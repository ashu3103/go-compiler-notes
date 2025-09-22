# Overview

When analyzing a function within a batch, we systematically examine every statement in its body. This comprehensive traversal ensures that no operation affecting program behavior is overlooked and lays the foundation for constructing precise data-flow information.

Among all statements, our primary focus is on **assignments** and **function calls**. These are the statements that actively manipulate data and influence program state, making them crucial for building an accurate **escape graph**. For example, consider a variable `x` assigned the result of a function call: `x = foo(y)`. Both the assignment and the call determine how `x` and `y` interact, how data may propagate, and whether any references escape the current function’s scope.

Rest of the blocks, statements or expressions are either discarded from the analysis or undergo nested analysis unless an assignment/call is found. For example:

While analysing an `if` block:
```go
type IfStmt struct {
	miniStmt
	Cond   Node
	Body   Nodes
	Else   Nodes
	Likely bool // code layout hint
}
```

- `Cond` is ignored from the analysis
- `Body` the nodes inside it are taken into account for analysis
- `Else` the nodes inside it are taken into account for analysis

## Intraprocedural Analysis

Intraprocedural analysis concentrates solely on the effects of **assignments within a single function**. By tracking these statements, we can determine how data flows through local variables and how it may potentially escape. The Go compiler often employs **optimistic or conservative strategies** here—for instance, assuming that certain variables do not escape unless proven otherwise or conservatively treating ambiguous cases as escaping to ensure correctness. These strategies balance precision and performance during analysis.

## Interprocedural Analysis

Interprocedural analysis extends the scope by considering interactions between functions. Each **callee function is analyzed first** and assigned tags that indicate how its parameters and return values might escape. These tags allow the analysis to **propagate escape information across function boundaries**, linking variables and memory locations in the caller with those in the callee. For example, if a function `foo(a)` returns a pointer that escapes, any assignment `x = foo(y)` in the caller can now be updated in the escape graph to reflect that `x` may also escape, ensuring a comprehensive view of data flow across the program.
