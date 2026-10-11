# Module 5 · Functions

Read alongside: King chapters 9, 10 and 15 · K&R sections 1.7–1.10 and chapter 4

Every call to a function gets a fresh set of boxes, a *stack frame*, on top
of the caller's. Seeing frames pile up and come off is the key to arguments,
scope and recursion, and later to pointers.

## 5.1 A function call is a new frame

**Read:** King 9.1–9.2, 9.4 · K&R 1.7, 4.1–4.2
**You will:** define and call a function, and follow its frame.

A function definition gives the type it returns, its name, its *parameters*
and its body. A call evaluates the arguments, makes a frame with a box for
each parameter and each local variable, and runs the body. `return` ends the
call and hands a value back; the frame is then gone.

```c run
#include <stdio.h>

/* power: base raised to the n-th power, n >= 0 */
int power(int base, int n) {
    int p = 1;
    for (int i = 0; i < n; i++)
        p = p * base;
    return p;
}

int main(void) {
    for (int i = 0; i < 4; i++)
        printf("%d %d %d\n", i, power(2, i), power(-3, i));
    return 0;
}
```

**Watch:**

- Each call to `power` puts a frame on top of `main`'s. It has its own `base`,
  `n`, `p` and its own `i`, a different box from `main`'s `i`.
- At `return p` the frame shows the value it returns, then disappears.
- `main`'s frame stays below the whole time, waiting at line 13.

### Check yourself

1. How many times is `power` called?
   <details><summary>Answer</summary>

   Eight: twice per round, four rounds.
   </details>

2. If `power` were defined *after* `main`, what would `main` need?
   <details><summary>Answer</summary>

   A *prototype* before it: `int power(int base, int n);`. A function must be
   declared before it is called, so the compiler can check the call.
   </details>

### Common mistake

Forgetting `return` in a function that promises a value. The caller gets
whatever was left in the register the value travels in. Compile with `-Wall`,
which warns about it.

## 5.2 Arguments are copies

**Read:** K&R 1.8 · King 9.3
**You will:** see that a function cannot change its caller's variables
through its parameters.

C passes arguments *by value*: the parameter is a new box that starts with a
copy of the argument. Changing it changes the copy.

```c run
#include <stdio.h>

void double_it(int x) {
    x = x * 2;
    printf("inside: %d\n", x);
}

int main(void) {
    int n = 21;
    double_it(n);
    printf("outside: %d\n", n);
    return 0;
}
```

**Watch:** at line 4, `x` in the top frame turns amber: 42. `n` in `main`'s
frame is still 21, and stays 21.

K&R point out the good side: since a parameter is a private copy, a function
can use it as a working variable without harming anything.

### Check yourself

1. How can `main` get the doubled value?
   <details><summary>Answer</summary>

   Return it (`int doubled(int x) { return x * 2; }` and `n = doubled(n);`),
   or pass `n`'s address (lesson 6.4).
   </details>

2. `scanf("%d", &n)` changes `n`. How, if arguments are copies?
   <details><summary>Answer</summary>

   It gets a copy of `n`'s *address*. A copy of an address still leads to the
   same box.
   </details>

### Common mistake

Writing a function that "returns" through a parameter, like
`void read_age(int age) { scanf("%d", &age); }`. It fills its own copy, which
vanishes.

## 5.3 Arrays as arguments

**Read:** King 9.3 (Array Arguments) · K&R 1.9
**You will:** know what a function really receives when you pass an array.

An array is never copied into a function. The function gets the address of the
array's first element, and `int a[]` in a parameter list really means `int *a`
(a pointer: module 6). Two consequences: the function can change the
caller's array, and it cannot know the array's length, so you pass it.

```c run
#include <stdio.h>

int sum(int a[], int n) {
    int s = 0;
    for (int i = 0; i < n; i++)
        s += a[i];
    return s;
}

void zero_first(int a[]) {
    a[0] = 0;
}

int main(void) {
    int nums[5] = {3, 1, 4, 1, 5};
    printf("%d\n", sum(nums, 5));
    zero_first(nums);
    printf("%d %d\n", nums[0], sum(nums, 5));
    return 0;
}
```

**Watch:** in `sum`'s frame, `a` is not a row of cells but an arrow into
`main`'s `nums`. In `zero_first`, the write at line 11 lights `main`'s
`nums[0]` amber.

### Check yourself

1. Predict the output.
   <details><summary>Answer</summary>

   `14`, then `0 11`.
   </details>

2. How would you promise that `sum` does not change the array?
   <details><summary>Answer</summary>

   `int sum(const int a[], int n)`: lesson 6.6.
   </details>

### Common mistake

Computing the length inside the function:

```c run
#include <stdio.h>

int count(int a[]) {
    return sizeof a / sizeof a[0];
}

int main(void) {
    int nums[5] = {3, 1, 4, 1, 5};
    printf("%zu %d\n", sizeof nums / sizeof nums[0], count(nums));
    return 0;
}
```

In `main`, `sizeof nums` is 20 bytes. In `count`, `a` is a pointer, 8 bytes,
so the "length" is 2.

## 5.4 Scope and lifetime

