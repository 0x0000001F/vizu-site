# Module 2 · Types and expressions

In the first module every variable had a type, `int` or `double`, and we
chose between them by a simple rule: whole numbers in an `int`, numbers
with a fraction in a `double`. That rule will carry you a long way, but
sooner or later a program prints a negative number where there should be
a large positive one, or adds 0.1 ten times and does not get 1. To
understand why, we have to look at what a type really is.

Inside the computer, memory holds nothing but *bits*. A bit is the
smallest piece of information there is: it is either on or off, which we
write as 1 or 0. Bits are grouped in eights, and a group of eight bits is a
*byte*. Every byte in memory has its own address. A variable occupies one
or more bytes next to each other, and the bits in those bytes are all that
it holds.

Bits by themselves do not mean anything. The eight bits `01000001` could
be the number 65, or the letter `A`, or part of a much larger number, or
part of a fraction. The *type* of a variable is what gives the bits their
meaning. It decides how many bytes the box takes, and how the bits inside
are to be read. This module goes through C's basic types one by one, looks
at what happens at their edges, and then at how C combines values of
different types in one expression.

## 2.1 Bits, bytes and the integer types

How many different values can a box hold? That depends only on how many
bits it has. One bit can be in two states, 0 or 1. Two bits can be in four:
00, 01, 10 and 11. Each extra bit doubles the number, because every
combination of the other bits can now come once with the new bit off and
once with it on. So 8 bits, one byte, can be in 2⁸ = 256 different states,
and 32 bits, four bytes, in 2³² states, a little over four thousand
million.

C has a family of integer types of different sizes, so that you can choose
a box as large as the numbers you need:

| Type | Size on today's computers | Range |
| --- | --- | --- |
| `char` | 1 byte | −128 to 127 |
| `short` | 2 bytes | −32 768 to 32 767 |
| `int` | 4 bytes | about −2.1 to 2.1 thousand million |
| `long` | 8 bytes | about ±9.2 × 10¹⁸ |
| `long long` | 8 bytes | the same |

The sizes in this table are the ones used on 64-bit Linux and macOS, and
by Vizu. The C standard does not fix them. It only promises that each type
is at least as large as the one above it in the table, and that an `int`
has at least 16 bits. On 64-bit Windows, for example, a `long` is only 4
bytes. C leaves this open so that each kind of computer can use the sizes
it handles best.

When the size matters, there is no need to guess. The operator `sizeof`
gives the size of a type or of a variable, in bytes.

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

The value of `sizeof` has a type of its own, `size_t`, an unsigned integer
type meant for sizes, and `%zu` is the conversion that prints it.

When you step through the program, look at the addresses written beside
the boxes in the memory picture. A variable's address is the address of its
first byte. On the stack, where these variables live, each new variable is
placed below the ones before it, at a lower address, and it needs as many
bytes as its type is long: the `char` takes one byte, the `long` eight. So
the addresses of two neighbours are always at least the size of the lower
one apart. (Sometimes they are a little further apart than that; module 9
explains why.)

### Exact sizes, and naming types

Sometimes a program needs a box of an exact size: a file format that
stores a number in exactly four bytes, for instance. For that the header
`<stdint.h>` provides types such as `int8_t`, `int16_t`, `int32_t` and
`int64_t`, which are exactly 8, 16, 32 and 64 bits on every computer, and
`uint8_t` to `uint64_t`, which are the same sizes without negative
numbers.

These names are not new types. Each is another name for one of the types
in the table, chosen by the system to have the right size. You can make
such names yourself with `typedef`:

```c
typedef int Celsius;
```

After this line, `Celsius` can be used wherever `int` could be:
`Celsius today = 21;`. The box is an ordinary `int`, but the program now
says what the number means. If a program later needs larger temperatures,
the `typedef` is the only line to change.

### Check yourself

1. Why does `10000000000` (ten thousand million) not fit in an `int`?
   <details><summary>Answer</summary>

   A 4-byte `int` has 32 bits, so it can be in 2³² states. Half of them
   are used for negative numbers, so the largest `int` is 2 147 483 647.
   Ten thousand million is far bigger.
   </details>

2. How many different values does a `short` have?
   <details><summary>Answer</summary>

   2¹⁶ = 65 536: it has 16 bits.
   </details>

