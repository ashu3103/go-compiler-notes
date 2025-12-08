# Mini Node

`miniNode` is a compact, embeddable minimal implementation of the compiler's Node interface. It is intended to be embedded as the first field of a larger node struct to save space (the `miniNode` value itself costs 12 bytes). By design `miniNode` is not a complete Node: the embedding type must implement a few methods to be a valid Node like:

- `func (n *MyNode) String() string { return fmt.Sprint(n) }`: string formatter
- `func (n *MyNode) rawCopy() Node { c := *n; return &c }`: make a shallow copy and return it as `Node`
- `func (n *MyNode) Format(s fmt.State, verb rune) { FmtNode(n, s, verb) }`: formatting hook

It centralizes common fields (`pos`, `op`, basic flags) and tiny helpers (`Typecheck`, `Walked`) so embedding types can focus on node-specific data.

## Methods 

**Read/write accessors**

- `func (n *miniNode) Op() Op`: returns the node opcode.
- `func (n *miniNode) Pos() src.XPos`: get position.
- `func (n *miniNode) SetPos(x src.XPos)`: set position.
- `func (n *miniNode) Esc() uint16`: get escape-analysis value.
- `func (n *miniNode) SetEsc(x uint16)`: set escape-analysis value.

**Typechecking & walked flags**

- `func (n *miniNode) Typecheck() uint8`: read 2-bit typecheck state (via bits.get2 at shift miniTypecheckShift). The two-bit field can represent 0..3, but SetTypecheck only allows 0..2.
- `func (n *miniNode) SetTypecheck(x uint8)`: set the 2-bit typecheck state. Panics if x > 2.
- `func (n *miniNode) Walked() bool`: read the `miniWalked` boolean flag (prevents/catches re-walking of nodes).
- `func (n *miniNode) SetWalked(x bool)`: set `miniWalked`

## Flag layout

* `miniTypecheckShift = 0`: the two-bit `Typecheck()` field is stored starting at bit 0 in `bits`.
    * `0`: the node is not typechecked
    * `1`: the node is completely typechecked
    * `2`: typechecking of the node is in progress
* `miniWalked = 1 << 2`: a single-bit mask at bit 2 (used by `Walked()/SetWalked`).

The `bits` field is 8 bits (type `bitset8`). The `get2` and `set2` helpers pack/unpack two-bit fields at a shift. This keeps `miniNode` small while giving room for several tiny flags.