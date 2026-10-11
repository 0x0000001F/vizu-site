# Module 1 · From C to C++

Read alongside: Stroustrup, *A Tour of C++* (3rd ed.), chapters 1–3

C++ began as "C with classes", and almost every C program in the C track is
also C++. This module covers what changes first: a new way to print, a new
kind of name for a box, functions that share a name, and a few modern
shorthands.

## 1.1 Hello, C++

**You will:** write and build a C++ program, and see what `std::` and `<<`
are.

```cpp
#include <iostream>
#include <string>

int main() {
    std::string name = "Augusta Ada King";
    int year = 1843;
    std::cout << "Hello, " << name << "! Notes from " << year << ".\n";
}
```

- `<iostream>` gives the stream objects: `std::cout` for output, `std::cin`
  for input. `std::` is the *namespace* the whole standard library lives in.
- `<<` sends a value into the stream. Each type knows how to print itself, so
  there is no `%d` to get wrong.
- `std::string` is a real string type: it knows its length and grows by
  itself. It still keeps its characters in an array; module 4 opens it up.
- `main` may end without `return 0;` in C++.

Build it with `g++ -std=c++20 -Wall hello.cpp -o hello`.

```memory
frame main
  name: std::string = {data: -> #1, size: 16, capacity: 16}
  year: int = 1843
heap #1: char[17] = "Augusta Ada King"
```

(A name of 15 characters or fewer would be kept inside the `std::string`
object itself, with no heap block: lesson 4.1.)

### Check yourself

1. What does `std::cout << 7 / 2 << '\n';` print?
   <details><summary>Answer</summary>

   `3`. Integer division is the same as in C.
   </details>

2. Why not write `using namespace std;` at the top?
   <details><summary>Answer</summary>

   It pulls hundreds of names into your program, some of which (`count`,
   `distance`, `data`) collide with yours. Writing `std::` is short and clear.
   </details>

### Common mistake

Mixing `printf` and `std::cout` on the same output and expecting the order to
hold: they are separate buffers unless synchronized (they are by default, at
some cost).

## 1.2 References

**You will:** use a reference, and see that it is another name for an
existing box.

A *reference* `int& r = x;` is a second name for `x`. It must be bound when it
is made, it can never be bound to another box, and it cannot be null. Reading
or writing `r` reads or writes `x`. That makes `swap` look the way you always
wanted it to:

```cpp
void swap(int& a, int& b) {
    int tmp = a;
    a = b;
    b = tmp;
}

int main() {
    int x = 1, y = 2;
    swap(x, y);     // no & at the call: the references bind to x and y
}
```

```memory
step: entering swap(x, y)
frame main
  x: int = 1
  y: int = 2
frame swap
  a: int& = ref x
  b: int& = ref y
  tmp: int = ?
step: after b = tmp;
frame main
  x: int = 2 !
  y: int = 1 !
frame swap
  a: int& = ref x
  b: int& = ref y
  tmp: int = 1
```

A reference is drawn with a dashed arrow: under the hood the compiler usually
passes an address, exactly like `int *` in C lesson 6.4, but the language
hides it. For big objects, `const T&` gives the speed of a pointer and the
safety of a copy: `void print(const std::string& s)`.

### Check yourself

1. After `int& r = x; r = y;`, what happened?
   <details><summary>Answer</summary>

   `x` got `y`'s value. `r` still refers to `x`; a reference cannot be
   re-bound.
   </details>

2. When should a function take `const std::string&` instead of
   `std::string`?
   <details><summary>Answer</summary>

   When it only reads the string: no copy is made. Small types (`int`,
   `double`) are passed by value.
   </details>

### Common mistake

Returning a reference to a local: `int& f() { int n = 1; return n; }`. Like C
lesson 6.5, the box is gone when the frame is; module 3.4 shows it.

## 1.3 Overloading and what the linker sees

**You will:** give several functions one name, and see how the linker tells
them apart.

C++ allows several functions with the same name if their parameters differ;
the compiler picks one by the arguments' types.

```cpp
int area(int w, int h) { return w * h; }
double area(double r) { return 3.14159 * r * r; }

int main() {
    int a = area(3, 4);     // the (int, int) one
    double b = area(2.0);   // the (double) one
}
```

The linker only knows plain names, so the compiler *mangles* each into one
that includes the parameter types. `nm` on the object file shows them:

```text
_Z4areaii    area(int, int)
_Z4aread     area(double)
```

That is why C++ code calling a C library writes `extern "C"` around its
declarations: it asks for the plain C name, `area`, not a mangled one.

### Check yourself

1. Can two functions differ only in their return type?
   <details><summary>Answer</summary>

   No. A call does not say what return type it wants, so the compiler could
   not choose.
   </details>

2. Why does a C++ program fail to link against a C library without
   `extern "C"`?
   <details><summary>Answer</summary>

   It looks for `_Z...` mangled names, while the library defines plain ones.
   </details>

### Common mistake

Overloads that make calls ambiguous: `f(long)` and `f(double)` called with an
`int`. Both need a conversion, and the compiler refuses to guess.

## 1.4 `auto`, range `for`, `nullptr`

**You will:** use the modern shorthands, and know when a loop copies.

- `auto x = expr;` takes the type of `expr`. Useful for long type names; the
  box is just as typed as before.
- `for (int v : values)` visits each element. `v` is a *copy*. To change the
  elements, use `for (int& v : values)`; to read big ones without copying,
  `for (const auto& v : values)`.
- `nullptr` is the null pointer, with a type of its own. `NULL` is just 0 and
  can be mistaken for an `int`.

```cpp
#include <vector>

int main() {
    std::vector<int> values{1, 2, 3};
    for (int v : values)
        v *= 10;            // changes the copy only
    for (int& v : values)
        v *= 10;            // changes the elements: 10, 20, 30
}
```

```memory
step: first loop, first round: v is a copy
frame main
  values: std::vector<int> = {data: -> #1, size: 3, capacity: 3}
  v: int = 10 !
heap #1: int[3] = [1, 2, 3]
step: second loop, first round: v is the element
frame main
  values: std::vector<int> = {data: -> #1, size: 3, capacity: 3}
  v: int& = ref #1[0]
heap #1: int[3] = [10 !, 2, 3]
```

### Check yourself

1. What type does `auto x = 5 / 2.0;` give `x`?
   <details><summary>Answer</summary>

   `double`: 2.5.
   </details>

2. Why does `f(NULL)` call `f(int)` when there are `f(int)` and `f(char*)`?
   <details><summary>Answer</summary>

   `NULL` is the integer 0. `f(nullptr)` calls the pointer version.
   </details>

### Common mistake

`for (auto v : big_strings)` copies every string on every round. Write
`const auto&`.
