# C and C++ memory constructs in Oura

Each entry shows a C or C++ construct and its spelling under the `from` / `by` design in
`RefinedIdea.md`. The Oura side uses the refined rules, not the current `.oura` files, so `@`
appears only on values, `ex` is gone, and teardown is structural.

Constructs that cannot be ported directly are collected in `CCppUnportable.md`.

## Assumptions used throughout

These are rules the design implies but does not state. `RefinedIdeaAnalysis.md` argues for each.

- A field without a `from` clause is `from this`.
- A field without a `by` clause is writable by anyone holding the record writable.
- For a reference slot `p`, `p! = @e` repoints, `p! = v` writes the referent, `p! = rel v` moves
  into the referent.
- A `!` on a place ends every reference anchored in that place.
- `use self!` is shorthand for `use self by <this function>`.

## Values and constants

### Struct by value

```c
struct Point { int x, y; };
struct Point p = {1, 2};
p.x = 3;
```

```oura
Point = proto struct(
    x : Int32
    y : Int32
)

p! = Point(x = 1, y = 2)
x'p! = 3
```

### `const` local, `const` field

```c
const int n = 5;
struct Id { const int value; };
```

```oura
n = 5                        /*/ unmarked binding, never written

Id = proto struct(
    value : Int32 by ()      /*/ nobody writes, stronger than const: no const_cast exists
)
```

### `const` member function

```cpp
struct Stack {
    size_t count() const;
    void push(int v);
};
```

```oura
Stack = proto (
    _self = this
    given use self  { count => : Count64 }        /*/ reads self
    given use self! { push(v : Int32) => : None } /*/ writes self
)
```

### `static` local counter

```c
int nextId(void) {
    static int counter = 0;
    return ++counter;
}
```

```oura
_counter! : Int32 by nextId = 0     /*/ module scope, only nextId may write it

nextId => Int32 {
    counter! + 1
    out counter
}
```

## Pointers to existing data

### Pointer to a local

```c
int a = 1;
int *p = &a;
*p = 2;
int b = 5;
p = &b;
```

```oura
a! = 1
p! = @a from (a | b)   /*/ region names both possible targets
p! = 2                 /*/ writes a
b! = 5
p! = @b                /*/ repoints
```

### Pointer to `const`, `const` pointer

```c
const int *p = &a;      /* may repoint, may not write a    */
int *const q = &a;      /* may not repoint, may write a    */
const int *const r = &a;
```

```oura
p! = @(a by ()) from a
q! = @a by this from a
r  = @a from a
```

### Nullable pointer

```c
struct Node *next = NULL;
if (next) proceed(next->value);
```

```oura
next : Node from list | None
if use next as Node { proceed(value'next) }
```

### Pointer to pointer, out parameter that repoints

```c
void push(struct Node **head, int v) {
    struct Node *n = malloc(sizeof *n);
    n->value = v; n->next = *head; *head = n;
}
```

The head slot survives past the return value, so the function takes the owner writable:

```oura
push(head! : Node from (rel this | out) | None, v : Int32) => {
    head! = Node(value = v, next = rel head)
}
```

### Function returning a reference into its argument

```cpp
int &at(std::vector<int> &v, size_t i) { return v[i]; }
```

```oura
at(use v, i : Index'v) => : Int32 from v      /*/ Ownership.oura:19 already does this
```

### View, span, `string_view`

```cpp
std::span<int> items(Block &b) { return {b.data, b.capacity}; }
```

```oura
items(use self from out) => items'data'self    /*/ Ownership.oura:20
```

## Heap ownership

### `unique_ptr`, owned heap field

```cpp
struct Tree {
    int value;
    std::unique_ptr<Tree> left, right;
};
```

```oura
Tree = proto struct(
    value : Int32
    left  : Tree from rel this | None
    right : Tree from rel this | None
)
```

`rel` moves it, structural teardown frees it, and a private `operator new Tree(Tree)` forbids
copying if the C++ type was move-only.

