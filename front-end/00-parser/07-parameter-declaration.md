# Parameter Declaration

The `paramDeclOrNil` function is responsible for parsing a single formal parameter or result in a function signature, including identifier lists, variadic syntax, and Go 1.18+ type set constraints.

## Grammer

The function enforces this combined grammar for parameters and type constraints:

| Component | Grammer | Example | Purpose |
|-----------|---------|---------|---------|
| Parameter Decl | `[ IdentifierList ] [ "..." ] Type` | `x, y int` or `...string` | Standard function arguments. |
| Embedded element | `MethodSpec \| EmbeddedTerm { "\|" EmbeddedTerm }` | `io.Reader \| Custom` | Defines a type constraint (for interfaces).|
| Embedded Term | `[ "~" ] Type` | `~int` or `interface{}` | Defines a constraint term (underlying type or specific type). |


## Parsing

### Variant-1: Enclosed in Parenthesis

The parameter list in enclosed parenthesis should contain either an identifier list followed by a type or an anonymous/embedded field.

|Step                               |Token                                                                                                                                                                                                                                                                                   |Action                             |Example Handled                                    |
|-----------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------|---------------------------------------------------|
|Name Check                         |`_Name` (Identifier)                                                                                                                                                                                                                                                                      |Consumes the identifier(s). The field now has a Name.|count `int`                                          |
|Name Extensions                    |`_Lbrack` (`[`) after name                                                                                                                                                                                                                                                                  |Parses Array, Slice or Type Arguments (e.g. `T[A]`).|`data [10]byte`                                      |
|                                   |`_Dot` (`.`) after name                                                                                                                                                                                                                                                                     |Parses a Qualified Identifier (e.g. `pkg.Type`).|`log.Logger`                                         |
|Variadic Check                     |_DotDotDot (`...`)                                                                                                                                                                                                                                                                        |Consumes `...`. Recursively calls `p.typeOrNil()` for the element type. Returns a `DotsType` AST node.|`...int`                                             |
|Standard Type                      |Any other type token (`_Star`, `_Map` etc.)                                                                                                                                                                                                                                                  |Calls the generic `p.typeOrNil()` routine to parse the full type expression.|`map[string]User`                                    |

### Variant-2: Enclosed in Brackets

The parameter list in enclosed brackets should contain an identifier list followed by a type, an anonymous/embedded field or type set constraints. They also take embedded elements in account

### Parsing Embedded Elements and Constraints

The `embeddedElem` and `embeddedTerm` functions are dedicated handlers for the Go `1.18+` type constraint syntax, which uses the binary operator `|` (union) and the unary operator `~` (underlying type).

`embeddedTerm` (`[ "~" ] Type`)
This function parses the single term that forms part of a type constraint:

* Tilde Operator (`~`): If the token is `~`, it is consumed. The function then recursively calls `p.type_()` to parse the type that follows (e.g., `~int`). An Operation node with `Op = Tilde` is returned, representing the "underlying type" constraint.
   * Example: `~int` (Matches all types whose underlying type is `int`).

* Simple Type: If `~` is not present, it simply calls `p.typeOrNil()` to parse a standard type expression.
   * Example: `string` (Matches only the type string).

`embeddedElem` (`EmbeddedTerm { "|" EmbeddedTerm }`)
This function handles the parsing of multiple terms connected by the union operator (`|`):

* First Term: It always starts by calling `p.embeddedTerm()` to get the initial type or constraint term.
* Union Loop: It enters a loop that continues as long as the current token is the `|` operator.
* Union Construction: Inside the loop, it consumes the `|` operator, calls `p.embeddedTerm()` to get the right-hand term, and constructs an Operation node with `Op = Or` to represent the union of the left and right terms.

This structure allows the parser to correctly build an AST for constraints like `~int | ~string | bool`.