### A common mistake

Assuming that a type has a certain size. A program that keeps an address
or a large count in an `int` may work on one computer and fail on
another. When it matters, ask `sizeof`, or use the exact types from
`<stdint.h>`.

## 2.2 Writing and printing integers

A number in a program, such as `100`, is called a *constant*. We usually
write constants in decimal, but C allows two other ways, and both come
from the fact that memory is made of bits.

A constant that begins with `0x` is written in *hexadecimal*, base 16,
with the digits 0 to 9 and then A to F for ten to fifteen. A constant that
begins with just `0` is in *octal*, base 8. Why would anyone want those?
Because they are short ways to write bits. One hexadecimal digit stands for
exactly four bits, so a byte is always two hexadecimal digits: `0xFF` is
`11111111`, all eight bits on. Programmers who work close to the hardware
use hexadecimal all the time. Module 12 returns to this.

`printf` can print in all three bases: `%d` in decimal, `%x` in
hexadecimal (`%X` with capital letters), and `%o` in octal.

```c run
#include <stdio.h>

int main(void) {
    int a = 255;
    int b = 0377;
    int c = 0xFF;
    printf("%d %d %d\n", a, b, c);
    printf("%x %X %o\n", a, a, a);

    long big = 3000000000L;
    unsigned int u = 4000000000u;
    printf("%ld %u\n", big, u);
    return 0;
}
```

The first line of output shows that `255`, `0377` and `0xFF` are the same
number, written three ways. In the memory picture the three boxes hold
exactly the same value; the base is only a matter of how the constant was
written in the program text.

The second half of the program shows two more details. A constant has a
type, just as a variable does. Normally it is an `int`, but a constant too
large for an `int` gets a larger type, and you can also ask for one with a
*suffix*: `L` makes a constant a `long`, `U` makes it unsigned, and `UL`
both. `printf` needs to be told about the larger types too: `%ld` prints a
`long`, `%lld` a `long long`, `%u` an `unsigned int`, and `%lu` an
`unsigned long`. The `l` in these is a *length modifier*: it tells
`printf` that the value takes more bytes than an `int`.

### Check yourself

1. What is `0x10` in decimal? And `010`?
   <details><summary>Answer</summary>

   16 and 8. The first is hexadecimal, the second octal.
   </details>

2. Why does a byte always take exactly two hexadecimal digits?
   <details><summary>Answer</summary>

   One hexadecimal digit has 16 possible values, which is exactly what four
   bits can hold, and a byte is eight bits.
   </details>

### A common mistake

Writing a leading zero to line numbers up, as in `int month = 09;`. A
leading zero means octal, and 9 is not an octal digit, so the compiler
refuses it. Worse, `010` is quietly accepted and means 8.

## 2.3 Overflow and unsigned numbers

Every integer type has a largest value, and it is natural to ask what
happens if we go past it. Think of a car's odometer with six digits: after
999 999 kilometres it shows 000 000 again. Binary counting in a fixed
number of bits behaves the same way: when all the bits are 1 and you add
one, every bit turns to 0 and the carry falls off the end.

C has two kinds of integer types, and they treat this situation very
differently.

The types we have seen so far are *signed*: they hold negative numbers as
well as positive ones. If a calculation with signed integers goes past the
largest value, that is called *overflow*, and the C standard says that
the behaviour is *undefined*. That is a strong statement. It does not mean
that the result is some unknown number; it means that C makes no promise
about the program at all from that point on. On most computers the number
simply wraps round to the most negative value, but a compiler that
optimizes the program is allowed to assume that overflow never happens,
and it may rearrange or even remove code on that assumption.

Each integer type also has an *unsigned* version, written with the word
`unsigned` in front: `unsigned int`, `unsigned char` and so on. An
unsigned type holds no negative numbers; all of its bit patterns are used
for zero and upwards, so an `unsigned int` goes from 0 to 4 294 967 295.
For unsigned types, wrapping round is not an error but the rule:
arithmetic is done like the odometer, and going below zero wraps round to
the largest value.

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

Step to line 5. Vizu warns that the addition overflows: this is the line
where the program stops having a meaning that C guarantees. Then step on
to line 7. There nothing is wrong. `u` was 0, and 0 minus 1 in an unsigned
type wraps round, by definition, to 4 294 967 295.

