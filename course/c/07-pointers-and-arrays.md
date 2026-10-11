# Module 7 · Pointers and arrays

In most languages, arrays and pointers (or references) are separate ideas.
In C they are bound together so closely that it is hard to talk about one
without the other. The connection is a single rule, which this module
builds on: if a pointer points to an element of an array, adding 1 to the
pointer makes it point to the next element.

From that one rule a great deal follows. A pointer can walk along an array
from one end to the other. The distance between two pointers into the same
array is a number of elements. And the subscript `a[i]` that we have used
since module 4 turns out to be pointer arithmetic in disguise. Once you
have watched an arrow slide along a row of boxes, all of this becomes
easy to picture.

## 7.1 Pointer arithmetic

Suppose `p` points to the first element of an array of `int`s. What is
`p + 1`? It cannot simply be the next byte, because the next byte is the
second byte of the same `int`, which is not an element at all. So C does
something cleverer: it adds one *element*. The compiler knows from the
pointer's type that an `int` takes 4 bytes, so `p + 1` is the address 4
bytes further on, `p + 2` the address 8 bytes further on, and so on. For a
pointer to `double`, each step is 8 bytes; for a pointer to `char`, 1.
That is why every pointer has a type.

Three kinds of arithmetic are allowed on pointers:

- Adding an integer to a pointer, or subtracting one from it, moves the
  pointer that many elements forward or back.
- Subtracting one pointer from another, when both point into the same
  array, gives the number of elements between them.
- Comparing two pointers into the same array, with `<`, `>` and the rest,
  tells which element comes first.

```c run
#include <stdio.h>

int main(void) {
    int a[5] = {10, 20, 30, 40, 50};
    int *p = &a[0];
    int *q = p + 3;
    printf("%d %d\n", *p, *q);
    p++;
    printf("%d\n", *p);
    printf("q - p = %ld\n", (long) (q - p));
    printf("p < q: %d\n", p < q);
    return 0;
}
```

Step through it and look at the arrows. After line 6, `p` points to `a[0]`
and `q` to `a[3]`, three elements on. On line 8, `p++` adds one to `p`, and
its arrow slides one box along, to `a[1]`. Now look at the address in `p`'s
box: it went up by 4, not by 1. Then `q - p` is 2, the number of elements
between them, not the number of bytes.

### Check yourself

1. If `p` is a `double *` holding the address 1000, what address does
   `p + 2` hold?
   <details><summary>Answer</summary>

   1016: two `double`s of 8 bytes each further on.
   </details>

2. For an array of 5 elements, is `p + 5` allowed? And `*(p + 5)`?
   <details><summary>Answer</summary>

   C allows a pointer to the place just past the last element to be
   computed and compared; loops use it to mark the end. But there is no
   element there, so reading or writing `*(p + 5)` is not allowed.
   </details>

### A common mistake

Subtracting or comparing pointers into two *different* arrays. Two arrays
lie wherever the compiler put them, so the distance between them means
nothing, and C does not define the result.

## 7.2 An array's name is a pointer to its start

Here is the second half of the connection. Whenever the name of an array
is used in an expression, it stands for a pointer to the array's first
element. If `a` is an array, then `a` in an expression means the same as
`&a[0]`.

And the subscript operator is *defined* in terms of pointers: `a[i]` means
`*(a + i)`. Read it slowly: take the pointer to the first element, move
`i` elements along, and use the box there. That is precisely the
calculation of lesson 4.1, start plus `i` times the size, now written in
C. It also means that the subscript works on any pointer, not only on an
array's name: if `p` points into an array, `p[i]` is `*(p + i)`.

```c run
#include <stdio.h>

int main(void) {
    int a[4] = {1, 2, 3, 4};
    int *pa = a;            /* the same as &a[0] */
    printf("%d %d %d %d\n", a[2], *(a + 2), pa[2], *(pa + 2));
    printf("%zu %zu\n", sizeof a, sizeof pa);
    pa++;                   /* fine: pa is a variable */
    /* a++; would not compile: an array name is not a variable */
    printf("%d\n", pa[0]);
    return 0;
}
```

