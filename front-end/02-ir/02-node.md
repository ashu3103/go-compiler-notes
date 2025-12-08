# Node

The `Node` interface is the central abstraction for IR/AST nodes in the compiler. Conceptually the interface provides these concerns:

- Formatting: `Format(s fmt.State, verb rune)`: custom printing/formatting for nodes.
- Source position: `Pos()` / `SetPos(x src.XPos)`: track original position for error reporting and inlining information.
- Copying: `copy()` / `rawCopy()` style methods (used for making node copies).
- Child traversal/editing: `doChildren`, `doChildrenWithHidden`, `editChildren`, `editChildrenWithHidden`: generic traversal/edit helpers used by passes that inspect or transform node children.
- Graph structure: `Op()` returns the node opcode; `Init()` returns the node's initialization list (an `ir.Nodes` slice) for nodes that carry initialization statements.
- Per-op fields accessors: `Type()`, `SetType()`, `Name()`, `Sym()`, `Val()`, `SetVal()`: these are present so the generic interface can access fields present only on specific node kinds (e.g., `ONAME`, `OLITERAL`, `OTYPE`). An implementation may panic if asked for fields it doesn't have.
- Analysis storage: `Esc()`/`SetEsc(uint16)` store escape analysis results on the node.
- Typechecking state: `Typecheck()`/`SetTypecheck(uint8)` (0 = not typechecked, 1 = done, 2 = in progress) and `NonNil()`/`MarkNonNil()` to record non-nil information used by checks and codegen.

Note: Not every node implements meaningful values for all per-op accessors; some methods are only valid for particular opcodes. Many node implementations embed `miniNode` to provide a small common backing for fields like `pos`, `op`, and a few flags.


# Node Ops

**Definition:** Op is the compiler's opcode enum (a uint8) that names every kind of IR/AST node the compiler handles.

**Purpose:** An Op classifies the shape and meaning of a Node. Passes (parser, order, typecheck, walk, ssa, backend) use the `Op` to:

- dispatch behavior (how to print, typecheck, or lower the node),
- decide which per-op fields are valid (`Type()`, `Name()`, `Val()` only make sense for some ops),
- pattern-match for rewrites/optimizations,
- generate the appropriate code (e.g., `OADD`: binary add; `OCALL`: call sequence)

