# Module 1 · First programs

A computer can do only a handful of very simple things. Its processor can
copy a number from one place to another, add two numbers, compare them, and
choose which instruction to carry out next. That is nearly all. Everything
else a computer seems to do, from showing this page to playing a film, is
built out of millions of those small steps, carried out one after another
very quickly.

The processor does not read words. Each of its instructions is a number,
and a program, as the processor sees it, is a long row of such numbers in
memory. People did once write programs that way, number by number, and it
was slow, tiring work in which one wrong digit could spoil everything. So
programming languages were invented. In a language like C we write the
program as text that a person can read, and a program called a *compiler*
translates that text into the processor's numbers for us.

This module starts with the shortest C program that does something you can
see, and ends with a program that reads numbers you type and does
arithmetic with them. On the way we meet the idea that the whole course is
built on: a variable is a box in the computer's memory. Every box has a
place, every box holds a value, and when the program runs, values are
written into boxes one step at a time. In this course you will not have to
imagine that happening. You will watch it.

## 1.1 Hello, world

The first program people write in a new language traditionally does
nothing more than print a greeting. That sounds too small to be worth
doing, but it is not. To get those two words onto the screen you have to
write the program text correctly, have it translated, run it, and find
where its output went. Once all of that works, the rest is a matter of
learning what to put inside the program.

Here is the program in C.

```c run
#include <stdio.h>

int main(void) {
    printf("hello, world\n");
    return 0;
}
```

It is only six lines long, yet it already contains the three kinds of
things that every C program is made of: a *directive*, a *function*, and
*statements*. Let us go through it from the top.

### The directive

The first line, `#include <stdio.h>`, is not really part of the program's
work. Lines that begin with `#` are messages to a helper called the
*preprocessor*, which reads the program before the compiler does. This one
asks the preprocessor to bring in the contents of a file named `stdio.h`.

Why do we need it? C itself has no way to print. There is no word in the
language that means "show this on the screen". Printing is done by
functions that come with C in its *standard library*, a collection of
ready-made code that every C system provides. `printf` is one of those
functions. The file `stdio.h` (short for "standard input and output") is a
*header*: it describes the library's input and output functions, what
their names are and what they expect to be given, so that the compiler can
check that we use them properly. Without the header the compiler would meet
the word `printf` and have no idea what it is.

### The function

Next comes `int main(void)`, followed by everything between `{` and `}`.
This is a *function*: a named piece of program that does a job. A C
program may contain many functions, but it must contain one named `main`,
because that is where the program starts. When you run a C program, the
first thing that happens is that `main` is called.

The words around the name tell us what `main` takes and gives. The `void`
in the parentheses means that `main` is given nothing to work with. The
`int` in front means that when `main` finishes, it hands back a whole
number, an *integer*. We will see in a moment who receives that number.

The braces `{` and `}` mark the beginning and the end of the function's
body. Everything between them belongs to `main`.

### The statements

Inside the body are two *statements*. A statement is one instruction to
the computer, and statements are carried out in the order they are
written, from top to bottom. Each statement in C ends with a semicolon,
the way a sentence ends with a full stop.

The first statement is

```c
printf("hello, world\n");
```

This is a *function call*. To call a function you write its name and then,
in parentheses, the things you give it, which are called its *arguments*.
Here we give `printf` one argument: the text between the double quotes.
Text in double quotes is called a *string*. `printf` prints the string it
is given.

Look closely at the end of the string. The two characters `\n` are not
printed as a backslash and an `n`. Together they stand for a single
character, the *newline*, which moves the output to the start of the next
line. We cannot simply press Enter inside the quotes, because a string must
begin and end on the same line, so C gives us this two-character way of
writing it. `printf` never moves to a new line by itself. If you leave out
the `\n`, whatever is printed next will continue on the same line.

The second statement, `return 0;`, ends `main`, and with it the whole
program. The 0 is the integer that `main` promised to hand back. It goes to
whoever started the program, usually the operating system or the command
line you typed into, and it reports how things went: 0 means that all went
well, and any other number means that something went wrong. A program that
runs another program can look at this number to decide what to do next.

### From text to a running program

On your own computer, you would save this text in a file, say `hello.c`,
and turn it into a program you can run. With the widely used `gcc`
compiler that is one command:

