# Module 11 · Functions as values, declarations, the preprocessor

This module gathers four tools that every serious C program uses, and that
beginners often find frightening when they first meet them in someone
else's code.

The first is the idea that a function, like a variable, lives at an
address in memory, and that a pointer can hold that address. A program can
then pass a function to another function, or keep a table of functions and
choose one while it runs. The standard library's sorting function is built
on exactly this.

The second and third are about reading C itself. C's declarations can
describe complicated types, such as an array of pointers to functions, and
there is a simple method for reading any of them. The words that can come
before a declaration, `static`, `extern`, `const` and their friends, each
say something precise about the variable, and we will collect them in one
place.

The fourth is the preprocessor, which we have been using since the first
`#include`. It works on the text of the program before the compiler sees
it, and it can do more than paste in headers: it can define small
"functions" called macros, and it can keep or drop whole parts of a
program depending on a condition.

## 11.1 Functions are values too

When a program is compiled, each function becomes a sequence of machine
instructions, and when the program runs, those instructions lie somewhere
in memory, like everything else. So a function has an address. A
*function pointer* is a variable that holds such an address, and calling
through it calls the function it points to.

The declaration of a function pointer looks like the function's own first
line, with the name replaced by `(*name)`:

```c
int (*f)(int);
```

reads "`f` is a pointer to a function that takes an `int` and returns an
`int`". The parentheses around `*f` are essential, as lesson 11.4 will
explain.

The name of a function, used without a call, turns into a pointer to the
function, just as the name of an array turns into a pointer to its first
element. So `f = twice;` makes `f` point to `twice`, and `f(21)` then calls
`twice(21)`.

```c run
#include <stdio.h>

int twice(int v) { return 2 * v; }
int square(int v) { return v * v; }

int apply(int (*f)(int), int v) {
    return f(v);
}

typedef int (*op_fn)(int, int);     /* op_fn: pointer to a function (int, int) -> int */

int add(int a, int b) { return a + b; }
int sub(int a, int b) { return a - b; }
int mul(int a, int b) { return a * b; }

int main(void) {
    printf("%d %d\n", apply(twice, 21), apply(square, 9));
    op_fn ops[3] = {add, sub, mul};
    const char *names[3] = {"add", "sub", "mul"};
    for (int i = 0; i < 3; i++)
        printf("%s(6, 3) = %d\n", names[i], ops[i](6, 3));
    return 0;
}
```

`apply` does not know which function it will call; it is told by its
caller. The same `apply` doubles a number in one call and squares it in
the next.

The second half of the program keeps a *table* of functions, an array of
function pointers, and calls them in a loop. A `typedef` gives the pointer
type a name, `op_fn`, which makes the array's declaration easy to read.
Tables like this are how many programs choose an action from input: a
calculator looks up the operator typed, an emulator looks up the
instruction it has to carry out.

Step through it. When `apply` is called, its frame holds `f`, and the box
of `f` shows the name of the function it points to. Then, when `f(v)` runs,
a frame for `twice` or `square` appears above `apply`'s.

### Check yourself

1. What does the program print?
   <details><summary>Answer</summary>

   `42 81`, then `add(6, 3) = 9`, `sub(6, 3) = 3` and `mul(6, 3) = 18`.
   </details>

2. Why does `apply(twice, 21)` need no `&` before `twice`?
   <details><summary>Answer</summary>

   A function's name used without a call already turns into a pointer to
   the function. `&twice` would mean the same.
   </details>

### A common mistake

Writing `int *f(int)` instead of `int (*f)(int)`. Without the parentheses,
it declares a *function* named `f` that returns a pointer to `int`, which is
something else entirely.

## 11.2 Sorting anything: `qsort`

The standard library has a sorting function, `qsort`, declared in
`<stdlib.h>`. It can sort an array of anything: numbers, strings,
structures. How can one function sort elements whose type it does not
know? It needs to be told only two things: how big each element is, so
that it can move elements around, and how to decide which of two elements
comes first. The second is a question only the caller can answer, so the
caller passes a *comparison function*.

`qsort(base, n, size, compare)` sorts the `n` elements of `size` bytes each
that begin at `base`. Whenever it needs to compare two elements, it calls
`compare` with pointers to them. The comparison must return a negative
number if the first should come before the second, zero if they are equal,
and a positive number if the first should come after.

