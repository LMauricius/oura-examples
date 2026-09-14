# C and C++ constructs without a direct Oura port

Each entry names the construct, states why the `from` / `by` design rejects it, and gives the
nearest rewrite. Rejections fall into three groups: address computation, user-defined teardown,
and references whose anchor cannot be named.

## Address computation

### `container_of`, `offsetof`

```c
struct Obj *o = container_of(link, struct Obj, link);
```

Recovers the owner from the address of a member: the reverse of `from`. References cannot be
punned, so there is no way to go from a member to its container.

**Rewrite:** the link holds an index into the owner's arena, so the owner is recovered by lookup.

### Tagged pointers, NaN-boxing of references

```c
uintptr_t tagged = (uintptr_t)p | 1;
```

A reference is not its bytes, so casting one to an integer is rejected. NaN-boxing of flat values
(integers, floats) is fine; boxing a reference is not.

**Rewrite:** put the tag bits in an index. `Index & Flags` in one word is a flat type.

### `void *` type erasure

```c
void qsort(void *base, size_t n, size_t sz, int (*cmp)(const void *, const void *));
```

Recovering the type from `void *` is an unchecked assertion.

**Rewrite:** generics. `sort(items! : Listable'Item, cmp : (Item, Item) -> : Ordering)`. At a C
ABI boundary, the cast stays inside `# ExternC`.

### Pointer arithmetic across objects, pointer comparison by address

```c
if (p < q) ...
ptrdiff_t d = q - p;
```

Records have no unique address; `?=` is structural.

**Rewrite:** indexes into one buffer. `Index'buf` compares and subtracts.

### Byte reinterpretation of non-flat types

```cpp
auto *raw = reinterpret_cast<unsigned char *>(&node);
```

Legal only for types whose fields are transitively `from this`. A record with a reference field
cannot be viewed as bytes.

**Rewrite:** serialise the references as indexes first, then the record is flat.

## User-defined teardown

Neither `operator rel` nor `operator del` is overridable. Everything below is a destructor that
does work other than freeing memory.

### `lock_guard`, scope guard, `defer`

```cpp
{ std::lock_guard<std::mutex> g(m); /* ... */ }   // unlocks on any exit
```

**Rewrite:** the guard is linear and `unlock(rel g)` is written on every exit path, including
each `out` and each error branch. The compiler rejects a missing call, so the guarantee is kept.
The convenience is lost.

### Destructor with side effects: flush, log, unregister

```cpp
struct Logger { ~Logger() { flush(); } };
```

**Rewrite:** linear type plus a named `close`. Same cost as above.

### Custom move constructor

```cpp
struct SmallString {
    char buf[16]; char *ptr;         // ptr points into buf when short
    SmallString(SmallString &&o) { ...; ptr = buf; }
};
```

`rel` is always a bit copy, and a reference into an inline sibling is exactly the pattern
`RefinedIdeaAnalysis.md` section 4.2 shows to be unsound under relocation.

**Rewrite:** replace the interior pointer with a discriminant. `SmallStack.oura` shows the shape:
a type-level `if` on a count chooses between an inline array and a heap buffer, and no pointer
into the inline half exists.

### Intrusive node that unlinks itself on destruction

```cpp
struct Node { ~Node() { prev->next = next; next->prev = prev; } };
```

**Rewrite:** removal is a method of the list, taking `self!`, which is where the invariant lives
anyway.

### Reference counting with a custom deleter

```cpp
std::shared_ptr<FILE> f(fopen(...), fclose);
```

`&` counts owners, but the teardown at zero is structural.

**Rewrite:** the shared payload is a linear wrapper record, and the last owner is responsible for
calling its `close`. Determining "last" statically needs the owners' deaths ordered; otherwise the
wrapper is held by one owner and the others hold `|` borrows.

## Anchors that cannot be named

### Reference to a peer created later

```cpp
Observer a; Subject s; s.attach(&a);   // s points at a, both are locals in one scope
Widget parent; Widget child; child.parent = &parent; parent.children.push_back(&child);
```

`from` must name a place that already exists, and a fixed declaration cannot be anchored to
something declared after it. Mutual references between two locals or two siblings have no anchor.

**Rewrite:** a common root declared first: a registry, an arena, or the enclosing record. Both
peers anchor to it and find each other by index.

### Standalone recursive node type with a parent pointer

```cpp
struct TreeNode { TreeNode *parent; std::unique_ptr<TreeNode> left, right; };
```

Nested inside a `Tree` record, `parent : TreeNode from root` works because the root is a lexical
name. As a top-level type, the node cannot name whoever will own the tree.

**Rewrite:** nest the node type in the container, or give it a region parameter. Region parameters
are the "generic referents" open question in `RefinedIdea.md`.

### Self-referential struct on the stack

```c
struct Ctx { char buf[256]; char *cursor; };   /* cursor points into buf */
struct Ctx c = { .cursor = c.buf };
```

Same relocation problem as the custom move constructor.

**Rewrite:** `cursor : Index'buf`.

### Global mutable state written from anywhere

```c
int g_debugLevel;
void anything(void) { g_debugLevel = 2; }
```

Not rejected, but the spelling is open. The `by` set for "every function in the program" has no
name yet, and the default for a global without a clause is undecided.

**Rewrite:** `g_debugLevel : Int32 by mod Self`, where `mod Self` is the current module, if the
writer is confined to one module. Otherwise wait for the decision.

### Static storage for string literals

```c
const char *s = "hello";   /* lives for the program */
```

A reference needs a region, and the program-lifetime region has no name.

**Rewrite:** `s : Text by () from mod Self` if a module counts as a region. Open.

## Control flow

### `setjmp` / `longjmp`, `goto` into cleanup

```c
if (setjmp(env)) goto cleanup;
```

`out` leaves a named lexical block; it cannot cross function frames or jump forward.

**Rewrite:** unions for errors, `#CatchStack` for the unwinding ABI, named blocks for early exit.
Cleanup that C reaches through `goto` is structural teardown or explicit linear drops.

### Exceptions escaping from a destructor, or thrown during unwinding

No implicit exits and no user teardown, so the question does not arise. Nothing to port.

## Intentionally excluded

These are C behaviours the design removes on purpose, listed so they are not mistaken for gaps.

- `const_cast`: `by ()` means no writer exists, and there is no way to add one after the fact.
- Uninitialised reads: storage without an object is `GarbageBytes` and is not the value's type.
- Use after `free`: `del` makes the place unreadable until reassigned.
- Double `free`: same.
- Iterator invalidation: a `!` on the container ends the iterator's alias.
- Dangling return of a local's address: a return without `from` cannot carry a reference.
- Reference cycles through `shared_ptr`: `from` names earlier places only.