```text
gcc hello.c -o hello
```

That one command does three jobs in turn. First the *preprocessor* carries
out the directives, so `#include <stdio.h>` is replaced by the text of the
header. Then the *compiler* translates the C into the processor's
instructions. Finally the *linker* joins those instructions with the
library's own code for `printf`, which was compiled long ago and sits on
your computer ready to be used. The result is a file named `hello` that
the operating system can load into memory and run:

```text
./hello
hello, world
```

### Running it here

In this course you never have to do that by hand. The box around the
program above is a *lab*: Vizu runs the program right in your browser, and
it stops after every step so that you can look at what happened.

The buttons under the program move through the run. The middle one plays
the whole run slowly; the arrows on either side of it go one step forward
or one step back, and the outer ones jump to the first step or the last.
The slider does the same thing, and when the lab has the focus, the left
and right arrow keys step too. The line marked in the code is the line that
runs next, and the sentence under the code says in words what is
happening.

Beside the code is the *flow* of the program. It draws only the places
where a program makes a decision; this program makes none, so its flow is
a single box. Under it is the *memory*: the stack, where a function's
variables live, and the heap, which comes much later. This program has no
variables, so both are empty for now. Lesson 1.3 fills them.

The most interesting part of this program is at the bottom. When
`printf` runs, the strip there shows what the computer does to get your
text onto the screen, in stages that you step through one at a time.
First `printf` builds the text. Then it does not send it to the screen
straight away; it copies it into a *buffer*, a waiting area in memory that
belongs to the C library. Sending text to the screen is the operating
system's job, and asking the operating system for anything is slow
compared with copying a few bytes, so the library prefers to collect text
and send it in one go. It sends what it has collected as soon as it sees a
newline. That is the next stage: the library finds the `\n`, and calls
`write`, which is a *system call*, a request to the operating system's
core, the *kernel*. The kernel then passes the text to the screen. If you
uncheck "Stop at each stage", a single press goes through all of that at
once; "Open in detail" shows the same journey on a larger drawing.

You do not need to remember all of this now. The point is only that even
`printf` is not magic: it is a series of ordinary steps, and you can see
every one of them.

### Check yourself

1. What does this print?
   ```c
   printf("one");
   printf("two\n");
   ```
   <details><summary>Answer</summary>

   `onetwo`, on one line. The first call prints no newline, so the second
   call's text continues where the first one stopped.
   </details>

2. Delete the `#include` line in the tool and run the program. What
   happens, and why?
   <details><summary>Answer</summary>

   Vizu warns that `printf` is used without its header. Without
   `stdio.h` the compiler has not been told what `printf` is, so it cannot
   check the call. A modern compiler warns about this or refuses the
   program.
   </details>

3. Which part of the program decides whether the operating system is told
   that everything went well?
   <details><summary>Answer</summary>

   `return 0;`. The number after `return` in `main` is the program's exit
   status, and 0 means success.
   </details>

### A common mistake

Text in C goes between double quotes: `"hello"`. Single quotes are for
exactly one character, such as `'h'`. Writing `printf('hello')` does not
give `printf` a string at all, and the compiler will complain.

## 1.2 Comments, names and the shape of a program

Before our programs grow, it is worth knowing what the compiler pays
attention to and what it ignores, because that decides how freely we may
arrange the text.

### Comments

A program is read by people as well as by the compiler, and people need
explanations that the compiler does not. For them C has *comments*. A
comment begins with `/*` and ends with `*/`, and it may run over as many
lines as you like. There is also a shorter kind: two slashes, `//`, begin a
comment that lasts until the end of the line. The preprocessor removes all
comments before the compiler starts, so a comment never changes what a
program does.

```c run
/* greet.c: prints a short greeting.
   Written to show comments and layout. */
#include <stdio.h>

int main(void) {
    // Each printf below prints one line.
    printf("Good morning.\n");
    printf("This line has a comment after it.\n");   // and here it is

    printf("Spaces    inside a string are kept.\n");
    return 0;
}
```

Step through this program and watch which lines are marked. The comment
lines are never marked, because by the time the program runs they no
longer exist. Neither are the blank lines: a blank line does nothing, it
only separates parts of the text for the reader's eye.

