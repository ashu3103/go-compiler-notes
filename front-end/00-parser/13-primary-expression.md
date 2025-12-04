# Primary Expression

This is the most critical stage of the compiler because it takes the simple building blocks (Operands) and allows you to chain them together to create complex operations like database.Query("users")[0].Name(). The Primary Expression is the central concept for expression construction.

## Grammer

```
PrimaryExpr =
	Operand |
	Conversion |
	PrimaryExpr Selector |
	PrimaryExpr Index |
	PrimaryExpr Slice |
	PrimaryExpr TypeAssertion |
	PrimaryExpr Arguments 


Selector       = "." identifier
Index          = "[" Expression "]"
Slice          = "[" ( [ Expression ] ":" [ Expression ] ) |
                     ( [ Expression ] ":" Expression ":" Expression )
                 "]"
TypeAssertion  = "." "(" Type ")"
Arguments      = "(" [ ( ExpressionList | Type [ "," ExpressionList ] ) [ "..." ] [ "," ] ] ")" 
```

## Parsing

The compiler uses a simple, continuous strategy to parse a Primary Expression:

- Start with the Core Block (The Operand): The compiler must first parse an Operand. This is your starting point: a literal value, a variable name, a function, or something in parentheses.
- Enter the Suffix Loop: The compiler then loops, continuously looking for "suffixes" or "attachments" to the expression it just built.
- Chain the Suffixes: If it finds a suffix (`.`, `[`, or `(`), it processes that operation, creates a new, larger expression node, and then immediately checks for another suffix. This continues until no more attachments are found.

The compiler's complexity comes from the ambiguity of the next token.

**`_Dot` Suffix**

The dot is ambiguous because it can mean either accessing a field or checking a type. The compiler has to look at the token after the dot to decide.

| Token | Action | Result | Example |
| -- | -- | -- | -- |
| `_Name` | If the token after the `.` is an Identifier (`_Name`), it's simple field access. | Returns a Selector Expression (the expression `X` selecting the field `Name`). | `p.age` |
| `Lparen` | If the token after the `.` is an Opening Parenthesis (`_Lparen`), it's a type operation. It looks at the next token: If the next token is `_Type` It's a Type Switch Guard (used only in switch statements). Otherwise, It parses the type expression inside the parentheses (e.g. `int`) and creates a Type Assertion Expression. | Returns a Type Switch Guard or a Type Assert expression | `value.(type)` or `interfaceValue.(string)` |

**`_Lbrack` Suffix**

The brackets can mean accessing a single element (Index) or taking a range of elements (Slice).

| Compiler Sees | Action | Result | Example |
| -- | -- | -- | -- |
| `X [ Expression ]` | If it parses the index expression (`i`) and the next token is the Closing Bracket (`_Rbrack`), it's a simple index. | Returns an Index Expression. | `mySlice[5]` |
| `X [ ... : ... ]` | If it encounters a Colon (`_Colon`) anywhere inside the brackets, it knows it is parsing a slice expression. It must consume the optional start, middle, and end indexes. Case-1 Partial Slice Example (`x[:j]`): If the very first token after `[` is a `:`, it’s a partial slice expression starting from the beginning. It then proceeds to parse the middle and end parts. Case-2 Full Slice Example (`x[i:j:k]`): The compiler ensures it parses three indexes and consumes two colons if it detects the three-index form | Returns a Slice Expression | `myArray[1:10]` or `myArray[1:10:15]` |

**`_Lparen` Suffix**

| Compiler Sees | Action | Result | Example |
| -- | -- | -- | -- |
| `X (` | If the token immediately after the Primary Expression is an Opening Parenthesis (`_Lparen`), it means the expression `X` is being invoked. It then parses the potentially long, comma-separated list of Arguments inside the parentheses. | Returns a Call Expression. | `Calculate(a, b, 5)` |

> Note: No suffix means the operand is itself returned as a primary expression.

## Example

Example of the Chain

In the expression `list[0].value()`, the parser proceeds:

- Start: Parses list (Operand).
- Loop 1: Sees `[`. Parses `[0]`. Creates an Index Expression (`list[0]`).
- Loop 2: Sees `.`. Parses `.value`. Creates a Selector Expression (`list[0].value`).
- Loop 3: Sees `(`. Parses the empty arguments `()`. Creates a Call Expression (`list[0].value()`).
- Stop: Sees nothing after `)`. The Primary Expression is complete.