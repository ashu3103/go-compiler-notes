# Simple Statement

A Simple Statement in Go is a concise, optional statement often used in control flow clauses (if, for, switch). It can take many forms, including basic expressions, assignments, variable declarations, and increment/decrement operations.

## Grammer

```
SimpleStmt = EmptyStmt | ExpressionStmt | SendStmt | IncDecStmt | Assignment | ShortVarDecl 
```

## Parsing

The parsing of a simple statement follows a two-pronged decision structure based on the LHS expression list and the operator that follows:

* Parse the Left-Hand Side (LHS):
    * The parser first tries to consume all expressions that could form the LHS of the statement (e.g., `i` in `i++` or `x`, `y` in `x, y = 1, 2`).

A. Single Expression LHS (Non-Assignment Forms)

This branch is taken only if the LHS is a single expression and the next token is not an assignment (`=`) or short declaration (`:=`)

|Next Token|Statement Type|Example Code|Resulting AST|
|--|--|--|--|
|Assignment Operator (`_AssignOp`)|Assignment Statement|`i += 5`|Creates an AssignStmt where the operation (`+=`) is resolved immediately.|
|Increment/Decrement (`_IncOp`)|Increment/Decrement Statement|`i++` or `j--`|Creates an AssignStmt where the operation is the increment (`++`) or decrement (`--`).|
|Channel Arrow (`_Arrow`)|Send Statement|`ch <- value`|Creates a `SendStmt`, setting the LHS as the Chan and the RHS expression as the Value.|
|Other (None of Above)|Expression Statement|`doSomething()`|Creates an `ExprStmt`, where the entire statement is just the evaluation of the expression.|

B Multiple Expression LHS or Assignment/Declaration

This branch is taken if:

- The LHS is a list of expressions (e.g., `x, y`).
- The next token is the Assignment (`_Assign`) or Short Variable Declaration (`_Define`) operator.

|Next Token|Statement Type|Example Code|Logic and Resulting AST|
|--|--|--|--|
|`:=` or `=`|Range Clause (Special)|`for i, v := range list { ... }`|If the statement is inside a for loop (`keyword == _For`) and the next token is _Range, it is parsed as a Range Clause statement.|
|:`:=` or `=`|Assignment / Short Var Decl|`x, y = 1, 2 or z := 3`|If not a range clause, the parser consumes the operator, parses the RHS expression list, and creates a final Assignment Statement (`AssignStmt`). If the operator was `:=`, the statement also acts as a Short Variable Declaration.|