```c run
#include <stdio.h>
#include <stdlib.h>

int ascending(const void *a, const void *b) {
    int x = *(const int *) a, y = *(const int *) b;
    return (x > y) - (x < y);
}

int main(void) {
    int nums[5] = {42, 7, 19, 3, 25};
    qsort(nums, 5, sizeof nums[0], ascending);
    for (int i = 0; i < 5; i++)
        printf("%d ", nums[i]);
    printf("\n");
    return 0;
}
```

Since `qsort` works with elements of any type, it can only pass untyped
addresses, `const void *`. The comparison knows that the elements are
`int`s, so it converts each pointer to `const int *` and reads the value.
The expression `(x > y) - (x < y)` is a safe way to produce −1, 0 or 1:
exactly one of the two comparisons is 1 when the numbers differ, and both
are 0 when they are equal.

Step through it. Every time `qsort` compares two elements, a frame of
`ascending` appears, with `a` and `b` as arrows into `nums`, and you can see
the array being sorted between the calls.

### Check yourself

1. How would you sort in descending order?
   <details><summary>Answer</summary>

   With a comparison that returns the opposite: `(x < y) - (x > y)`, or
   simply swap `a` and `b` in the existing one.
   </details>

2. Why not simply `return x - y;`?
   <details><summary>Answer</summary>

   The subtraction can overflow when `x` and `y` are far apart, such as a
   large positive number minus a large negative one, and the sign of the
   result is then wrong.
   </details>

### A common mistake

Writing a comparison that reads the wrong type, such as `*(const int *) a`
for an array of `double`s. `qsort` cannot check, so the program compares
garbage and the array ends up in a meaningless order.

## 11.3 Storage classes and qualifiers

A declaration can begin with words that say where a variable lives, who
can see it, and what may be done with it. We have met most of them one at
a time; here they are together.

| Word | On | Says |
| --- | --- | --- |
| `auto` | a local variable | the default: made when its block starts, gone when it ends. Nobody writes it. |
| `static` | a local variable | one box for the whole run, which keeps its value between calls (lesson 5.5) |
| `static` | a global variable or a function | visible only in this source file (lesson 5.8) |
| `extern` | a declaration | "this is defined somewhere else": no box is made here |
| `register` | a local variable | a hint to keep it in a processor register; its address cannot be taken. Modern compilers ignore the hint. |
| `const` | anything | read-only after its initialization |
| `volatile` | anything | may change behind the program's back, through hardware or another part of the system, so it must be read from memory every time |

The first four are *storage classes*: they decide where and how long a
variable's box exists, and where its name can be used. The last two are
*type qualifiers*: they add a property to the type.

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

`clamp` uses `limit` although its definition comes later in the file. The
`extern` declaration at the top tells the compiler that a variable of that
name and type exists; the definition on line 10 is where its box is made.
Step through it and notice that `limit` is in the globals area from the
very first step, before `main` has even started: global variables get their
boxes, and their first values, before the program begins.

### Declarations and definitions

The difference between those two lines is worth stating carefully. A
*declaration* tells the compiler that a name exists and what type it has. A
*definition* also creates the thing: the box of a variable, or the
instructions of a function. A name may be declared as often as you like,
but defined only once. A function prototype is a declaration; the function
with its body is the definition.

### Check yourself

1. In which of these is a box made: `extern int count;`, `int count;` at
   the top of a file, `static int count;` inside a function?
   <details><summary>Answer</summary>

   In the second and the third. The first only declares; the box must be
   defined somewhere else.
   </details>

2. Why does `volatile` exist?
   <details><summary>Answer</summary>

   A compiler may keep a variable's value in a register and not read memory
   again if the program itself does not change it. For a variable that
   hardware changes, such as a clock or a device's status, that would be
   wrong. `volatile` forbids the shortcut.
   </details>

### A common mistake

Expecting `register` or `inline` to make code faster. Today's compilers
decide for themselves what to keep in registers and which functions to
copy into their callers. Measure before believing that such a word helps.

## 11.4 Reading declarations

C's declarations are built to mirror how the variable is used. `int *p`
says that `*p` is an `int`; `int a[10]` says that `a[i]` is an `int`. That is
elegant, but it means that a complicated declaration has to be read from
the inside out, and that is what makes some of them look like puzzles.

