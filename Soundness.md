# Soundness of the `from` / `by` core

This document formalises a core of the design in `RefinedIdea.md` and proves that well-typed
programs never dereference dead storage, never read uninitialised or destroyed storage, free every
heap cell exactly once, and never write a place outside its declared writer set.

The proof is for the design **as amended** by four rules that `RefinedIdeaAnalysis.md` shows to be
necessary. Without them the counterexamples in that document are executable, so no proof exists.
The amendments are stated as premises in section 2.4 and are the only additions to the text.

What is proved: the single-threaded core with owned, heap-owned and borrowed storage, aliases,
moves, in-place destruction, structural teardown, calls, and writer sets. What is treated as
extensions with a proof sketch only: `&` shared ownership, threads. What is out of scope: bit
casts, `# ExternC`, arithmetic, refinements.

## 1. The core calculus

### 1.1 Syntax

```
Base types      B   ::= Int | Bool | ...                    flat, no references inside
Regions         R   ::= this | A                            A is a place, see below
Storage kinds   k   ::= inline | heap                       from this | from rel this
Types           τ   ::= B
                      | S                                   record, declared once
                      | ref τ from A                        reference anchored at A
                      | τ | None

Record decl     S = struct( f_1 : τ_1 [k_1] [by W_1] ; ... ; f_n : τ_n [k_n] [by W_n] )
Writer sets     W   ::= ()  |  {g_1, ..., g_m}              a set of function names
Places          p   ::= x | p.f                             surface: f'p
Values          v   ::= c | S{f_i = v_i} | None | loc
Expressions     e   ::= v | p | rel p | @p | S{f_i = e_i} | g(a_1, ..., a_n)
Arguments       a   ::= e | p!                              p! passes the place writable
Statements      s   ::= x : τ = e                           declare, x fresh
                      | x : τ                               declare uninitialised
                      | p! = e                              write, R1 chooses slot vs referent
                      | del p
                      | { s* }                              scope
                      | out e
Function decl   g(x_1 : τ_1 [!] , ...) [by-membership] => τ_ret [from x_i] { s* }
```

Restrictions that make this a core rather than the language:

- Every declared field, parameter and local carries an explicit `from` and `by` clause. Surface
  defaults (`from this`, universal writer) are elaborated before typing.
- Anchors `A` are places built only from names declared earlier in the same or an enclosing scope,
  or `this` inside a record.
- Parameters are consumed unless marked `!`. Borrowed parameters (`use x`) are elaborated to a
  local `x : ref τ from a` where `a` is the argument place, introduced at the call site; see 3.5.
- `out` returns from the current function only. Named block exits are elaborated to nested
  functions; they do not affect the argument.

### 1.2 Runtime structures

```
Cell ids        ℓ
Cell state      C ::= Val v | Ref ℓ | Garbage | Dead
Store           σ : ℓ ⇀ (τ, C)
Ownership       own : ℓ ⇀ ℓ                     parent cell, absent for root cells
Frames          F ::= [x_1 ↦ ℓ_1, ..., x_n ↦ ℓ_n]   one per open scope
Configuration   ⟨σ, own, F̄, s̄⟩
```

A record value occupies one cell per field. An inline field's cell is a child of the record's
cell; a heap field's cell is also a child, but its identity is stable under relocation. A `ref`
field's cell holds `Ref ℓ'` and is a child of the record cell; `ℓ'` is **not** owned by it.

`subtree(ℓ)` is the set of cells reachable from `ℓ` through `own⁻¹`. `root(ℓ)` is the cell with no
parent above `ℓ`. `cell(p)` resolves a place to a cell by following `own⁻¹` for field steps, and
following `Ref` for a field whose cell holds one.

### 1.3 Dynamic semantics

Only the transitions that touch the invariant are listed. All others are standard.

