# Module 3 · Control flow

The programs so far have mostly gone straight through, one statement after
another from the top of `main` to the bottom, with a single loop now and
then. Real programs are not like that. They look at the data and decide
what to do: print a warning if a number is too large, take one path for
the letters and another for the digits, keep reading until the input runs
out. The order in which a program's statements are carried out is called
its *control flow*, and this module is about the statements that steer it.

There are really only two kinds. A *selection* statement chooses between
paths: `if` and `switch`. A *loop* goes round the same path again: `while`,
`do` and `for`. Add a few statements that jump out of the usual order,
`break`, `continue` and `goto`, and you have every way a C program can
move.

At the heart of all of them is a question that the program asks itself:
is this condition true? Vizu makes every such question a step of its own.
When a condition is tested, the sentence under the code asks the question
in words, with the current values filled in, and gives the answer; the
flow beside the memory lights up the path that the answer leads to. This
module teaches you to read those steps, and by the end you will be able to
predict them.

## 3.1 True and false

Before we can write `if` statements we need conditions, and to write
conditions we need to know how C thinks about truth.

### Comparing

The *relational operators* compare two values: `<` (less than), `>`
(greater than), `<=` (less than or equal to) and `>=` (greater than or
equal to). The *equality operators* are `==` (equal to) and `!=` (not equal
to). Notice that "equal to" is written with two equals signs. A single `=`
is assignment, as we saw in module 2, and confusing the two is one of the
most common mistakes in C.

What does a comparison give? In many languages it gives a special value,
true or false. In C it gives a number: 1 if the comparison holds, and 0 if
it does not. So `3 < 5` is 1, and `3 > 5` is 0.

The other direction works too: wherever C expects a condition, any number
will do. The number 0 counts as false, and every other number counts as
true. `if (n)` means the same as `if (n != 0)`.

### Combining