### Check yourself

1. What is the largest value of an `unsigned char`?
   <details><summary>Answer</summary>

   255. A `char` has 8 bits, so it has 2⁸ = 256 values, from 0 to 255 when
   none of them are spent on negative numbers.
   </details>

2. This loop is meant to count down from `n - 1` to 0. What does it do
   instead?
   ```c
   for (unsigned int i = n - 1; i >= 0; i--)
   ```
   <details><summary>Answer</summary>

   It never ends. An unsigned number is always at least 0, so `i >= 0` is
   always true; after 0, `i--` wraps round to the largest value and the loop
   carries on.
   </details>

### A common mistake

Checking for overflow after it has happened. `if (a + b < a)` looks like a
clever test, but for signed integers the addition itself is already
undefined, and the compiler may remove the test. Check before adding
instead: `if (a > INT_MAX - b)`, with `INT_MAX` from `<limits.h>`.

## 2.4 Characters are numbers

How does a computer store a letter? It cannot store the shape of the
letter, only bits. So every character is given a number, its *code*, and
the computer stores that number. The table of codes used almost everywhere
is called ASCII: in it, `A` is 65, `B` is 66, the digit `0` is 48, a space
is 32, and the newline is 10.

In C a character is therefore simply a small integer. The type `char` is
an integer type one byte wide, and a *character constant*, a character
between single quotes such as `'A'`, is nothing but another way of
writing its code: `'A'` and `65` are the same number. This means we can do
arithmetic with characters.

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

`c + 1` is 66, which is the code of `B`. `'7' - '0'` is a well-known trick:
the digits have consecutive codes, `'0'` is 48 and `'7'` is 55, so
subtracting the code of `'0'` from the code of any digit gives the value of
that digit. Programs that read numbers character by character use exactly
this.

`%c` prints a value as the character with that code, and `%d` prints the
same value as a number. Look at the memory picture: Vizu draws a `char`
box with both readings at once, the character and its code, because both
are true at the same time.

### Reading and writing one character

The library has two small functions for single characters. `getchar()`
reads one character from the input and gives back its code, and
`putchar(c)` prints the character whose code is `c`. Here is a program
that reads one character and prints it, followed by the next one in the
alphabet:

```c run input="m\n"
#include <stdio.h>

int main(void) {
    int c = getchar();
    putchar(c);
    putchar(c + 1);
    putchar('\n');
    return 0;
}
```

Notice that `c` is an `int`, not a `char`. That is deliberate. When there
is no more input, `getchar` returns a special value called `EOF`, "end of
file", which must be different from every possible character, so it does
not fit in a `char`. Module 3 uses this to read a whole text.

Escape sequences work in character constants just as they do in strings:
`'\n'` is the newline, `'\t'` a tab, `'\''` a single quote, and `'\\'` a
backslash.

### Check yourself

1. What does `printf("%d\n", 'a' - 'A');` print?
   <details><summary>Answer</summary>

   32. In ASCII the lower-case letters come exactly 32 places after the
   capitals, so adding or subtracting 32 changes a letter's case.
   </details>

2. Why does `'7' - '0'` work for every digit?
   <details><summary>Answer</summary>

   The C standard guarantees that the codes of `'0'` to `'9'` follow each
   other without gaps. (It makes no such promise for the letters, although
   ASCII happens to keep them together too.)
   </details>

### A common mistake

Mixing up `'0'`, `0` and `'\0'`. The first is the digit zero, code 48. The
second is the number zero. The third is the character whose code is 0,
which C uses to mark the end of a string, as module 8 explains.

## 2.5 Floating point

Integers are exact, but they cannot hold 2.5 or 0.001. For numbers with a
fractional part C has the *floating-point* types: `float`, which takes 4
bytes and keeps about 7 significant digits, and `double`, which takes 8
bytes and keeps about 16. "Floating point" means that the decimal point is
not in a fixed place: the number is kept as its significant digits and a
separate exponent, much like scientific notation, where 1 230 000 is
written 1.23 × 10⁶. That lets the same box hold very large and very small
numbers.