```
[Read]     ⟨p⟩       →  v                where σ(cell(p)) = (τ, Val v),
                                          and every Ref cell crossed by cell(p) is live in Δ
[Alias]    ⟨@p⟩      →  Ref cell(p)
[Move]     ⟨rel p⟩   →  v ; σ(cell(p)) := Dead for cell(p) and all of subtree(cell(p)),
                                          then the value's cells are re-owned under the target
[Write-v]  ⟨p! = v⟩  →  teardown(subtree(cell(p))) ; σ(cell(p)) := Val v ; re-own v's cells
[Write-r]  ⟨p! = @q⟩ →  σ(cell(p)) := Ref cell(q)              cell(p) is a Ref cell
[Del]      ⟨del p⟩   →  teardown(subtree(cell(p))) ; σ(cell(p)) := Dead
[Scope]    end of scope with frame F  →  for each x ↦ ℓ in F, in reverse declaration order:
                                          if σ(ℓ) = Dead then skip else teardown(subtree(ℓ))
[Call]     g(ā)      →  push frame, bind parameters (3.5), run body, pop, rebind ! arguments
```

`teardown(T)` walks the cells of `T` in post-order, frees heap cells, and marks every cell `Dead`.
A cell already `Dead` is skipped, which is what makes partial moves free of double frees once the
static side guarantees the walk never reaches a `Dead` cell that still has live children.

A configuration is **stuck** if a step requires a cell state it does not have: `[Read]` on
`Garbage` or `Dead`, `[Read]` through a `Ref ℓ'` with `σ(ℓ') = Dead`, `teardown` reaching a freed
heap cell twice, or a write executed by a function outside the cell's writer set. Soundness means
a well-typed program never reaches a stuck configuration.

## 2. Static semantics

### 2.1 Typing environment

```
Γ  : names ⇀ (τ, A_decl)           declared type and, for references, declared anchor
Δ  : places ⇀ {Init, Moved, Deleted, Uninit}     definite assignment state
L  : set of reference bindings currently live       killed by writes and moves
π  : the function being checked                     for writer-set membership
```

`Δ` is prefix-closed for `Init`: if `p.f` is `Init` then `p` is `Init`. `Moved` and `Deleted` on a
prefix make every extension unreadable.

### 2.2 Anchor of an expression

`anchor(p)` is the owner-side place a reference to `p` is tied to:

```
anchor(x)     = x
anchor(p.f)   = anchor(p)                       if cell of p.f is inline or heap under p
anchor(p.f)   = A                               if p.f is a ref field declared from A,
                                                 resolved relative to p (this ↦ p)
```

The third line is what makes an alias taken *through* a reference anchor at the reference's own
anchor rather than at the reference: `@next'cur` where `cur : ref ListNode from list` anchors at
`list`.

### 2.3 Overlap and kill

Two places overlap, `p ⋈ q`, when one is a prefix of the other. The kill operation removes from
`L` every reference binding whose anchor overlaps `p`:

```
kill(p, L) = { r ∈ L | anchor(r) ⋈̸ p }
```

### 2.4 Premises added to the text

These four are the amendments. Each is written as a typing side condition.

```
[R1]   In p! = e, if cell(p) is a Ref cell, then e is @q (repoint) or e is a value (write referent).
       Both are writes to p for writer-set purposes. A value written through a Ref cell goes to
       the pointee, and the pointee's declared writer set is the one checked (the inner by).
[R2]   Every rule that writes, moves or deletes p sets L := kill(p, L).
[R3]   A record field f_i : ref τ from f_j'this is well formed only if k_j = heap.
[R4]   Passing an argument to a parameter declared by W requires the argument's declared
       writer set to include W. A parameter marked ! has writer set {g} for its own function g.
```

R4 is the viral reading of `by`; under it the "trust point" in the text is checked, not trusted.

### 2.5 Typing rules for statements

