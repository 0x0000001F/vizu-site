# Module 2 · Types and expressions

Read alongside: King chapters 4 and 7 · K&R chapter 2

A box's *type* decides two things: how many bytes it takes, and what its bits
mean. This module looks at C's basic types, what happens at their edges, and
how C combines them in expressions.

## 2.1 Integer types and `sizeof`

**Read:** King 7.1, 7.6 · K&R 2.2
**You will:** know how big each integer type is, and how to ask.

C has several integer types, from small to large: `char`, `short`, `int`,
`long`, `long long`. The standard only says each is at least as big as the one
before it. On today's 64-bit Linux and macOS (and in Vizu) they are 1, 2, 4, 8
and 8 bytes. `sizeof` tells you the size of a type or a variable, in bytes.

```c run
#include <stdio.h>

int main(void) {
    char c = 'A';
    short s = 1000;
    int i = 100000;
    long l = 10000000000;
    printf("char %zu, short %zu, int %zu, long %zu\n",
           sizeof c, sizeof s, sizeof i, sizeof l);
    return 0;
}
```

`sizeof` gives a value of type `size_t`, printed with `%zu`.

**Watch:** the four boxes have different widths. A box's width is its size;
the addresses of neighbouring boxes are at least that far apart.

### Check yourself

1. Why does `10000000000` not fit in an `int`?
   <details><summary>Answer</summary>

   A 4-byte `int` holds 2³² different values, from −2 147 483 648 to
   2 147 483 647. Ten billion is bigger.
   </details>

2. Is `long` always 8 bytes?
   <details><summary>Answer</summary>

   No. On 64-bit Windows `long` is 4 bytes. When the size matters, use the
   exact types from `<stdint.h>`: `int32_t`, `int64_t`, `uint8_t`.
   </details>

### Common mistake

Assuming sizes. Code that stores a pointer in an `int`, or a count of bytes in
a `short`, works on one machine and breaks on another. Ask `sizeof`.

## 2.2 Overflow and unsigned

**Read:** King 7.1 (Integer Overflow) · K&R 2.2
**You will:** see what happens past the largest value.

An `int` can only hold so much. Adding one to the largest `int` is *signed
overflow*, and the C standard says the result is *undefined*: the program may
do anything. In practice it usually wraps to the most negative number, but an
optimizing compiler is allowed to assume it never happens and remove your
checks.

`unsigned` types have no negative numbers, and they never overflow: arithmetic
wraps around, by definition. Zero minus one is the largest value.

```c run
#include <stdio.h>

int main(void) {
    int big = 2147483647;
    int bigger = big + 1;
    unsigned int u = 0;
    u = u - 1;
    printf("%d %u\n", bigger, u);
    return 0;
}
```

**Watch:** at line 5 Vizu warns about the overflow. At line 7 nothing is
wrong: `u` wraps to 4294967295 by the rules.

### Check yourself

1. What is the largest value of an `unsigned char`?
   <details><summary>Answer</summary>

   255: one byte is 8 bits, 2⁸ − 1.
   </details>

2. Is `u = u - 1` for an `unsigned` 0 a bug?
   <details><summary>Answer</summary>

   It is well defined, but often a bug anyway: a loop like
   `for (unsigned i = n - 1; i >= 0; i--)` never ends, because an unsigned
   `i` is always `>= 0`.
   </details>

### Common mistake

Checking for overflow after it has happened: `if (a + b < a)` is undefined for
signed `int`s. Check before: `if (a > INT_MAX - b)`.

## 2.3 Characters are numbers

**Read:** King 7.3 · K&R 1.5, 2.3
**You will:** use `char` as the small integer it is.

A `char` is a one-byte integer. A character constant such as `'A'` is just a
number: its code in the character set, 65 in ASCII. So you can do arithmetic
with characters: the next letter, or the value of a digit.

```c run
#include <stdio.h>

int main(void) {
    char c = 'A';
    char next = c + 1;
    int digit = '7' - '0';
    printf("%c %d %c %d\n", c, c, next, digit);
    return 0;
}
```

`%c` prints a value as a character, `%d` as a number. The same box, two
readings.

**Watch:** Vizu shows a `char` box with both: the character and its code.

### Check yourself

1. What does `printf("%d\n", 'a' - 'A');` print?
   <details><summary>Answer</summary>

   32. Lower-case letters come 32 after upper-case ones in ASCII; that is how
   `toupper` can work.
   </details>

2. Why is `'7' - '0'` equal to 7?
   <details><summary>Answer</summary>

   The digits '0' to '9' have consecutive codes (48 to 57). The standard
   guarantees this for digits, not for letters.
   </details>

### Common mistake

