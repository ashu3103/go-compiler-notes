# Operand

An operand is a value, variable, or expression that an operator acts upon while evaluating an expression. It can be literals (simple and composite), identifier or method expression.

## Grammer

```
Operand     = Literal | OperandName | MethodExpr | "(" Expression ")" .
Literal     = BasicLit | CompositeLit | FunctionLit .
BasicLit    = int_lit | float_lit | imaginary_lit | rune_lit | string_lit .
OperandName = identifier | QualifiedIdent.
```

## Parsing

The Go compiler uses the first token it encounters to jump to the right section of code. Here is how the compiler identifies and processes the different kinds of operands:

| Token | Action | Example |
| ----- | ------ | ------- |
| `_Name` | It returns this name as the expression itself. | `age` |
| `_Literal` | It returns this fixed value as the expression. | `10`, `3.14` or `"Hello World"` |
| `_Lparen` | It consumes (gets rid of) the opening `(`. It calls itself recursively to parse everything inside as a complete sub-expression (e.g., parsing `y - 2` in our example). It consumes the closing `)`. The final result of the inner expression becomes the operand. | In `(10 + 2)` compiler sees `(` and parses `10 + 2` as a single expression |
| `_Func` | It has to make a critical sub-decision: is this just a definition of a function's structure (its type), or is it a function that does work right here. It consumes func and then parses the Function Signature Type (the contract of what the function takes and returns). This signature is an expression on its own. It looks at the very next token: 1. If its `{` (a closure function should be returned) Parse the entire block of code inside the braces (the function Body). Bundle the Signature Type and the Body together into a Function Literal expression and returns this complete function. 2. Otherwise simply return the function signature type that was parsed earlier | Closure: `func(a int) int { return a * 2 }` or Function type: `func(a int) (int, error)` |
| `_Lbrack`, `_Chan`, `_Map`, `_Struct` or `_Interface` | It simply calls a helper function called `p.type_()` (which means "parse the rest of this as a complete type definition") because in Go, types themselves can be used as operands (for things like type conversions or declarations). | `map[string]int`, `struct { name string }` |
