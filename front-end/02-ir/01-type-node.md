# Type Node

This file provides Node wrappers for `*types.Type` values in the Go compiler's intermediate representation (IR). It enables types to be represented as IR nodes during compilation, which is essential for type checking, generics instantiation, and runtime type operations.

**Structure**:
```go
type typeNode struct {
    miniNode
    typ *types.Type
}
```

**Fields**:
- `miniNode` — Embeds the minimal node implementation (12 bytes)
- `typ` — The wrapped `*types.Type` value

**Constructor**:
```go
func newTypeNode(typ *types.Type) *typeNode
```
- Creates a new typeNode wrapper
- Sets position to `src.NoXPos` (no position)
- Sets opcode to `OTYPE`
- Marks as typechecked (`SetTypecheck(1)`)

**Methods**:
- `Type() *types.Type`: Returns the wrapped type
- `Sym() *types.Sym`: Returns the type's symbol

**Usage**: Internal to this package; use `TypeNode()` function to obtain type nodes.