```
[Decl]      Γ ⊢ e : τ     x ∉ Γ     every A named in τ is declared before x
            ⟹  Γ, x : τ ; Δ[x ↦ Init] ; L ∪ {x if τ is a ref}

[Decl-U]    x : τ     ⟹  Γ, x : τ ; Δ[x ↦ Uninit] ; L

[Read]      Δ(p) = Init     every ref binding on the path of p is in L
            ⟹  Γ ⊢ p : τ_p

[Alias]     Γ ⊢ p readable     ⟹  Γ ⊢ @p : ref τ_p from anchor(p)

[Move]      Δ(p) = Init     the path of p crosses no Ref cell, or crosses one whose
            declared inner writer set contains π
            ⟹  Γ ⊢ rel p : τ_p ;  Δ[p ↦ Moved] ;  L := kill(p, L)

[Write]     p resolves, π ∈ W_last(p), premises of R1
            Γ ⊢ e : τ compatible with the slot or referent per R1
            for a repoint p! = @q:  anchor(q) is A_decl(p) or a place below it
            ⟹  Δ[p ↦ Init] ;  L := kill(p, L)

[Del]       Δ(p) = Init     π ∈ W_last(p)     operator del of τ_p is visible to π
            ⟹  Δ[p ↦ Deleted] ;  L := kill(p, L)

[Scope]     for each x declared in the scope:  Δ(x) ∈ {Init, Moved, Deleted}
            and for every sub-place q of x with Δ(q) ∈ {Moved, Deleted}:  Δ(x) ≠ Init
            i.e. a partially moved or deleted binding must be refilled or wholly moved
            ⟹  pop x from Γ, Δ, L

[Out]       Γ ⊢ e : τ_ret     if τ_ret carries a ref anchored at A then A is a parameter
            named in the from clause of the signature, or A is out
            every linear local in scope is Moved or Deleted
```

`W_last(p)` is the writer set of the declaration of the last hop of `p`, resolved through R1 to
the pointee's declaration when the last hop is a `Ref` cell and the right-hand side is a value.

### 2.6 Well-formed records and functions

```
[Rec]   S = struct(...) is well formed when
        every from clause names this or an earlier field of S,
        R3 holds for every ref field,
        every by clause is a set of function names.

[Fun]   g(...) => τ_ret from x_i { body } is well formed when
        body checks under Γ = parameters, π = g,
        every from clause in the signature names a parameter or out,
        the body's final Δ satisfies [Scope] for the parameter frame,
        and each ! parameter is Init at every out.
```

## 3. The invariant

`Inv(σ, own, Γ, Δ, L)` is the conjunction of:

```
(I1) Forest.      own is a forest. Every live cell not a root has exactly one parent.
                  Every heap cell that is not Dead is in subtree(ℓ) for some root ℓ bound in a frame.

(I2) Agreement.   Δ(p) = Init      ⟹  σ(cell(p)) = (τ_p, Val v) or (τ_p, Ref ℓ) with v, ℓ well typed
                  Δ(p) ∈ {Moved, Deleted}  ⟹  σ(cell(p)) = Dead and subtree(cell(p)) is Dead
                  Δ(p) = Uninit    ⟹  σ(cell(p)) = Garbage

(I3) Live refs.   r ∈ L, σ(cell(r)) = Ref ℓ  ⟹  σ(ℓ) ≠ Dead,  ℓ ∈ subtree(cell(anchor(r))),
                  and Δ(anchor(r)) = Init.

(I4) Order.       For every ref binding r, anchor(r) is declared in the same scope before r,
                  or in an enclosing scope. Hence the frame of anchor(r) is popped no earlier
                  than the frame of r.

(I5) Dead refs.   A ref binding not in L is never on the path of a [Read], [Alias], [Move],
                  [Write] or [Del]. (Syntactic: the rules require membership in L.)

(I6) Writers.     Every cell carries the writer set of its declaration. Every executed
                  [Write] or [Del] on cell(p) is performed by a function in that set.
```

## 4. Lemmas

**Lemma 1 (Anchors are owner paths).** For every place `p`, `cell(anchor(p))` is an ancestor of
`cell(p)` in `own`, or equal to it.

*Proof.* Induction on `p`. For `x`, trivial. For `p.f` inline or heap, `cell(p.f)` is a child of
`cell(p)` and `anchor(p.f) = anchor(p)`, so the inductive hypothesis applies. For `p.f` a ref field
declared `from A`, the new cell is `σ(cell(p.f)) = Ref ℓ`, and by (I3) `ℓ ∈ subtree(cell(A))`;
`anchor(p.f) = A`. ∎

**Lemma 2 (Overlap captures containment).** If `ℓ ∈ subtree(cell(p))` and some live `r` has
`σ(cell(r)) = Ref ℓ`, then `anchor(r) ⋈ p`.

*Proof.* By (I3), `ℓ ∈ subtree(cell(anchor(r)))`. Subtrees of a forest (I1) are nested or disjoint.
`ℓ` lies in both, so `cell(anchor(r))` and `cell(p)` are comparable under `own`, and by Lemma 1
both are owner paths from the same root, so one place is a prefix of the other. ∎