The method is simple and always works. Start at the name. Read to the right
as long as you can: `[n]` means "array of n", and `(…)` means "function
taking … and returning". When you reach a closing parenthesis or the end,
read to the left: `*` means "pointer to". When you have read everything
inside a pair of parentheses, step out of them and continue the same way.
The type at the far left comes last.

```c
int *a[10];         /* a: array of 10 pointers to int */
int (*b)[10];       /* b: pointer to an array of 10 ints */
int *f(void);       /* f: function returning a pointer to int */
int (*g)(void);     /* g: pointer to a function returning int */
void (*h[3])(int);  /* h: array of 3 pointers to functions taking an int and returning nothing */
```

Take `int (*b)[10]`. The name is `b`. To its right is a closing
parenthesis, so we read left: `*`, "pointer to". We have finished the
parentheses, so we step out and read right again: `[10]`, "an array of
10". Then left: `int`. Together: `b` is a pointer to an array of 10 `int`s.
Compare `int *a[10]`: here nothing stops us going right first, so `a` is an
array of 10 first, and only then pointers to `int`.

When a declaration becomes hard to read, a `typedef` can name a piece of
it, as `op_fn` did in lesson 11.1. Then `void (*h[3])(int)` can be written
`typedef void (*handler)(int);` followed by `handler h[3];`.

### Check yourself

1. Read `char **argv`.
   <details><summary>Answer</summary>

   `argv` is a pointer to a pointer to `char`.
   </details>

2. Read `const char *const names[]`.
   <details><summary>Answer</summary>

   `names` is an array of constant pointers to constant `char`s: neither the
   pointers nor the characters may be changed.
   </details>

### A common mistake

Reading from left to right. `int *a[10]` is not "a pointer to an array of
10": the brackets bind more tightly than the `*`, so the name meets `[10]`
first, and `a` is an array.

## 11.5 Macros

The preprocessor, as module 1 said, reads the program before the compiler
does and carries out the lines that begin with `#`. One of the things it
does is replace *macros*. A macro is defined with `#define`, and from then
on, wherever its name appears, the preprocessor puts the macro's
replacement text in its place.

A macro can also have parameters, which makes it look like a small
function:

```c
#define SQUARE(x) ((x) * (x))
```

Now `SQUARE(n + 1)` is replaced by `((n + 1) * (n + 1))`. The argument is
pasted in exactly as it was written, wherever the parameter appears. That
is the key fact about macros, and the source of all their traps: the
preprocessor works on text, not on values, and it knows nothing about C.

```c run
#include <stdio.h>

#define SQUARE_BAD(x) x * x
#define SQUARE(x) ((x) * (x))
#define PRINT_INT(n) printf(#n " = %d\n", n)

int main(void) {
    int total = 6;
    printf("%d %d\n", SQUARE_BAD(2 + 3), SQUARE(2 + 3));
    PRINT_INT(total);
    PRINT_INT(SQUARE(total) / 4);
    printf("this is line %d\n", __LINE__);
    return 0;
}
```

`SQUARE_BAD(2 + 3)` becomes `2 + 3 * 2 + 3`, and multiplication comes
first, so the result is 11, not 25. The cure is parentheses: around every
use of a parameter, so that an argument stays in one piece, and around the
whole replacement, so that the result stays in one piece inside a larger
expression. `SQUARE` has both.

`PRINT_INT` uses the operator `#`, which turns a macro's argument into a
string: `#n` becomes the argument as it was written, in double quotes. So
`PRINT_INT(total)` becomes `printf("total" " = %d\n", total)`, and two
strings side by side are joined into one. A macro like this saves typing
when debugging, because it prints the expression together with its value.

There is also an operator `##`, which glues two pieces of text into one
name: with `#define NAME(a, b) a ## b`, `NAME(count, 2)` becomes `count2`.
It is rarely needed.

Finally, the preprocessor defines a few macros of its own. `__LINE__` is
the number of the line it appears on, and `__FILE__` is the name of the
source file. They are useful in error messages that say where something
went wrong.

When you step through this program, the macro names never appear as steps
of their own: by the time the program runs, they have all been replaced.
Line 9 is a single `printf`, and so are lines 10 and 11.

### Macros or functions?

