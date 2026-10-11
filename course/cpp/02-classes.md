# Module 2 · Classes and objects

Read alongside: Stroustrup, *A Tour of C++*, chapters 5–6

A class is a C struct that also owns the functions that work on it, and
controls how it is created, copied and destroyed. Each of those controls
becomes plain C in memory; this module shows which.

## 2.1 Member functions and `this`

**You will:** see that a member function is an ordinary function with a
hidden pointer argument.

```cpp
struct Point {
    int x, y;
    void move(int dx, int dy) {
        x += dx;        // means this->x += dx
        y += dy;
    }
};

int main() {
    Point p{3, 4};
    p.move(1, 1);
}
```

`p.move(1, 1)` compiles to roughly `Point_move(&p, 1, 1)`: the object's
address is passed as a hidden parameter called `this`, and `x` inside the
function means `this->x`. The object holds only `x` and `y`; the code of
`move` exists once, not once per object.

```memory
step: inside p.move(1, 1)
frame main
  p: Point = {x: 3, y: 4}
frame Point::move
  this: Point* = -> p
  dx: int = 1
  dy: int = 1
step: after y += dy;
frame main
  p: Point = {x: 4 !, y: 5 !}
frame Point::move
  this: Point* = -> p
  dx: int = 1
  dy: int = 1
```

### Check yourself

1. What is `sizeof(Point)`?
   <details><summary>Answer</summary>

   8: two `int`s. Member functions take no space in the object.
   </details>

2. What does a `const` member function, `int norm() const`, promise?
   <details><summary>Answer</summary>

   That `this` is a `const Point*`: the function may read the members, not
   change them. It can be called on `const` objects.
   </details>

### Common mistake

A parameter with the same name as a member: `void set(int x) { x = x; }`
assigns the parameter to itself. Write `this->x = x;` or rename.

## 2.2 Invariants, `private`, constructors

**You will:** make an object that is always valid.

An *invariant* is a rule an object must always satisfy: a fraction's
denominator is never 0 and is positive. Making the members `private` means
only the class's own functions can change them, and a *constructor* makes sure
every object starts valid.

```cpp
#include <numeric>      // std::gcd
#include <stdexcept>

class Fraction {
public:
    Fraction(int num, int den) : num_(num), den_(den) {
        if (den_ == 0)
            throw std::invalid_argument("zero denominator");
        if (den_ < 0) { num_ = -num_; den_ = -den_; }
        int g = std::gcd(num_, den_);
        num_ /= g;
        den_ /= g;
    }
    int num() const { return num_; }
    int den() const { return den_; }
private:
    int num_, den_;
};

int main() {
    Fraction half(3, -6);   // stored as -1/2
}
```

The `: num_(num), den_(den)` part is the *member initializer list*: members
are built there, before the body runs. `class` is `struct` with `private` as
the default.

```memory
step: end of the constructor
frame main
  half: Fraction = {num_: -1 !, den_: 2 !}
frame Fraction::Fraction
  this: Fraction* = -> half
  num: int = 3
  den: int = -6
  g: int = 3
```

### Check yourself

1. Can code in `main` write `half.den_ = 0;`?
   <details><summary>Answer</summary>

   No: `den_` is private. The only way to make a `Fraction` is the
   constructor, which checks.
   </details>

2. Is `private` a protection at run time?
   <details><summary>Answer</summary>

   No. It is checked by the compiler only; in memory the object is the same
   struct as in C.
   </details>

### Common mistake

Members are initialized in the order they are *declared*, not the order of the
initializer list. If one member's initializer uses another declared later, it
reads garbage.

## 2.3 Destructors and RAII

**You will:** tie a resource's life to an object's scope.

A *destructor*, `~Buffer()`, runs automatically when an object's life ends: at
the closing `}` of its scope, in reverse order of construction. Acquire a
resource in the constructor, release it in the destructor, and it can no
longer leak, even on early `return`s. This idea is called *RAII* (Resource
Acquisition Is Initialization), and it is the heart of C++.

```cpp
#include <cstdlib>

class Buffer {
public:
    explicit Buffer(int n) : size_(n), data_(new int[n]) {}
    ~Buffer() { delete[] data_; }
    int& operator[](int i) { return data_[i]; }
private:
    int size_;
    int* data_;
};

int main() {
    Buffer b(4);
    b[0] = 7;
}   // ~Buffer runs here: the block is given back
```

```memory
step: after Buffer b(4);
frame main
  b: Buffer = {size_: 4, data_: -> #1}
heap #1: int[4] = [?, ?, ?, ?]
step: after b[0] = 7;
frame main
  b: Buffer = {size_: 4, data_: -> #1}
heap #1: int[4] = [7 !, ?, ?, ?]
step: at the closing }: ~Buffer() ran
frame main
heap #1: int[4] (freed) = [7, ?, ?, ?]
```

Compare with C lesson 10.1: the `free` is still there, but nobody can forget
to call it.

### Check yourself

1. Two objects `a` then `b` in one scope: in which order are they destroyed?
   <details><summary>Answer</summary>

   `b`, then `a`: reverse order, so later objects may depend on earlier ones.
   </details>

