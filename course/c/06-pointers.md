# Module 6 · Pointers

Read alongside: King chapter 11 · K&R sections 5.1–5.2

You have seen addresses on every box since lesson 1.2. A pointer is a box that
holds one of them. Vizu draws a pointer as an arrow from its box to the box
whose address it holds, the same picture King and K&R draw by hand, but for
every line.

## 6.1 Every box has an address

**Read:** King 11.1 · K&R 5.1
**You will:** take a variable's address and print it.

Memory is a long row of bytes, each with a number: its address. A variable
occupies one or more bytes; its address is the address of its first byte. The
`&` operator gives it.

```c run
#include <stdio.h>

int main(void) {
    int i = 7;
    double d = 2.5;
    char c = 'x';
    printf("i is at %p\n", (void *) &i);
    printf("d is at %p\n", (void *) &d);
    printf("c is at %p\n", (void *) &c);
    return 0;
}
```

`%p` prints an address, usually in hexadecimal; `printf` wants it as a
`void *`, hence the cast.

**Watch:** the printed addresses are the numbers on the boxes. `i` takes 4
bytes, `d` 8, `c` 1. On the stack, later variables often get *lower*
addresses: the stack grows downwards.

Vizu keeps addresses short and round so they are easy to follow: string
literals from `0x1000`, globals from `0x2000`, the heap from `0x4000`, the
stack downwards from `0x8000`. A real 64-bit machine prints much longer ones,
like `0x7ffd5c2e8a1c`, but the rules are the same: each byte has its own
number, and the bytes of one variable sit next to each other.

### Check yourself

1. Can you take the address of `i + 1`?
   <details><summary>Answer</summary>

   No. `&` needs something that has a box (a variable, an array element, a
   struct member). `i + 1` is a value that lives nowhere.
   </details>

2. Why does the address of the same variable change between runs on a real
   machine?
   <details><summary>Answer</summary>

   The operating system places the stack at a random address each run (ASLR),
   to make attacks harder.
   </details>

### Common mistake

Printing an address with `%d`. An address is 8 bytes on a 64-bit machine; `%d`
reads 4 of them. Use `%p`.

## 6.2 Pointers and `*`

**Read:** King 11.2 · K&R 5.1
**You will:** declare a pointer, point it at a variable, and use it.

`int *p;` declares `p` as a *pointer to int*: a box that holds the address of
an `int`. After `p = &i;`, `p` points to `i`. Then `*p` (the *indirection*
operator) means "the box at the address in `p`": while `p` points to `i`, `*p`
is another name for `i`. Reading `*p` reads `i`; writing `*p` writes `i`.

```c run
#include <stdio.h>

int main(void) {
    int i;
    int *p;
    p = &i;
    i = 1;
    printf("%d %d\n", i, *p);
    *p = 2;
    printf("%d %d\n", i, *p);
    return 0;
}
```

**Watch:**

- After line 6, an arrow goes from `p` to `i`. `i` is still `?`: `p` points to
  it, but nothing has been written yet.
- Line 9 is the moment to stop at: the write goes along the arrow, and `i`
  turns amber.

This is the sequence King draws step by step in section 11.2.

### Check yourself

1. In `int *p, q;`, what type is `q`?
   <details><summary>Answer</summary>

   `int`. The `*` belongs to `p` alone. Write one declaration per line to
   avoid the trap.
   </details>

2. What is `*&i`? And `&*p`?
   <details><summary>Answer</summary>

   `i`, and `p`. `*` and `&` undo each other.
   </details>

### Common mistake

Using a pointer that points nowhere yet:

```c run expect="error"
#include <stdio.h>

int main(void) {
    int *p;
    *p = 5;
    printf("%d\n", *p);
    return 0;
}
```

`p` holds garbage, so `*p = 5` writes 5 to a random address. Vizu stops at
line 5: `p` has no arrow to follow.

## 6.3 Copying pointers or what they point to

**Read:** King 11.3
**You will:** tell `p = q` from `*p = *q`.

Both are assignments, and they do very different things:

- `*p = *q` copies the *value* `q` points to into the box `p` points to. The
  arrows stay; a box changes.
- `p = q` copies the *address*. Now both point to the same box; no value
  changes.

```c run
#include <stdio.h>

int main(void) {
    int i = 1, j = 2;
    int *p = &i, *q = &j;
    *p = *q;        /* copies the value: i becomes 2 */
    printf("i=%d j=%d\n", i, j);
    i = 1;
    p = q;          /* copies the address: p now points to j */
    *p = 7;
    printf("i=%d j=%d\n", i, j);
    return 0;
}
```

**Watch:** line 6 lights `i` amber and leaves the arrows alone. Line 9 lights
nothing but swings `p`'s arrow from `i` to `j`; line 10 then writes `j`.

### Check yourself

1. Predict both lines of output.
   <details><summary>Answer</summary>

   `i=2 j=2`, then `i=1 j=7`.
   </details>

2. After `p = q;`, how many names does the box `j` have?
   <details><summary>Answer</summary>

   Three: `j`, `*p` and `*q`. Several pointers to one box are called
   *aliases*; a change through any of them is seen through all.
   </details>

### Common mistake

Writing `p = 7;` when `*p = 7;` was meant. It stores 7 as an *address* in `p`.
The compiler warns ("makes pointer from integer"); never ignore that warning.

## 6.4 `swap`, done right

**Read:** King 11.4 · K&R 5.2
**You will:** let a function change its caller's variables.

