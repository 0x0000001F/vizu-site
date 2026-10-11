# Module 6 · Pointers

Pointers have a reputation. Many people who learn C say that everything
went well until pointers, and that after pointers nothing made sense. If
you have followed this course, you are in a better position than they
were, because you have already seen the main idea dozens of times.

Since lesson 1.3, every box in the memory picture has had a small number
beside it: its address. Memory is a long row of bytes, each with its own
number, and every variable lives at some address. A *pointer* is nothing
more than a variable whose value is one of those numbers. It holds the
address of another box, and so it *points to* that box.

Why would anyone want that? Because an address lets a program reach a box
without knowing its name. A function can change its caller's variables if
it is given their addresses, which it could not do with copies of their
values. A program can walk through an array by moving an address along
it. And, in module 10, a program can ask for new memory while it runs,
memory that has no name at all and can only be reached through its
address. Nearly everything interesting that C can do goes through
pointers.

Vizu draws a pointer as an arrow, from the pointer's box to the box whose
address it holds. Books draw the same arrows by hand, a few times per
chapter; here they are drawn for every line of every program, so you can
always see where each pointer leads.

## 6.1 Every box has an address

Picture the computer's memory as a very long street of houses, each house
holding one byte, and each with a house number. The numbers start at 0
and go up one by one. A variable of type `char` lives in one house; an
`int` occupies four houses next to each other; a `double` eight. The
address of a variable is the number of the first house it occupies.

C lets a program find out a variable's address with the *address
operator*, `&`, which we have used since the first `scanf`: `&i` is the
address of `i`. `printf` can print an address with the conversion `%p`.

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

The `(void *)` before each address converts it to the general kind of
address that `%p` expects; we will meet `void *` properly in module 10.

Step through the program and compare the printed addresses with the
numbers beside the boxes in the memory picture. They are the same numbers.
Vizu keeps addresses short and round so that they are easy to read and to
compare: string constants are placed from `0x1000`, global variables from
`0x2000`, memory from `malloc` from `0x4000`, and the stack grows downwards
from `0x8000`. A real 64-bit computer prints much longer addresses, such as
`0x7ffd5c2e8a1c`, but they work in exactly the same way: each byte has its
own number, and the bytes of one variable sit next to each other.

### Check yourself

1. Can you take the address of `i + 1`?
   <details><summary>Answer</summary>

   No. `&` needs something that has a box: a variable, an element of an
   array, a member of a structure. `i + 1` is a value that has been worked
   out but is not kept anywhere, so it has no address.
   </details>

2. On a real computer, the address of the same variable changes from one
   run to the next. Why?
   <details><summary>Answer</summary>

   The operating system places the stack, and other parts of the program, at
   a slightly different random place each time it starts a program. This is
   called address space layout randomization, and it makes some attacks
   much harder.
   </details>

### A common mistake

Printing an address with `%d`. An address on a 64-bit computer takes 8
bytes, but `%d` expects a 4-byte `int`, so it prints only part of it, or
nonsense. Addresses are printed with `%p`.

## 6.2 Pointer variables and `*`

An address is a value, so it can be stored in a variable. A variable that
holds an address is called a *pointer variable*, or simply a pointer, and
it is declared with a `*` before its name:

```c
int *p;
```

This declares `p` as a *pointer to `int`*: a box that can hold the address
of an `int`. The type matters. A pointer always knows what kind of box it
points to, so that when it is used, the computer knows how many bytes to
read and how to read them.

Giving a pointer a value means giving it an address:

```c
p = &i;
```

Now `p` holds the address of `i`, and we say that `p` *points to* `i`.

To use the box that a pointer points to, we put `*` in front of the
pointer. `*p` means "the box whose address is in `p`". This `*` is called
the *indirection operator*, because it reaches a box indirectly, by way of
its address. While `p` points to `i`, `*p` is simply another name for `i`:
reading `*p` reads `i`, and assigning to `*p` assigns to `i`.

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

This program is worth stepping through slowly, one line at a time.

After line 6, there is an arrow from `p` to `i`. Notice that `i` still
holds a question mark: `p` points to it, but nothing has been written into
it yet. A pointer can point to a box regardless of what the box holds.

Line 7 writes 1 into `i` in the ordinary way, and line 8 prints it twice,
once by its name and once through the pointer.

Line 9 is the moment to stop at. `*p = 2` does not change `p` at all; `p`
still holds the same address, and its arrow does not move. The write goes
*along the arrow*, into the box at the other end, and it is `i` that turns
amber. That is all that indirection means.

### A pointer that points nowhere

A pointer that has been declared but not given a value holds whatever bits
happen to be in its box, just like any other variable. Those bits are some
number, and if the program uses them as an address, it reaches some
random place in memory.

