# Function Signature Type

"function signature type" is simply the method's full contract i.e. the types of its parameters and results in order.

## Parsing

The signature is defined by two lists:

* Parameter List: What the method takes in (e.g. (`p []byte`)). The parser just calls another helper function (`paramList`) to handle this part.
* Result List: What the method gives back. This can be:
     * A full list of named or unnamed return parameters (e.g. `(n int, err error)`).
     * A single, unnamed return type (e.g. `int`).
     * An empty list (e.g. `()`).