Mixing up `'0'` (the character, code 48), `0` (the number) and `'\0'` (the
character with code 0, which ends strings: module 8).

## 2.4 Floating point

**Read:** King 7.2 · K&R 2.2
**You will:** see why `0.1 + 0.2` is not quite `0.3`.

`float` (4 bytes, about 7 significant digits) and `double` (8 bytes, about 16)
hold numbers with a fractional part, in binary. Most decimal fractions,
0.1 among them, have no exact binary form, just as 1/3 has no exact decimal
form. Each addition rounds a little.

```c run
#include <stdio.h>

int main(void) {
    double sum = 0.0;
    for (int i = 0; i < 10; i++)
        sum = sum + 0.1;
    printf("%.17f\n", sum);
    if (sum == 1.0)
        printf("equal\n");
    else
        printf("not equal\n");
    return 0;
}
```

**Watch:** `sum` after each round, then the comparison: its card shows the two
values side by side.

### Check yourself

1. How should two `double`s be compared?
   <details><summary>Answer</summary>

   With a tolerance: `fabs(a - b) < 1e-9` (from `<math.h>`), the tolerance
   chosen for the problem.
   </details>

2. Why is money often counted in whole cents, in an integer type?
   <details><summary>Answer</summary>

   Integers are exact; 0.10 + 0.20 in `double` is not.
   </details>

### Common mistake

Printing a `double` with `%d`, or an `int` with `%f`. `printf` cannot know the
types of its values, so it reads the wrong bytes. Vizu warns when the
placeholder and the value do not match.

## 2.5 Operators, precedence and side effects

**Read:** King 4 · K&R 2.5–2.12
**You will:** read an expression the way the compiler does.

`*` and `/` bind tighter than `+` and `-` (*precedence*); operators of the same
level group left to right (*associativity*): `a - b - c` is `(a - b) - c`.
`i++` gives the old value and then adds one; `++i` adds one and gives the new
value. `x += 3` is short for `x = x + 3`.

```c run
#include <stdio.h>

int main(void) {
    int i = 5;
    int a = i++;
    int b = ++i;
    int c = 2 + 3 * 4;
    int d = (2 + 3) * 4;
    i += 10;
    printf("a=%d b=%d c=%d d=%d i=%d\n", a, b, c, d, i);
    return 0;
}
```

**Watch:** at line 5 two boxes change in one step: `a` gets 5 and `i` becomes 6.
That is a *side effect*: the expression did more than give a value.

### Check yourself

1. Predict the output before running.
   <details><summary>Answer</summary>

   `a=5 b=7 c=14 d=20 i=17`.
   </details>

2. What does `i = i++;` do?
   <details><summary>Answer</summary>

   It is undefined: `i` is changed twice with nothing to say which happens
   first. Never change a variable twice in one expression, or read it in the
   same expression where you change it.
   </details>

### Common mistake

Trusting the order of evaluation. In `f() + g()` C does not say which function
runs first; in `printf("%d %d", i, i++)` the result is undefined.

## 2.6 Conversions and casts

**Read:** King 7.4 · K&R 2.7
**You will:** predict what type an expression has, and change it on purpose.

When an operator gets two different types, C converts the "smaller" one: an
`int` plus a `double` is a `double`. When a value is stored, it is converted
to the box's type: a `double` stored in an `int` loses its fraction. A *cast*,
`(double) x`, converts on purpose.

```c run
#include <stdio.h>

int main(void) {
    int total = 7, count = 2;
    double wrong = total / count;
    double right = (double) total / count;
    int truncated = 3.99;
    printf("%.2f %.2f %d\n", wrong, right, truncated);
    if (-1 < 1u)
        printf("as expected\n");
    else
        printf("surprise: -1 < 1u is false\n");
    return 0;
}
```

**Watch:**

- Line 5: `total / count` is computed in `int` (3) and only then stored in a
  `double`: 3.00.
- Line 6: the cast makes `total` a `double` first, so the division is in
  `double`: 3.50.
- Line 9: comparing a signed and an unsigned `int` converts the signed one to
  unsigned. −1 becomes 4294967295, which is not less than 1.

### Check yourself

1. What is `(int) 2.9`? And `(int) -2.9`?
   <details><summary>Answer</summary>

   2 and −2: conversion to an integer cuts towards zero.
   </details>

2. How do you compute the average of `int` scores as a `double`?
   <details><summary>Answer</summary>

   Cast before dividing: `(double) sum / n`.
   </details>

### Common mistake

Comparing a signed value with the result of `sizeof` or `strlen` (both
unsigned): `if (i < strlen(s) - 1)` misbehaves when the string is empty, since
`0 - 1` wraps to a huge number.