A good comment says *why* something is done, or what a piece of code is
for. A comment that only repeats the code in words, such as `x = 5; /* set
x to 5 */`, adds nothing; the reader can already see that.

### Names

The things we name in a program, such as functions and variables, are
called *identifiers*. An identifier may contain letters, digits and the
underscore `_`, but it must not begin with a digit. So `total`, `max_value`
and `line2` are fine, while `2nd_line` is not. Upper and lower case are
different letters in C: `total`, `Total` and `TOTAL` are three different
names. It is usual to write names in lower case and to separate words with
underscores.

A few words are reserved by the language itself and may not be used as
names. They are the *keywords*, such as `int`, `return`, `if`, `while`,
`for` and `void`. You will meet them all as the course goes on; there are
only a few dozen.

### Layout

C cares very little about how the text is spread out. Between the words
and symbols of a program you may put as many spaces, tabs and line breaks
as you like, and the meaning stays the same. This program is exactly the
same as `hello, world` to the compiler:

```c
#include <stdio.h>
int main(void){printf("hello, world\n");return 0;}
```

It is also much harder to read. There are only two places where layout
matters. A directive such as `#include` must be on a line of its own, and a
string must be complete on one line. Everywhere else, the layout is for
people, and good layout is a kindness to whoever reads the program next,
which is often you, a few weeks later. That is why the body of a function
is indented: the indentation shows at a glance which lines belong inside
the braces.

### Check yourself

1. Which of these are valid names: `count`, `_count`, `count_2`, `2count`,
   `int`?
   <details><summary>Answer</summary>

   `count`, `_count` and `count_2` are valid. `2count` begins with a digit,
   and `int` is a keyword.
   </details>

2. Does adding comments make a program run more slowly?
   <details><summary>Answer</summary>

   No. Comments are removed before the program is translated, so the
   finished program does not contain them at all.
   </details>

### A common mistake

Forgetting to close a comment. Everything after `/*` is ignored until the
next `*/`, so a missing `*/` silently swallows the code that follows it, and
the error the compiler then reports may appear many lines further down.

## 1.3 Variables are boxes

So far our programs have only printed text that was already written in
them. To do anything more, a program has to remember values: a number
typed in, a result worked out halfway through. It keeps them in
*variables*.

The best way to think of a variable is as a box in the computer's memory.
The box has a name, so that we can refer to it. It has a *type*, which says
what sort of value it holds and how big the box must be. It holds a value,
which can be replaced by another while the program runs. And it has a
place in memory, called its *address*.

### Declaring a variable

Before a variable can be used, it must be *declared*: we state its type and
its name. The declaration

```c
int height;
```

asks for a box that can hold an `int`, a whole number such as 8 or −250,
and names it `height`. On today's computers an `int` takes 4 bytes of
memory. If we need numbers with a fractional part, like 2.75, we use the
type `double` instead, which takes 8 bytes.

To put a value into the box we use *assignment*:

```c
height = 8;
```

Read the `=` as "becomes", not as "equals": the statement means that
`height` becomes 8. Whatever was in the box before is thrown away.
Declaring a variable and giving it its first value can also be done in one
go, `int height = 8;`. The value after the `=` is then called an
*initializer*.

### A program with variables

This program works out how much a box-shaped container holds. Its sides
are measured in centimetres, so the volume comes out in cubic centimetres,
and since one litre is 1000 cubic centimetres, dividing by 1000 gives
litres.

```c run
#include <stdio.h>

int main(void) {
    int length = 30;
    int width = 20;
    int height;
    height = 10;
    int volume = length * width * height;
    double litres = volume / 1000.0;
    printf("Volume: %d cubic centimetres\n", volume);
    printf("That is %.1f litres.\n", litres);
    return 0;
}
```

The new thing in the `printf` calls is the *conversion specification*.
When `printf` meets `%d` in its string, it does not print those two
characters. Instead it takes the next value given after the string and
prints it there, as a decimal integer. `%.1f` does the same for a `double`,
printing it with one digit after the decimal point. Lesson 1.6 looks at
these in detail.

### Watching the boxes

This is the first program with variables, so step through it slowly.

When line 4 runs, a box named `length` appears in the stack, and 30 is
written into it. The box is marked in amber. Amber always means the same
thing in this course: this was written in the step that just ran. One step
later, the amber moves to `width`.

