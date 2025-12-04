# Variable Declaration

The Variable Declaration Specification (`VarSpec`) defines the grammar for declaring one or more variables. This process is similar to constant declaration but offers a different structure regarding the optional Type and initialization Expression List.

## Grammer

```
VarSpec = IdentifierList ( Type [ “=” ExpressionList ] ∣ “=” ExpressionList )
```

## Parsing

The parser function (`varDecl` in the provided code) generates a `VarDecl` Abstract Syntax Tree (AST) node by following these steps:

* Parse Identifier List:
    * The parser consumes the comma-separated list of variable names (identifiers).
    * This list is assigned to the `NameList` field of the `VarDecl` node.
* Check for Initialization (Type Omitted):
    * The parser first checks if the next token is the assignment operator (`_Assign` or `=`).
    * If `=` is present: This matches the second grammar option (`“=”ExpressionList`). The Type is omitted and will be inferred. The parser consumes the = and parses the value expressions, setting the `Values` field.
* Handle Type and Optional Initialization (Type Provided):
    * If `=` is not present (the else block): This implies the first grammar option (`Type[“=”ExpressionList]`).
    * The parser must now parse a Type expression and assign it to the `Type` field.
    * After the Type is parsed, the parser checks again if the next token is the assignment operator (`_Assign` or `=`).
    * If `=` is present now: The parser consumes the = and parses the value expressions, setting the `Values` field.