2. What other resources fit RAII?
   <details><summary>Answer</summary>

   Files (`std::ifstream` closes itself), locks (`std::lock_guard` unlocks),
   sockets, database connections: anything that must be given back.
   </details>

### Common mistake

This `Buffer` is still dangerous to copy: lesson 2.4 shows why.

## 2.4 Copying: shallow and deep

**You will:** see why copying a class that owns memory needs care.

By default, copying an object copies each member. For `Buffer` that copies
the *pointer*, not the block: two objects own one block, and both destructors
free it. It is C's double free (lesson 10.3), now hidden in a `}`.

```cpp
int main() {
    Buffer a(4);
    Buffer b = a;   // member-wise copy: b.data_ == a.data_
}   // ~b frees the block, then ~a frees it again
```

```memory
step: after Buffer b = a; (default copy)
frame main
  a: Buffer = {size_: 4, data_: -> #1}
  b: Buffer = {size_: 4 !, data_: -> #1 !}
heap #1: int[4] = [?, ?, ?, ?]
```

The fix is a *copy constructor* that copies the block itself (a *deep* copy),
and a *copy assignment operator* that does the same for `b = a;`:

```cpp
Buffer(const Buffer& other)
    : size_(other.size_), data_(new int[other.size_]) {
    for (int i = 0; i < size_; i++)
        data_[i] = other.data_[i];
}
```

```memory
step: after Buffer b = a; (deep copy)
frame main
  a: Buffer = {size_: 4, data_: -> #1}
  b: Buffer = {size_: 4 !, data_: -> #2 !}
heap #1: int[4] = [?, ?, ?, ?]
heap #2: int[4] = [?, ?, ?, ?]
```

### Check yourself

1. Which classes need a hand-written copy constructor?
   <details><summary>Answer</summary>

   Those that own something through a raw pointer or handle. Classes made of
   values and of members that copy themselves (`std::string`,
   `std::vector`) do not.
   </details>

2. How can you forbid copying altogether?
   <details><summary>Answer</summary>

   `Buffer(const Buffer&) = delete;` and the same for `operator=`. Then
   `Buffer b = a;` does not compile.
   </details>

### Common mistake

Writing the copy constructor and forgetting `operator=`. `b = a;` on existing
objects then still does the shallow copy, and leaks `b`'s old block too.

## 2.5 Moving

**You will:** transfer ownership instead of copying.

Often the source of a copy is about to die anyway (a temporary, a local being
returned). Copying its block and then freeing the original is wasted work. A
*move constructor* steals the pointer and leaves the source empty:

```cpp
Buffer(Buffer&& other) noexcept
    : size_(other.size_), data_(other.data_) {
    other.size_ = 0;
    other.data_ = nullptr;      // the source no longer owns anything
}

int main() {
    Buffer a(4);
    Buffer b = std::move(a);    // a is now empty
}
```

`Buffer&&` is an *rvalue reference*: it binds to objects that may be stolen
from. `std::move(a)` does not move anything; it is a cast that says "you may
steal from `a`".

```memory
step: before the move
frame main
  a: Buffer = {size_: 4, data_: -> #1}
heap #1: int[4] = [?, ?, ?, ?]
step: after Buffer b = std::move(a);
frame main
  a: Buffer = {size_: 0 !, data_: null !}
  b: Buffer = {size_: 4 !, data_: -> #1 !}
heap #1: int[4] = [?, ?, ?, ?]
```

No new block, no copying of elements: one pointer changed hands.

### Check yourself

1. What may you do with `a` after the move?
   <details><summary>Answer</summary>

   Destroy it or assign a new value to it. Its content is "valid but
   unspecified"; do not read it.
   </details>

2. Why `noexcept`?
   <details><summary>Answer</summary>

   `std::vector` only moves its elements when it grows if their move cannot
   throw; otherwise it copies them to stay safe.
   </details>

### Common mistake

Forgetting to null the source. Then both destructors free the block: the
double free again.

## 2.6 The rules of zero, three and five

**You will:** decide which special members a class should write.

- **Rule of three** (C++98): a class that needs a destructor, a copy
  constructor or a copy assignment almost certainly needs all three.
- **Rule of five** (C++11): add the move constructor and move assignment.
- **Rule of zero**: best of all, write none of them. Build the class from
  members that already manage themselves, and the compiler's defaults are
  correct.

```cpp
#include <vector>

class Buffer {          // rule of zero: copy, move, destroy all just work
public:
    explicit Buffer(int n) : data_(n) {}
    int& operator[](int i) { return data_[i]; }
private:
    std::vector<int> data_;
};
```

Lessons 2.3 to 2.5 wrote by hand what `std::vector` already does. Knowing
what is inside is the point of this course; writing it again in every class is
not.

### Check yourself

1. Which members of the rule-of-zero `Buffer` does the compiler generate?
   <details><summary>Answer</summary>

   All five, each calling the matching operation of `std::vector`.
   </details>

2. When is the rule of five still needed?
   <details><summary>Answer</summary>

   When writing a resource-owning type itself: a container, a smart pointer,
   a wrapper around a C library handle.
   </details>

### Common mistake

Declaring only a destructor "to log something". That silently disables the
implicit move operations, and every move becomes a copy.
