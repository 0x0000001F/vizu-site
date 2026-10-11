# Module 11 · Functions as values, declarations, the preprocessor

Read alongside: King chapters 14, 17.7 and 18 · K&R sections 4.11 and 5.11–5.12

Four tools that make C programs flexible, and that make C declarations look
frightening until you learn to read them.

## 11.1 Function pointers and `qsort`

**Read:** King 17.7 · K&R 5.11
**You will:** pass a function to another function.

A function has an address too: its machine code lives somewhere in memory.
A *function pointer* holds that address, and calling through it calls the
function. The standard `qsort` sorts any array because you hand it the
comparison: a function that takes pointers to two elements and returns a
negative number, zero or a positive number.

```c run
#include <stdio.h>
#include <stdlib.h>

int ascending(const void *a, const void *b) {
    int x = *(const int *) a, y = *(const int *) b;
    return (x > y) - (x < y);
}

int apply(int (*f)(int), int v) {
    return f(v);
}

int twice(int v) {
    return 2 * v;
}

int main(void) {
    int nums[5] = {42, 7, 19, 3, 25};
    qsort(nums, 5, sizeof nums[0], ascending);
    for (int i = 0; i < 5; i++)
        printf("%d ", nums[i]);
    printf("| %d\n", apply(twice, 21));
    return 0;
}
```

`int (*f)(int)` reads: `f` is a pointer to a function taking an `int` and
returning an `int`. A function's name, like an array's, turns into a pointer
to it, so `apply(twice, 21)` needs no `&`.

**Watch:** every call `qsort` makes to `ascending` is a frame of its own, with
`a` and `b` as arrows into `nums`. The array gets sorted under your eyes.

### Check yourself

1. Why does the comparison take `const void *` and not `int *`?
   <details><summary>Answer</summary>

   `qsort` works for any element type, so it can only hand over untyped
   addresses. The comparison knows the real type and casts.
   </details>

2. Why not `return x - y;`?
   <details><summary>Answer</summary>

   It overflows when `x` and `y` are far apart (a large positive minus a large
   negative). `(x > y) - (x < y)` gives −1, 0 or 1 safely.
   </details>

### Common mistake

`int *f(int)` instead of `int (*f)(int)`: without the parentheses it declares
a function returning an `int *`.

## 11.2 Storage classes and qualifiers

**Read:** King 18.2–18.3 · K&R 4.4–4.7
**You will:** know what each of `auto`, `static`, `extern`, `register`,
`const` and `volatile` says.

| Word | On | Means |
| --- | --- | --- |
| `auto` | a local | the default: made at block entry, gone at exit (never written today) |
| `static` | a local | one box for the whole run (lesson 5.4) |
| `static` | a global or function | visible in this file only (lesson 5.6) |
| `extern` | a declaration | "defined elsewhere": no box is made here |
| `register` | a local | a hint to keep it in a register; its address cannot be taken; ignored by modern compilers |
| `const` | any | read-only after initialization |
| `volatile` | any | may change behind the program's back (hardware, a signal handler): read it from memory every time |

```c run
#include <stdio.h>

extern int limit;           /* declared here, defined below */

int clamp(int v) {
    register int top = limit;   /* a hint; compilers decide for themselves */
    return v > top ? top : v;
}

int limit = 100;            /* the definition: the box is made here */

int main(void) {
    const double rate = 0.2;    /* read-only after this line */
    volatile int ticks = 0;     /* always read from memory */
    ticks++;
    printf("%d %d %.1f %d\n", clamp(250), clamp(42), rate, ticks);
    return 0;
}
```

**Watch:** `limit` sits with the globals from the start, before `main` runs.

### Check yourself