Conditions can be joined with the *logical operators*. `a && b` ("a and
b") is true when both `a` and `b` are true. `a || b` ("a or b") is true when
at least one of them is. `!a` ("not a") is true when `a` is false.

`&&` and `||` have a property that is used all the time: they work out
their left side first, and if that already decides the answer, they do not
look at the right side at all. If the left side of an `&&` is false, the
whole thing must be false, whatever the right side says; if the left side
of an `||` is true, the whole thing must be true. This is called
*short-circuit evaluation*, and it makes conditions like this one safe:

```c
if (n != 0 && total / n > 10)
```

When `n` is 0, the left side is false, so the division on the right is
never carried out, and the program never divides by zero.

### A type for truth

Since truth is just a number, C programs used to keep yes-or-no values in
an `int`. Modern C also has a type meant only for them, `bool`, with the
values `true` and `false`, provided by the header `<stdbool.h>`. A `bool`
holds only 1 or 0: whatever value you store in it, it becomes 1 if the
value is not zero.

```c run
#include <stdio.h>
#include <stdbool.h>

int main(void) {
    int age = 20;
    bool adult = age >= 18;
    bool teen = age >= 13 && age <= 19;
    printf("adult: %d, teen: %d\n", adult, teen);
    printf("3 < 5 is %d, 3 > 5 is %d\n", 3 < 5, 3 > 5);
    return 0;
}
```

Step through it and look at the memory: `adult` holds 1 and `teen` holds 0.
The comparisons have produced ordinary numbers, stored in ordinary boxes.

### Check yourself

1. What is the value of `5 > 3 > 1`?
   <details><summary>Answer</summary>

   0. It is read as `(5 > 3) > 1`. `5 > 3` is 1, and `1 > 1` is 0. To ask
   whether three numbers are in order, write `5 > 3 && 3 > 1`.
   </details>

2. In `if (n != 0 && total / n > 10)` with `n` equal to 0, how much of the
   condition is worked out?
   <details><summary>Answer</summary>

   Only `n != 0`. It is false, so `&&` already knows the answer. Vizu shows
   the right side as not checked.
   </details>

### A common mistake

Writing a range the way it is written in mathematics: `if (0 < x < 10)`.
This compiles, but it compares `0 < x`, which gives 0 or 1, with 10, and
both are always less than 10. The condition is true for every `x`. Write
`if (0 < x && x < 10)`.

## 3.2 `if` and `else`

The `if` statement carries out a statement only when a condition is true:

```c
if (condition)
    statement
```

If you want several statements to depend on the condition, group them with
braces into a *compound statement*, also called a *block*. A block counts as
a single statement wherever C expects one:

```c
if (temperature > 30) {
    printf("It is hot.\n");
    hot_days++;
}
```

An `if` may have an `else` part, which is carried out when the condition is
false. Exactly one of the two parts runs, never both and never neither.

### Choosing among many

Often there are more than two cases. Since the statement after `else` can
itself be an `if`, we can chain them, and it is the custom to write such a
chain with each `else if` at the same depth, as a list of cases:

```c run
#include <stdio.h>

int main(void) {
    int score = 72;
    char grade;
    if (score >= 90)
        grade = 'A';
    else if (score >= 80)
        grade = 'B';
    else if (score >= 70)
        grade = 'C';
    else
        grade = 'F';
    printf("%d -> %c\n", score, grade);
    return 0;
}
```

The conditions are tested in order, from the top, and the first one that
is true wins: its statement is carried out, and the rest of the chain is
skipped. The final `else` catches every case that no condition matched.

Step through it and count the condition steps. There are three. The first
asks whether 72 is at least 90, and the answer is no, so the program goes
on to the next `else if`. The second answers no as well. The third answers
yes, `grade` becomes `'C'`, and the program jumps straight past the rest of
the chain to the `printf`. In the flow you can see the three decisions
stacked one under the other, with a red "no" leading from each to the next
and the green "yes" leading out to the assignment.

The order of the conditions matters. A score of 95 is also at least 80 and
at least 70; it gets an A only because `score >= 90` is tested first.

### Which `if` does an `else` belong to?

Look at this fragment:

```c
if (x != 0)
    if (y > 0)
        printf("both\n");
else
    printf("x is zero\n");
```

The indentation suggests that the `else` belongs to the first `if`. But
the compiler does not read indentation. Its rule is that an `else` belongs
to the nearest `if` that does not have one yet, which here is the second.
So when `x` is 0, nothing at all is printed. This is called the *dangling
else* problem, and the cure is simple: when `if` statements are nested, use
braces, and the question cannot arise.

### The conditional expression

For a choice between two *values*, rather than two statements, C has a
compact operator, the only one that takes three operands:

```c
int max = a > b ? a : b;
```

Read it as "if `a > b`, then `a`, otherwise `b`". The condition before the
`?` is tested, and the whole expression takes the value of either the part
before the `:` or the part after it. It is handy in small places like this;
for anything longer, an `if` is easier to read.

### Check yourself

1. With `score = 95`, how many conditions are tested?
   <details><summary>Answer</summary>

   One. `score >= 90` is true, and once an `if` in the chain has been
   chosen, none of the conditions after it is looked at.
   </details>

2. What does `printf("%s\n", n == 1 ? "item" : "items");` print when `n` is
   3?
   <details><summary>Answer</summary>

   `items`. The condition is false, so the expression takes the value after
   the colon.
   </details>

### A common mistake

Writing `=` where `==` was meant:

```c run
#include <stdio.h>

int main(void) {
    int x = 3;
    if (x = 5)
        printf("x is five\n");
    printf("x is now %d\n", x);
    return 0;
}
```

`x = 5` is not a comparison but an assignment. It puts 5 into `x`, and its
value, 5, is not zero, so the condition is always true. Step to line 5 and
watch `x` turn amber: the "test" has changed the variable. Many
programmers write a constant on the left, `if (5 == x)`, precisely so that
a missing `=` becomes an error.

## 3.3 `switch`

An `if` chain that compares one value against many constants, such as a
month number against 1, 2, 3 and so on, is long and repetitive. For this
situation C has the `switch` statement.

```c run
#include <stdio.h>

int main(void) {
    int month = 4;
    int days;
    switch (month) {
    case 2:
        days = 28;
        break;
    case 4: case 6: case 9: case 11:
        days = 30;
        break;
    default:
        days = 31;
        break;
    }
    printf("month %d has %d days\n", month, days);
    return 0;
}
```

The value in parentheses after `switch` is worked out once. Then the
program jumps to the `case` label whose constant is equal to it. If no
label matches, it jumps to `default`, and if there is no `default`, it
skips the whole `switch`. Several labels may stand before the same
statements, as `case 4: case 6: case 9: case 11:` does here, so that all
of those months share one piece of code.

The labels are only places to jump to, not walls. Once the program has
jumped to a label, it carries on from there, through the statements of
the following cases as well, until it meets a `break`, which leaves the
`switch`. That is why every case above ends with `break`.

Step through the program. The `switch` is a single step: it shows the
value, 4, and the label it chose. The very next step is already inside
`case 4`, and after `days = 30` the `break` takes the program to the
`printf`.

The constants after `case` must really be constants: numbers, characters
such as `'a'`, or names made with `#define`. A `case` cannot test a
variable or a range.

### Check yourself

1. Which months get 31 days in this program?
   <details><summary>Answer</summary>

   All the months not listed: 1, 3, 5, 7, 8, 10 and 12, caught by
   `default`. (A program that also handled wrong month numbers would need
   a separate check.)
   </details>

2. Could the `switch` be written as an `if` chain?
   <details><summary>Answer</summary>

   Yes: `if (month == 2) … else if (month == 4 || month == 6 || …) … else
   …`. A `switch` says the same more clearly when one value is compared with
   many constants.
   </details>

### A common mistake

Forgetting a `break`:

```c run
#include <stdio.h>

int main(void) {
    int level = 2;
    switch (level) {
    case 1:
        printf("one ");
    case 2:
        printf("two ");
    case 3:
        printf("three ");
    }
    printf("\n");
    return 0;
}
```

It prints `two three`. The program jumped to `case 2`, printed, and then
simply went on into `case 3`, because nothing told it to stop. This is
called *falling through*. Occasionally it is what you want; when it is,
write a comment saying so, because the next reader will otherwise assume
that a `break` was forgotten.

## 3.4 `while` and `do`

A loop repeats a statement, its *body*, for as long as a condition holds.
We met `while` in module 1; now let us look at it more closely, and at its
sibling, `do`.

### `while`: test first

```c
while (condition)
    body
```

The condition is tested before every round, including the first. So if the
condition is false from the start, the body is never carried out at all.
That is usually exactly right: a loop that adds up the numbers in the input
should do nothing if there are no numbers.

Here is such a loop. It reads numbers until `scanf` can no longer read
one, and adds them up:

```c run input="8 15 4 23\n"
#include <stdio.h>

int main(void) {
    int n, sum = 0, count = 0;
    while (scanf("%d", &n) == 1) {
        sum += n;
        count++;
    }
    printf("%d numbers, sum %d\n", count, sum);
    return 0;
}
```

The condition uses what `scanf` gives back, the number of values it
filled. As long as it manages to read a number, it returns 1 and the loop
goes on. When the input runs out, `scanf` cannot read anything, it returns
a different value, and the loop ends. Step through it and watch the
conditions: each "yes" is a number read, and the last condition, the one
that says "no", is the moment the input was found to be empty.

### `do`: test last

Sometimes the body must run at least once, whatever happens. For that
there is the `do` statement, which tests its condition after each round
instead of before:

```c
do
    body
while (condition);
```

Counting the digits of a number is a good example. Every number has at
least one digit, even 0, so we take one digit off and count it before we
ask whether any are left:

```c run
#include <stdio.h>

int main(void) {
    int n = 4096;
    int digits = 0;
    do {
        n /= 10;
        digits++;
    } while (n > 0);
    printf("%d digits\n", digits);
    return 0;
}
```

Dividing by 10 drops the last digit: 4096 becomes 409, then 40, then 4,
then 0. Watch `n` shrink in the memory picture while `digits` counts up.
The condition is tested four times, and only the fourth test, `0 > 0`,
says no. With a `while` loop, the number 0 would have been counted as
having no digits at all.

### Loops that never end

If nothing in the body ever makes the condition false, the loop goes on for
ever. Usually that is a mistake, but sometimes it is meant: a program that
serves requests waits for the next one, for ever. It is then written
honestly as `while (1)` or `while (true)`, and somewhere in the body a
`break` or a `return` leaves the loop when the time comes (lesson 3.6).

### Check yourself

1. How many times does the body of `while (x < 0) x++;` run when `x` is 5?
   <details><summary>Answer</summary>

   None. The condition is false at the very first test.
   </details>

2. What does the digit-counting program print for `n = 0`?
   <details><summary>Answer</summary>

   `1 digits`. The body runs once before the first test.
   </details>

### A common mistake

Changing the wrong variable in the body. In `while (i < 10) { sum += i; }`
nothing changes `i`, so the loop never ends. Vizu stops such a program
after 5000 steps, on the line of the loop that was still running, and asks
whether it is a loop that never ends.

## 3.5 The `for` statement

We met `for` in module 1. It is the loop C programmers write most often,
because most loops count: through the numbers from 1 to 100, through the
elements of an array, through the characters of a string. A `for` puts the
three parts of a counting loop side by side, where the reader sees them at
once.

```c
for (start; condition; change)
    body
```

is the same as

```c
start;
while (condition) {
    body
    change;
}
```

### The usual forms

A handful of patterns cover almost every counting loop, and it is worth
knowing them by sight:

```c
for (i = 0; i < n; i++)      /* counts 0, 1, …, n-1: n rounds */
for (i = 1; i <= n; i++)     /* counts 1, 2, …, n:   n rounds */
for (i = n - 1; i >= 0; i--) /* counts down from n-1 to 0 */
```

The first is the one you will see most, because C counts the elements of
an array from 0, as module 4 explains.

Each of the three parts may be left out. Without a start or a change, the
loop simply does not do that part. Without a condition, the condition
counts as always true, so `for (;;)` is a loop that never ends by itself,
another way of writing `while (1)`.

The counter can be declared in the `for` itself, as in `for (int i = 0; …)`.
It then exists only inside the loop, which is usually what you want.

### Two counters at once

The *comma operator* lets one part of a `for` do two things. `a, b` works
out `a`, then `b`, and takes the value of `b`. In a `for` it is used to
start or change two variables together. This loop walks two counters
towards each other:

```c run
#include <stdio.h>

int main(void) {
    for (int i = 0, j = 10; i < j; i++, j--)
        printf("%d-%d ", i, j);
    printf("\n");
    return 0;
}
```

Watch `i` and `j` in the memory: on each round, the change part makes both
of them amber at once, one going up and the other down, until they meet.

### Loops inside loops

The body of a loop can be another loop. The inner loop then runs all its
rounds for every single round of the outer one. This program prints a
small multiplication table:

```c run
#include <stdio.h>

int main(void) {
    for (int row = 1; row <= 3; row++) {
        for (int col = 1; col <= 4; col++)
            printf("%4d", row * col);
        printf("\n");
    }
    return 0;
}
```

Count the condition steps as you go. The inner condition is tested five
times for each row (four yeses and a no), and the outer one four times in
all (three yeses and a no). Notice also that `col` is created anew for each
row and disappears at the end of it.

### Check yourself

1. How many times does `for (int i = 10; i > 0; i -= 3)` run its body?
   <details><summary>Answer</summary>

   Four times, with `i` equal to 10, 7, 4 and 1. Then `i` becomes −2, and
   the test fails.
   </details>

2. How many times is `printf("%4d", …)` called in the multiplication
   table?
   <details><summary>Answer</summary>

   Twelve: four columns in each of three rows.
   </details>

### A common mistake

Writing `<=` where `<` was meant, or the other way round. The loop then
runs one round too many or one too few, an *off-by-one error*. When you
write a loop, check it at its edges: what happens on the first round, and
what happens on the last?

## 3.6 Leaving a loop early

A loop's condition is tested at one place, at the top (or, in a `do`, at
the bottom). Sometimes the natural place to decide that a loop is finished
is in the middle of its body. For that C has `break`, and for skipping just
the rest of one round it has `continue`.

