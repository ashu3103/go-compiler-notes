# Outlives Analysis

The Outlives analysis determines whether the value stored at a particular location (`root`) can persist beyond the lifetime of another location (`other`). This is especially relevant for stack-allocated variables, where lifetimes are limited by function or block scope.

## Constraints

- If the location `root` is marked with the `attrEscapes` attribute, it is guaranteed to outlive `other`.
- Go cannot reliably predict what a caller does with returned values (function result parameters). Therefore, it conservatively assumes that any such value may escape to the heap. Consequently, if `l.paramOut == true`, `root` is considered to outlive `other`. [Easter Egg]
- When both `root` and `other` are in the same function, `root` will outlive `other` if it was declared outside the scope of `other`. For example:
    ```go
    var root *int
    for {
        root = new(int) // other location: new(...)
    }
    ```

    In this example, the memory location created by `new(int)` (the `ONEW` node) is considered to be "outlived" by `root`, since `root` was declared outside the `for` loop.

    ```go
    if root.curfn == other.curfn && root.loopDepth < other.loopDepth {
		return true
	}
    ```

> Note: All global variables are implicitly marked with attrEscape. Therefore, any location that flows to a global variable is also considered to escape.