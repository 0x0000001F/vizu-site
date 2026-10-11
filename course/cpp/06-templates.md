# Module 6 · Templates

Read alongside: Stroustrup, *A Tour of C++*, chapters 7–8

C wrote generic code with `void *` and sizes (`qsort`, lesson 11.1). C++
writes it once as a *template*, and the compiler writes a typed copy for each
type used. Nothing about a template exists at run time; its cost and its
errors are all at compile time.

## 6.1 Function templates

**You will:** write a function for any type, and see the copies the compiler
makes.

```cpp
template <typename T>
T max_of(T a, T b) {
    return a > b ? a : b;
}

int main() {
    int i = max_of(3, 7);               // makes max_of<int>
    double d = max_of(2.5, 1.5);        // makes max_of<double>
}
```

The compiler deduces `T` from the arguments and *instantiates* a function for
each `T`. The program then contains two ordinary functions, as `nm` shows:

```text
_Z6max_ofIiET_S0_S0_    int max_of<int>(int, int)
_Z6max_ofIdET_S0_S0_    double max_of<double>(double, double)
```

### Check yourself

1. Why does `max_of(3, 2.5)` not compile?
   <details><summary>Answer</summary>

   `T` cannot be both `int` and `double`. Write `max_of<double>(3, 2.5)`.
   </details>

2. What happens with a type that has no `>`?
   <details><summary>Answer</summary>

   The instantiation fails to compile, with an error pointing into the
   template. Lesson 6.3's concepts make that error readable.
   </details>

### Common mistake

Putting a template's definition in a `.cpp` file and only its declaration in
the header. Each file that uses it must see the whole template to instantiate
it; templates live in headers.

## 6.2 Class templates

**You will:** write a container for any element type.

```cpp
#include <vector>
#include <stdexcept>

template <typename T>
class Stack {
public:
    void push(const T& v) { items_.push_back(v); }
    T pop() {
        if (items_.empty())
            throw std::out_of_range("pop on an empty stack");
        T v = items_.back();
        items_.pop_back();
        return v;
    }
    bool empty() const { return items_.empty(); }
private:
    std::vector<T> items_;
};

int main() {
    Stack<int> numbers;
    numbers.push(1);
    numbers.push(2);
    Stack<std::string> words;   // a second, separate class
}
```

Compare with C lesson 14.1: the same stack, but for any `T`, with no
`void *`, and with an error instead of a crash on an empty `pop`.

```memory
frame main
  numbers: Stack<int> = {items_: {data: -> #1, size: 2, capacity: 2}}
  words: Stack<std::string> = {items_: {data: null, size: 0, capacity: 0}}
heap #1: int[2] = [1, 2]
```

### Check yourself

1. Are `Stack<int>` and `Stack<double>` related types?
   <details><summary>Answer</summary>

   No: two unrelated classes generated from one template. A function taking
   `Stack<int>&` does not accept a `Stack<double>`.
   </details>

2. Is `pop` instantiated for `Stack<std::string>` above?
   <details><summary>Answer</summary>

   No. Member functions of a class template are only instantiated when used.
   </details>

### Common mistake

Expecting templates to shrink the program. Each instantiation is separate
machine code; many types times a big template is a big binary.

## 6.3 `constexpr` and concepts

**You will:** run code at compile time, and constrain a template.

`constexpr` marks functions and values that can be computed by the compiler.
When the arguments are constants, the result is computed during compilation
and the running program only sees the answer:

```cpp
constexpr long factorial(int n) {
    return n <= 1 ? 1 : n * factorial(n - 1);
}

static_assert(factorial(10) == 3628800);   // checked by the compiler
constexpr long f10 = factorial(10);        // no call happens at run time
```

A *concept* names the requirements a template has on its type. A call with
the wrong type then fails with a short message naming the requirement,
instead of pages of errors from deep inside the template.

```cpp
#include <concepts>

template <typename T>
concept Ordered = requires(T a, T b) {
    { a > b } -> std::convertible_to<bool>;
};

template <Ordered T>
T max_of(T a, T b) {
    return a > b ? a : b;
}
```

### Check yourself

1. Where is the memory for `factorial(10)`'s recursion when it is computed
   by the compiler?
   <details><summary>Answer</summary>

   Inside the compiler. The program contains only the constant 3628800; Vizu
   would show no frames at all.
   </details>

2. What does `max_of` say now if called with a type without `>`?
   <details><summary>Answer</summary>

   That the type does not satisfy `Ordered`, at the call.
   </details>

### Common mistake

Thinking `constexpr` forces compile-time evaluation. A `constexpr` function
called with run-time values simply runs at run time; use `consteval` when it
must be compile time.