`break` leaves the innermost loop at once, and the program carries on
after it. (We have already seen `break` leave a `switch`; it does the same
for loops.) `continue` skips the rest of the body for this round only: the
loop goes on to its next test, and in a `for`, to its change part first.

```c run input="5 -3 8 0 4\n"
#include <stdio.h>

int main(void) {
    int n, sum = 0;
    for (;;) {
        scanf("%d", &n);
        if (n == 0)
            break;
        if (n < 0)
            continue;
        sum += n;
    }
    printf("sum of the positives: %d\n", sum);
    return 0;
}
```

This program adds up the positive numbers it reads, stops at the first 0,
and ignores negative numbers. Step through it. When −3 is read, the
`continue` sends the program straight back to the top of the loop, and
`sum` is not touched. When 0 is read, the `break` leaves the loop for good,
so the 4 that follows in the input is never read at all.

### `goto`

C also has a statement that jumps anywhere within a function: `goto
label;`, where `label` names a statement, as in `done: printf(…);`. In the
early days of programming, programs were full of jumps like this, and they
became very hard to follow; the loops and `if` statements we have seen were
invented to replace them. In modern C, `goto` is used rarely, mostly to
leave several nested loops at once, which a `break` cannot do.

### The empty statement

A semicolon on its own is a statement that does nothing, the *empty
statement*. Occasionally that is useful: a loop whose whole work is done in
its condition needs an empty body. But it is far more often an accident.