Then look at line 6. It declares `height` but gives it no value, so the box
appears with a question mark in it. The question mark means that nothing
has been written there yet. In a real computer the box is never truly
empty: it holds whatever bits happened to be left in that piece of memory
by earlier work. C does not clear it for you. Only on line 7 does the 10
arrive.

Line 8 reads three boxes and writes the product into a fourth. Line 9 is
worth a second look: `volume / 1000.0` divides an `int` by a number with a
decimal point, so the division is done with fractions and the result, 6.0,
goes into the `double` box `litres`.

Notice also the small number on each box. That is its address, the place
in memory where the box lives. Vizu uses short, round addresses so that
they are easy to read. You will not need them until the module on
pointers, but they are there from the very start, because every variable
has one.

### Check yourself

1. How many bytes of memory do the five variables use altogether?
   <details><summary>Answer</summary>

   24: four `int`s of 4 bytes each, and one `double` of 8 bytes.
   </details>

2. What would be printed if line 7 were removed?
   <details><summary>Answer</summary>

   Nothing you could rely on. `height` would be read before anything was
   written into it, so the volume would be built from whatever happened to
   be in that box. Vizu warns you on line 8.
   </details>

3. After `int a = 5; a = 7; a = a + 1;`, what is in `a`?
   <details><summary>Answer</summary>

   8. Each assignment replaces the old value; the last one reads 7, adds 1,
   and writes 8 back into the same box.
   </details>

### A common mistake

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

Most compilers accept this without a word. Step to line 5 and you will see
why the result is meaningless: `count` still holds its question mark, and
Vizu warns that it is being read before it was written.

## 1.4 Arithmetic and a temperature table

Now that we can keep values in boxes, we can let a program work out many
values in a row. Our example is a table of temperatures: Fahrenheit
degrees on the left, and the same temperatures in Celsius on the right.
The formula is C = 5 × (F − 32) / 9.

To make a table, the program must do the same calculation over and over,
each time with a different Fahrenheit value. Repeating a piece of program
is called a *loop*, and the simplest loop in C is `while`.

```c run
#include <stdio.h>

/* Print a table of Fahrenheit temperatures and their Celsius
   equivalents, from 0 to 100 degrees in steps of 20. */
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

### How `while` works

A `while` loop consists of a *condition* in parentheses and a *body*,
here the three statements between the braces. The loop works like this:
first the condition is tested. If it is true, the body is carried out once,
and then the program goes back and tests the condition again. This goes on
until the test finds the condition false; then the loop is finished, and
the program continues with whatever comes after the body.

Our condition is `fahr <= 100`, "fahr is less than or equal to 100". At
the start `fahr` is 0, so the condition holds and the body runs: it works
out the Celsius value, prints a line of the table, and adds 20 to `fahr`.
Then the test comes round again with 20, and so on. When `fahr` reaches
120, the test fails, and the program goes on to `return 0`.

Notice that the body must change something that the condition depends on.
If it did not, the condition would stay true for ever, and the loop would
never end.

### Arithmetic

C has the arithmetic operators you would expect: `+` for addition, `-` for
subtraction, `*` for multiplication and `/` for division. There is also
`%`, which gives the *remainder* of a division: `17 % 5` is 2. As in
ordinary mathematics, multiplication and division are done before addition
and subtraction, and parentheses can change that order. In
`5 * (fahr - 32) / 9` the subtraction in parentheses comes first; then the
multiplication and the division are done from left to right.

There is one important difference from ordinary arithmetic. When both
numbers in a division are integers, the result is an integer too, and any
fraction is simply cut off. So `7 / 2` is 3, not 3.5, and `-7 / 2` is −3.
Our table relies on this: the Celsius values are whole numbers, rounded
towards zero.

The `%3d` in `printf` prints an integer at least three characters wide,
with spaces in front if the number is shorter. That keeps the columns of
the table neatly lined up.

### Watching the loop

Step through the program and look at the flow beside the memory. It
shows the loop's one decision, `fahr <= 100`. Each time the condition is
tested, the test is a step of its own: the sentence under the code asks the
question in words, with the numbers filled in, and gives the answer. A yes
lights the path into the body in green. The way back up to the test is
drawn in grey, and the final no lights the way out of the loop in red.

In the memory, `fahr` turns amber every time line 10 adds 20 to it. And
`celsius` behaves in an interesting way: because it is declared inside the
braces of the loop, its box is made fresh at the start of each round and
disappears at the end of it.

### Check yourself

1. How many lines does the table have?
   <details><summary>Answer</summary>

   Six: for 0, 20, 40, 60, 80 and 100. The seventh test, `120 <= 100`, is
   false.
   </details>

2. What are `7 / 2`, `7 % 2` and `-7 / 2` in C?
   <details><summary>Answer</summary>

   3, 1 and −3. Integer division cuts the fraction off, towards zero, and
   `%` gives what is left over.
   </details>

### A common mistake

Writing the formula in the order it is often said, "five ninths of":

```c run
#include <stdio.h>

