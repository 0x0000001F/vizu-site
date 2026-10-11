# Module 1 · First programs

Read alongside: King chapters 2–3 · K&R sections 1.1–1.4

Five lessons that get you from an empty file to a program that reads numbers
and prints a table. Along the way you meet the one picture this whole course
is built on: a variable is a box in memory.

## 1.1 Hello, world

**Read:** King 2.1–2.2 · K&R 1.1
**You will:** run a C program and know what each of its lines is for.

Every C program is made of *functions*, and every program has one called
`main`: that is where it starts. A function is a list of *statements* between
`{` and `}`, run one after another.

```c run
#include <stdio.h>

int main(void) {
    printf("hello, world\n");
    return 0;
}
```

Line by line:

- `#include <stdio.h>` is not C code: it is an instruction to the
  *preprocessor*, which pastes in the declarations of the standard input and
  output functions, `printf` among them.
- `int main(void)` says that `main` takes nothing and gives back an `int`.
- `printf("hello, world\n");` prints the text between the quotes. `\n` is one
  character, the newline; without it the next output would carry on on the
  same line.
- `return 0;` ends `main` and tells whoever started the program that it went
  well. Any other number means something went wrong.

On your own machine a C program is first *compiled* (turned into machine
instructions) and *linked* (joined with the library code for `printf`), then
run: `gcc hello.c -o hello` and then `./hello`. Vizu runs the same program
without that step, so that it can stop after every line.

**Watch:** the program is over in a few steps. Under the memory, the strip
shows what the computer does for `printf`: the text goes into *stdout's
buffer* in the C library, the newline sends it on with the `write` system
call, and the kernel puts `hello, world` on the screen. The last step says
that `main` returned 0.

### Check yourself

1. What does this print?
   ```c
   printf("one");
   printf("two\n");
   ```
   <details><summary>Answer</summary>

   `onetwo` on one line: only the second call prints a newline.
   </details>

2. Delete the `#include` line and run it again. What does Vizu say?
   <details><summary>Answer</summary>

   It warns that `printf` is used without its header. A C99 compiler rejects
   or warns about the call, because it no longer knows what `printf` looks
   like.
   </details>

### Common mistake

Text in C goes between double quotes: `"hello"`. Single quotes are for one
character: `'h'`. `printf('hello')` is not a string at all.

## 1.2 Variables are boxes

**Read:** King 2.4 · K&R 1.2
**You will:** see that a variable is a named box in memory, with a type, a
size and an address.