**Lemma 3 (Kill restores I3).** If `Inv` holds before a step that makes every cell of
`subtree(cell(p))` `Dead` or relocated, then after `L := kill(p, L)`, (I3) holds.

*Proof.* Any `r ∈ L` pointing into the affected subtree has `anchor(r) ⋈ p` by Lemma 2 and is
removed. Any `r` that survives points outside `subtree(cell(p))`, whose cells are unchanged. The
condition `Δ(anchor(r)) = Init` survives because `anchor(r)` neither overlaps `p` nor is
below it. ∎

**Lemma 4 (Relocation preserves internal references).** Let `v` be a record value moved from
`cell(p)` to `cell(q)`. Every `Ref` cell inside `v` that pointed inside `v` still points to a live
cell owned by `v` after the move.

*Proof.* By R3, a ref field inside `S` anchored at a sibling requires the sibling to be a heap
field. Heap cells have stable identity under `[Move]`; only their `own` parent changes. A ref
field anchored at a deeper place `f_j.g...` resolves through `f_j` first, so the same argument
applies at the first hop. Ref fields anchored *outside* `S` point to cells not in the moved
subtree and are unaffected, and their anchors remain declared before `q` because `q`'s declared
type names them (rule [Decl], "every `A` named in `τ` is declared before `x`"). ∎

**Lemma 5 (Scope death is safe).** When a frame is popped, no reference binding anchored at a
binding of that frame is in `L`.

*Proof.* By (I4), a reference anchored at `x` is declared after `x` in the same scope or in an
inner scope. Inner scopes are already popped. Same-scope bindings are removed by `[Scope]` in
reverse order, and `[Scope]` pops them from `L`. ∎

**Lemma 6 (Teardown is exactly once).** `teardown(subtree(ℓ))` frees each heap cell in the
subtree exactly once, and never encounters a cell that is `Dead` but has live children.

*Proof.* Exactly once: `own` is a forest (I1), so post-order visits each cell once, and a cell
visited is marked `Dead`, which the walk skips on any later encounter. No dead-with-live-children:
`[Scope]` requires a binding with a moved or deleted sub-place to be wholly `Moved`, `Deleted`, or
refilled to `Init`; by (I2) the subtree of a `Moved` or `Deleted` place is entirely `Dead`, and a
refilled place has a fresh subtree with no `Dead` cells. ∎

**Lemma 7 (Writer sets are syntactic).** Every runtime write to a cell originates from a `[Write]`
or `[Del]` statement in some function `π`, or from the call-site rebinding of a `!` argument.

*Proof.* Inspection of the transitions. `[Call]` rebinding is the desugaring `p! = g(rel p, ...)`
in `README.md`, which is a `[Write]` by the caller on its own place. ∎

## 5. Theorems

**Theorem 1 (Preservation).** If `Inv(σ, own, Γ, Δ, L)` holds and `⟨σ, own, F̄, s⟩ → ⟨σ', own', F̄', s'⟩`
under the typing rules of section 2, then `Inv(σ', own', Γ', Δ', L')` holds for the post-state
computed by the rules.

*Proof.* By cases on the step.

- `[Decl]`. A fresh cell, `Init`, with a well-typed value: (I2). If it is a reference, its anchor is
  declared earlier by the side condition: (I4); the value came from `[Alias]` on a readable place,
  so the pointee is live and inside `subtree(cell(anchor))` by Lemma 1: (I3).
- `[Read]`. No store change. The rule requires every ref on the path to be in `L`, so by (I3) each
  hop reaches a live cell, and `Δ(p) = Init` with (I2) gives a `Val` or `Ref`, not `Garbage` or
  `Dead`.
- `[Alias]`. No store change; the produced reference satisfies (I3) by Lemma 1.
- `[Move]`. The source subtree becomes `Dead`; `Δ[p ↦ Moved]` gives (I2). Lemma 3 with `kill(p, L)`
  gives (I3) for outside references; Lemma 4 gives correctness of references inside the moved
  value. (I1): the moved cells are re-parented under the target, which is a single new parent.