int main(void) {
    int fahr = 212;
    int celsius = 5 / 9 * (fahr - 32);
    printf("%d\n", celsius);
    return 0;
}
```

`5 / 9` is a division of two integers, so it is 0, and 0 times anything is
0: water would boil at 0 degrees. Either multiply first, as the table
program does, or write `5.0 / 9.0`, which divides with fractions (lesson
2.4 explains why that works).

## 1.5 `for` loops and names for constants

The temperature table works, but it has two weaknesses. The three parts
that control the loop, where `fahr` starts, when the loop stops, and how
`fahr` changes, are spread over three different lines. And the numbers 0,
100 and 20 appear in the middle of the program with nothing to say what
they mean. This lesson fixes both.

```c run
#include <stdio.h>

#define LOWER 0     /* the lowest temperature in the table */
#define UPPER 100   /* the highest */
#define STEP  20    /* the difference between two rows */

int main(void) {
    for (int fahr = LOWER; fahr <= UPPER; fahr = fahr + STEP)
        printf("%3d %6.1f\n", fahr, (5.0 / 9.0) * (fahr - 32));
    return 0;
}
```

### The `for` loop

A `for` loop gathers the three controlling parts into one line, separated
by semicolons:

```c
for (start; condition; change)
    body
```

The *start* is carried out once, before anything else. Then comes the same
round as in a `while` loop: test the *condition*, and if it is true, carry
out the *body*; then carry out the *change*, and test again. When the
condition is false, the loop ends. Our loop is therefore exactly the same
as the `while` loop of the last lesson, only written more compactly.

Which loop should you use? When a loop counts, with a clear start, end
and step, `for` is the natural choice, because a reader sees all three at
once. When a loop goes on until something happens, for instance until the
input runs out, `while` usually reads better.

The body here is a single statement, so it needs no braces. Braces are
only needed to group several statements into one.

### Names for constants

The lines that begin with `#define` give names to numbers. `#define UPPER
100` tells the preprocessor: wherever the word `UPPER` appears in the
program, put `100` in its place. This happens before the compiler sees the
program, so the compiler only ever sees the numbers.

Names like these are called *symbolic constants*, and they make a program
easier both to read and to change. `fahr <= UPPER` says what it means, and
if the table should one day go up to 300, there is a single line to change.
By custom, symbolic constants are written in capital letters, so they
cannot be confused with variables.

A symbolic constant is not a variable. It has no box in memory, and its
value can never change while the program runs.

### Watching it

Step through the program and look at the memory: `LOWER`, `UPPER` and
`STEP` never appear in it, because by the time the program runs, they
have been replaced by numbers. `fahr` does appear, and it lives only as
long as the loop: after the last test its box is gone, because it was
declared inside the `for`.

The output also looks different. `%6.1f` prints a `double` six characters
wide with one digit after the decimal point, and since `5.0 / 9.0` divides
with fractions, the Celsius values are now exact to one decimal: −17.8
instead of −17.

### Check yourself

1. How many times does `printf` run?
   <details><summary>Answer</summary>

   Six times, as before: once for each value of `fahr` from 0 to 100 in
   steps of 20.
   </details>

2. What happens if `STEP` is defined as 0?
   <details><summary>Answer</summary>

   `fahr` never changes, so `fahr <= UPPER` stays true and the loop never
   ends. Vizu stops such a program after a few thousand steps and tells you
   why.
   </details>

3. Rewrite the loop so that the table runs backwards, from 100 down to 0.
   <details><summary>Answer</summary>

   `for (int fahr = UPPER; fahr >= LOWER; fahr = fahr - STEP)`. The start,
   the test and the change all turn round.
   </details>

### A common mistake

Putting a semicolon at the end of a `#define`. With `#define UPPER 100;`
every `UPPER` is replaced by `100;`, semicolon and all, so the line
`fahr <= UPPER;` becomes `fahr <= 100;;` and the compiler reports an error
on a line that looks perfectly correct. A directive is not a statement and
takes no semicolon.

