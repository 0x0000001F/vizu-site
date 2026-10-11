# Module 3 · Owning memory

Read alongside: Stroustrup, *A Tour of C++*, chapters 6 and 15

In C, every `malloc` has an owner only in the programmer's head. C++ lets the
type system say who owns a heap block, so that ownership can be moved,
shared, and given back automatically.

## 3.1 `new` and `delete`

**You will:** know what `new` and `delete` add to `malloc` and `free`.

`new T(args)` allocates memory for a `T` *and runs its constructor*;
`delete p` runs the destructor *and frees the memory*. For arrays, `new T[n]`
and `delete[] p`. A `new` without its `delete` leaks, a second `delete` is a
double free, and mixing the forms (`delete` on a `new[]`, `free` on a `new`)
is undefined.

```cpp
#include <string>

int main() {
    std::string* s = new std::string("on the heap, with a constructor");
    int* nums = new int[3]{1, 2, 3};
    delete s;           // ~string() runs, then the memory is freed
    delete[] nums;
}
```

```memory
step: after both new
frame main
  s: std::string* = -> #1
  nums: int* = -> #3
heap #1: std::string = {data: -> #2, size: 31, capacity: 31}
heap #2: char[32] = "on the heap, with a constructor"
heap #3: int[3] = [1, 2, 3]
step: after delete s;
frame main
  s: std::string* = -> #1
  nums: int* = -> #3
heap #1: std::string (freed) = {data: -> #2, size: 31, capacity: 31}
heap #2: char[32] (freed) = "on the heap, with a constructor"
heap #3: int[3] = [1, 2, 3]
```

`delete s` freed *two* blocks: the string's destructor freed its character
buffer first. With `free(s)` that inner block would have leaked.

### Check yourself

1. Why can `delete[]` find out how many destructors to run?
   <details><summary>Answer</summary>

   For types with destructors, `new[]` stores the count just before the
   elements, much like the header of C lesson 10.6.
   </details>

2. When should you write `new` in modern C++?
   <details><summary>Answer</summary>

   Almost never directly: use containers, `std::make_unique` and
   `std::make_shared` (lessons 3.2–3.3).
   </details>

### Common mistake

`delete` on memory from `new[]`. With `int` it may seem to work; with
`std::string` elements, only one destructor runs and the rest leak, or the
allocator is corrupted.

## 3.2 `unique_ptr`

**You will:** give a heap object exactly one owner.

`std::unique_ptr<T>` is a small object holding a pointer, whose destructor
deletes what it points to. It cannot be copied (two owners would delete
twice), but it can be moved: ownership passes on, and the source becomes
empty.

```cpp
#include <memory>

struct Sensor {
    int id;
    double value;
};

std::unique_ptr<Sensor> make_sensor(int id) {
    return std::make_unique<Sensor>(Sensor{id, 0.0});
}

int main() {
    std::unique_ptr<Sensor> a = make_sensor(7);
    std::unique_ptr<Sensor> b = std::move(a);   // b owns it now
    b->value = 21.5;
}   // b's destructor deletes the Sensor; a owns nothing
```

```memory
step: after a = make_sensor(7)
frame main
  a: unique_ptr<Sensor> = {ptr: -> #1}
heap #1: Sensor = {id: 7, value: 0.0}
step: after b = std::move(a)
frame main
  a: unique_ptr<Sensor> = {ptr: null !}
  b: unique_ptr<Sensor> = {ptr: -> #1 !}
heap #1: Sensor = {id: 7, value: 0.0}
step: at the closing }
frame main
heap #1: Sensor (freed) = {id: 7, value: 21.5}
```

A `unique_ptr` is exactly the size of a raw pointer and costs nothing at run
time; the rules exist only for the compiler.

### Check yourself

1. Why does `std::unique_ptr<Sensor> c = b;` not compile?
   <details><summary>Answer</summary>

   Its copy constructor is deleted: two owners of one object are not allowed.
   </details>