The first `printf` reaches the same element four ways, and they all print
3.

### An array is not a pointer

Despite all this, an array and a pointer are different things, and the
memory picture shows the difference clearly. `a` is the row of four boxes
itself. There is no box anywhere that holds the address of `a`; the
compiler simply knows where the array is. `pa`, on the other hand, is a
box of its own, 8 bytes, holding an address, drawn with an arrow. That is
why `sizeof a` is 16, the size of the whole array, while `sizeof pa` is 8.
And it is why `pa++` is allowed, since `pa` is a variable whose value can
change, while `a++` is not: there is no variable `a` holding an address
that could be changed.

So an array *turns into* a pointer when it is used in an expression, but
it *is* not one. The two places where it does not turn into a pointer are
`sizeof a`, which sees the whole array, and `&a`, which gives the address
of the whole array.

### Array parameters, again

This explains what we saw in lesson 5.4. When an array is passed to a
function, the argument is an expression, so the array's name turns into a
pointer to its first element, and that pointer is what the function
receives. A parameter written `int a[]` is in fact a pointer parameter,
exactly the same as `int *a`; C allows the brackets only because they
suggest how the parameter is meant to be used. That is why `sizeof` inside
the function gave 8.

### Check yourself

1. What does the program print?
   <details><summary>Answer</summary>

   `3 3 3 3`, then `16 8`, then `2`.
   </details>

2. `2[a]` compiles. Why, and what does it mean?
   <details><summary>Answer</summary>

   `2[a]` means `*(2 + a)`, which is the same as `*(a + 2)`, that is,
   `a[2]`. It is a curiosity that follows from the definition. Never write
   it.
   </details>

### A common mistake

Saying "an array is a pointer" and believing it. It leads to surprises
with `sizeof`, and to code that tries to assign to an array. An array is a
row of boxes; its name only turns into a pointer when it is used.

## 7.3 Walking an array with a pointer

With pointer arithmetic, a loop can go through an array without an index
at all: a pointer starts at the first element and moves along one element
per round. C programmers write such loops all the time, especially for
strings, so it is important to be able to read them.

The usual form uses an expression that combines `*` and `++`:

```c
s += *p++;
```

Since `++` binds more tightly than `*`, `*p++` means `*(p++)`. And since
`p++` gives the old value of `p` (lesson 2.6), the `*` is applied to the
old address. So the expression reads the element that `p` points to, and
moves `p` on to the next one, both in one go.

```c run
#include <stdio.h>

int sum(const int *p, int n) {
    int s = 0;
    const int *end = p + n;
    while (p < end)
        s += *p++;
    return s;
}

int main(void) {
    int nums[5] = {3, 1, 4, 1, 5};
    printf("%d\n", sum(nums, 5));
    return 0;
}
```

`end` points to the place just past the last element, which is the
allowed position of lesson 7.1. The loop goes on while `p` has not reached
it.

Step into `sum` and watch the two arrows from its frame. `end`'s arrow
points just beyond the last box of `nums` and stays there. `p`'s arrow
starts at the first box and walks along, one box per round, reading each
value as it goes, until it meets `end` and the condition says no.

### Two pointers from both ends

Some jobs are done neatly with two pointers moving towards each other.
Reversing an array in place is one: swap the first and last elements, then
the second and the second-to-last, and so on, until the pointers meet in
the middle.

```c run
#include <stdio.h>

int main(void) {
    int a[] = {1, 2, 3, 4, 5};
    int *lo = a, *hi = a + 4;
    while (lo < hi) {
        int t = *lo;
        *lo++ = *hi;
        *hi-- = t;
    }
    for (int i = 0; i < 5; i++)
        printf("%d ", a[i]);
    printf("\n");
    return 0;
}
```

`*lo++ = *hi` stores the value at `hi` into the box at `lo`, and then
moves `lo` forward. `*hi-- = t` stores `t` at `hi`, and moves `hi` back.
Watch the two arrows close in on each other from the ends of the array,
with a pair of boxes turning amber each round.