### `malloc` and `free`

```c
int *buf = malloc(n * sizeof(int));
/* ... */
free(buf);
```

```oura
buf = Buffer(Int32, n)       /*/ from rel this on the local; Ownership.oura:37
/*/ ... */
/*/ freed when buf dies unmoved, or explicitly: */
del buf
```

### `realloc`

```c
buf = realloc(buf, 2 * n * sizeof(int));
```

```oura
buf = resized(rel buf, 2 * n)      /*/ Ownership.oura:45
```

### `shared_ptr`

```cpp
auto s = std::make_shared<Config>();
Worker a{s}, b{s};
```

```oura
Worker = proto struct(
    config : Config from rel this & other'self   /*/ other declared earlier in the enclosing scope
)
```

`&` is refcounted when the two owners' deaths are not ordered at compile time. A cycle cannot be
formed because the declaration-level ownership graph is checked to be acyclic.

### `shared_ptr` cycles: the group owner

```cpp
struct Widget {
    std::shared_ptr<Widget> parent;               // leaks: parent and child keep each other alive
    std::vector<std::shared_ptr<Widget>> children;
};
```

The intent behind a cycle is always a group that lives as one unit while anything outside still
needs it. Oura spells the group as one owner and the cross-links as borrows into it:

```oura
Tree = proto struct(
    _nodes : Arena'Widget from rel this           /*/ the group owner: every widget lives here

    _Widget = proto struct(
        parent   : Widget from nodes'tree not from this | None
        children : Listable(Widget from nodes'tree not from parent'this)
    )
)
```

The tree dies when its binding dies, every widget with it, and no count exists. `not from this`
keeps `parent` pointing strictly upward, so removing a subtree invalidates only references into
that subtree. The ownership-graph check rejects the literal port (`parent` and `children` both
owning) as a cycle, which is the leak C++ would have had.

Where the C++ code used `weak_ptr` to break the cycle, the Oura version needs nothing extra: the
borrow already cannot outlive the group.

### `weak_ptr`

No direct form. Use an arena and a generation-checked index:

```oura
Handle = proto struct(
    index      : Index'arena
    generation : Unsigned32
)

get(use arena, h : Handle) => Item from arena | None {
    out item(arena, index'h) if generation(arena, index'h) ?= generation'h else None
}
```

### Arena, region allocator

```c
struct Arena a; arena_init(&a, 1 << 20);
struct Node *n = arena_alloc(&a, sizeof *n);
```

```oura
arena = Arena(1 << 20)
n = alloc(arena!, Node(...)) : Node from arena    /*/ the arena owns, n refers
```

Everything allocated from `arena` is anchored to it and dies with it.

## Manual lifetime

### Placement new, explicit destructor

```cpp
alignas(T) unsigned char slot[sizeof(T)];
T *t = new (slot) T(args);
t->~T();
```

```oura
slot! : T | GarbageBytes'size'Bytes'T     /*/ storage declared without an object
slot! = T(args)                            /*/ construct in place
del slot                                   /*/ destroy in place; slot is invalid until reassigned
```

### Buffer with a live prefix

```c
struct Vec { size_t len, cap; T *data; };  /* data[0..len) constructed, rest raw */
```

```oura
/*/ SmallStack.oura:6 does exactly this */
filledItems : Array(Item, count)
freeItems = GarbageBytes'freeCount
```

### Free-list arena with scattered live slots

```c
union Slot { T value; union Slot *nextFree; };
```

```oura
Slot = proto struct(
    live  : Bool
    value : T if live'this else Index'pool    /*/ type-level if, checked on each access
)
```

Each read is a narrowing rather than a proof, as the text says.

### Flexible array member, VLA

```c
struct Msg { size_t len; char body[]; };
void f(size_t n) { int tmp[n]; }
```

```oura
Msg = proto struct(
    len  : Count
    body : Array(Byte, len)
)

f(n : Count) => { tmp! = Array(Int32, n) }
```

## Mutability control

### `volatile` hardware register