To say clearly that a pointer points to nothing, C has a special value,
`NULL`, defined in `<stdio.h>` and several other headers. No real box ever
has this address, so a program can test `if (p != NULL)` before following a
pointer, and a pointer that has nothing to point to yet should be set to
`NULL`. Following a `NULL` pointer is still an error, but a much better one:
on a real computer the program stops at once, instead of quietly reading or
writing somewhere random.

### Check yourself

1. In `int *p, q;`, what is the type of `q`?
   <details><summary>Answer</summary>

   `int`, not a pointer. The `*` belongs to `p` alone. Declaring one
   pointer per line avoids the trap.
   </details>

2. What are `*&i` and `&*p`?
   <details><summary>Answer</summary>

   `i` and `p`. `&` takes an address and `*` follows one, so each undoes the
   other.
   </details>

### A common mistake

Using a pointer before giving it an address:

```c run expect="error"
#include <stdio.h>

int main(void) {
    int *p;
    *p = 5;
    printf("%d\n", *p);
    return 0;
}
```

`p` holds garbage, so `*p = 5` would write 5 into whatever place in memory
that garbage happens to name. On a real computer that may crash the
program, or, worse, quietly change some other variable. Vizu stops at line
5, because `p` has no arrow to follow.

## 6.3 Copying pointers, or what they point to

Two assignments look almost the same and do completely different things,
and telling them apart is the most important skill with pointers. Suppose
`p` and `q` are pointers to `int`.

```c
*p = *q;
```

copies a *value*. It takes the `int` that `q` points to and stores it in
the `int` that `p` points to. The pointers do not change; their arrows
stay exactly where they were. A box at the end of an arrow gets a new
value.

```c
p = q;
```

copies an *address*. It puts the address held in `q` into `p`, so that `p`
now points to the same box as `q`. No `int` anywhere changes its value;
only an arrow moves.

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

Step through it and watch for the two kinds of change. Line 6 makes `i`
turn amber, and the arrows do not move. Line 9 is the opposite: no number
changes, but the arrow from `p` swings away from `i` and over to `j`. From
then on, writing through `p`, as line 10 does, changes `j`.

When several pointers point to the same box, the box has several names:
after line 9, `j`, `*p` and `*q` are all the same box. Such names are
called *aliases*. A change made through any of them is seen through all of
them, which is useful, and also a common source of surprise.

### Check yourself

1. What does the program print?
   <details><summary>Answer</summary>

   `i=2 j=2`, and then `i=1 j=7`.
   </details>

2. After line 9, how many names does the box `j` have?
   <details><summary>Answer</summary>

   Three: `j`, `*p` and `*q`.
   </details>

### A common mistake

Writing `p = 7;` when `*p = 7;` was meant. That stores the number 7 in `p`
as if it were an address, so that `p` now points to the house numbered 7,
which belongs to no variable of yours. The compiler warns about it, with a
message about making a pointer from an integer. Never ignore that warning.

## 6.4 Pointers as arguments

Lesson 5.3 showed that a function receives copies of its arguments, so it
cannot change its caller's variables. Pointers are the way round that
rule. If we give a function the *address* of a variable, the function
receives a copy of the address, and a copy of an address still leads to
the same box. Through it, the function can change the caller's variable.

The classic example is a function that swaps two variables. Here it is
written twice, once wrongly and once correctly:

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

Step through `swap_wrong` first. Its frame gets copies of 1 and 2 in `x`
and `y`, and it swaps them perfectly. But they are its own boxes, in its
own frame, and when the frame disappears, the swapped values go with it.
`a` and `b` in `main`'s frame were never touched.

Now step through `swap`. The call passes `&a` and `&b`, so the frame of
`swap` holds two pointers, and their arrows reach down into `main`'s
frame, to `a` and `b`. The assignments `*px = *py` and `*py = tmp` go along
those arrows, and the amber lands on `a` and `b` themselves.

This is also the explanation of `scanf`, finally. `scanf("%d", &n)` gives
`scanf` the address of `n`, and `scanf` stores the number it reads through
that address.

### Giving back more than one value

A function can return only one value, but with pointers it can deliver as
many as it likes. The caller provides the boxes, and the function fills
them. This function finds both the smallest and the largest element of an
array:

```c run
#include <stdio.h>

void min_max(const int a[], int n, int *smallest, int *largest);

int main(void) {
    int temps[6] = {14, 9, 21, 17, 6, 19};
    int lo, hi;
    min_max(temps, 6, &lo, &hi);
    printf("coldest %d, warmest %d\n", lo, hi);
    return 0;
}

void min_max(const int a[], int n, int *smallest, int *largest) {
    *smallest = *largest = a[0];
    for (int i = 1; i < n; i++) {
        if (a[i] < *smallest)
            *smallest = a[i];
        else if (a[i] > *largest)
            *largest = a[i];
    }
}
```

