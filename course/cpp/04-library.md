# Module 4 · The library, in memory

Read alongside: Stroustrup, *A Tour of C++*, chapters 10–13

The standard library's containers are ordinary C++ classes built on the
pointers, arrays and heap blocks of the C track. Knowing their layout tells
you what is fast, what is slow, and what invalidates what.

## 4.1 `std::string` and the small-string buffer

**You will:** see where a string's characters live.

A `std::string` object holds a pointer to its characters, its size and its
capacity. Allocating a heap block for every short string would be wasteful,
so implementations keep short strings *inside the object*, in a small buffer
that the pointer points into (the *small-string optimization*). In GCC's
library that is up to 15 characters, in a 32-byte object.

```cpp
#include <string>

int main() {
    std::string tiny = "pointer";               // 7 chars: inside the object
    std::string big = "dynamic memory allocation"; // 25 chars: on the heap
}
```

```memory
frame main
  tiny: std::string = {data: -> tiny.buf, size: 7, buf: "pointer"}
  big: std::string = {data: -> #1, size: 25, capacity: 25}
heap #1: char[26] = "dynamic memory allocation"
```

### Check yourself

1. What is `sizeof(std::string)` with GCC, whatever the text?
   <details><summary>Answer</summary>

   32 bytes: pointer, size, and a 16-byte area that is either the small
   buffer or the capacity.
   </details>

2. Why can moving a short string not just steal a pointer?
   <details><summary>Answer</summary>

   Its characters are inside the source object; they must be copied. Moving a
   long string steals the heap block.
   </details>

### Common mistake

Keeping a `const char*` from `s.c_str()` and then changing `s`. Appending may
reallocate the characters, and the pointer dangles.

## 4.2 `std::vector`: size, capacity, reallocation

**You will:** predict when a vector reallocates, and avoid it.

A `std::vector<T>` holds three things: a pointer to a heap array, the number
of elements in use (`size`), and the number that fit (`capacity`). When
`push_back` finds it full, it allocates a bigger array (GCC doubles; MSVC
grows by half), moves the elements over, and frees the old one: C lesson
10.2, packaged.

```cpp
#include <vector>

int main() {
    std::vector<int> v;
    for (int i = 1; i <= 5; i++)
        v.push_back(i * 10);
}
```

```memory
step: after the 1st push_back
frame main
  v: std::vector<int> = {data: -> #1, size: 1, capacity: 1}
heap #1: int[1] = [10 !]
step: after the 2nd push_back
frame main
  v: std::vector<int> = {data: -> #2 !, size: 2 !, capacity: 2 !}
heap #1: int[1] (freed) = [10]
heap #2: int[2] = [10, 20 !]
step: after the 3rd push_back
frame main
  v: std::vector<int> = {data: -> #3 !, size: 3 !, capacity: 4 !}
heap #2: int[2] (freed) = [10, 20]
heap #3: int[4] = [10, 20, 30 !, ?]
step: after the 5th push_back
frame main
  v: std::vector<int> = {data: -> #4 !, size: 5 !, capacity: 8 !}
heap #3: int[4] (freed) = [10, 20, 30, 40]
heap #4: int[8] = [10, 20, 30, 40, 50 !, ?, ?, ?]
```

If you know how many elements are coming, `v.reserve(n)` allocates once.

### Check yourself

1. How many reallocations for 1000 `push_back`s when the capacity doubles?
   <details><summary>Answer</summary>

   About ten (1, 2, 4, …, 1024). Each element is moved about once on average:
   `push_back` is fast "amortized".
   </details>

2. What does `v.clear()` do to the capacity?
   <details><summary>Answer</summary>

   Nothing: the size becomes 0, the block stays for reuse.
   </details>

### Common mistake

Keeping pointers, references or iterators to elements across a `push_back`:
module 3.4.

## 4.3 Iterators are pointers, grown up

**You will:** see what `begin()` and `end()` are.

An iterator marks a position in a container. `begin()` is the first element;
`end()` is one past the last, exactly like the `end` pointer of C lesson 7.3.
`*it` reads the element and `++it` moves on. For a vector, an iterator is a
pointer in a thin wrapper; for a list, `++it` follows a `next` pointer from
node to node. The same loop works for both.

```cpp
#include <list>
#include <vector>

int main() {
    std::vector<int> v{3, 1, 4};
    int sum = 0;
    for (auto it = v.begin(); it != v.end(); ++it)
        sum += *it;
}
```