```c
volatile uint32_t *status = (uint32_t *)0x4000;
while (!(*status & READY)) ;
```

```oura
status : Unsigned32 by mod Device       /*/ mod Device is declared # ExternC
while not (status & READY) { }          /*/ never cached across iterations
```

### `mutable` cache

```cpp
struct Shape {
    mutable std::optional<double> areaCache;
    double area() const;
};
```

```oura
Shape = proto struct(
    _areaCache : Real | None by MemoCache
    _MemoCache = (area)

    area(use self by MemoCache) => Real {
        if use areaCache'self as Real { out areaCache'self }
        areaCache'self! = compute'self
        out areaCache'self
    }
)
```

### `friend`

```cpp
class Stack { friend class StackDebugger; size_t count; };
```

```oura
count : Unsigned64 by CountChange       /*/ SmallStack.oura:72
_CountChange = (push, pop, concat, ensureCapacity, StackDebugger)
```

### Read-public, write-private

```cpp
class C { int n_; public: int n() const { return n_; } };
```

```oura
_n : Int32
n => n'self          /*/ every use site stays count'self, count'list
```

### Deleted copy, defaulted move

```cpp
struct File {
    File(const File &) = delete;
    File(File &&) = default;
};
```

```oura
File = proto struct(
    use _operator new File(File)       /*/ copy is private: affine
    /*/ rel is always available and always a bit copy */
)
```

## Resources

### RAII memory

```cpp
struct Block { int *data; ~Block() { delete[] data; } };
```

```oura
Block = proto struct(
    _data : Buffer'Item     /*/ from rel this; structural teardown frees it
)
```

The explicit destructor at `Ownership.oura:25` disappears.

### RAII non-memory resource: linear type

```cpp
struct Socket { ~Socket() { ::close(fd); } };
```

```oura
Socket = proto struct(
    _fd : Int32
    use _operator new Socket(Socket)
    use _operator del                       /*/ linear: must go through close
)

close(socket : Socket) => { sysClose(fd'socket); del socket }

/*/ caller */
s = connect(addr)
send(s!, data)
close(rel s)                                 /*/ required on every exit path
```

### Mutex protecting data

```cpp
std::mutex m; int shared;
{ std::lock_guard<std::mutex> g(m); shared++; }
```

```oura
Mutex(T : proto Any) = proto struct(
    _payload : T from rel this
    use _operator del                        /*/ guard below is linear too
)

Guard(T) = proto struct(
    payload : T from mutex                   /*/ writable through the guard
    use _operator new Guard(Guard)
    use _operator del
)

lock(use m from(out)) => : Guard'T
unlock(g : Guard'T) => { del g }

/*/ caller */
g = lock(m)
payload'g! + 1
unlock(rel g)
```

The lock owns the payload, so the region transfers with the guard and the release ordering is
implied.

### Transaction that must commit or roll back

```cpp
struct Tx { ~Tx() { if (!done) rollback(); } };
```

```oura
Tx = proto struct( use _operator del )
commit(tx : Tx) => { ...; del tx }
rollback(tx : Tx) => { ...; del tx }
/*/ dropping tx without either is a compile error */
```

## Aggregates and reinterpretation

### Tagged union

```c
struct V { enum { INT, ERR } tag; union { int i; struct Err e; }; };
```

```oura
Int32 | OutOfBoundsError            /*/ Bounds.oura:20
```

### Untagged union, `bit_cast`

```cpp
float f; uint32_t bits = std::bit_cast<uint32_t>(f);
```

```oura
bits = Bits32'f                     /*/ conversion is a call; legal because Float32 is flat
```

### Bitfields and flags

```c
struct Flags { unsigned ready : 1, error : 1, mode : 2; };
```

```oura
Flags = proto flags(Unsigned8)(
    ready : Bit
    error : Bit
    mode  : Bits'2
)
use rawByte as Flags else { out BadRegisterValue }     /*/ checked narrowing, reserved bits verified
```

### Network header