1. What is the difference between a declaration and a definition?
   <details><summary>Answer</summary>

   A declaration tells the compiler a name and its type; a definition also
   makes the box (or the function's code). A variable may be declared many
   times and defined once.
   </details>

2. Why is `volatile` not a way to share data between threads?
   <details><summary>Answer</summary>

   It stops the compiler from caching the value, but it does not make updates
   atomic or ordered between CPUs. Threads need `_Atomic` or locks.
   </details>

### Common mistake

Expecting `register` or `inline` to make code faster. The optimizer decides;
measure before believing.

## 11.3 Reading declarations

**Read:** King 18.4 · K&R 5.12
**You will:** read any C declaration, from the name outwards.

Start at the name. Go right while you can (`[]` "array of", `()` "function
returning"), then left (`*` "pointer to"), stepping out of each pair of
parentheses when you hit it. End with the type on the far left.

```c
int *a[10];         /* a: array of 10 pointers to int */
int (*b)[10];       /* b: pointer to an array of 10 ints */
int *f(void);       /* f: function returning a pointer to int */
int (*g)(void);     /* g: pointer to a function returning int */
void (*h[3])(int);  /* h: array of 3 pointers to functions taking int, returning nothing */
char (*(*x(void))[5])(void);
/* x: function returning a pointer to an array of 5 pointers to
      functions returning char. Nobody should write this. */
```

`typedef` gives a type a name, which makes such declarations readable:

```c run
#include <stdio.h>

typedef int (*op_fn)(int, int);     /* op_fn: pointer to a function (int, int) -> int */

int add(int a, int b) { return a + b; }
int sub(int a, int b) { return a - b; }
int mul(int a, int b) { return a * b; }

int main(void) {
    op_fn ops[3] = {add, sub, mul};
    const char *names[3] = {"add", "sub", "mul"};
    for (int i = 0; i < 3; i++)
        printf("%s(6, 3) = %d\n", names[i], ops[i](6, 3));
    return 0;
}
```

K&R write a program, `dcl`, that turns declarations into words like the
comments above; it is a good exercise in recursive parsing.

### Check yourself

1. Read `char **argv`.
   <details><summary>Answer</summary>

   `argv`: pointer to pointer to `char`.
   </details>

2. Read `const char *const names[]`.
   <details><summary>Answer</summary>

   `names`: array of constant pointers to constant `char`s. Neither the
   pointers nor the characters may be changed.
   </details>

### Common mistake

Reading left to right. `int *a[10]` is not "a pointer to an array": `[]` binds
tighter than `*`, so the name meets `[10]` first.

## 11.4 The preprocessor

**Read:** King 14 · K&R 4.11
**You will:** write macros that do not bite, and compile code in or out.

Before the compiler sees a file, the preprocessor handles every line starting
with `#`: it pastes headers in (`#include`), replaces macros (`#define`), and
keeps or drops code (`#if`, `#ifdef`). It works on text, not on C, and that is
where its traps come from: a macro's argument is pasted in as written.

```c run
#include <stdio.h>

#define SQUARE_BAD(x) x * x
#define SQUARE(x) ((x) * (x))
#define DEBUG 1

int main(void) {
    printf("%d %d\n", SQUARE_BAD(2 + 3), SQUARE(2 + 3));
#if DEBUG
    printf("debug build\n");
#endif
    return 0;
}
```

`SQUARE_BAD(2 + 3)` becomes `2 + 3 * 2 + 3`, which is 11. Parentheses around
each use of the parameter and around the whole body fix it.

### Check yourself

1. What is wrong with `SQUARE(i++)`, even with the parentheses?
   <details><summary>Answer</summary>

   It becomes `((i++) * (i++))`: `i` is changed twice in one expression,
   which is undefined. A function (or an `inline` function) evaluates its
   argument once.
   </details>

2. How do you turn `DEBUG` off without editing the file?
   <details><summary>Answer</summary>

   Use `#ifdef DEBUG` in the code and compile with or without `-DDEBUG`.
   </details>

### Common mistake

A macro with several statements, `#define SWAP(a, b) t = a; a = b; b = t`,
after an `if` without braces: only the first statement is inside the `if`.
Wrap such macros in `do { ... } while (0)`.