There is one thing about these types that surprises almost everyone. The
digits are kept in *binary*, not in decimal, and most decimal fractions
cannot be written exactly in binary. You know the same problem from
decimal: 1/3 is 0.3333… and never ends, so any decimal you write down is
slightly wrong. In binary, 1/10 is such a fraction. The `double` closest to
0.1 is very close, but it is not 0.1, and every calculation with it rounds
a little more.

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

Watch `sum` grow in the memory picture, one tenth at a time. After ten
rounds you would expect exactly 1, and `%.17f`, which prints seventeen
digits after the point, shows that it is not: the tiny errors of each
addition have added up. So when the program asks whether `sum` equals
1.0, the honest answer is no. Step to that comparison: the sentence under
the code shows it as `0.9999999999999999 == 1.0`, with every digit of
`sum`, and the answer is no.

### Writing and reading floating-point numbers

A constant with a decimal point or an exponent is a `double`: `2.5`,
`1e6` (one million), `1.5e-3` (0.0015). With an `f` at the end, as in
`0.1f`, it is a `float`. For printing, `printf` has `%f`, `%e` and `%g`, as
lesson 1.6 showed. A `float` is printed with the same conversions, since
`printf` turns it into a `double` first.

Reading is different. `scanf` must know exactly how big the box is that it
writes into, so it reads a `float` with `%f` and a `double` with `%lf`.

```c run input="2.5\n"
#include <stdio.h>

int main(void) {
    double side;
    printf("Side of the square: ");
    scanf("%lf", &side);
    printf("Its area is %.2f\n", side * side);
    return 0;
}
```

### Check yourself

1. How should a program test whether two `double`s are equal?
   <details><summary>Answer</summary>

   Not with `==`. Test whether they are close enough instead:
   `fabs(a - b) < 1e-9`, with `fabs` from `<math.h>` and a tolerance that
   suits the problem.
   </details>

2. Why do programs that handle money often count in whole cents, in an
   integer type?
   <details><summary>Answer</summary>

   Integers are exact. 0.10 + 0.20 in a `double` is not exactly 0.30, while
   10 + 20 cents is exactly 30 cents.
   </details>

### A common mistake

Reading a `double` with `%f` in `scanf`. `%f` tells `scanf` that the box
is a 4-byte `float`, so it writes only half of the 8-byte `double`, and the
value comes out wrong. In `printf`, `%f` is right for both; in `scanf`, a
`double` needs `%lf`.

## 2.6 Operators and expressions

An *expression* is anything that has a value: a constant such as `5`, a
variable such as `x`, or a combination of them joined by operators, such
as `x * 2 + 1`. Most of the work in a program is done by expressions, so
it pays to know exactly how C reads them.

### Which operator goes first

When an expression has several operators, C needs rules to decide which
one is applied first. The first rule is *precedence*: some operators bind
more tightly than others. `*`, `/` and `%` bind more tightly than `+` and
`-`, so `2 + 3 * 4` means `2 + (3 * 4)`, which is 14. The second rule is
*associativity*: when operators of the same precedence meet, most of them
are applied from left to right, so `10 - 4 - 3` means `(10 - 4) - 3`, which
is 3, and not `10 - (4 - 3)`. When in doubt, use parentheses: they always
win, and they make the meaning clear to the reader as well.

### Assignment is an expression too

In C, assignment is an operator like any other, and `x = 5` is an
expression whose value is the value assigned. That is why

```c
a = b = 0;
```

works: assignment groups from right to left, so this is `a = (b = 0)`.
First `b` becomes 0, and the value of that, 0, is assigned to `a`.

Changing a variable by some amount is so common that C has short forms for
it. `x += 3` means `x = x + 3`, and there are `-=`, `*=`, `/=` and `%=` in
the same way. These are called *compound assignments*.

The left side of an assignment must be something that has a box, such as
a variable. C calls that an *lvalue*, because it can stand on the left of
an `=`. `x = 5` is fine, but `5 = x` and `x + 1 = 5` make no sense, since
neither `5` nor `x + 1` is a place where a value can be stored.

### Increment and decrement

Adding or subtracting one is the most common change of all, and C has two
operators just for it: `++` adds one to a variable and `--` subtracts one.
Each can be written before or after the variable, and there is a
difference. `++i` adds one to `i`, and the value of the expression is the
new value. `i++` also adds one to `i`, but the value of the expression is
the old value, the one `i` had before.

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