- `[Write-v]`. The old subtree is torn down (Lemma 6, exactly once) and replaced. Lemma 3 restores
  (I3). (I6): the rule checked `π ∈ W_last(p)`; under R1 the last hop resolves through a `Ref`
  cell to the pointee, whose writer set is the inner `by`, which is what `W_last` reads.
- `[Write-r]`. The `Ref` cell changes target. The rule requires `anchor(q)` to be the declared
  anchor of the slot or a place below it, so `subtree(cell(anchor(q))) ⊆ subtree(cell(A_decl))`,
  which keeps (I3) for the slot. `kill(p, L)` handles references *to* the slot cell itself.
- `[Del]`. As `[Write-v]` without the replacement; `Δ[p ↦ Deleted]` gives (I2).
- `[Scope]`. Lemma 5 shows no live reference is left dangling by the pop. Lemma 6 shows teardown
  is exactly once and never reaches a dead cell with live children. (I1) is preserved because
  whole root subtrees are removed.
- `[Call]`. The callee frame is typed under `[Fun]`, with its own `Γ`, `Δ`, `L`, so the callee body
  preserves `Inv` by the same cases. At return: references anchored at callee locals are not in
  `L` (Lemma 5); a returned reference is anchored at a parameter named in the `from` clause, which
  the call site re-anchors at the corresponding argument place, and that argument place is `Init`
  and declared before the binding receiving the result, restoring (I3) and (I4). A `!` argument is
  `Moved` for the duration and rewritten at return by a `[Write-v]` in the caller, which applies
  the `[Write-v]` case including `kill`. ∎

**Theorem 2 (Progress).** A well-typed configuration is either terminal or can step, and the step
is not stuck.

*Proof.* The only stuck conditions are `[Read]` on `Garbage` or `Dead`, `[Read]` through a dead
pointee, double free, and a write outside the writer set. The first two are excluded by (I2) and
(I3) with (I5). Double free is excluded by Lemma 6. Writer violations are excluded by (I6) and
Lemma 7. ∎

**Theorem 3 (Memory safety).** No execution of a well-typed program dereferences a `Dead` cell,
reads `Garbage`, frees a heap cell twice, or leaves a heap cell unfreed at program end.

*Proof.* The first three follow from Theorems 1 and 2. For the last: by (I1) every live heap cell
lies in the subtree of some root bound in a frame. At program end every frame is popped. A root
popped `Init` has its subtree torn down. A root popped `Moved` had its subtree re-owned under
another root, which is popped later or earlier by the same argument. Linear types cannot be
dropped without `del` by `[Out]` and `[Scope]`, so they do not escape the count. ∎

**Theorem 4 (Writer discipline).** If a cell is declared `by W`, then every write to it during
execution is performed by a function in `W`. In particular, a cell declared `by ()` holds its
initial value for its entire life.

*Proof.* (I6) is part of `Inv`; Theorem 1 preserves it; `[Decl]` initialisation is not a write.
The R4 side condition ensures a `by W` parameter receives only arguments whose owner declared `W`,
so no chain of calls widens the set. ∎

**Corollary (Caching).** In the single-threaded core, two reads of a place `p` with no intervening
call to any function in `W_last(p)` and no intervening `[Write]` to a place overlapping `p` return
the same value. This is the justification for treating `by ()` as stronger than `const`, and the
reason an asynchronous writer set (the old `ex`) must be marked so the compiler can exclude it from
this corollary.

## 6. Extensions with proof sketches

### 6.1 Shared ownership, `from A & B`

Add a cell state `Shared(n, v)` with a count. `own` becomes a DAG restricted so that a shared cell's
parents are exactly the cells of the places named in its `from` clause. Teardown decrements and
frees at zero.

*Sketch.* (I1) weakens to: the ownership graph is a DAG. Acyclicity: every `from` clause names
places declared earlier (rule `[Rec]`, `[Decl]`), and a repoint or move cannot change the *owner
set* of a shared cell, only which cell occupies a slot. A cycle would require some cell to be
owned by a place declared after the cell's own declaration, contradicting the ordering. Lemma 6
becomes "freed exactly once, on the last decrement", which follows from the count equalling the
in-degree, itself preserved by each owner death decrementing once. The design's statement that
`del` destroys in place must be read as "decrement" on shared cells for this to hold.

### 6.2 Borrows from an unknown source, `from A | B`