| Op | Comment + example |
|---|---|
| OXXX | (placeholder/invalid) |
| ONAME | var or func name — example: `var x int // ONAME` |
| ONONNAME | Unnamed arg or return value; unresolved package id — example: function parameter placeholder `func(_ int)` |
| OTYPE | type name — example: `type T struct{}` |
| OLITERAL | literal — example: `42` |
| ONIL | nil — example: `var p *T = nil` |
| OADD | X + Y — example: `a + b` |
| OSUB | X - Y — example: `a - b` |
| OOR | X | Y — example: `a | b` (bitwise or) |
| OXOR | X ^ Y — example: `a ^ b` |
| OADDSTR | +{List} (string addition) — example: `s1 + s2` |
| OADDR | &X — example: `&x` |
| OANDAND | X && Y — example: `a && b` |
| OAPPEND | append(Args) — example: `append(s, x)` |
| OBYTES2STR | Type(X) (Type is string, X is a []byte) — example: `string(b)` |
| OBYTES2STRTMP | Type(X) (Type is string, X is a []byte, ephemeral) — example: ephemeral helper during conversion |
| ORUNES2STR | Type(X) (Type is string, X is a []rune) — example: `string([]rune("x"))` |
| OSTR2BYTES | Type(X) (Type is []byte, X is a string) — example: `[]byte(s)` |
| OSTR2BYTESTMP | Type(X) (Type is []byte, X is a string, ephemeral) — example: temporary helper for `[]byte(s)` |
| OSTR2RUNES | Type(X) (Type is []rune, X is a string) — example: `[]rune(s)` |
| OSLICE2ARR | Type(X) (Type is [N]T, X is a []T) — example: slice->array conversion |
| OSLICE2ARRPTR | Type(X) (Type is *[N]T, X is a []T) — example: slice->array-pointer conversion |
| OAS | X = Y or (if Def=true) X := Y; Init includes DCL for X — example: `x = y` |
| OAS2 | Lhs = Rhs (x, y = f()) or (if Def=true) Lhs := Rhs — example: `a, b = f()` |
| OAS2DOTTYPE | Lhs = Rhs (x, ok = I.(int)) — example: `v, ok := i.(T)` |
| OAS2FUNC | Lhs = Rhs (x, y = f()) — example: `a, b = f()` |
| OAS2MAPR | Lhs = Rhs (x, ok = m["foo"]) — example: `v, ok := m[k]` |
| OAS2RECV | Lhs = Rhs (x, ok = <-c) — example: `v, ok := <-ch` |
| OASOP | X AsOp= Y (x += y) — example: `x += y` |
| OCALL | X(Args) (function call, method call or type conversion) — example: `f(x)` |
| OCALLFUNC | X(Args) (function call f(args)) — example: `f(x)` |
| OCALLMETH | X(Args) (direct method call x.Method(args)) — example: `obj.Method()` |
| OCALLINTER | X(Args) (interface method call x.Method(args)) — example: `i.Method()` |
| OCAP | cap(X) — example: `cap(s)` |
| OCLEAR | clear(X) — example: `clear(m)` |
| OCLOSE | close(X) — example: `close(ch)` |
| OCLOSURE | func Type { Func.Closure.Body } (func literal) — example: `func(){}` |
| OCOMPLIT | Type{List} (composite literal, not yet lowered) — example: `T{...}` |
| OMAPLIT | Type{List} (map literal) — example: `map[string]int{"a":1}` |
| OSTRUCTLIT | Type{List} (struct literal) — example: `T{F:1}` |
| OARRAYLIT | Type{List} (array literal) — example: `[3]int{1,2,3}` |
| OSLICELIT | Type{List} (slice literal), Len is slice length — example: `[]int{1,2}` |
| OPTRLIT | &X (X is composite literal) — example: `&T{...}` |
| OCONV | Type(X) (type conversion) — example: `int64(x)` |
| OCONVIFACE | Type(X) (conversion, to interface) — example: `var a any = x` |
| OCONVNOP | Type(X) (no effect) — example: `x.(T)` when conversion not needed |
| OCOPY | copy(X, Y) — example: `copy(dst, src)` |
| ODCL | var X (declares X of type X.Type) — example: `var x T` |
| ODCLFUNC | func f() or func (r) f() (parser-only) — example: parser node for function decl |
| ODELETE | delete(Args) — example: `delete(m, k)` |
| ODOT | X.Sel (struct selector) — example: `s.f` |
| ODOTPTR | X.Sel (pointer to struct selector) — example: `p.f` |
| ODOTMETH | X.Sel (non-interface method) — example: `t.Method` |
| ODOTINTER | X.Sel (interface method) — example: `i.Method` |
| OXDOT | X.Sel (before rewrite) — example: intermediate selector |
| ODOTTYPE | X.Ntype or X.Type; after walk Itab contains descriptors — example: `x.(T)` |
| ODOTTYPE2 | X.Ntype or X.Type (on rhs of OAS2DOTTYPE) — example: rhs of type assertion |
| OEQ | X == Y — example: `a == b` |
| ONE | X != Y — example: `a != b` |
| OLT | X < Y — example: `a < b` |
| OLE | X <= Y — example: `a <= b` |
| OGE | X >= Y — example: `a >= b` |
| OGT | X > Y — example: `a > b` |
| ODEREF | *X — example: `*p` |
| OINDEX | X[Index] (array or slice) — example: `a[i]` |
| OINDEXMAP | X[Index] (map) — example: `m[k]` |
| OKEY | Key:Value (in literal) — example: `"k": v` |
| OSTRUCTKEY | Field:Value (after type checking) — example: `F: v` in `T{F:v}` |
| OLEN | len(X) — example: `len(s)` |
| OMAKE | make(Args) (before typing) — example: `make(T, n)` |
| OMAKECHAN | make(Type[, Len]) (chan) — example: `make(chan int, 1)` |
| OMAKEMAP | make(Type[, Len]) (map) — example: `make(map[int]int)` |
| OMAKESLICE | make(Type[, Len[, Cap]]) (slice) — example: `make([]int, n)` |
| OMAKESLICECOPY | makeslicecopy(Type, Len, Cap) (created by order pass) — example: generated for `s := make(T,n); copy(s,t)` |
| OMUL | X * Y — example: `a * b` |
| ODIV | X / Y — example: `a / b` |
| OMOD | X % Y — example: `a % b` |
| OLSH | X << Y — example: `a << b` |
| ORSH | X >> Y — example: `a >> b` |
| OAND | X & Y — example: `a & b` |
| OANDNOT | X &^ Y — example: `a &^ b` |
| ONEW | new(X) — example: `new(T)` |
| ONOT | !X — example: `!ok` |
| OBITNOT | ^X — example: `^x` |
| OPLUS | +X — example: `+x` |
| ONEG | -X — example: `-x` |
| OOROR | X || Y — example: `a || b` |
| OPANIC | panic(X) — example: `panic("err")` |
| OPRINT | print(List) — example: `print(x)` |
| OPRINTLN | println(List) — example: `println(x)` |
| OPAREN | (X) — example: `(x)` |
| OSEND | Chan <- Value — example: `ch <- v` |
| OSLICE | X[Low : High] (slice) — example: `s[i:j]` |
| OSLICEARR | X[Low : High] (pointer to array) — example: `p[i:j]` |
| OSLICESTR | X[Low : High] (string) — example: `str[i:j]` |
| OSLICE3 | X[Low : High : Max] — example: `s[i:j:k]` |
| OSLICE3ARR | X[Low : High : Max] (array pointer) — example: `p[i:j:k]` |
| OSLICEHEADER | sliceheader{Ptr, Len, Cap} — example: internal slice header node |
| OSTRINGHEADER | stringheader{Ptr, Len} — example: internal string header node |
| ORECOVER | recover() — example: `recover()` |
| ORECV | <-X — example: `<-ch` |
| ORUNESTR | Type(X) (Type is string, X is rune) — example: `string(r)` |
| OSELRECV2 | like OAS2 with ORECV — example: `v, ok := <-ch` in select |
| OMIN | min(List) — example: `min(a,b)` |
| OMAX | max(List) — example: `max(a,b)` |
| OREAL | real(X) — example: `real(c)` |
| OIMAG | imag(X) — example: `imag(c)` |
| OCOMPLEX | complex(X, Y) — example: `complex(a,b)` |
| OUNSAFEADD | unsafe.Add(X, Y) — example: `unsafe.Add(ptr, off)` |
| OUNSAFESLICE | unsafe.Slice(X, Y) — example: `unsafe.Slice(ptr, len)` |
| OUNSAFESLICEDATA | unsafe.SliceData(X) — example: `unsafe.SliceData(s)` |
| OUNSAFESTRING | unsafe.String(X, Y) — example: `unsafe.String(ptr, len)` |
| OUNSAFESTRINGDATA | unsafe.StringData(X) — example: `unsafe.StringData(s)` |
| OMETHEXPR | X(Args) (method expression T.Method(args)) — example: `T.Method` |
| OMETHVALUE | X.Sel (method value, not called) — example: `t.Method` |
| OBLOCK | { List } (block) — example: `{ x := 1; x }` |
| OBREAK | break [Label] — example: `break` |
| OCASE | case List: Body — example: `case 1: ...` |
| OCONTINUE | continue [Label] — example: `continue` |
| ODEFER | defer Call — example: `defer f()` |
| OFALL | fallthrough — example: `fallthrough` |
| OFOR | for Init; Cond; Post { Body } — example: `for i:=0; i<n; i++ {}` |
| OGOTO | goto Label — example: `goto L` |
| OIF | if Init; Cond { Then } else { Else } — example: `if x>0 {}` |
| OLABEL | Label: — example: `L:` |
| OGO | go Call — example: `go f()` |
| ORANGE | for Key, Value = range X { Body } — example: `for k,v := range m {}` |
| ORETURN | return Results — example: `return x` |
| OSELECT | select { Cases } — example: `select { case v := <-ch: ... }` |
| OSWITCH | switch Init; Expr { Cases } — example: `switch x {}` |
| OTYPESW | OTYPESW: X := Y.(type) — example: `switch v := i.(type) {}` |
| OINLCALL | intermediary representation of an inlined call — example: internal inline node |
| OMAKEFACE | construct an interface value — example: `iface := any(x)` |
| OITAB | rtype/itab pointer of an interface value — example: internal itab node |
| OIDATA | data pointer of an interface value — example: internal data pointer node |
| OSPTR | base pointer of a slice or string — example: internal base pointer node |
| OCFUNC | reference to c function pointer (not go func value) — example: C func pointer ref |
| OCHECKNIL | emit code to ensure pointer/interface not nil — example: nil-check helper |
| ORESULT | result of a function call; Xoffset is stack offset — example: internal result node |
| OINLMARK | start of an inlined body, with file/line of caller — example: inline marker node |
| OLINKSYMOFFSET | offset within a name — example: symbol offset node |
| OJUMPTABLE | jump table structure for dense switches — example: backend jump table node |
| OINTERFACESWITCH | a type switch with interface cases — example: interface specialized type switch |
| OMOVE2HEAP | Promote a stack-backed slice to heap — example: escape promotion of slice |
| ODYNAMICDOTTYPE | x = i.(T) where T is a type parameter — example: generic type assertion |
| ODYNAMICDOTTYPE2 | x, ok = i.(T) where T is a type parameter — example: `v, ok := i.(T)` |
| ODYNAMICTYPE | a type node for type switches — example: runtime dynamictype node |
| OTAILCALL | tail call to another function — example: tail-call node |
| OGETG | runtime.getg() (read g pointer) — example: internal getg node |
| OGETCALLERSP | internal/runtime/sys.GetCallerSP() — example: get caller SP node |
| OEND | end marker |

