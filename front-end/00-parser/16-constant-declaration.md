# Constant Declaration

The Constant Declaration Specification (`ConstSpec`) defines the grammar for declaring one or more constants.

## Grammer

```
ConstSpec = IdentifierList [ [ Type ] "=" ExpressionList ]
```

## Parsing

The goal of the parser function (`constDecl` in the provided code) is to translate the source code matching the `ConstSpec` grammar into an Abstract Syntax Tree (AST) node, specifically a `ConstDecl` structure.

* Parse Identifier List:
    * The parser first consumes the comma-separated list of constant names (identifiers).
    * This list is stored in the `NameList` field of the `ConstDecl` node.
* Check for Type (Optional):
    * After the identifier list, the parser checks the next token. If the token is not a statement terminator (like end-of-file `_EOF`, semicolon `_Semi`, or right parenthesis `_Rparen`), it implies an optional type expression might follow.
    * The parser attempts to parse a type expression and assigns it to the `Type` field. If no type is present, the field remains `nil`.
* Check for Assignment and Values (Optional):
    * If a type was potentially parsed (or if the previous step did not terminate the declaration), the parser then checks if the next token is the assignment operator (`_Assign` or `=`).
    * If an assignment operator is present, the parser consumes it and proceeds to parse a list of expressions (ExpressionList). This list contains the values assigned to the constants.
    * The value expressions are stored in the `Values` field.

## Examples

|Source Code|Grammar Match|NameList|Type|Values|Description|
| -- | -- | -- | -- | -- | -- |
|const PI = 3.14|ID “=” Expr|[PI]|nil|[3.14]|Assigns a value; the type is implicitly inferred.|
|const MaxValue int = 100|ID Type  “=” Expr|[MaxValue]|int|[100]|Explicitly sets the type and assigns a value.|
|const Flag|ID|[Flag]|nil|nil|No explicit type or value; relies on implicit context (often used in groups).|
|const X, Y = 1, 2|IDList “=” ExprList|[X, Y]|nil|[1, 2]|Declares multiple constants with corresponding values.|