A reference with anchor set `{A, B}` is in `L` only while both `A` and `B` are `Init`. `kill(p, L)`
removes it if `p` overlaps either. Lemma 3 goes through unchanged because the condition is a
conjunction.

### 6.3 Threads

Add frames per thread and a `[Spawn]` rule whose arguments are moved (`rel`) or are references to
cells declared `by ()`, or are lock cells. A lock cell owns its payload; `[Lock]` returns a linear
guard anchored at the lock and marks the payload's writer set as `{holder}`; `[Unlock]` consumes
the guard.

*Sketch.* Data race freedom: a cell writable by two threads would need two writer paths, one per
thread. Moved cells have one owner in one thread. `by ()` cells have no writer. Lock payloads have
writer set `{holder}` and at most one live guard, since the guard is linear and `[Lock]` blocks
while one exists. Refcounts on shared cells crossing threads must be atomic; this is an
implementation obligation, not a typing one. The ordering edge published by unlock is the
region transfer, as the text says.

### 6.4 Region subtraction, `not from`

Syntax: a `from` clause may be followed by `not from N_1, ..., N_k`, where each `N_i` is a place.
The region of the reference is `subtree(A) \ (subtree(N_1) ∪ ... ∪ subtree(N_k))`. The
motivating declaration:

```oura
_ListNode = proto struct(
    value : Item
    prev  : ListNode from head'list not from this | None
    next  : ListNode from rel this | None
)
```

`prev` may point anywhere under `head'list` except into the node itself or anything the node
owns, which is every node after it. The positive region is `head'list` rather than the list root,
so that `tail'self! = @prev'tail'self` type checks: the source region must be a subset of the
target's declared region, and `tail` is declared `from head'self`.

**Changes to the calculus.** A region is now a pair `(P, N)` of path sets. Three definitions
generalise:

```
region(r)          = subtree(P) \ ⋃ subtree(N_i)
kill(p, L)         = { r ∈ L | region(r) ∩ subtree(cell(p)) = ∅ }
subset check       at p! = @q:  region(q) ⊆ region_decl(p)
```

Both tests are decidable by prefix comparison. `region(r) ∩ subtree(p) ≠ ∅` holds iff some
positive path overlaps `p` and no negative path is a prefix of `p`. `region(q) ⊆ region(p)` holds
iff the positive path of `q` lies below the positive path of `p`, and every negative path of `p`
that overlaps the positive path of `q` is either a prefix of it (then `q` is excluded entirely and
the check fails) or is itself below some negative path of `q`.

**Soundness.** (I3) becomes "the pointee lies in `region(r)`". Lemma 2 is restated: if `ℓ ∈
subtree(cell(p))` and a live `r` points to `ℓ`, then `region(r) ∩ subtree(cell(p)) ≠ ∅`. The proof
is immediate, since `ℓ` is in both sets. Lemma 3 follows as before. The only new obligation is
that a pointee, once in the region, stays there: the region is a set of cells fixed by the
declaration, and the sole way a cell leaves it is by being moved, which is a `[Move]` on a place
whose subtree contains it, and that kills `r` by Lemma 2. Repointing re-runs the subset check.
So Theorems 1 to 3 go through unchanged.

**Construction.** In a record literal the cell for `this` does not exist yet, so `subtree(this)`
is empty relative to every existing place. Any alias to a pre-existing place therefore satisfies
`not from this` at construction. `LinkedList.oura:38`, `prev = @tail'self`, passes for this
reason.

**Precision gained.** Under `from root` alone, every write into the list overlaps every
back-reference, since `root` is a prefix of everything. Under `from head'list not from this`,
removing the tail node `n` writes `next'(n-1)`, whose subtree is `{n}`. For each earlier node `j`,
`n ∈ subtree(j)`, so `n` is outside `region(prev_j)`, and no back-reference dies. This is the
per-node invalidation the text wanted and could not get from root anchoring.

**Precision not gained.** After construction, `prev'n! = @tail'self` is rejected, because
`region(tail) = subtree(head)` is not a subset of `subtree(head) \ subtree(n)`. Relinking through
sources whose region already excludes `n`, such as `@prev'n` or `@prev'prev'n`, is accepted.
A source whose region is "the whole list" can never be shown to avoid one node statically.

### 6.5 Forward anchors

