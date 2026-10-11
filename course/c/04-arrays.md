# Module 4 · Arrays

Read alongside: King chapter 8 · K&R sections 1.6 and 5.7

An array is many boxes of one type, side by side in memory, under one name.
Side by side is the important part: it is why indexing is fast, and why going
one past the end lands in someone else's box.

## 4.1 Arrays are rows of boxes

**Read:** King 8.1 · K&R 1.6
**You will:** declare, fill and walk an array, and know where its elements
are.

`int scores[5];` asks for five `int` boxes in a row. They are numbered from 0:
`scores[0]` to `scores[4]`. An initializer fills them in order; if it is
shorter than the array, the rest are 0. `sizeof scores` is the size of the
whole array, so `sizeof scores / sizeof scores[0]` is its length.

```c run
#include <stdio.h>

int main(void) {
    int scores[5] = {90, 75, 60, 85, 70};
    int sum = 0;
    for (int i = 0; i < 5; i++)
        sum += scores[i];
    printf("average: %.1f\n", sum / 5.0);
    printf("scores takes %zu bytes\n", sizeof scores);
    return 0;
}
```

**Watch:**

- The five rows of `scores` are drawn from the start; each round, the cell
  being read gets a blue edge, `[0]` first, then `[1]` and on.
- The addresses of neighbouring cells differ by 4, the size of an `int`.

### Check yourself

1. What is in `int a[5] = {1, 2};`?
   <details><summary>Answer</summary>

   1, 2, 0, 0, 0.
   </details>

2. What is the address of `scores[3]` if `scores[0]` is at 1000?
   <details><summary>Answer</summary>

   1012: three `int`s of 4 bytes further.
   </details>

### Common mistake

Writing the length twice (`int a[5]` and `i < 5`) and changing only one. Use
a `#define`, or compute it with `sizeof`.

## 4.2 Past the end

**Read:** King 8.1 (the warning on array bounds)
**You will:** see what C does not check.

C does not check indices. `scores[4]` in a four-element array is simply the
four bytes after the array: maybe another variable, maybe nothing the program
owns. Reading it gives garbage; writing it changes some other variable, and
the bug shows up somewhere else, later.

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

Vizu remembers which variable every byte belongs to, so it stops at the first
read past the end and says which array and which index.

**Watch:** at the last step a red `?` appears in the bytes just past
`scores[3]`: the four bytes `scores[4]` would be. They are drawn from the
start, because memory is there; they just do not belong to `scores`.

### Check yourself

1. Which character in the program is the bug?
   <details><summary>Answer</summary>

   The `=` in `i <= 4`. The loop must stop before 4.
   </details>

2. Why would a real program often *not* crash here?
   <details><summary>Answer</summary>

   The bytes after the array usually belong to the program (another variable,
   the stack frame), so the read succeeds and returns whatever is there.
   Nothing tells you it was wrong.
   </details>

### Common mistake

Thinking a crash would tell you. Out-of-bounds bugs mostly do not crash; they
give wrong numbers. Tools like Vizu, `-fsanitize=address` and Valgrind exist
because of this.

## 4.3 Counting with an array

**Read:** K&R 1.6 · King 8.1 (repdigit)
**You will:** use a value as an index.

To count how often each digit appears, keep ten counters in an array and use
the digit itself as the index. `c - '0'` turns the character `'7'` into the
number 7 (lesson 2.3).

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

**Watch:** each digit read lights up, in amber, the counter it belongs to:
the cell `ndigit[c - '0']`.

### Check yourself

1. What does `ndigit[9]` end up as?
   <details><summary>Answer</summary>

   3: the 9 in "9 cats" and the two in "99".
   </details>

2. What would happen without the `c >= '0' && c <= '9'` check?
   <details><summary>Answer</summary>

   A letter such as `'c'` would give `c - '0'` = 51, an index far past the
   end: lesson 4.2's bug.
   </details>

### Common mistake

Forgetting to start counters at zero. `int ndigit[10];` inside a function holds
garbage; `= {0}` clears all ten.

## 4.4 Two dimensions, one row of memory

**Read:** King 8.2 · K&R 5.7
**You will:** see how a table is laid out in memory.

`int grid[2][3]` is an array of 2 rows, each an array of 3 `int`s. Memory is
one long row of bytes, so the rows are stored one after the other: all of row
0, then all of row 1 (*row-major* order).

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

**Watch:** the table is drawn as rows, but the addresses run on: the first
cell of row 1 comes right after the last cell of row 0.

### Check yourself

1. How many bytes from `grid[0][0]` to `grid[1][2]`?
   <details><summary>Answer</summary>

   20: element number 1 × 3 + 2 = 5, times 4 bytes.
   </details>

2. Why does looping `r` outside and `c` inside matter on a real machine?
   <details><summary>Answer</summary>

   It reads memory in order, which the CPU's cache is built for. Swapping the
   loops jumps a whole row at each step; on big tables it can be several
   times slower. (The CPU track comes back to this.)
   </details>

### Common mistake

`grid[1, 2]` is not `grid[1][2]`. The comma operator throws away the 1, so it
means `grid[2]`, a row that does not exist.
