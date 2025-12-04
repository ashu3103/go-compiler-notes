# Type Declaration

The Type Declaration Specification (TypeSpec) defines the grammar for declaring a new type or an alias for an existing type.

## Grammer

```
TypeSpec = identifier [ TypeParams ] [ "=" ] Type 
```

## Parsing

The parser function (`typeDecl`) creates a `TypeDecl` AST node. The logic is complex due to the ambiguity introduced by the shared use of square brackets (`[ ]`) for both Type Parameters and Array/Slice dimensions.

* Parse Type Name:
    * The parser first consumes the identifier that names the new type and assigns it to the `Name` field.

* Check for Square Brackets (`_Lbrack`): Handling Ambiguity
    * If the next token is an opening square bracket (`[` or `_Lbrack`), the parser enters a complex branch to determine if the brackets introduce a Type Parameter List or an Array/Slice Type definition.
        * Case A: Slice Type (`[]T`): If the parser sees `[` immediately followed by `]` (`_Rbrack`), it is a Slice Type. (Example: `type MySlice = []int`)
        * Case B: Array Type (`[N]T`) or Type Parameter List (`[P T]`): If the token after `[` is an identifier (`_Name`), the ambiguity is highest. The parser must look ahead and analyze the expression inside the brackets:
            * If the expression inside the brackets matches the structure of a Type Parameter (e.g., P or P T), it is parsed as a Type Parameter List (`TParamList`). (Example: `type List[T any] struct { ... }`)
            * Otherwise, it is parsed as an Array Type (where `N` is the array length). (Example: `type Array5 [5]int`)
        * Case C: Array Type with Ellipsis (`[...]T`) or Array with No Length
            * If the token after `[` is anything else (e.g., `...`), it defaults to an Array Type. (Example: `type ArrayUnsized [...]int`)
* Handle Simple Declarations (No Brackets):
    * If the token is not an opening square bracket, the declaration is straightforward.
    * The parser checks for the alias operator (`=` or `_Assign`) and sets the Alias flag if present.
    * Then, it parses the underlying Type and assigns it to the `Type` field. (Example: `type Kilometers int`)
* Final Validation:
    * If the parsing fails to produce a Type (e.g., if the declaration was empty or malformed), an error is reported.

## Defining Type vs Aliasing Type

### Type Define:

```go
type MyInt int
```

- Defined a new type `Myint`
- `Myint` is not same as `int`
- Methods on `MyInt` do not apply to `int`
- They behave like separate types in type-checking

```go
type MyInt int

func main() {
    var a int = 10
    var b MyInt = 20

    // var c int = b   // compile error
    var c int = int(b) // explicit conversion
}
```

Usage:

- To create strong types (e.g. `UserID`, `Distance` etc.)
- To attach methods to basic types

### Type Alias:

```go
type MyInt = int
```

- `MyInt` and `int` are exactly the same type
- No conversions needed
- Methods automatically apply to both
- Mostly useful for compatibility during refactoring

```go
type MyInt = int

func main() {
    var a int = 10
    var b MyInt = 20

    var c int = b   // fine
    var d MyInt = a // also fine
}
```

Usage:

- To migrate between types without breaking code
- To simplify long or nested type names
- To provide compatibility layers (e.g., rune = int32)