Before running it, try to predict the five numbers it prints. Then step
through and watch line 5 closely: in a single step, two boxes turn amber.
`a` receives 5, the old value of `i`, and `i` itself becomes 6. An
expression that changes something, besides giving a value, is said to have
a *side effect*. On line 6, `i` first becomes 7, and then `b` receives that
new value.

### Check yourself

1. What are the values of `a`, `b`, `c`, `d` and `i` at the end?
   <details><summary>Answer</summary>

   `a=5 b=7 c=14 d=20 i=17`.
   </details>

2. What does `i = i++;` do?
   <details><summary>Answer</summary>

   Nobody can say. It changes `i` twice in one expression, and C does not
   define which change happens last, so the behaviour is undefined. The
   rule to remember: never change a variable twice in one expression, and
   do not read it elsewhere in an expression that changes it.
   </details>

### A common mistake

Relying on the order in which the parts of an expression are worked out.
In `f() + g()`, C does not say whether `f` or `g` is called first, and in
`printf("%d %d\n", i, i++)` the result is undefined. Put side effects in
statements of their own.

## 2.7 Conversions and casts

What happens when an expression mixes types, such as an `int` multiplied
by a `double`? The processor cannot do that directly: it has separate
instructions for integer arithmetic and for floating-point arithmetic,
and both numbers must be in the same form. So C first converts one of the
values to the other's type, following a few rules.

### The rules

When an operator is given two values of different types, the "narrower"
one is converted to the type of the "wider" one, so that no information is
lost: an `int` combined with a `double` is converted to `double`, and the
result is a `double`. Small types, `char` and `short`, are always turned
into `int` before any arithmetic is done on them. These are called the
*usual arithmetic conversions*.

When a value is assigned to a variable, it is converted to the type of the
variable, whatever that is. That can lose information: a `double` stored in
an `int` loses its fraction, and a large `int` stored in a `char` loses its
upper bytes.

Sometimes you want a conversion that C would not make by itself. A *cast*
asks for one explicitly: the name of a type in parentheses before a value,
as in `(double) total`, converts the value to that type.

```c run
#include <stdio.h>

int main(void) {
    int total = 7, count = 2;
    double wrong = total / count;
    double right = (double) total / count;
    int cut = 3.99;
    printf("%.2f %.2f %d\n", wrong, right, cut);
    if (-1 < 1u)
        printf("as expected\n");
    else
        printf("surprise: -1 < 1u is false\n");
    return 0;
}
```

Step through it slowly; each line has a lesson in it.

On line 5, `total / count` divides one `int` by another, so it is integer
division and gives 3. Only then is the 3 converted to `double` for storing
in `wrong`, which therefore holds 3.00. The conversion came too late to
help.

On line 6 the cast converts `total` to `double` before the division. Now
the division mixes a `double` with an `int`, so `count` is converted too,
and the division is done with fractions: 3.50.

On line 7, 3.99 is stored in an `int`, and the fraction is simply cut
off: 3. Conversion to an integer always cuts towards zero; it never
rounds.

Line 8 is the strangest. `1u` is an `unsigned int`, and when a signed and
an unsigned `int` meet, the rules convert the signed one to unsigned. −1
becomes 4 294 967 295, which is certainly not less than 1, so the
comparison is false. Step to it and you will see exactly that: Vizu shows
the comparison as `4294967295 < 1`, with −1 already converted. (In the tool,
the explanation beside the code also spells the conversion out:
`(unsigned int) -1`.)

### Check yourself

1. What are `(int) 2.9` and `(int) -2.9`?
   <details><summary>Answer</summary>

   2 and −2. Converting to an integer cuts towards zero.
   </details>

2. `sum` and `n` are `int`s. How do you compute their average as a
   `double`?
   <details><summary>Answer</summary>

   Cast before dividing: `(double) sum / n`. Writing `(double) (sum / n)`
   is too late, since the integer division has already thrown the fraction
   away.
   </details>

### A common mistake

Comparing a signed number with the result of `sizeof` or `strlen`, which
are unsigned. `if (i < strlen(s) - 1)` misbehaves when the string is
empty: 0 − 1 in an unsigned type is not −1 but the largest possible value.