### Check yourself

1. What does the program above print?
   <details><summary>Answer</summary>

   `sum of the positives: 13`, that is, 5 + 8.
   </details>

2. Inside two nested loops, what does a `break` in the inner loop leave?
   <details><summary>Answer</summary>

   Only the inner loop. The outer loop goes on with its next round. To leave
   both, use a variable that the outer loop checks, a `return`, or a
   `goto`.
   </details>

### A common mistake

A semicolon after the parentheses of a loop:

```c run
#include <stdio.h>

int main(void) {
    int i;
    for (i = 0; i < 3; i++);
    {
        printf("i is %d\n", i);
    }
    return 0;
}
```

The semicolon is an empty statement, and it is the whole body of the loop.
The loop counts `i` up to 3, doing nothing each time; then the block in
braces, which is not part of the loop at all, runs once and prints `i is
3`. Step through it and you will see the loop go round three times without
ever visiting line 7.

## 3.7 Reading text: copying and counting

With loops and conditions we can now write small but genuinely useful
programs. One family of them reads text, one character at a time, and
does something with each character. They are built on `getchar` and on a
loop that goes on until the input runs out.

### Copying the input

The simplest such program copies its input to its output, unchanged:

```c run input="Hi there\n"
#include <stdio.h>

int main(void) {
    int c;
    while ((c = getchar()) != EOF)
        putchar(c);
    return 0;
}
```

