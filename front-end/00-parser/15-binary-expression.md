# Binary Expression

The binary expression is responsible for evaluating expressions that involve two operands, like mathematical calculations (`+`, `*`) and logical comparisons (`==`, `&&`). Its primary difficulty is correctly handling operator precedence (which operation happens first).

## Grammer

```
Expression = UnaryExpr | Expression binary_op Expression
```

## Parsing

The parser uses a simple, elegant mechanism based on recursion to build the Abstract Syntax Tree (AST) while respecting precedence rules.

**Left Operand**

The function begins by ensuring it has the leftmost operand (`x`) of the expression. It retrieves this by calling the `p`.`unaryExpr()` function, which handles single-operand operations like negation (`-`), pointer creation (`&`), and simple literals

**The Precedence Loop (The Balancing Act)**

The parser enters a `for` loop that runs repeatedly, consuming new operators and operands as long as the new operator's precedence is higher than the current context's precedence.

```go
// Continue as long as we see an operator AND the new operator's
// precedence (p.prec) is STRICTLY GREATER than the current
// precedence level we are enforcing (prec).
for (p.tok == _Operator || p.tok == _Star) && p.prec > prec {
    // ... setup and consume the new, higher precedence operator (t)
    // ...
    t.Y = p.binaryExpr(nil, tprec) // Recurse: Parse the right side
    x = t // Replace the current expression (x) with the new, high-precedence expression (t)
}
```

- High Precedence (New Op > Current Context): If the parser sees `*` after parsing the `+` operation, the `*` has higher precedence. The parser consumes `*`, starts a new Operation node, and uses recursion to parse the entire right side (`t.Y = p.binaryExpr(nil, tprec)`). This forces the high-precedence expression to become a child of the low-precedence expression, correctly grouping them.
- Low/Equal Precedence (New Op <= Current Context): If the parser sees another `+` after parsing an initial `+` (equal precedence), or if it sees `+` after parsing `*` (low precedence), the loop condition `p.prec > prec` fails. The loop terminates, and the parser returns the fully formed expression (`x`) to the caller, which can then incorporate it into a larger, lower-precedence operation.