The text requires an anchor to be declared before the reference. Section 2 encoded this in
`[Decl]`, `[Rec]` and (I4). This section drops the requirement: a `from` or `not from` clause may
name a field or local declared later, and a reference may be initialised from one.

**Where the order was used.** Only in two places.

1. Lemma 5, scope death. A reference anchored at `y` must not be live after `y` is popped.
   Ordering guaranteed that `r` was popped first.
2. Section 6.1, acyclicity of shared ownership. Ordering guaranteed that no cell is owned by a
   place declared after the cell's own declaration.

Nothing else depends on it. In particular Lemma 1 (anchors are owner paths), Lemma 2 (overlap),
Lemma 4 (relocation) and R3 are order-free, and structural teardown never follows a reference,
because `operator del` is not overridable. That last fact is what makes the change cheap: within
a record all fields die together, and teardown of a `Ref` cell is a no-op, so no teardown order
can observe a dangling reference.

**Replacement for Lemma 5.** Popping a binding is a write to it. `[Scope]` performs
`L := kill(y, L)` for each popped `y`, which R2 already prescribes for every other change to a
place. A reference anchored at `y` that is still in scope becomes dead in `L`. By (I5) it can no
longer be read, aliased, moved through or written through until it is reassigned, and
reassignment re-runs the subset check against an anchor that must be `Init`. (I4) is deleted;
(I3) alone carries the obligation. The `[Decl]` side condition "every `A` named in `τ` is
declared before `x`" becomes "every `A` named in `τ` is a place in the same or an enclosing
scope"; the anchor need not be `Init` at declaration, only at each `[Alias]`, which `[Alias]`
already demands.

**Replacement for acyclicity.** Borrowing regions (`from A` without `rel`) own nothing, so cycles
among them are harmless. Owning regions (`from rel this`, and `from rel this & B` in 6.1) form
the declaration-level ownership graph: an edge from field `f` to field `g` whenever `g`'s clause
names `f` as an owner. This graph is finite and static, so the rule "declared earlier" is
replaced by "the ownership graph has no cycle", which is checked per program. Under lexical
ordering the graph was acyclic by construction; under forward anchors it is acyclic by check.
With that, the argument in 6.1 is unchanged.

**Theorem 1 with forward anchors.** The `[Scope]` case now cites the kill rather than Lemma 5.
The `[Decl]` case no longer establishes (I4) and instead establishes (I3) at the first `[Alias]`
into the binding. All other cases are unchanged. ∎

**What this permits.** Mutually referring siblings, a parent field declared after the children
that reference it, and a local reference declared before the value it will point at and filled
later. It does not permit two peers to *own* each other, which is the cycle the graph check
rejects.

## 7. What the proof does not cover

- Bit casts. Soundness of `Bits32'f` depends on the target type accepting every bit pattern, or
  the cast being a checked `as` narrowing. Neither is in the calculus.
- `# ExternC`. Anything behind it is an axiom.
- Refinement checking. Bounds such as `Index'arr` are assumed decided correctly by the refinement
  checker; the calculus treats indexed access as a call with a precondition.
- Arithmetic overflow and stack depth.
- Generators. `items'self` in `LinkedList.oura` holds `cur` across yields. The calculus covers it
  only if the generator is elaborated to a record holding `cur : ref ListNode from self`, in which
  case R2 kills the generator on any `self!` and the iterator-invalidation class is rejected.
- Region parameters on standalone recursive types. Not in the calculus because their syntax is
  undecided.

## Unresolved questions

- R1 exact: `p! = rel v` through a ref: move into referent, or forbidden?
- Kill granularity: overlap on places (Rust-like, as proved) or whole root (coarser, simpler)?
- Generators: elaborate to records so R2 applies?
- `del` on shared: decrement, or forbidden?
- `not from` inside a `|` region, `from A | B not from N`: subtract from each, or from the union?
- Forward-anchored local with deferred init: unreadable until aliased, or `| None` required?
- Ownership-graph acyclicity across mutually recursive record types: check on the instantiated types, or on declarations with type parameters as opaque?
- `[Out]` on a returned ref anchored at a callee local: reject, or auto-move the local into `out`?
- Refcount atomicity across threads: implementation rule, or type-level distinction (`Rc` vs `Arc`)?
