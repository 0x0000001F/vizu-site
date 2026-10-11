# Module 7 · Errors and the capstone

Read alongside: Stroustrup, *A Tour of C++*, chapter 4

Exceptions leave a function in the middle; RAII is what makes that safe. The
capstone then rebuilds `std::vector` from the C track's dynamic array.

## 7.1 Exceptions and unwinding

**You will:** watch frames come off the stack when an exception is thrown.

`throw` creates an exception object and leaves the current function. The
runtime then *unwinds* the stack: frame by frame, it destroys every local
object (running destructors, so RAII objects give their resources back) until
it finds a `try` block with a matching `catch`.

```cpp
#include <stdexcept>
#include <vector>

double ratio(int a, int b) {
    std::vector<int> scratch(100);          // owns a heap block
    if (b == 0)
        throw std::invalid_argument("b is zero");
    return double(a) / b;
}

int main() {
    try {
        double r = ratio(1, 0);
    } catch (const std::invalid_argument& e) {
        // e.what() is "b is zero"
    }
}
```

```memory
step: at the throw
frame main
frame ratio
  a: int = 1
  b: int = 0
  scratch: std::vector<int> = {data: -> #1, size: 100, capacity: 100}
heap #1: int[100] = [0, 0, 0, 0, 0, 0, 0, 0]
step: unwinding: ratio's frame is destroyed
frame main
heap #1: int[100] (freed) = [0, 0, 0, 0, 0, 0, 0, 0]
step: in the catch block
frame main
  e: const invalid_argument& = ref #2
heap #2: invalid_argument = {what: "b is zero"}
```

(The drawing shows the first cells of the 100-element block.)

### Check yourself

1. Why must destructors not throw?
   <details><summary>Answer</summary>

   They run during unwinding. A second exception while one is in flight calls
   `std::terminate` and the program ends.
   </details>

2. Why catch by `const&`?
   <details><summary>Answer</summary>

   Catching by value copies the exception and slices derived exception types
   (lesson 5.3).
   </details>

### Common mistake

`new` in a function that can throw before the matching `delete`: the
exception skips the `delete` and leaks. With `unique_ptr` or a container,
unwinding frees it.

## 7.2 Capstone: a vector of your own

**You will:** write a small `std::vector<int>` and connect both tracks.

C lesson 10.2 grew an array by hand with `realloc`. Turn it into a class:

```cpp
#include <cstddef>
#include <utility>

class IntVector {
public:
    IntVector() = default;
    ~IntVector() { delete[] data_; }

    IntVector(const IntVector& o)
        : data_(o.size_ ? new int[o.size_] : nullptr), size_(o.size_), cap_(o.size_) {
        for (std::size_t i = 0; i < size_; i++) data_[i] = o.data_[i];
    }
    IntVector(IntVector&& o) noexcept
        : data_(std::exchange(o.data_, nullptr)),
          size_(std::exchange(o.size_, 0)),
          cap_(std::exchange(o.cap_, 0)) {}
    IntVector& operator=(IntVector o) noexcept {     // copy or move, then swap
        std::swap(data_, o.data_);
        std::swap(size_, o.size_);
        std::swap(cap_, o.cap_);
        return *this;
    }

    void push_back(int v) {
        if (size_ == cap_) grow();
        data_[size_++] = v;
    }
    int& operator[](std::size_t i) { return data_[i]; }
    std::size_t size() const { return size_; }

private:
    void grow() {
        std::size_t cap = cap_ ? 2 * cap_ : 1;
        int* bigger = new int[cap];
        for (std::size_t i = 0; i < size_; i++) bigger[i] = data_[i];
        delete[] data_;
        data_ = bigger;
        cap_ = cap;
    }
    int* data_ = nullptr;
    std::size_t size_ = 0;
    std::size_t cap_ = 0;
};
```

```memory
step: after three push_back calls
frame main
  v: IntVector = {data_: -> #2, size_: 3, cap_: 4}
heap #1: int[2] (freed) = [1, 2]
heap #2: int[4] = [1, 2, 3 !, ?]
step: after IntVector w = std::move(v);
frame main
  v: IntVector = {data_: null !, size_: 0 !, cap_: 0 !}
  w: IntVector = {data_: -> #2 !, size_: 3 !, cap_: 4 !}
heap #2: int[4] = [1, 2, 3, ?]
```

Every piece of it comes from a lesson: the growth from C 10.2, the destructor
from C++ 2.3, deep copy from 2.4, the stolen pointer from 2.5.

### Your project

1. Add `at(i)`, which throws `std::out_of_range` past the end (C lesson 4.2's
   bug, now an exception).
2. Make it a template, `Vector<T>` (lesson 6.2). Why does `new T[cap]` become
   a problem for types without a default constructor? (Real vectors allocate
   raw memory and construct elements one by one.)
3. Add `begin()` and `end()` returning `int*`, and use your class in a range
   `for` (lesson 4.3).

### Common mistake

Self-assignment, `v = v;`. The copy-and-swap `operator=` above handles it;
a version that first frees its own block and then copies from `o` destroys
the data it is about to copy.