The condition of the loop does three things in one line, and it is worth
taking apart, because you will see it in a great many C programs.
`getchar()` reads the next character. `c = getchar()` stores it in `c`; and
since an assignment is an expression whose value is the value assigned,
the whole of `(c = getchar())` has the character as its value. That value
is then compared with `EOF`. So the loop reads a character, keeps it, and
checks whether it really was a character or the end of the input, all
before the body runs.

The parentheses around the assignment are necessary. `!=` binds more
tightly than `=`, so without them `c = getchar() != EOF` would mean
`c = (getchar() != EOF)`, and `c` would receive the result of the
comparison, 1 or 0, instead of the character.

### Counting lines, words and characters

Now something more interesting: a program that counts the lines, words and
characters of its input. Counting lines is easy, since each line ends with
a newline character. Counting words takes a little thought. Let us say
that a word is a run of characters with no space, tab or newline in it. To
count words, the program remembers whether it is currently inside a word or
outside one. Each time it moves from outside to inside, a new word has
begun.

```c run input="the quick brown\nfox jumps\n"
#include <stdio.h>

#define IN  1   /* inside a word */
#define OUT 0   /* outside a word */

int main(void) {
    int c, lines = 0, words = 0, chars = 0;
    int state = OUT;
    while ((c = getchar()) != EOF) {
        chars++;
        if (c == '\n')
            lines++;
        if (c == ' ' || c == '\n' || c == '\t')
            state = OUT;
        else if (state == OUT) {
            state = IN;
            words++;
        }
    }
    printf("%d %d %d\n", lines, words, chars);
    return 0;
}
```

A variable like `state`, which remembers where the program is in its work,
is a very common tool. Here it has two possible values, given names with
`#define` so that the code reads well.

Before you step through it, try to predict the three numbers it prints.
Then watch `state` in the memory: it flips to 1 at the first letter of
each word and back to 0 at the space or newline after it, and `words` turns
amber only at the moments when `state` flips from 0 to 1. Watch the strip
at the bottom as well: `getchar` takes its characters one at a time out of
stdin's buffer, and only when the buffer is empty does the library ask the
kernel for more.

### Check yourself

1. What does the counting program print for its input?
   <details><summary>Answer</summary>

   `2 5 26`: two newlines, five words, and 16 + 10 characters (each line's
   newline counts as a character).
   </details>

2. What would change if the words were separated by two spaces instead of
   one?
   <details><summary>Answer</summary>

   Only the count of characters. At the second space, `state` is already
   `OUT`, so nothing else happens.
   </details>

### A common mistake

Keeping the character in a `char`: `char c; while ((c = getchar()) !=
EOF)`. `EOF` is a value chosen to be different from every character, and a
`char` cannot hold it. Depending on the computer, the loop then either
never ends or ends early at a perfectly ordinary character. Keep `c` an
`int`.
