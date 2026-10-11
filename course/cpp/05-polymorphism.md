# Module 5 · Polymorphism

Read alongside: Stroustrup, *A Tour of C++*, chapter 5

A `Shape*` that calls the right `area()` for a circle or a square looks like
magic. It is one hidden pointer per object and one table per class. This
module draws both.

## 5.1 Inheritance and object layout

**You will:** see a derived object as a base object with more after it.

A derived class contains its base class as its first part (a *base
subobject*), followed by its own members. That is why a `Circle*` can be
converted to a `Shape*`: in single inheritance both point to the same address,
and a `Shape*` simply sees the first part.

```cpp
struct Shape {
    int x, y;
};

struct Circle : Shape {
    int r;
};

int main() {
    Circle c;
    c.x = 1; c.y = 2; c.r = 5;
    Shape* s = &c;      // points to the Shape part of c
}
```

```memory
frame main
  c: Circle = {Shape: {x: 1, y: 2}, r: 5}
  s: Shape* = -> c.Shape
```

### Check yourself

1. What is `sizeof(Circle)`?
   <details><summary>Answer</summary>

   12: the base's two `int`s and its own one.
   </details>

2. Through `s`, can you read `r`?
   <details><summary>Answer</summary>

   Not without a cast: a `Shape*` only knows the `Shape` part.
   </details>

### Common mistake

Inheriting to reuse code ("a `Stack` is a `vector`"). Inherit when the derived
class *is a* base and can be used wherever one is expected; otherwise hold the
other class as a member.

## 5.2 Virtual functions: the vptr and the vtable

**You will:** see how a call through a base pointer finds the right function.

A `virtual` function can be overridden by derived classes, and a call through
a base pointer or reference runs the derived version. The usual mechanism:

- the compiler makes one *vtable* per class: a table of addresses of that
  class's virtual functions, in read-only memory;
- every object of a class with virtual functions gets a hidden first member,
  the *vptr*, pointing to its class's table;
- `s->area()` loads the vptr, picks the entry for `area`, and calls through it
  (C lesson 11.1's function pointers, in a table).

```cpp
#include <memory>
#include <vector>

struct Shape {
    virtual double area() const = 0;    // pure: Shape is abstract
    virtual ~Shape() = default;
};

struct Circle : Shape {
    double r;
    explicit Circle(double r) : r(r) {}
    double area() const override { return 3.14159 * r * r; }
};

struct Square : Shape {
    double side;
    explicit Square(double s) : side(s) {}
    double area() const override { return side * side; }
};

int main() {
    std::vector<std::unique_ptr<Shape>> shapes;
    shapes.push_back(std::make_unique<Circle>(1.0));
    shapes.push_back(std::make_unique<Square>(2.0));
    double total = 0;
    for (const auto& s : shapes)
        total += s->area();
}
```

```memory
step: the shapes, their vptrs, the tables
area vtables (read-only)
  vtable for Circle: table = [Circle::area, Circle::~Circle]
  vtable for Square: table = [Square::area, Square::~Square]
frame main
  shapes: vector<unique_ptr<Shape>> = {data: -> #1, size: 2, capacity: 2}
  total: double = 0
heap #1: unique_ptr<Shape>[2] = [-> #2, -> #3]
heap #2: Circle = {vptr: -> vtable for Circle, r: 1.0}
heap #3: Square = {vptr: -> vtable for Square, side: 2.0}
```

The cost: 8 bytes per object, and one extra memory load per call that the
compiler often cannot inline.

### Check yourself

1. What is `sizeof(Circle)` here?
   <details><summary>Answer</summary>

   16: the vptr (8) and `r` (8).
   </details>

2. Why does `Shape` have a virtual destructor?
   <details><summary>Answer</summary>

   The `unique_ptr<Shape>`s delete through a `Shape*`; lesson 5.3 shows what
   happens without it.
   </details>

### Common mistake

Calling a virtual function from a constructor. While `Shape`'s constructor
runs, the vptr still points to `Shape`'s table: the derived override is not
called.

## 5.3 Slicing and virtual destructors

**You will:** avoid the two classic polymorphism bugs.

**Slicing.** Passing or assigning a derived object *by value* to a base
variable copies only the base part. The derived members, and the derived
vptr, are cut off.

```cpp
void print_area(Shape s);          // by value: would slice (and does not compile here, Shape is abstract)
void print_area(const Shape& s);   // by reference: no copy, no slicing
```

**Virtual destructors.** `delete` through a base pointer runs the destructor
it finds through that pointer. If the base destructor is not virtual, only
the base part is destroyed: the derived class's members, and whatever they
own, leak.

```cpp
struct Base { ~Base() {} };                 // not virtual
struct Logger : Base { std::string file = "app.log"; };

int main() {
    Base* b = new Logger;
    delete b;       // ~Base runs, ~Logger does not: file's buffer leaks
}
```

```memory
step: after delete b; with a non-virtual destructor
frame main
  b: Base* = -> #1
heap #1: Logger (freed) = {Base: {}, file: {data: -> #1.file.buf, size: 7, buf: "app.log"}}
```

(Here the string is short enough to live inside the object, so only the
object's block is involved; with a longer file name its heap buffer would be
left behind, unfreed.)

The rule: a class meant to be a base, deleted through base pointers, gets a
`virtual` destructor.

### Check yourself

1. Why is slicing silent?
   <details><summary>Answer</summary>

   Copying a derived object into a base object is a legal conversion; nothing
   tells the compiler you wanted the whole object.
   </details>

2. Does a class with no virtual functions need a virtual destructor?
   <details><summary>Answer</summary>

   Only if it will be deleted through a pointer to a base. A plain value type
   does not.
   </details>

### Common mistake

Storing derived objects in a `std::vector<Shape>`. Every element is sliced to
a `Shape`; store `std::unique_ptr<Shape>` instead, as in lesson 5.2.