A *declaration* such as `int height;` asks for a box big enough for an `int`
(4 bytes on today's machines) and gives it a name. An *assignment* such as
`height = 8;` writes a value into the box. You can do both at once: `int
height = 8;` is a declaration with an *initializer*.

```c run
#include <stdio.h>

int main(void) {
    int height = 8;
    int length = 12;
    int width;
    width = 10;
    int volume = height * length * width;
    printf("volume: %d cubic inches\n", volume);
    return 0;
}
```

`%d` in `printf`'s text is a placeholder: it is replaced by the next value
after the text, printed as a decimal integer.

**Watch:**

- Step to line 7. `width` already has a box, but its value is `?`: nobody has
  written to it yet. In C such a box holds whatever bits were left there.
- Step once more. Line 7 writes 10, and the box lights up amber: amber always
  means "this changed in the last step".
- Every box has a small number on it: its *address*, where it lives in memory.
  Module 6 is about those numbers.

### Check yourself

1. How many bytes do the four variables take together?
   <details><summary>Answer</summary>

   16: four `int`s of 4 bytes each.
   </details>

2. What is printed if line 7 is removed?
   <details><summary>Answer</summary>

   Nothing you can rely on. `width` is read before anything was written to
   it; Vizu warns at line 8 and carries on, as a real program would, with
   whatever happened to be in the box.
   </details>

### Common mistake

Using a variable before giving it a value:

```c run
#include <stdio.h>

int main(void) {
    int count;
    int next = count + 1;
    printf("%d\n", next);
    return 0;
}
```

The compiler often accepts this. Vizu shows `count` as `?` and warns at line 5.

## 1.3 Arithmetic and a temperature table

**Read:** K&R 1.2 · King 2.6
**You will:** write a loop, and meet integer division.

K&R's second program prints a table of Fahrenheit temperatures and their
Celsius equivalents, using the formula C = 5/9 × (F − 32).

```c run
#include <stdio.h>

/* print a Fahrenheit-Celsius table, 0 to 100 by 20 */
int main(void) {
    int fahr = 0;
    while (fahr <= 100) {
        int celsius = 5 * (fahr - 32) / 9;
        printf("%3d %4d\n", fahr, celsius);
        fahr = fahr + 20;
    }
    return 0;
}
```

A `while` loop checks its condition; if it is true the body runs, and then
the condition is checked again. `%3d` prints a number at least 3 characters
wide, so the columns line up.

**Watch:**

- Each check of `fahr <= 100` is a step of its own, with the question in
  words and its answer. The loop's last check is the one that says "no".
- `celsius` is a new box inside the loop's `{ }`: it is made for each round.
- `fahr` goes 0, 20, 40 … and the box lights amber each time.
- The flow beside the memory draws only the decision, `fahr <= 100`; lines
  7–9 are one box. A yes sends the light down green into the body, the way
  back round the loop is grey, and the last check's no sends it red to
  `return 0`.

### Check yourself

1. How many rows does the table have?
   <details><summary>Answer</summary>

   Six: 0, 20, 40, 60, 80 and 100. The seventh check (`120 <= 100`) is false.
   </details>

2. What is `7 / 2` in C? And `-7 / 2`?
   <details><summary>Answer</summary>

   `3` and `-3`. Dividing two integers gives an integer; the fraction is cut
   off, towards zero.
   </details>

### Common mistake

Writing the formula the way it reads, 5/9 first:

```c run
#include <stdio.h>

int main(void) {
    int fahr = 212;
    int celsius = 5 / 9 * (fahr - 32);
    printf("%d\n", celsius);
    return 0;
}
```

`5 / 9` is integer division, so it is 0, and every temperature becomes 0
degrees. Multiply first, or use `5.0 / 9.0` (lesson 2.4).

## 1.4 `for` and named constants

**Read:** K&R 1.3–1.4 · King 2.6, 6.3
**You will:** use a `for` loop and `#define`.

The same table, with the three parts of the loop on one line and the numbers
given names:

```c run
#include <stdio.h>

#define LOWER 0     /* lowest temperature in the table */
#define UPPER 100   /* highest */
#define STEP  20    /* step between rows */

int main(void) {
    for (int fahr = LOWER; fahr <= UPPER; fahr = fahr + STEP)
        printf("%3d %6.1f\n", fahr, (5.0 / 9.0) * (fahr - 32));
    return 0;
}
```

`for (start; condition; next)` runs `start` once, then: check the condition,
run the body, run `next`, check again. Use `for` when the loop has a counter;
`while` when it runs until something happens.

`#define UPPER 100` tells the preprocessor to replace the word `UPPER` by `100`
before the compiler sees the program. A name like this is not a variable: it
has no box.

**Watch:**

- `LOWER`, `UPPER` and `STEP` never appear in the memory picture. `fahr` does.
- `fahr` exists only while the loop runs: after the last check its box is gone.
- `%6.1f` prints a floating-point number 6 characters wide with one digit
  after the point. `5.0 / 9.0` is not integer division, so the values are
  right this time.

### Check yourself

1. How many times does `printf` run?
   <details><summary>Answer</summary>

   Six, as before.
   </details>

2. What happens if `STEP` is 0?
   <details><summary>Answer</summary>

   `fahr` never changes, so `fahr <= UPPER` is always true: a loop that never
   ends. Vizu stops it after 5000 steps and says why.
   </details>

### Common mistake

A semicolon after `#define UPPER 100;` becomes part of the replacement: every
`UPPER` turns into `100;`, and the error appears far from the `#define`.

## 1.5 Reading input

**Read:** King 2.5, 3.2 · K&R 1.5 (and 7.4 later)
**You will:** read numbers with `scanf`, and meet your first address.

```c run input="3 4\n"
#include <stdio.h>

int main(void) {
    int a, b;
    printf("Two numbers: ");
    scanf("%d %d", &a, &b);
    printf("%d + %d = %d\n", a, b, a + b);
    return 0;
}
```

`scanf` is `printf` backwards: the text says what to expect, and the values
read are stored in the variables. But a function cannot change its caller's
variables by itself (lesson 5.2 shows why). So we do not give `scanf` the
values of `a` and `b`; we give it their *addresses*, `&a` and `&b`, and it
writes into the boxes at those addresses.

**Watch:**

- Before line 6, `a` and `b` are `?`.
- At line 6 both turn amber: `scanf` wrote 3 and 4 through their addresses.
- The input panel shows what was read.

### Check yourself

1. The input is `10 x`. What does `scanf` store, and what does it return?
   <details><summary>Answer</summary>

   It stores 10 in `a`, stops at `x`, and leaves `b` untouched. It returns 1,
   the number of values it stored. Checking that number is how a program
   notices bad input.
   </details>

2. Why does `printf` not need `&`?
   <details><summary>Answer</summary>

   `printf` only reads the values; it has no need to change the boxes.
   </details>

### Common mistake

Forgetting the `&`:

```c run input="5" expect="error"
#include <stdio.h>

int main(void) {
    int n;
    scanf("%d", n);
    printf("%d\n", n);
    return 0;
}
```

`scanf` gets the value in `n` (unknown) and treats it as an address. On a real
machine this writes to a random place in memory, or crashes. Vizu stops at
line 5 and explains it.