In the frame of `min_max` there are three arrows going down into `main`:
one to the array, and one each to `lo` and `hi`. Watch `lo` and `hi`
change while `main` is not even running, every time the function finds a
colder or a warmer day.

### Check yourself

1. What would go wrong with `swap(a, b);`, without the `&`?
   <details><summary>Answer</summary>

   The values 1 and 2 would be used as addresses. The compiler warns that
   an `int` is given where an `int *` is expected; Vizu stops at the first
   `*px`, because there is no box at address 1.
   </details>

2. Why are `lo` and `hi` not given values before the call?
   <details><summary>Answer</summary>

   They do not need any: `min_max` writes into both before it reads them.
   Their first values come from the function, through the pointers.
   </details>

### A common mistake

Writing `*px++` where `(*px)++` was meant. `++` binds more tightly than
`*`, so `*px++` moves the pointer to the next box instead of adding one to
the value it points to. Lesson 7.3 uses exactly that on purpose.

## 6.5 Returning a pointer, and a dead local

A function can also return a pointer. This one returns a pointer to the
larger of two integers, so that the caller can then change the larger one,
whichever it was:

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

`larger` compares the two values, and, using the conditional expression
of lesson 3.2, returns whichever of the two *pointers* leads to the larger
one. `m` therefore ends up pointing to `y`, and `*m = 0` sets `y` to 0.

Returning a pointer is safe as long as the box it points to still exists
after the function has returned. A pointer that the function was given, as
here, is fine. So is a pointer to a global or `static` variable, and, in
module 10, to memory obtained from `malloc`.

What is never safe is returning the address of a local variable of the
function itself:

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

`n` lives in the frame of `make_number`, and the frame disappears when the
function returns. The address is still a number, and `p` still holds it,
but the box that was there no longer exists.

Step through it and watch the arrow. While `make_number` runs, the arrow
goes to `n` in its frame. When the frame is removed, the arrow is left
pointing into empty space, and it turns red: it points to memory that now
belongs to nobody. Such a pointer is called a *dangling pointer*. Vizu
stops at the line that follows it.

### Check yourself

1. Why might the bad program print 42 on a real computer anyway?
   <details><summary>Answer</summary>

   When a frame is removed, its bytes are not wiped; they are only marked
   as free to be used again. Until the next function call reuses them, the
   old 42 is still there. The program is wrong even when it seems to work,
   and it will stop working at the worst moment.
   </details>

2. Name two correct ways to write `make_number`.
   <details><summary>Answer</summary>

   Return the `int` itself rather than a pointer to it; or let the caller
   pass the address of a box to fill, `void make_number(int *out)`. Making
   `n` `static` would also work, at the cost of one shared box for all
   calls.
   </details>

### A common mistake

The same bug with an array: a function that builds a string in a local
`char buf[100]` and returns `buf`. The array dies with the frame. Write
into an array that the caller provides instead, or into memory from
`malloc`.

## 6.6 `const` and pointers

The word `const` makes a box read-only: the compiler refuses any statement
that would change it. With a pointer there are two boxes involved, the
pointer itself and the box it points to, so there are two different things
that can be made read-only, and the position of `const` in the declaration
says which.

The trick for reading such declarations is to start at the name and read
outwards:

- `const int *p` reads "`p` is a pointer to an `int` that is `const`". The
  `int` is read-only *through `p`*: `*p = 3` is refused. But `p` itself may
  be changed, to point somewhere else.
- `int *const q` reads "`q` is a `const` pointer to an `int`". Now the
  pointer is read-only: `q` will always point to the same box. But the box
  can be changed through it, so `*q = 3` is allowed.

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

The first kind is by far the most useful, and its main use is in function
parameters. `total` receives a pointer to the caller's array. Declaring it
`const int *a` is a promise, checked by the compiler, that `total` only
reads the array and never changes it. The caller can rely on that promise,
and a reader of the program knows it at a glance from the first line of
the function.

In the memory picture, `p` and `q` both have arrows to `x`. The write on
line 15 goes along `q`'s arrow, and the read on line 16 through `p` sees
the new value.

### Check yourself

1. Which of these does the compiler refuse: `*p = 3;`, `p = &nums[0];`,
   `q = &nums[0];`?
   <details><summary>Answer</summary>

   `*p = 3;`, because the `int` is read-only through `p`, and
   `q = &nums[0];`, because `q` itself is read-only. Moving `p` to another
   box is allowed.
   </details>

2. Why does `min_max` in lesson 6.4 have `const int a[]` but plain `int
   *smallest`?
   <details><summary>Answer</summary>

   It only reads the array, so it promises not to change it. It must write
   through `smallest` and `largest`, so those cannot be `const`.
   </details>

### A common mistake

Thinking that `const int *p` makes the `int` constant everywhere. It only
says that the `int` cannot be changed *through `p`*. If the variable itself
is not `const`, `x = 5;` still works, and `*p` will then read 5.