Is walking with a pointer faster than using an index? On old compilers it
often was. A modern compiler turns both into the same machine
instructions, so choose whichever is clearer for the job at hand.

### Check yourself

1. What is the difference between `*p++`, `(*p)++` and `*++p`?
   <details><summary>Answer</summary>

   `*p++` reads the value and then moves the pointer. `(*p)++` adds one to
   the value and leaves the pointer where it is. `*++p` moves the pointer
   first and then reads the value it now points to.
   </details>

2. Why is `end` declared `const int *`, like `p`?
   <details><summary>Answer</summary>

   It is computed from `p`, which points to read-only `int`s, and the
   result has the same type. Assigning it to a plain `int *` would quietly
   throw the promise away, and the compiler would warn.
   </details>

### A common mistake

A loop that stops only when it finds a certain value, such as `while (*p++
!= 0)`, run over an array that does not contain that value. Nothing stops
the pointer at the end of the array, and it reads on into whatever comes
next.

## 7.4 Pointers into two dimensions

A two-dimensional array is stored as one long row, row after row (lesson
4.5). So an ordinary pointer to `int` can walk through all of its elements,
from the first to the last, as if it were one long array. And because each
row is itself an array, a row's name turns into a pointer to its first
element, just like any array's name: `m[1]` is a pointer to `m[1][0]`.

There is also a pointer that points to a whole row. Its type is "pointer
to an array of 4 `int`s", written `int (*row)[4]`. The parentheses are
essential. Without them, `int *row[4]` would be something quite different:
an array of four pointers. Adding 1 to a pointer to a row moves it a whole
row on, 16 bytes, because one element of `m` is one row.

Such a pointer is just what is needed to go *down* a column, which an
ordinary loop over the long row cannot do conveniently. Here is a program
that uses all three kinds of walk:

```c run
#include <stdio.h>

#define ROWS 3
#define COLS 4

int main(void) {
    int m[ROWS][COLS] = {
        {1, 2, 3, 4},
        {5, 6, 7, 8},
        {9, 10, 11, 12},
    };
    int sum = 0;
    for (int *p = &m[0][0]; p <= &m[ROWS - 1][COLS - 1]; p++)
        sum += *p;
    printf("all elements add up to %d\n", sum);

    int col_sum = 0;
    int (*row)[COLS];
    for (row = &m[0]; row < &m[ROWS]; row++)
        col_sum += (*row)[2];
    printf("column 2 adds up to %d\n", col_sum);

    for (int *p = m[1]; p < m[1] + COLS; p++)
        *p = 0;
    printf("row 1 is now %d %d %d %d\n", m[1][0], m[1][1], m[1][2], m[1][3]);
    return 0;
}
```

The first loop treats the table as one row of twelve `int`s. Watch its
arrow cross from the end of row 0 into the start of row 1 without any
jump: in memory, the rows touch.

The second loop moves `row` one whole row at a time. `*row` is the row it
points to, an array, and `(*row)[2]` is element 2 of that row. So the loop
visits 3, 7 and 11, the third column.

The third loop uses `m[1]`, the name of row 1, as a pointer to its first
element, and walks along that one row, clearing it.

### Check yourself

1. If `row` points to `m[1]`, where does `row + 1` point?
   <details><summary>Answer</summary>

   To `m[2]`, 16 bytes further on: one step is a whole row of four `int`s.
   </details>

2. What is the difference in memory between `char *names[3]` and `char
   names[3][10]`?
   <details><summary>Answer</summary>

   The first is an array of three pointers, each pointing to a string kept
   somewhere else, as long as it needs to be. The second is one block of
   three rows of ten characters each, whatever the lengths of the names.
   Lesson 8.5 draws both.
   </details>

### A common mistake

Passing a two-dimensional array to a function declared `void f(int **m)`.
A two-dimensional array is not an array of pointers; there are no pointers
in it anywhere. The parameter must say how long a row is, `int m[][COLS]`
or `int (*m)[COLS]`, so that the function can work out where each row
begins.