## 1.6 Printing with `printf`

We have been using `printf` since the first lesson, a little at a time.
It is time to look at it properly, since almost every program in this
course prints its results with it.

`printf` is given a string, called the *format string*, followed by any
number of values. It prints the format string character by character,
except in two cases. A `%` begins a conversion specification, which is
replaced by the next value, formatted as the specification says. And a
backslash `\` in the string, as we have seen, begins an *escape sequence*,
a way of writing a character that cannot be typed directly between quotes.

### Conversion specifications

The letter at the end of a conversion specification says what kind of
value to expect and how to show it:

| Written | Prints |
| --- | --- |
| `%d` | an `int`, as a decimal number |
| `%f` | a `double`, with six digits after the point |
| `%e` | a `double` in scientific notation, like `1.234500e+03` |
| `%g` | a `double`, in whichever of the two is shorter |
| `%c` | one character |
| `%s` | a string |
| `%%` | a `%` sign itself |

Between the `%` and the letter you may give a *width* and a *precision*.
The width is the least number of characters to print; a shorter value is
padded with spaces on the left, or on the right if a minus sign comes
before the width. The precision, written after a point, is the number of
digits after the decimal point for `%f` and `%e`. The following program
tries several of them. The square brackets are only there to show exactly
where each printed value begins and ends.

```c run
#include <stdio.h>

int main(void) {
    int n = 42;
    double x = 3.14159;
    printf("[%d] [%5d] [%-5d] [%05d]\n", n, n, n, n);
    printf("[%f] [%.2f] [%8.3f] [%-8.1f]\n", x, x, x, x);
    printf("[%e] [%g]\n", 1234.5, 0.0001);
    printf("[%c] [%s] [100%%]\n", 'A', "text");
    return 0;
}
```

Run it and compare each pair of brackets with its specification. `%5d`
puts three spaces before 42, `%-5d` puts them after, and `%05d` fills with
zeros instead. `%.2f` rounds 3.14159 to 3.14, and `%8.3f` prints 3.142
inside a field eight characters wide.

`printf` trusts you completely about the values. It cannot see what you
gave it; it only reads the format string, and takes the next value to be
whatever the specification says. Give it an `int` where the string says
`%f` and it will misread the bytes and print nonsense. Vizu warns you when
a value does not match its specification, but a compiler may not, so it
pays to check every `printf` call by eye.

### Escape sequences

These are the escape sequences you will use most:

| Written | Means |
| --- | --- |
| `\n` | newline: go to the start of the next line |
| `\t` | tab: jump to the next tab stop |
| `\"` | a double quote, which would otherwise end the string |
| `\\` | a backslash itself |

```c run
#include <stdio.h>

int main(void) {
    printf("Item\tPrice\n");
    printf("Tea\t2.50\n");
    printf("She said \"thank you\".\n");
    printf("A folder: C:\\Users\\me\n");
    return 0;
}
```

When you step through this program, watch the strip at the bottom. Each
`printf` puts its finished text into the buffer, and each `\n` sends the
buffer on to the screen. The tabs line the two columns up, the escaped
quotes are printed as quotes, and each pair of backslashes becomes one.

### Check yourself

1. What does `printf("[%4d]", 7)` print? And `printf("[%-4d]", 7)`?
   <details><summary>Answer</summary>

   `[   7]` and `[7   ]`: four characters wide, padded on the left, and
   then on the right.
   </details>

2. How do you print the text `50% off` with `printf`?
   <details><summary>Answer</summary>

   `printf("50%% off\n");`. A single `%` would begin a conversion
   specification.
   </details>

### A common mistake

