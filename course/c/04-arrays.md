# Module 4 · Arrays

Suppose a program has to read the marks of thirty students, work out the
average, and then print every mark that is below it. With what we know so
far, it would need thirty variables, `mark1`, `mark2`, and so on to
`mark30`, thirty `scanf` calls, and thirty `if` statements. That is
clearly not the way. We need a single name for the whole group of values,
and a way to say "the fifth one" or "the i-th one" where `i` is a variable.

That is what an *array* gives us. An array is a row of boxes of the same
type, side by side in memory, under one name. The boxes are called the
array's *elements*, and each is reached by its position, its *index*.

"Side by side" is the important part, and this module keeps coming back to
it. Because the elements follow each other in memory without gaps, the
computer can find any element with one quick calculation, however long the
array is. And for the same reason, if a program asks for an element one
place beyond the end, it gets the box that happens to lie there, which
belongs to something else. C does not stop it. Seeing that happen, and
learning not to let it happen, is one of the most useful things this
course can teach.

## 4.1 Arrays are rows of boxes

An array is declared like a variable, with the number of elements in
square brackets after the name:

```c
int scores[5];
```

This asks for five `int` boxes in a row and calls the row `scores`. The
elements are numbered from 0, not from 1: they are `scores[0]`,
`scores[1]`, `scores[2]`, `scores[3]` and `scores[4]`. There is no
`scores[5]`. Writing an index in square brackets after the array's name is
called *subscripting*, and an element reached this way can be used exactly
like an ordinary variable: read, assigned, incremented, or given to
`scanf` with an `&`.

### Why from 0?

Counting from 0 looks odd at first, but it comes straight from how the
computer finds an element. The array's name stands for the address where
the array begins. Element number `i` lies `i` elements further on. So if
`scores` begins at address 1000 and an `int` takes 4 bytes, then
`scores[0]` is at 1000 + 0 × 4, `scores[1]` at 1000 + 1 × 4, and
`scores[i]` at 1000 + i × 4. The index is simply the distance from the
start, counted in elements, and the first element is no distance at all
from the start. This one multiplication and addition is all it takes to
find any element, which is why arrays are so fast.

### Walking an array

The index can be any integer expression, and that is what makes arrays
powerful: a loop can visit every element in turn.

```c run
#include <stdio.h>

int main(void) {
    int scores[5] = {90, 75, 60, 85, 70};
    int sum = 0;
    for (int i = 0; i < 5; i++)
        sum += scores[i];
    printf("average: %.1f\n", sum / 5.0);
    return 0;
}
```

The braces after `=` give the elements their first values, in order. The
loop is the most common loop in C: `i` runs from 0 while it is less than
the length, so it takes the values 0 to 4, exactly the indexes of the
array.

Step through the program and watch the memory. The array is drawn as five
boxes, one under the other, each with its own address. Look at the
addresses: neighbouring elements are exactly 4 apart, the size of an `int`.
On each round of the loop, the element being read is marked, `[0]` first,
then `[1]`, and so on down the array, while `sum` turns amber as it grows.

### Reading into an array, and reading it back

Here is something that was impossible without arrays: reading a series of
numbers and printing them in the opposite order. The program cannot print
anything until it has seen the last number, so it must keep all of them.

```c run input="34 82 49 102 7\n"
#include <stdio.h>

#define N 5

int main(void) {
    int numbers[N];
    printf("Enter %d numbers: ", N);
    for (int i = 0; i < N; i++)
        scanf("%d", &numbers[i]);
    printf("In reverse order:");
    for (int i = N - 1; i >= 0; i--)
        printf(" %d", numbers[i]);
    printf("\n");
    return 0;
}
```

Two details deserve attention. The length is given a name with `#define`,
so that it is written only once; if the program should handle ten numbers,
one line changes. And `scanf` is given `&numbers[i]`, the address of one
element, because it writes a single number into a single box, just as it
did into an ordinary variable.

Step through the first loop and watch the elements fill one by one, each
turning amber as `scanf` writes into it. Then the second loop reads them
back from the last to the first.

### Check yourself

1. What are the indexes of an array declared as `double t[24];`?
   <details><summary>Answer</summary>

   0 to 23. An array of n elements has indexes 0 to n − 1.
   </details>

2. If `scores[0]` is at address 1000, where is `scores[3]`?
   <details><summary>Answer</summary>

   At 1012: three `int`s of 4 bytes past the start.
   </details>

### A common mistake

Writing the length in two places, as in `int a[5];` and `for (i = 0; i <
5; i++)`, and then changing only one of them. Give the length a name, or
compute it as the next lesson shows.

## 4.2 Giving an array its values

An array can be given its first values in its declaration, with an
*initializer*: a list of values in braces, separated by commas.

```c
int primes[5] = {2, 3, 5, 7, 11};
```