```memory
step: second round
frame main
  v: std::vector<int> = {data: -> #1, size: 3, capacity: 3}
  sum: int = 3
  it: vector<int>::iterator = -> #1[1]
heap #1: int[3] = [3, 1, 4]
```

### Check yourself

1. Why `it != v.end()` and not `it < v.end()`?
   <details><summary>Answer</summary>

   `!=` works for every iterator; `<` only for those that can jump (vectors,
   arrays), not for lists.
   </details>

2. What is the range `for` of lesson 1.4 made of?
   <details><summary>Answer</summary>

   Exactly this loop: the compiler writes `begin()`, `end()`, `!=`, `++` and
   `*` for you.
   </details>

### Common mistake

Erasing inside a loop with `v.erase(it); ++it;`: `erase` invalidates `it`. Use
`it = v.erase(it);`, which returns the next valid position.

## 4.4 Algorithms and lambdas

**You will:** sort with a lambda, and see what a lambda is in memory.

`<algorithm>` has `std::sort`, `std::find`, `std::count_if` and dozens more,
all working on iterator ranges. A *lambda* is a function written in place.
Unlike C's comparison function for `qsort` (lesson 11.1), it is typed, and
the compiler can inline it.

```cpp
#include <algorithm>
#include <vector>

int main() {
    std::vector<int> v{42, 7, 19, 3, 25};
    int pivot = 20;
    std::sort(v.begin(), v.end(), [](int a, int b) { return a > b; });
    auto big = std::count_if(v.begin(), v.end(),
                             [&pivot](int x) { return x > pivot; });
}
```

A lambda is an object of a class the compiler writes: each captured variable
becomes a member. `[pivot]` copies it in; `[&pivot]` stores a reference, an
address in disguise.

```memory
step: inside count_if
frame main
  v: std::vector<int> = {data: -> #1, size: 5, capacity: 5}
  pivot: int = 20
frame std::count_if
  pred: lambda = {pivot: ref pivot}
heap #1: int[5] = [42, 25, 19, 7, 3]
```

### Check yourself

1. What does `big` end up as?
   <details><summary>Answer</summary>

   2: 42 and 25.
   </details>

2. When is capturing by reference dangerous?
   <details><summary>Answer</summary>

   When the lambda outlives the variable: stored in a member, or run later on
   another thread after the function returned.
   </details>

### Common mistake

A comparison that is not a strict ordering, like `a >= b`. `std::sort` may
then read past the end of the range.

## 4.5 Which container: memory decides

**You will:** choose a container by its layout.

| Container | Layout | Good at | Costs |
| --- | --- | --- | --- |
| `vector` | one contiguous array | walking, indexing, appending | inserting in the middle moves the rest |
| `list` | a node per element, linked both ways | inserting in the middle, given an iterator | a heap block per element, scattered memory |
| `map` | a balanced binary tree of nodes | ordered keys, lookup in log *n* | a node per element, pointer chasing |
| `unordered_map` | an array of buckets, each a chain of nodes | lookup in constant time on average | memory overhead, no order |

```memory
step: vector<int>{1, 2, 3}: one block
frame main
  v: std::vector<int> = {data: -> #1, size: 3, capacity: 3}
heap #1: int[3] = [1, 2, 3]
step: list<int>{1, 2, 3}: a block per element
frame main
  l: std::list<int> = {head: -> #1, size: 3}
heap #1: node = {value: 1, next: -> #2, prev: null}
heap #2: node = {value: 2, next: -> #3, prev: -> #1}
heap #3: node = {value: 3, next: null, prev: -> #2}
```

Modern CPUs read memory in 64-byte *cache lines* and prefetch what comes
next. A vector's elements arrive sixteen `int`s at a time; a list's nodes are
wherever the allocator put them. For most sizes, a vector beats a list even
for inserting in the middle. Measure; but start with `vector`.

### Check yourself

1. How many heap blocks does a `std::list<int>` of a million elements use?
   <details><summary>Answer</summary>

   A million, each with two pointers and allocator overhead around a 4-byte
   `int`.
   </details>

2. When is `map` better than `unordered_map`?
   <details><summary>Answer</summary>

   When you need the keys in order, or ranges of keys, or a guarantee against
   bad hashing.
   </details>

### Common mistake

Choosing `list` "because inserting is O(1)". Finding the place to insert is
O(*n*) and slow in cache; the vector usually wins.