Giving `printf` fewer values than its format string asks for, as in
`printf("%d and %d\n", a);`. The second `%d` takes a value that was never
given, and prints whatever happens to be where the value should have been.

## 1.7 Reading input with `scanf`

A program that always works with the same numbers is not much use. We
want to give it numbers while it runs. The standard library's `scanf` reads
from the *standard input*, which is normally the keyboard, and it is the
mirror image of `printf`: its format string says what the input should look
like, and the values it reads are stored in variables.

```c run input="3 4\n"
#include <stdio.h>

int main(void) {
    int a, b;
    printf("Enter two numbers: ");
    scanf("%d %d", &a, &b);
    printf("%d + %d = %d\n", a, b, a + b);
    return 0;
}
```

### Why the `&`?

The strange part is the `&` in front of `a` and `b`. Remember that a
variable is a box with an address. If we wrote `scanf("%d %d", a, b)`, we
would be handing `scanf` the *values* in the boxes, which at this point
are still question marks, and `scanf` could do nothing useful with them.
What `scanf` needs to know is *where* to put the numbers it reads. `&a`
means "the address of `a`", and with that address `scanf` can find the
box and write into it. Module 5 shows why a function cannot simply change
its caller's variables, and module 6 is entirely about addresses; for now
the rule is simply that each variable given to `scanf` needs a `&`.

### How `scanf` reads

`scanf` works through its format string from left to right, and the input
along with it. For each `%d` it first skips any spaces, tabs and newlines
in the input. Then it reads characters for as long as they can be part of
a whole number: an optional sign, then digits. At the first character that
cannot belong to the number it stops, and leaves that character in the
input for whatever reads next. A space in the format string matches any
amount of white space in the input, including none.

Other characters in the format string must appear in the input exactly as
written. That makes it easy to read input in a fixed shape, such as two
fractions joined by a plus sign:

```c run input="1/2 + 1/3\n"
#include <stdio.h>

int main(void) {
    int num1, den1, num2, den2;
    printf("Enter two fractions, like 1/2 + 1/3: ");
    scanf("%d/%d + %d/%d", &num1, &den1, &num2, &den2);
    int num = num1 * den2 + num2 * den1;
    int den = den1 * den2;
    printf("The sum is %d/%d\n", num, den);
    return 0;
}
```

The format `"%d/%d + %d/%d"` says: a number, a slash, a number, then a plus
sign with any spaces around it, then the same again. The arithmetic is the
one you learned at school: to add two fractions, bring them to the same
denominator first.

### Watching the input

Step through either program and watch three things. Before the `scanf`
line, the variables still hold question marks. When `scanf` runs, they
turn amber all together: `scanf` has written into them through their
addresses. And in the strip at the bottom you can see where the input came
from: the keys you typed went first to the kernel, which keeps them until
a whole line has been typed, then into a buffer of the C library, called
stdin's buffer, and `scanf` took its characters from there, one at a time.
The input for these labs is given with the program; in the tool you can
type your own.

### What `scanf` gives back

`scanf` returns a value too: the number of variables it managed to fill.
If you type `10 x` into the first program, `scanf` reads the 10 into `a`,
reaches the `x`, which cannot begin a number, and stops. It leaves `b`
untouched and returns 1. A careful program checks this number before
trusting the variables, and we will do so from module 3 on.

### Check yourself

1. The input is `  12   34`. What do `a` and `b` get?
   <details><summary>Answer</summary>

   12 and 34. Every `%d` skips the white space in front of its number,
   however much there is.
   </details>

2. Why does `printf` not need `&` in front of its variables?
   <details><summary>Answer</summary>

   `printf` only needs the values, to print them. It never has to change
   the boxes, so it does not need to know where they are.
   </details>

3. With the format `"%d/%d"`, what happens if the input is `3 / 4`?
   <details><summary>Answer</summary>

   `scanf` reads 3, then expects a slash right away but finds a space, and
   stops there. It has filled one variable and returns 1. To allow spaces
   around the slash, write `"%d /%d"`: the space in the format matches the
   spaces in the input.
   </details>

### A common mistake

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

Here `scanf` is given the value in `n`, which is unknown, and treats it as
an address. On a real computer it then writes the 5 into some random place
in memory, or the program crashes. Vizu stops at line 5 and explains what
went wrong.