The initializer may be shorter than the array. The values then go into the
first elements, and all the remaining elements are set to zero. This gives
a short way to clear a whole array: `int counts[100] = {0};` sets the first
element to 0 explicitly and the other ninety-nine to 0 by the rule.

If an initializer is given, the length may be left out, and the compiler
counts the values: `int primes[] = {2, 3, 5, 7, 11};` has five elements.

Sometimes only a few elements need values, scattered through a large
array. For that you can say which element gets which value, with a
*designated initializer*: `int a[10] = {[2] = 5, [7] = 1};` sets `a[2]` to
5 and `a[7]` to 1, and every other element to 0.

### How long is an array?

`sizeof` applied to an array gives the size of the whole array in bytes.
Dividing that by the size of one element gives the number of elements:

```c run
#include <stdio.h>

int main(void) {
    int primes[] = {2, 3, 5, 7, 11, 13};
    int a[10] = {[2] = 5, [7] = 1};
    int n = sizeof primes / sizeof primes[0];
    printf("primes takes %zu bytes and has %d elements\n", sizeof primes, n);
    for (int i = 0; i < n; i++)
        printf("%d ", primes[i]);
    printf("\n%d %d %d\n", a[0], a[2], a[7]);
    return 0;
}
```

`primes` has six `int`s of 4 bytes, so `sizeof primes` is 24, and 24
divided by 4 is 6. The great advantage of computing the length this way is
that it stays right when you add values to the initializer.

Look at `a` in the memory picture: ten boxes, all holding 0 except the two
that the designated initializer filled.

### Arrays that do not change

If an array holds values that should never change, such as the names of
the months or the number of days in each, declare it `const`:

```c
const int days_in_month[12] = {31, 28, 31, 30, 31, 30, 31, 31, 30, 31, 30, 31};
```

The compiler then refuses any statement that tries to change an element.
The word documents the intention, and it lets the compiler catch a mistake
for you.

### Arrays whose length is known only while running

The length of an array is usually a constant. Since 1999, C also allows a
length that is worked out while the program runs, as in `int v[n];`, where
`n` is a variable. Such a *variable-length array* is useful when the size
depends on the input, but it lives on the stack, which is not large, so
it is only safe for modest sizes. Module 10 shows the usual way to get
memory of any size while a program runs.

### Check yourself

1. What does `int a[5] = {1, 2};` contain?
   <details><summary>Answer</summary>

   1, 2, 0, 0, 0. Elements without a value in the initializer become 0.
   </details>

2. Why does `sizeof primes / sizeof primes[0]` work for an array of any
   type?
   <details><summary>Answer</summary>

   The whole array's size divided by one element's size is the number of
   elements, whatever that size is.
   </details>

### A common mistake

Expecting an array without an initializer to start at zero. `int c[10];`
inside a function holds whatever bits were left in that memory. Only an
initializer, even just `= {0}`, clears it.

## 4.3 Past the end

What happens if a program uses an index that is too large, or negative? In
many languages the program is stopped with an error message. C does
nothing of the kind. It does not check indexes at all, ever. It calculates
the address exactly as it always does, start plus index times size, and
reads or writes whatever is at that address.

That address is not empty. Memory is one long row of bytes, and the bytes
after an array belong to something: another variable, information the
program keeps about function calls, or memory that the program is not
allowed to touch at all. Reading there gives a meaningless number. Writing
there quietly changes some other variable, and the program goes wrong
later, somewhere else, in a way that seems to have nothing to do with the
array. Such bugs are among the hardest to find.

```c run expect="error"
#include <stdio.h>

int main(void) {
    int scores[4] = {90, 75, 60, 85};
    int total = 0;
    for (int i = 0; i <= 4; i++)
        total += scores[i];
    printf("%d\n", total);
    return 0;
}
```

The array has four elements, indexes 0 to 3, but the loop runs while
`i <= 4`, so its last round reads `scores[4]`. On a real computer that read
would succeed, and `total` would simply be wrong.

Vizu is stricter than C. It remembers which variable every byte of memory
belongs to, so it notices the moment a program reaches past the end of an
array, stops there, and says which array and which index. Step to the end
of the run and look at the memory: just past `scores[3]` there are four
bytes drawn with a red question mark. Those are the bytes that `scores[4]`
would be. They exist, memory is there, but they do not belong to `scores`.

### Check yourself

1. Which single character in the program is the bug?
   <details><summary>Answer</summary>

   The `=` in `i <= 4`. With `i < 4`, the loop stops after `scores[3]`.
   </details>

2. Why would the real program usually not crash?
   <details><summary>Answer</summary>

   The bytes after the array usually belong to the same program, so reading
   them is allowed by the computer. The read succeeds and returns whatever
   happens to be there, and nothing tells you that it was wrong.
   </details>

### A common mistake

Trusting that a crash would warn you. Most out-of-bounds bugs do not crash;
they just produce wrong results. That is why tools exist that watch every
memory access, such as the compiler option `-fsanitize=address` and the
program Valgrind, and it is why Vizu stops you.

