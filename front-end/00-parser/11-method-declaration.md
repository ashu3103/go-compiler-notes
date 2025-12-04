# Method Declaration

Let's say the compiler sees the word Reader. Inside the interface, `Reader` could mean one of two things:

- A Method: The interface requires a method called `Reader`. (e.g., `Reader(p []byte) (n int, err error)`).
- An Embedded Type: The interface is simply inheriting all the requirements from another, pre-existing interface called `Reader`.

The Go compiler has to solve this ambiguity immediately after seeing the first word.

## Grammer

```
MethodSpec        = MethodName Signature | InterfaceTypeName
MethodName        = identifier
InterfaceTypeName = TypeName
```

## Parsing

The moment the compiler consumes the first word (which must be an identifier, like `Reader`), it applies a simple, but crucial, look-ahead rule:

The Parser's Rule: Look at the token immediately following the identifier.

1 It's a Method Declaration (Named and Typed Field)

If the compiler sees an opening parenthesis (`(`) right after the identifier, it must be a method declaration.

- The identifier consumed (e.g. `Reader`) is the name of the method.
- Everything that follows the name i.e. the parentheses, the parameter list, and the result list is used to build the method's function signature type. This signature is the full "contract" of the method (what it takes in and what it returns).

As a result the parser creates a new `Field` node that is named (with the method's name) and typed (with the method's signature).

Example:
```Go

interface {
    // Compiler sees "Read" and then "("
    // It knows: This is a method.
    Read(p []byte) (n int, err error) 
}
```

2 It's an Embedded Interface (Unnamed and Typed Field)

If the compiler sees ANYTHING else immediately following the identifier (like a newline, a comma, or the closing brace `}`), it assumes the identifier is the name of another interface being embedded.

- The identifier consumed (e.g., `io.Reader`) is the type itself, not the name of a method and therefore field is unnamed, it doesn't have a new name inside the current interface.

As a result the parser creates a new `Field` node that is unnamed and typed (where the type is the qualified name of the identifier).

Example:
```Go

interface {
    // Compiler sees "io.Reader" but no "(" next.
    // It knows: This is an embedded interface type.
    io.Reader 
}
```