Lesson 5.2 showed that a function only gets copies. To let it change the
caller's variables, give it their addresses: the copies of the addresses still
lead to the caller's boxes.

```c run
#include <stdio.h>

void swap_wrong(int x, int y) {
    int tmp = x;
    x = y;
    y = tmp;
}

void swap(int *px, int *py) {
    int tmp = *px;
    *px = *py;
    *py = tmp;
}

int main(void) {
    int a = 1, b = 2;
    swap_wrong(a, b);
    printf("after swap_wrong: a=%d b=%d\n", a, b);
    swap(&a, &b);
    printf("after swap:       a=%d b=%d\n", a, b);
    return 0;
}
```

**Watch:**

- In `swap_wrong` the boxes `x` and `y` swap, in the top frame. Then the
  frame is gone, and `a` and `b` never changed.
- In `swap`, `px` and `py` are arrows reaching down into `main`'s frame. The
  amber at lines 11 and 12 lands on `a` and `b` themselves.

### Check yourself

1. Write the prototype of a function that finds the smallest and the largest
   element of an array and gives both back.
   <details><summary>Answer</summary>

   `void min_max(const int a[], int n, int *min, int *max);` called as
   `min_max(nums, 5, &lo, &hi);`. King builds exactly this in 11.4.
   </details>

2. What goes wrong with `swap(a, b);` (no `&`)?
   <details><summary>Answer</summary>

   The values 1 and 2 are used as addresses. The compiler warns that an `int`
   is passed where an `int *` is expected; Vizu stops at the first `*px`.
   </details>

### Common mistake

`*px++` when `(*px)++` was meant. `++` binds tighter than `*`, so `*px++` moves
the pointer, not the value (lesson 7.3 uses that on purpose).

## 6.5 Returning pointers, and a dead local

**Read:** King 11.5
**You will:** return a pointer safely, and recognize one that points to a
frame that is gone.

A function can return a pointer, as long as what it points to outlives the
call: one of the arguments, a global, a `static`, or heap memory (module 10).

```c run
#include <stdio.h>

int *larger(int *a, int *b) {
    return *a > *b ? a : b;
}

int main(void) {
    int x = 3, y = 8;
    int *m = larger(&x, &y);
    *m = 0;
    printf("x=%d y=%d\n", x, y);
    return 0;
}
```

`cond ? a : b` is the *conditional expression*: `a` if `cond` is true, else
`b`. `m` ends up pointing to `y`, and `*m = 0` zeroes `y`.

Returning the address of a *local* is the bug:

```c run expect="error"
#include <stdio.h>

int *make_number(void) {
    int n = 42;
    return &n;
}

int main(void) {
    int *p = make_number();
    printf("%d\n", *p);
    return 0;
}
```

**Watch:** while `make_number` runs, the arrow goes to `n` in its frame. When
the frame goes, the arrow stays, and turns red: it points to memory that
belongs to nobody. Vizu stops at the `*p` that follows it.

### Check yourself

1. Why might the bad program print 42 on a real machine anyway?
   <details><summary>Answer</summary>

   The frame's bytes are not wiped when it ends; they are just free to be
   reused. Until the next call reuses them, the old 42 is still there. The
   program is wrong even when it seems to work.
   </details>

2. Name two ways to fix `make_number`.
   <details><summary>Answer</summary>

   Return the `int` itself, not a pointer; or let the caller pass the address
   of a box to fill (`void make_number(int *out)`). (A `static int n` would
   also work, at the price of one shared box.)
   </details>

### Common mistake

The same bug with an array: returning a local `char buf[100]` from a function
that builds a string. The array dies with the frame; return a pointer to the
caller's buffer or to heap memory instead.

## 6.6 `const` and pointers

**Read:** King 11.4 (Using const to Protect Arguments), 18.3
**You will:** read and write the two kinds of `const` pointer.

`const` makes a box read-only. With pointers there are two things that can be
read-only, the box pointed to or the pointer itself, and the position of
`const` says which. Read the declaration from the name outwards:

- `const int *p`: `p` is a pointer to a `const int`. You may move `p`, but not
  change `*p` through it.
- `int *const q`: `q` is a `const` pointer to an `int`. You may change `*q`,
  but `q` always points to the same box.

```c run
#include <stdio.h>

int total(const int *a, int n) {
    int s = 0;
    for (int i = 0; i < n; i++)
        s += a[i];
    return s;
}

int main(void) {
    int nums[3] = {4, 5, 6};
    int x = 1;
    const int *p = &x;      /* x cannot be changed through p */
    int *const q = &x;      /* q cannot point anywhere else */
    *q = 2;
    printf("%d %d\n", total(nums, 3), *p);
    return 0;
}
```

**Watch:** `p` and `q` both point to `x`. The write at line 15 goes through
`q`; reading `*p` at line 16 sees it.

### Check yourself

1. Which of these does the compiler reject: `*p = 3;`, `p = &nums[0];`,
   `q = &nums[0];`?
   <details><summary>Answer</summary>

   `*p = 3;` (the `int` is read-only through `p`) and `q = &nums[0];` (`q` is
   read-only). Moving `p` is fine.
   </details>

2. Why write `const` in `total`'s parameter?
   <details><summary>Answer</summary>

   It is a promise, checked by the compiler, that `total` only reads the
   array. The caller can pass it read-only data, and the reader of the code
   knows at a glance.
   </details>

### Common mistake

Thinking `const int *p` makes the `int` constant everywhere. It only says you
cannot change it *through `p`*; `x = 5;` still works, and `*p` then reads 5.