**Read:** King 10.1–10.4 · K&R 1.10, 4.3–4.6
**You will:** tell where a name can be used (scope) from how long its box
lives (lifetime).

- A variable declared inside a block is *local*: usable only in that block,
  and its box is made when the block starts and gone when it ends
  (*automatic* lifetime).
- `static` on a local keeps its scope but gives it a box for the whole run,
  made and initialized once.
- A variable declared outside all functions is *external* (global): usable in
  the whole file, alive for the whole run.

```c run
#include <stdio.h>

int calls = 0;              /* external: lives for the whole run */

int next_id(void) {
    static int id = 100;    /* static: made once, keeps its value */
    int step = 1;           /* automatic: made at each call */
    calls++;
    id += step;
    return id;
}

int main(void) {
    int a = next_id();
    int b = next_id();
    int c = next_id();
    printf("%d %d %d after %d calls\n", a, b, c, calls);
    return 0;
}
```

**Watch:** `calls` and `id` live in the globals area, above the stack, and
keep their values between calls. `step` appears in each new frame and goes
with it.

### Check yourself

1. Predict the output.
   <details><summary>Answer</summary>

   `101 102 103 after 3 calls`.
   </details>

2. Can `main` use `id`?
   <details><summary>Answer</summary>

   No. Its box lives all the time, but its name is only known inside
   `next_id`.
   </details>

### Common mistake

Using global variables to pass data between functions. It works in a small
program and makes a big one impossible to follow: any function may have
changed it. King lists the pros and cons in 10.2; prefer parameters.

## 5.5 Recursion

**Read:** King 9.6 · K&R 4.10
**You will:** watch a function call itself, frame by frame.

A function may call itself. Each call gets its own frame, with its own copy
of the parameters, so the calls do not disturb each other. A recursive
function needs a case that does not recurse, or it never stops.

```c run
#include <stdio.h>

int fact(int n) {
    if (n <= 1)
        return 1;
    return n * fact(n - 1);
}

int main(void) {
    printf("%d\n", fact(5));
    return 0;
}
```

**Watch:** five `fact` frames pile up, `n` = 5, 4, 3, 2, 1. Then they come off
one by one, each showing what it returns: 1, 2, 6, 24, 120.

Recursion can also do work on the way *back*. To print a number's digits in
order, print the digits before the last one first:

```c run
#include <stdio.h>

void print_digits(int n) {
    if (n / 10)
        print_digits(n / 10);
    putchar(n % 10 + '0');
}

int main(void) {
    print_digits(4096);
    putchar('\n');
    return 0;
}
```

### Check yourself

1. How many frames are on the stack, at most, for `print_digits(4096)`?
   <details><summary>Answer</summary>

   Five: `main` and four `print_digits`, one per digit.
   </details>

2. Which digit is printed first, and why?
   <details><summary>Answer</summary>

   4. The deepest call, with `n` = 4, is the first to reach `putchar`.
   </details>

### Common mistake

No case that stops:

```c run expect="error"
int fact(int n) {
    return n * fact(n - 1);
}

int main(void) {
    return fact(3);
}
```

Every call makes another; the stack fills up. A real program dies with a
*stack overflow*. Vizu stops it and says the recursion never ends.

## 5.6 Programs in many files

**Read:** King 15 · K&R 4.5
**You will:** split a program into a header and source files, and know what
the linker does.

A bigger program is split into `.c` files, each compiled on its own into an
*object file*; the *linker* then joins them and the library into one program.
A `.h` header holds what the files share: prototypes, types, `#define`s,
`extern` declarations. It is included where it is needed.

```c
/* counter.h */
#ifndef COUNTER_H
#define COUNTER_H

void counter_add(int n);
int counter_value(void);

#endif
```

```c
/* counter.c */
#include "counter.h"

static int total = 0;     /* static: invisible outside this file */

void counter_add(int n) { total += n; }
int counter_value(void) { return total; }
```

```c
/* main.c */
#include <stdio.h>
#include "counter.h"

int main(void) {
    counter_add(5);
    counter_add(7);
    printf("%d\n", counter_value());
    return 0;
}
```

Build it with `gcc -c counter.c`, `gcc -c main.c`, then
`gcc counter.o main.o -o counter`, or all at once with
`gcc counter.c main.c -o counter`. A `Makefile` records these steps so only
changed files are rebuilt.

The `#ifndef COUNTER_H` lines are an *include guard*: if the header is included
twice, the second copy is skipped.

Vizu runs one file today; this lesson is for reading.

### Check yourself

1. What does `static` mean on `total` here?
   <details><summary>Answer</summary>

   The variable is private to `counter.c`: other files cannot name it, even
   with `extern`. Here `static` is about visibility, not lifetime.
   </details>

2. A program compiles but the linker says "undefined reference to
   `counter_add`". What is missing?
   <details><summary>Answer</summary>

   `counter.c` (or `counter.o`) in the link command. The header promised the
   function exists; the linker could not find its code.
   </details>

### Common mistake

Defining a variable in a header (`int total = 0;` in `counter.h`). Every file
that includes it defines its own `total`, and the linker complains about
duplicates. Headers *declare* (`extern int total;`); one `.c` file defines.
