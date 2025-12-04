# Field Declaration

The Go compiler’s job when reading a struct body (`{...}`) is to resolve an inherent ambiguity: Is a given identifier starting a field declaration the name of a new field or the type of an embedded field?

The parser uses a decision tree based on the very next token (a lookahead) to determine the structural intent.

## Grammer

Field declaration are used to in structs and follows the following grammer:

```
FieldDecl = (IdentifierList Type | AnonymousField) [Tag]
IdentifierList = identifier { "," identifier }
AnonymousField = [ "*" ] TypeName
Tag = string_literal
```

## Parsing

**Step-1:** The parser scans the first token of the line. There are two primary starting paths:

Path-A: If the first token is an asterisk (`*`), the parser immediately knows this is an Anonymous Field (embedding a pointer). The `*` must be followed by a Qualified TypeName.

Example:

```go
struct {
    *ArbitraryType
}
```

Path-B: If the first token is an identifier, the compiler enters the ambiguity zone. The identifier could be:

- A Field Name (e.g., ID int).
- An Embedded Type Name (e.g., Address).

The compiler must now perform a lookahead to decide.

**Step-2:** Resolving the Identifier Ambiguity

|Next Token                         |Resolution Strategy                                                                                                                                                                                                                                                                     |Example                            |Resulting Field Type                               |
|-----------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------|---------------------------------------------------|
|`,` (Comma)                          |Named Field List: This is the easiest case. The parser confirms a list of names is being defined.                                                                                                                                                                                       |X, Y, Z float64                    |Named Field (Z is a name, float64 is the type).    |
|`[` (Left Bracket)                   |Complex Type Check: Ambiguous! It could be an array/slice (Name [5]int) or a generic type instantiation (Map[int]string). The parser must consume tokens until it finds the actual type definition to confirm if the identifier is a name or a type.                                    |Data []byte or Map[string]int|Depends on the resolved type structure.            |
|String Literal ("...")             |Simple Embedded Type: A string literal can only be a Tag. If an identifier is immediately followed by a tag, it cannot be a named field (since a named field requires a type before the tag). The parser assumes the identifier is the type name, making it an anonymous/embedded field.|Address "json:\"addr\""            |Anonymous Field (the type Address is embedded).    |
|Terminator (`_Rbrace`, `_Semi`)        |Simple Embedded Type: Similar to the tag check, if the struct definition ends here, the identifier must be an embedded type name.                                                                                                                                                       |Address                            |Anonymous Field (the type Address is embedded).    |
|Any other token (like another Type)|Named Field: If the next token is unambiguously a type (e.g., the keyword string or int), the first identifier is treated as the field name.                                                                                                                                            |Name string                        |Named Field (Name is the name, string is the type).|


**Step-3:** Tag Consumption

Regardless of whether the parser generated a named field or an anonymous field, the final step is to check for metadata:

Check for String Literal, If the very next token is a string_literal, this literal is consumed and attached to the field as its Tag.

## Examples

Example-1:

```go
struct {
    *ast.Node       // embedded field (with package qualified type)
    Employee        // embedded field
    Map[int, int]   // embedded field (with generics)
    ...
    a, b float64    // typename followed by an identifier list
    c int           // typename followd by an identifier
}
```
