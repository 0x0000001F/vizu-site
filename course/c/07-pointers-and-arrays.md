# Module 7 · Pointers and arrays

Read alongside: King chapter 12 · K&R sections 5.3–5.4 and 5.7–5.9

In C, arrays and pointers are so closely tied that K&R discuss them together.
The tie is one rule: adding 1 to a pointer moves it to the next element. Once
you can see an arrow slide along a row of cells, the rest follows.

## 7.1 Pointer arithmetic

**Read:** King 12.1 · K&R 5.4
**You will:** move a pointer along an array and measure distances with it.

If `p` points to an element of an array, `p + 1` points to the next one,
whatever the element's size: the compiler adds `sizeof *p` bytes to the
address. Subtracting two pointers into the same array gives the number of
elements between them; comparing them tells which comes first.

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

**Watch:** at line 8 the arrow from `p` slides one cell to the right, and its
address goes up by 4, not by 1.

### Check yourself

1. If `p` is a `double *` at address 1000, where is `p + 2`?
   <details><summary>Answer</summary>

   1016: two `double`s of 8 bytes.
   </details>

2. Is `p + 5` allowed for a 5-element array? And `*(p + 5)`?
   <details><summary>Answer</summary>

   `p + 5`, one past the end, may be computed and compared (loops use it as
   the end marker). Reading or writing through it is not.
   </details>

### Common mistake

Subtracting or comparing pointers into *different* arrays. The result means
nothing; the standard leaves it undefined.

## 7.2 `a[i]` is `*(a + i)`

**Read:** King 12.3 · K&R 5.3
**You will:** see what an array name means in an expression.

In almost every expression, the name of an array turns into a pointer to its
first element: `a` means `&a[0]`. And `a[i]` is defined as `*(a + i)`. So
indexing is pointer arithmetic, and a pointer can be indexed like an array.

The two exceptions: `sizeof a` is the size of the whole array, and `&a` is the
address of the whole array. And an array name is not a variable: `a++` and
`a = p` do not compile.

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

**Watch:** `pa` is an 8-byte box with an arrow; `a` is the row of cells
itself. After line 8, `pa[0]` is the second cell.

### Check yourself

1. Predict the output.
   <details><summary>Answer</summary>

   `3 3 3 3`, then `16 8`, then `2`.
   </details>

2. Why does `2[a]` compile, and what is it?
   <details><summary>Answer</summary>

   `2[a]` is `*(2 + a)`, which is `*(a + 2)`: `a[2]`. A curiosity; never write
   it.
   </details>

### Common mistake

Believing "an array is a pointer". It is not: it has no box holding an
address, and `sizeof` sees its full size. It only *turns into* a pointer when
used in an expression or passed to a function (lesson 5.3).

## 7.3 Walking an array with a pointer

**Read:** King 12.2 · K&R 5.4–5.5
**You will:** read the `*p++` idiom.

`*p++` means `*(p++)`: use the value `p` points to, then move `p` on. It is
the heart of countless C loops. A pointer one past the end makes a clean stop
condition.

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

**Watch:** `end`'s arrow points just past the last cell and stays there; `p`'s
arrow walks up to meet it, reading one cell per step.

King makes a useful point here: on a modern compiler this is not faster than
`a[i]`. The two give the same machine code. Choose the clearer one.

### Check yourself

1. What is the difference between `*p++`, `(*p)++` and `*++p`?
   <details><summary>Answer</summary>

   `*p++`: value, then move the pointer. `(*p)++`: add one to the value, the
   pointer stays. `*++p`: move first, then the value.
   </details>

2. Why is `end` a `const int *` too?
   <details><summary>Answer</summary>

   It points into the same read-only view of the array as `p`; a plain
   `int *` would need a cast to drop the `const`.
   </details>

### Common mistake

`while (*p++ != 0)` on an array with no zero in it: nothing stops the pointer
at the end.

## 7.4 Pointers into two dimensions

**Read:** King 12.4 · K&R 5.7–5.9
**You will:** walk a 2-D array as one row, and point to a whole row.

A 2-D array is stored row after row (lesson 4.4), so an `int *` can walk all
of it from the first element to the last. `m[i]` is a row, an array of 4
`int`s, so a pointer to a row has the type "pointer to array of 4 `int`":
`int (*row)[4]`. The parentheses matter: `int *row[4]` is an array of four
pointers.

```c run
#include <stdio.h>

int main(void) {
    int m[3][4] = {
        {1, 2, 3, 4},
        {5, 6, 7, 8},
        {9, 10, 11, 12},
    };
    int sum = 0;
    for (int *p = &m[0][0]; p <= &m[2][3]; p++)
        sum += *p;
    int (*row)[4] = &m[1];      /* a pointer to a whole row */
    printf("sum %d, row 1 starts with %d\n", sum, (*row)[0]);
    return 0;
}
```

**Watch:** one arrow crosses from row 0 into row 1 without a jump: the rows
touch in memory.

### Check yourself

1. If `row` points to `m[1]`, where does `row + 1` point?
   <details><summary>Answer</summary>

   To `m[2]`: 16 bytes further, a whole row of four `int`s.
   </details>

2. K&R contrast `char *names[]` with `char names[][15]`. What is the
   difference in memory?
   <details><summary>Answer</summary>

   The first is an array of pointers to strings stored elsewhere, each as long
   as it needs; the second is one block of rows of 15 characters each,
   whatever the names' lengths. Lesson 8.5 draws both.
   </details>

### Common mistake

Passing a 2-D array to a function declared `void f(int **m)`. A 2-D array is
not an array of pointers; the function must take `int m[][4]` (or
`int (*m)[4]`), with the row length written in.
