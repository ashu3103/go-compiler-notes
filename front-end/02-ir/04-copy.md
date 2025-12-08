# Node Copying

Certain IR nodes are considered "shared" and cannot be copied, and how shallow and deep copying work in the Go compiler's intermediate representation (IR).

## What Are Shared Nodes?

**Shared nodes** are IR nodes that may appear in **multiple locations** within the Abstract Syntax Tree (AST). These nodes represent entities that have **identity semantics** rather than **value semantics**. Copying them would violate their semantic meaning because their identity (pointer equality) is significant.

The shared node types are:
- `ONAME` — Named declarations (variables, parameters, functions)
- `ONONAME` — Unresolved identifiers (before type checking)
- `OTYPE` — Type nodes
- `OLITERAL` — Literal constants
- `ONIL` — The nil literal

### Example: Named Declaration `ONAME`

**Why It's Shared**: Represents a **unique entity** in the program (variable, parameter, constant, function).

**Identity Matters Because**:
- Every reference to the same variable must point to the **same `Name` node**
- The `Name` node holds critical metadata:
  - Symbol table entry (`sym`)
  - Storage class (`Class` — PAUTO, PEXTERN, PPARAM, etc.)
  - Escape analysis state (`Esc`)
  - Frame offset (`Offset_`)
  - Definition site (`Defn`)
  - Enclosing function (`Curfn`)
  - Closure variable chain (`Outer`)

**Example**:
```go
func example() {
    x := 10        // Name node created for 'x'
    y := x + 1     // References the SAME Name node for 'x'
    z := x * 2     // References the SAME Name node for 'x'
}
```

**AST Structure**:
```
FuncDecl "example"
├─ DeclStmt: x := 10
│  └─ Name{sym: "x", Class: PAUTO, Offset: 0}  ← Original
├─ AssignStmt: y := x + 1
│  └─ BinaryExpr: x + 1
│     └─ Name{sym: "x"} ← SAME pointer as above
└─ AssignStmt: z := x * 2
   └─ BinaryExpr: x * 2
      └─ Name{sym: "x"} ← SAME pointer as above
```

**Proof It Can't Be Copied**:
```go
// From name.go
func (n *Name) copy() Node { panic(n.no("copy")) }
```

## Shallow Copy

**Shallow copy** creates a new node struct with the **same child references** as the original.

### Visual Example

**Original Tree**:
```
AssignStmt (node A)
├─ miniNode {pos, op, bits, esc}
├─ Lhs: Name{sym: "x"}  ← Node B
└─ Rhs: BinaryExpr      ← Node C
    ├─ Op: OADD
    ├─ X: Name{sym: "y"}  ← Node D
    └─ Y: BasicLit{val: 1} ← Node E
```

**After `Copy(node A)`**:
```
Original AssignStmt (node A)          Copy AssignStmt (node A')
├─ miniNode {pos, op, bits, esc}      ├─ miniNode {pos, op, bits, esc}  ← NEW
├─ Lhs: Name{sym: "x"}  ← Node B      ├─ Lhs: Name{sym: "x"}  ← SAME Node B
└─ Rhs: BinaryExpr      ← Node C      └─ Rhs: BinaryExpr      ← SAME Node C
    ├─ Op: OADD                           ├─ Op: OADD
    ├─ X: Name{sym: "y"}  ← Node D        ├─ X: Name{sym: "y"}  ← SAME Node D
    └─ Y: BasicLit{val: 1} ← Node E       └─ Y: BasicLit{val: 1} ← SAME Node E
```

## Deep Copy

**Deep copy** recursively copies the entire subtree, creating **new nodes** at each level, **except** for shared nodes which are preserved.

### Visual Example

**Original Tree**:
```
BinaryExpr (node A)
├─ Op: OADD
├─ X: BinaryExpr (node B)
│   ├─ Op: OMUL
│   ├─ X: Name{sym: "x"} (node C) ← ONAME (shared)
│   └─ Y: BasicLit{val: 2} (node D) ← OLITERAL (shared)
└─ Y: Name{sym: "y"} (node E) ← ONAME (shared)
```

**After `DeepCopy(newPos, node A)`**:

```
Original Tree                          Deep Copied Tree
─────────────────                      ────────────────────
BinaryExpr (node A)                    BinaryExpr (node A')  ← NEW
├─ Op: OADD                            ├─ Op: OADD
├─ Pos: oldPos                         ├─ Pos: newPos       ← UPDATED
├─ X: BinaryExpr (node B)              ├─ X: BinaryExpr (node B') ← NEW
│   ├─ Op: OMUL                        │   ├─ Op: OMUL
│   ├─ Pos: oldPos                     │   ├─ Pos: newPos   ← UPDATED
│   ├─ X: Name{sym: "x"} (node C)      │   ├─ X: Name{sym: "x"} (node C) ← SAME (shared!)
│   └─ Y: BasicLit{val: 2} (node D)    │   └─ Y: BasicLit{val: 2} (node D) ← SAME (shared!)
└─ Y: Name{sym: "y"} (node E)          └─ Y: Name{sym: "y"} (node E) ← SAME (shared!)
```