## 4.4 Counting with an array

Here is a different way to use an array: as a set of counters, where the
index says what is being counted. To count how many times each of the ten
digits appears in a text, we keep ten counters, one per digit, in an array
of ten elements. When the program reads the digit 7, it adds one to
counter number 7.

The only question is how to turn the character `'7'` into the number 7.
Lesson 2.4 gave the answer: the digits have consecutive codes, so
`c - '0'` is the value of the digit `c`.

```c run input="2026: 9 cats, 3 dogs, 99 bottles\n"
#include <stdio.h>

int main(void) {
    int c, ndigit[10] = {0};
    int nother = 0;
    while ((c = getchar()) != EOF) {
        if (c >= '0' && c <= '9')
            ndigit[c - '0']++;
        else
            nother++;
    }
    printf("digits:");
    for (int i = 0; i < 10; i++)
        printf(" %d", ndigit[i]);
    printf(", other: %d\n", nother);
    return 0;
}
```

The condition `c >= '0' && c <= '9'` decides whether the character is a
digit at all. Only then is it used as an index. Without that test, a
letter such as `'c'` would give `c - '0'` = 51, and the program would add one
to a counter far beyond the end of the array.

Step through it and watch the array. Each time a digit is read, the
counter that belongs to it turns amber: the 2 goes to `ndigit[2]`, the 0 to
`ndigit[0]`, and so on. Every other character goes to `nother`.

The same idea works for many problems. To find out whether a number has a
repeated digit, keep an array of ten `bool`s, one per digit, all false at
first. Take the number apart digit by digit, as in lesson 3.4; for each
digit, if its `bool` is already true, the digit is repeated, and otherwise
set its `bool` to true.

### Check yourself

1. What is in `ndigit[9]` at the end?
   <details><summary>Answer</summary>

   3: the 9 in "9 cats" and the two in "99".
   </details>

2. What would `ndigit[c - '0']++` do if the input contained a minus sign?
   <details><summary>Answer</summary>

   Nothing bad, because the test keeps it out: `'-'` is not between `'0'`
   and `'9'`. Without the test it would give a negative index, `'-' - '0'`
   is −3, which is just as wrong as one past the end.
   </details>

### A common mistake

Forgetting to start the counters at zero. `int ndigit[10];` inside a
function is full of leftover bits, and adding one to leftover bits gives
nonsense. `= {0}` clears all ten.

## 4.5 Two dimensions, one row of memory

Many things come in tables: a chessboard, the seats in a cinema, a
spreadsheet, the pixels of an image. C lets you declare an array with two
indexes:

```c
int grid[2][3];
```

This is a table of 2 rows and 3 columns, and its elements are written
`grid[row][column]`. In fact it is an array of arrays: `grid` is an array of
2 elements, and each of those elements is itself an array of 3 `int`s.

But memory is not a table. It is one long row of bytes, each with the next
number as its address. So C has to lay the table out in a line, and it does
so row by row: first all of row 0, then all of row 1. This is called
*row-major* order. `grid[1][0]` comes right after `grid[0][2]`.

```c run
#include <stdio.h>

int main(void) {
    int grid[2][3] = {
        {1, 2, 3},
        {4, 5, 6},
    };
    int sum = 0;
    for (int r = 0; r < 2; r++)
        for (int c = 0; c < 3; c++)
            sum += grid[r][c];
    printf("sum %d, size %zu bytes\n", sum, sizeof grid);
    return 0;
}
```

The initializer is written as a list of rows, each row in its own braces,
which makes the table easy to see in the program text. Two nested loops
visit the elements: the outer loop chooses a row, the inner loop walks
along it.

Step through it and compare the picture with the addresses. Vizu draws the
two rows separately so the table is easy to read, but the addresses tell
the truth: the first element of row 1 comes immediately after the last
element of row 0, 4 bytes on. To find `grid[r][c]`, the computer works out
r × 3 + c, the element's position in the one long row, and then proceeds
exactly as for an ordinary array.

### Check yourself

1. How many bytes lie between the start of `grid[0][0]` and the start of
   `grid[1][2]`?
   <details><summary>Answer</summary>

   20. `grid[1][2]` is element 1 × 3 + 2 = 5 of the long row, and five
   `int`s take 20 bytes.
   </details>

2. Why does it matter, on a real computer, that the loop over `r` is
   outside and the loop over `c` inside?
   <details><summary>Answer</summary>

   That order reads memory in the order it is laid out, which is what the
   processor's cache is built for. With the loops swapped, each step jumps a
   whole row ahead; on large tables that can be several times slower. The
   track on the machine comes back to this.
   </details>

### A common mistake

Writing `grid[1, 2]` for `grid[1][2]`. Inside the brackets, the comma is
the comma operator of lesson 3.5: it throws the 1 away and leaves 2, so
the expression means `grid[2]`, a row that does not exist.
