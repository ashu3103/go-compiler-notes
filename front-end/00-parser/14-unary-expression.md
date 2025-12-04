# Unary Expression

A Unary Expression is an expression where a single operator acts on a single value. In programming, this means the operator only needs one thing (one operand) to do its job.

The tricky part comes when you chain these operations, like saying `---x` or `*&v`. The Go compiler has to read this chain correctly, from left to right, to figure out the final instruction.

## Grammer

```
UnaryExpr = PrimaryExpr | unary_op UnaryExpr 
```

## Parsing

**Path 1: It's a Primary Expression (No Operator Ahead)**

If the compiler sees a token that is a value, a variable name, or a starting parenthesis, it means there is no unary operator applied right now.

For example it can be `age`, `10`, `(x + y)`, or a function call `fmt.Println()`.

**Path 2: It's a Unary Operator (Start the Chain)**

If the compiler sees a token that is one of the designated Unary Operators, it knows an operation is about to begin. This is where the stacking happens.

* Consume Operator: The parser reads the first operator (e.g. `*`).
* Recurse: The parser does not stop. It immediately calls itself recursively to parse the next thing. This next thing might be another unary operator (like `&`), or it might finally be the primary value (like `v`).
* Build the Node: This recursive process creates a stack of operators until it hits a primary value.
* Result: The parser returns an Operation Expression node. This node contains two main fields:
    * `Op`: The current operator (e.g. `*`).
    * `X`: The result of the next, inner unary expression (the rest of the chain)

## Allowed Unary Operators

| Operator | Token | Meaning | Analogy |
| -- | -- | -- | -- |
|`+`|`_Operator`|Unary Plus (Rarely used, usually for sign)|Affirms a positive value |
|`-`|`_Operator`|Unary Minus (Negation)|Flips the sign of a value |
|`~`|`_Operator`|Bitwise NOT (Introduced in Go 1.20)|Flips all the bits |
|`!`|`_Operator`|Logical NOT|Flips a boolean (True → False) |
|`&`|`_And`|Address-of Operator|Gets the memory location (address) of a variable |
|`*`|`_Star`|Dereference Operator|Gets the value stored at a memory location |

## Example

Let's look at the expression *&v (Dereference the address of variable v)

|Step|Token Consumed|Recursive Call Target|Node Created (Bottom-up)|
| -- | -- | -- | -- |
|1|* (Dereference)|&v|Creates a * node, waiting for its operand X.|
|2|& (Address-of)|v|Creates an & node, waiting for its operand X.|
|3|v (Primary)|N/A (Stops recursion)|Creates a Primary Expression node for the variable v.|