```c
struct Hdr { uint16_t len; uint16_t type; } __attribute__((packed));
struct Hdr *h = (struct Hdr *)buf;
```

```oura
Hdr = proto struct(len : Unsigned16BE, type : Unsigned16BE)
use items(buf, range(0, 4)) as Hdr else { out Truncated }
```

### Downcast

```cpp
if (auto *c = dynamic_cast<Circle *>(shape)) ...
```

```oura
if use shape as Circle { ... }
```

## Functions

### Function pointer with context

```c
void forEach(struct List *l, void (*f)(void *ctx, int v), void *ctx);
```

```oura
forEach(use l, f : (Int32) -> : None) => : None
forEach(list, (v) -> { total! + v })        /*/ closure captures total by reference
```

### Virtual dispatch

```cpp
struct Shape { virtual double area() const = 0; };
```

```oura
Shape = proto struct(
    area : (use self) -> : Real                 /*/ Functions.oura ImplRecord fills such slots
)
```

### Out parameters

```c
void divmod(int a, int b, int *q, int *r);
```

```oura
divmod(a : Int32, b : Int32, q! : Int32, r! : Int32) => : None
divmod(7, 2, q!, r!)
```

### Exceptions

```cpp
try { f(); } catch (const Error &e) { handle(e); }
```

```oura
Error = Exceptional'class        /*/ Exception.oura: #CatchStack picks unwinding as the ABI
f() else { handle(Error) }
```

## Linked structures

### Singly linked list

```c
struct Node { int v; struct Node *next; };
```

```oura
Node = proto struct(
    v    : Int32
    next : Node from rel this | None
)
```

### Doubly linked list

```c
struct Node { struct Node *prev, *next; };
```

```oura
/*/ LinkedList.oura, back-reference anchored to the list root */
_ListNode = proto struct(
    value : Item
    prev  : ListNode from list | None
    next  : ListNode from rel this | None
)
```

A back-reference is valid while the list is, not while its node is. Removing a node happens only
in methods that take `self!`, which ends every outside alias.

### Tree with parent pointer

```c
struct T { struct T *parent, *left, *right; };
```

```oura
Tree = proto struct(
    Root = this                                  /*/ the outermost owner
    _root : Tree from rel this
    _TreeNode = proto struct(
        parent : TreeNode from root | None
        left   : TreeNode from rel this | None
        right  : TreeNode from rel this | None
    )
)
```

The node type is nested so it can name the root. A standalone node type cannot; see
`CCppUnportable.md`.

### Intrusive list, object in two lists

```c
struct Obj { struct ListHead byName, byTime; };
```

```oura
Obj = proto struct(
    byName : Index'nameList | None
    byTime : Index'timeList | None
)
```

Both lists are arenas; the object holds indexes into each. Mutation goes through one owner at a
time.

### Union-find

```c
int find(int *parent, int x) { while (parent[x] != x) x = parent[x] = parent[parent[x]]; return x; }
```

```oura
find(parent! : Array'Index, x : Index) => Index {
    x! = x
    while item(parent, x) != x {
        item(parent!, x) = item(parent, item(parent, x))
        x! = item(parent, x)
    }
    out x
}
```

Indexes make the aliasing sequential; nothing changes shape.

### Pointer equality

```c
if (a == b) /* same object */
```

```oura
if ia ?= ib                     /*/ indexes; ?= on records is structural
```

## Threads

### Spawn with moved data

```cpp
std::thread t([buf = std::move(buf)] { work(buf); });
```

```oura
t = spawn((buf) -> work(buf), rel buf)       /*/ ownership crosses; no alias remains here
```

### Signal handler flag

```c
volatile sig_atomic_t stop = 0;
void onSig(int) { stop = 1; }
```

```oura
stop : Bool by mod Signals                    /*/ mod Signals is # ExternC
```

### DMA buffer

```c
volatile uint8_t rxBuf[256];
```

```oura
rxBuf : Array(Byte, 256) by mod Device
```