Macros were once used for speed, since they avoid the cost of a function
call. Today a function, perhaps marked `inline`, is almost always better:
it checks the types of its arguments, it works out each argument exactly
once, and it shows up properly when debugging. Use macros for what only
the preprocessor can do: constants, `#` and `##`, `__LINE__`, and the
conditional compilation of the next lesson.

### Check yourself

1. What is wrong with `SQUARE(i++)`, even with all the parentheses?
   <details><summary>Answer</summary>

   It becomes `((i++) * (i++))`, which changes `i` twice in one expression,
   and that is undefined (lesson 2.6). A function would work out its
   argument once.
   </details>

2. What does `PRINT_INT(SQUARE(total) / 4)` print?
   <details><summary>Answer</summary>

   `SQUARE(total) / 4 = 9`. The `#` uses the argument as written, before
   `SQUARE` is replaced, and 6 × 6 / 4 is 9 in integer division.
   </details>

### A common mistake

A macro made of several statements, such as `#define SWAP(a, b) t = a; a =
b; b = t`, used after an `if` without braces. Only the first statement ends
up inside the `if`. Such macros are wrapped in `do { … } while (0)`, which
turns them into a single statement.

## 11.6 Conditional compilation

The preprocessor can also keep or drop parts of a program, depending on a
condition that it works out before the compiler starts. The dropped lines
are not compiled at all, as if they had never been written. This is
called *conditional compilation*, and it is how one source file can serve
several purposes: a debugging build and a release build, or versions for
different operating systems.

```c run
#include <stdio.h>

#define DEBUG 1
#define VERSION 3

int main(void) {
    int balance = 120;
#if DEBUG
    printf("[debug] balance is %d\n", balance);
#endif
#if 0
    this is not C at all, and it is never compiled
#endif
#if VERSION >= 3
    printf("version 3 or later\n");
#elif VERSION == 2
    printf("version 2\n");
#else
    printf("an old version\n");
#endif
#ifdef NDEBUG
    printf("checks are off\n");
#else
    printf("checks are on\n");
#endif
    return 0;
}
```

`#if` is followed by a condition, which may use numbers, macros and the
usual operators. If it is not zero, the lines up to the matching `#endif`
are kept; otherwise they are dropped. `#elif` and `#else` work like `else
if` and `else`. `#ifdef NAME` keeps its lines if `NAME` has been defined as
a macro at all, whatever its value, and `#ifndef NAME` if it has not. Inside
an `#if`, `defined(NAME)` asks the same question, so conditions such as
`#if defined(DEBUG) && VERSION > 2` are possible.

Look at lines 11 to 13. Text that is not C at all sits between `#if 0` and
`#endif`, and the program compiles and runs anyway, because the
preprocessor removes those lines before the compiler sees them. `#if 0` is
a quick way to switch off a piece of code without deleting it.

Step through the program and see which lines become steps. Only the kept
lines ever run. The others are still there in the text, but for the
compiler they do not exist.

### Defining macros from outside

A macro can also be defined when the program is compiled, without
changing the source: `gcc -DDEBUG prog.c` compiles the program as if it
began with `#define DEBUG 1`. Together with `#ifdef DEBUG`, this lets the
same file be built with or without its debugging code. The standard
library uses the same idea: `assert` (module 13) is switched off when the
macro `NDEBUG` is defined.

The include guards of lesson 5.8 are conditional compilation too: the
whole header sits inside `#ifndef COUNTER_H`, and the first thing the
header does is define `COUNTER_H`, so its contents are kept only the first
time.

Finally, `#error message` stops the compilation with the message. It is
used inside a conditional to refuse a combination of settings that cannot
work, such as `#if VERSION < 2` followed by `#error this program needs
version 2`.

### Check yourself

1. Which lines does the program print?
   <details><summary>Answer</summary>

   `[debug] balance is 120`, `version 3 or later` and `checks are on`.
   </details>

2. What would change if line 3 were `#define DEBUG 0`?
   <details><summary>Answer</summary>

   `#if DEBUG` would be false, so the debug line would be dropped. With
   `#ifdef DEBUG` it would still be kept, since `DEBUG` would still be
   defined, only with the value 0.
   </details>

### A common mistake

Forgetting an `#endif`. Everything up to the end of the file then belongs
to the unfinished `#if`, and the error message may point far away from the
place where the `#endif` is missing. Vizu says which `#if` was never
closed.