2. How do you pass a `Sensor` to a function that only uses it?
   <details><summary>Answer</summary>

   As `Sensor&` or `const Sensor&` (`f(*b)`), or a raw `Sensor*` that does not
   own. Pass the `unique_ptr` itself (by value, moved) only to hand ownership
   over.
   </details>

### Common mistake

Using `a` after moving from it: `a->id` dereferences a null pointer.

## 3.3 `shared_ptr` and `weak_ptr`

**You will:** share an object among several owners, and avoid the cycle that
leaks.

`std::shared_ptr<T>` lets several owners share one object. Next to the object
lives a *control block* with a count of owners; copying a `shared_ptr` adds
one, destroying one subtracts one, and the last owner deletes the object.

```cpp
#include <memory>
#include <string>

int main() {
    auto doc = std::make_shared<std::string>("report");
    {
        auto reader = doc;      // a second owner: count 2
    }                           // reader gone: count 1
}                               // doc gone: count 0, the string is deleted
```

```memory
step: inside the inner block
frame main
  doc: shared_ptr<string> = {ptr: -> #1.object, ctrl: -> #1}
  reader: shared_ptr<string> = {ptr: -> #1.object, ctrl: -> #1}
heap #1: control block + string = {owners: 2 !, weak: 0, object: "report"}
step: after the inner block
frame main
  doc: shared_ptr<string> = {ptr: -> #1.object, ctrl: -> #1}
heap #1: control block + string = {owners: 1 !, weak: 0, object: "report"}
```

The trap is a cycle: two objects that own each other through `shared_ptr`s
keep each other's counts above zero forever. Make one of the links a
`std::weak_ptr`, which points without owning; `lock()` gives a
`shared_ptr` if the object still exists.

### Check yourself

1. Why does `make_shared` allocate once, while
   `shared_ptr<T>(new T)` allocates twice?
   <details><summary>Answer</summary>

   `make_shared` puts the control block and the object in one heap block, as
   drawn above; with `new T` the object already exists, so the control block
   needs a block of its own.
   </details>

2. Parent and child nodes point to each other. Which pointer should be weak?
   <details><summary>Answer</summary>

   The child's pointer to its parent: parents own children, not the other
   way round.
   </details>

### Common mistake

Using `shared_ptr` everywhere "to be safe". Shared ownership makes it hard to
know when things die, and every copy updates an atomic counter. Prefer
`unique_ptr`; share only what is truly shared.

## 3.4 Dangling references

**You will:** recognize the C++ versions of C's dangling pointer.

A reference or an iterator is an address in disguise, and it dangles as
easily as a pointer:

```cpp
#include <vector>

const int& first(const std::vector<int>& v) { return v[0]; }

int main() {
    std::vector<int> v{1, 2, 3};
    const int& r = first(v);    // r refers to v's first element
    v.push_back(4);             // capacity was 3: the elements move
    int x = r;                  // r refers to freed memory
}
```

```memory
step: after const int& r = first(v)
frame main
  v: std::vector<int> = {data: -> #1, size: 3, capacity: 3}
  r: const int& = ref #1[0]
heap #1: int[3] = [1, 2, 3]
step: after v.push_back(4)
frame main
  v: std::vector<int> = {data: -> #2 !, size: 4 !, capacity: 6 !}
  r: const int& = ref #1[0]
heap #1: int[3] (freed) = [1, 2, 3]
heap #2: int[6] = [1, 2, 3, 4 !, ?, ?]
```

It is C lesson 10.2's "pointer into the old block across a `realloc`", hidden
inside `push_back`.

### Check yourself

1. Which other operations can move a vector's elements?
   <details><summary>Answer</summary>

   Anything that may grow it: `push_back`, `emplace_back`, `insert`,
   `resize`, `reserve`. `erase` shifts the elements after the erased one.
   </details>

2. Why is `std::string_view sv = std::string("temp");` a bug?
   <details><summary>Answer</summary>

   The temporary string dies at the end of the statement; `sv` points into its
   freed buffer.
   </details>

### Common mistake

Returning a reference to a local, `const std::string& name() { std::string s
= "x"; return s; }`: the compiler warns; the result always dangles.
