# Module 10 · Dynamic memory

Every variable we have used so far has had its size fixed when the program
was written. An array of 100 `int`s holds 100, no more and no less, and if
the input turns out to have 101 numbers, the program is stuck. The lifetimes
have been fixed too. A local variable lives as long as its function's
frame, and disappears when the function returns; a global lives for the
whole run. There has been nothing in between.

Real programs constantly need memory that does not fit those rules. A text
editor does not know how long the file will be until it opens it. A program
that reads a list of names does not know how many there will be. A function
that builds a new string must hand it to its caller, so the string has to
outlive the function's frame.

For all of these, C lets a program ask for memory while it runs, in exactly
the amount it needs, from a separate region called the *heap*. Memory from
the heap stays until the program itself gives it back, whatever functions
come and go in the meantime. That freedom comes with a responsibility that
C does not take off your hands: everything taken from the heap must be
given back exactly once, and never used afterwards. This module shows how
to use the heap, the classic mistakes, and finally how the heap works
inside.

Vizu draws the heap next to the stack. Every block appears there when it is
made, labelled with the line that made it, with arrows from every pointer
that leads to it, and it fades when it is freed.

## 10.1 The heap: `malloc` and `free`

The library function `malloc`, declared in `<stdlib.h>`, reserves a block
of memory on the heap. It is given the number of bytes wanted, and it
returns the address of the first byte. If there is not enough memory, it
returns `NULL` instead.

What type is that address? `malloc` does not know what the memory will be
used for, so it returns a `void *`, a "pointer to anything". A `void *`
cannot be used to reach the memory directly, but it can be assigned to a
pointer of any other type, which then decides how the bytes are used.

When the block is no longer needed, the program gives it back with
`free`, passing the same address that `malloc` returned. Between the two
calls, the block belongs to the program, and nothing will free it
automatically.

```c run
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    int n = 5;
    int *a = malloc(n * sizeof *a);
    if (a == NULL) {
        printf("out of memory\n");
        return 1;
    }
    for (int i = 0; i < n; i++)
        a[i] = i * i;
    printf("%d %d\n", a[0], a[4]);
    free(a);
    a = NULL;
    return 0;
}
```

Look at how the size is written: `n * sizeof *a`. `*a` is what `a` points
to, an `int`, so this asks for room for `n` `int`s, 20 bytes. Written this
way, the line stays correct even if the type of `a` is changed later.

Once `a` points to the block, it is used exactly like an array: `a[i]` is
`*(a + i)`, as lesson 7.2 explained, and that works for any pointer.

Step through it and watch the right-hand side of the memory picture. At
line 6, a block of 20 bytes appears in the heap, labelled with the line
that made it. Its boxes hold question marks, because `malloc` does not
clear the memory it hands out. An arrow leads to it from `a`, which lives on
the stack in `main`'s frame. The loop fills the boxes. At line 14, `free`
gives the block back, and it fades. `a` still holds the old address, which
now leads to memory that is no longer the program's, until line 15 sets it
to `NULL`.

### Check yourself

1. Where are `n` and `a`, and where are the five `int`s?
   <details><summary>Answer</summary>

   `n` and `a` are ordinary local variables, on the stack in `main`'s
   frame. The five `int`s are on the heap. `a` only holds their address.
   </details>

2. Why set `a` to `NULL` after `free`?
   <details><summary>Answer</summary>

   So that a later mistaken use of `a` fails clearly, as the use of a null
   pointer, instead of quietly reaching memory that has been given back.
   </details>

### A common mistake

Writing `malloc(n)` when `n` elements were meant. `malloc` counts bytes,
so this reserves `n` bytes, a quarter of what `n` `int`s need, and the loop
then writes past the end of the block.

## 10.2 Strings on the heap

The heap is the natural home for strings whose length is known only while
the program runs. Here is a function that joins two strings into a new
one. It cannot use a local array for the result, because the array would
die with the function's frame (lesson 6.5), and it cannot know in advance
how long the result will be. So it measures, asks the heap for exactly the
right amount, and returns a pointer to it.

```c run
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

char *concat(const char *s1, const char *s2);

int main(void) {
    char *full = concat("Grace ", "Hopper");
    if (full == NULL)
        return 1;
    printf("%s (%zu characters)\n", full, strlen(full));
    free(full);
    return 0;
}

char *concat(const char *s1, const char *s2) {
    char *result = malloc(strlen(s1) + strlen(s2) + 1);
    if (result == NULL)
        return NULL;
    strcpy(result, s1);
    strcat(result, s2);
    return result;
}
```

The size asked for is the length of both strings plus one, for the `'\0'`
at the end. Forgetting that one byte is the most common mistake with
strings on the heap.

Step through it and follow the block. It is made inside `concat`, and
`result`'s arrow leads to it from `concat`'s frame. `strcpy` and `strcat`
fill it, character by character. Then `concat` returns, and its frame
disappears, but the block stays: the heap does not care about frames. The
arrow now comes from `full`, in `main`'s frame. The caller has received a
string that outlived the function that made it, and from now on it is the
caller's job to free it, as line 12 does.

### Check yourself

1. How many bytes does `concat("Grace ", "Hopper")` ask for?
   <details><summary>Answer</summary>

   13: six characters, six characters, and one for the `'\0'`.
   </details>

2. Who must free the string, and how would they know?
   <details><summary>Answer</summary>

   The caller, `main`. C has no way to say this in the code, so it has to
   be written in the function's documentation: "returns a new string,
   which the caller must free".
   </details>

### A common mistake

Asking for `strlen(s)` bytes to copy a string `s`. The copy needs one
byte more for its `'\0'`, and `strcpy` writes that byte one past the end of
the block.

## 10.3 Growing arrays: `calloc` and `realloc`

Two more functions complete the set. `calloc(n, size)` reserves room for
`n` elements of `size` bytes each, like `malloc(n * size)`, but it also
fills the whole block with zeros. `realloc(p, new_size)` changes the size of
a block that came from the heap. If there is room after the block, it may
simply grow it where it is. If not, it reserves a new block of the new
size elsewhere, copies the old contents into it, frees the old block, and
returns the new address. Either way, the program must use the address that
`realloc` returns from then on.

With `realloc`, a program can keep an array that grows as data arrives.
The usual strategy is to double the capacity whenever the array is full;
the copying then happens rarely, however many elements arrive.

```c run input="4 8 15 16 23 42\n"
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    int cap = 2, n = 0, x;
    int *a = malloc(cap * sizeof *a);
    if (a == NULL)
        return 1;
    while (scanf("%d", &x) == 1) {
        if (n == cap) {
            cap *= 2;
            int *bigger = realloc(a, cap * sizeof *a);
            if (bigger == NULL) {
                free(a);
                return 1;
            }
            a = bigger;
        }
        a[n++] = x;
    }
    printf("%d numbers, capacity %d, last %d\n", n, cap, a[n - 1]);
    free(a);
    return 0;
}
```

`n` counts the numbers stored and `cap` the room there is. Before storing
a number, the program checks whether the array is full, and if so, doubles
it.

This is a lab worth watching closely. At each `realloc`, a new, larger block
appears in the heap with the old values already copied into it, the old
block fades, and `a`'s arrow swings over to the new one. It happens twice,
when the capacity goes from 2 to 4 and from 4 to 8. The C++ track will
show that its `std::vector` does exactly this inside.

### Check yourself

1. How many times does `realloc` run for the six numbers?
   <details><summary>Answer</summary>

   Twice: the capacity goes from 2 to 4, then from 4 to 8.
   </details>

2. Why is the result of `realloc` put into `bigger` instead of straight
   into `a`?
   <details><summary>Answer</summary>

   If `realloc` fails, it returns `NULL` and leaves the old block where it
   was. `a = realloc(a, …)` would then overwrite the only pointer to the
   old block with `NULL`, and the block could never be freed.
   </details>

### A common mistake

Keeping a pointer into the old block across a `realloc`, such as `int
*third = &a[2];`. If the block has moved, `third` points into memory that
has been freed.

## 10.4 Leaks, double frees and dangling pointers

The heap gives a program control over memory, and with control come
three classic ways to get it wrong. Each has a name, and it is worth
learning them, because together they cause a large share of the bugs and
security holes in C programs.

- A **dangling pointer** still holds the address of a block that has been
  freed. Using it is a *use after free*: the memory may already have been
  handed out again, for something else.
- A **double free** gives the same block back twice. It confuses the
  allocator's records, and the damage shows up later, somewhere else.
- A **leak** loses the last pointer to a block before the block is freed.
  The block can then never be given back.

The first is often hidden behind a second pointer to the same block:

```c run expect="error"
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    int *p = malloc(sizeof *p);
    *p = 7;
    int *q = p;         /* a second name for the block */
    free(p);
    printf("%d\n", *q);
    return 0;
}
```

Step through it. After `free(p)`, both arrows turn red, `p`'s and `q`'s,
although nothing was ever done to `q`. A pointer does not know that its
block has gone; it keeps the old address. Vizu stops at line 9, where the
freed block is read through `q`.

A leak, on the other hand, stops nothing. The program simply never gives
the memory back:

```c run
#include <stdlib.h>

int main(void) {
    for (int i = 0; i < 3; i++) {
        int *p = malloc(100);
        p[0] = i;
    }
    return 0;
}
```

On each round, `p` receives a new block, and the block from the round
before is left with no arrow pointing to it at all. Nothing in the
program can reach it any more, so nothing can free it. When the program
ends, Vizu marks the three blocks as leaked.

### Check yourself

1. Is a leak a problem in a program that finishes a second later?
   <details><summary>Answer</summary>

   Hardly: when a program ends, the operating system takes back all its
   memory. But a server, an editor or a game runs for days, and its leaks
   add up until memory runs out.
   </details>

2. How does setting a pointer to `NULL` after `free` help against double
   frees?
   <details><summary>Answer</summary>

   `free(NULL)` does nothing, so freeing the same pointer a second time
   becomes harmless. It does not help with other copies of the pointer,
   such as `q`.
   </details>

### A common mistake

Freeing something that did not come from `malloc`, such as a local array
or a string literal, or a pointer into the middle of a block. Only the
exact address that `malloc`, `calloc` or `realloc` returned may be given to
`free`.

## 10.5 Linked lists

The heap makes possible a structure that arrays cannot provide: a list
that grows and shrinks one element at a time, where adding an element at
the front costs the same whether the list has three elements or three
million.

Each element of such a list is a structure called a *node*. It holds a
value and a pointer to the next node. The last node's pointer is `NULL`,
which marks the end. The list as a whole is just a pointer to its first
node. Since each node points to the next, the nodes can lie anywhere on
the heap; they are linked by their pointers, not by being side by side.

```c run
#include <stdio.h>
#include <stdlib.h>

struct node {
    int value;
    struct node *next;
};

struct node *push(struct node *head, int value) {
    struct node *n = malloc(sizeof *n);
    n->value = value;
    n->next = head;
    return n;
}

int main(void) {
    struct node *list = NULL;
    for (int i = 1; i <= 3; i++)
        list = push(list, i * 10);
    for (struct node *p = list; p != NULL; p = p->next)
        printf("%d -> ", p->value);
    printf("NULL\n");
    while (list != NULL) {
        struct node *next = list->next;
        free(list);
        list = next;
    }
    return 0;
}
```

The structure `struct node` contains a pointer to its own type. That is
allowed, because a pointer has a fixed size whatever it points to; a
structure could not contain a whole copy of itself.

`push` puts a new value at the front of the list. It makes a node, stores
the value, makes the node's `next` point to the old first node, and
returns the new node, which becomes the new first node.

The three loops in `main` do the three basic things to a list. The first
builds it. The second walks it: `p` starts at the first node and follows
the `next` pointers until it reaches `NULL`. The third frees it, node by
node.

Step through it and watch the heap. Each `push` makes a node whose `next`
arrow leads to the previous first node, and then `list`'s arrow swings to
the new node. In the second loop, `p`'s arrow hops from node to node along
the chain. In the third, notice the order of the work: the pointer to the
next node is saved *before* the current node is freed, because after
`free(list)`, the node, and its `next`, may no longer be read.

### Check yourself

1. In which order are the values printed?
   <details><summary>Answer</summary>

   `30 -> 20 -> 10 -> NULL`. Each `push` puts its value in front of the
   ones before.
   </details>

2. Write a function that returns the first node holding `v`, or `NULL`.
   <details><summary>Answer</summary>

   ```c
   struct node *find(struct node *p, int v) {
       while (p != NULL && p->value != v)
           p = p->next;
       return p;
   }
   ```
   The order of the two tests matters: thanks to `&&`, `p->value` is read
   only when `p` is not `NULL`.
   </details>

### A common mistake

Freeing the list in a `for` loop: `for (p = list; p != NULL; p = p->next)
free(p);`. After the body frees `p`, the loop reads `p->next` from the node
that was just freed.

## 10.6 Pointers to pointers

The `push` of the last lesson returns the new first node, and the caller
must remember to write `list = push(list, …)`. It would be nicer if `push`
could update `list` itself. But `list` is a variable of the caller's, and
lesson 6.4 taught us how a function changes a caller's variable: it is
given the variable's address. `list` is a `struct node *`, so its address
is a `struct node **`, a pointer to a pointer.

```c run
#include <stdio.h>
#include <stdlib.h>

struct node {
    int value;
    struct node *next;
};

void push(struct node **head, int value) {
    struct node *n = malloc(sizeof *n);
    n->value = value;
    n->next = *head;
    *head = n;
}

void remove_value(struct node **link, int value) {
    while (*link != NULL && (*link)->value != value)
        link = &(*link)->next;
    if (*link != NULL) {
        struct node *dead = *link;
        *link = dead->next;
        free(dead);
    }
}

int main(void) {
    struct node *list = NULL;
    push(&list, 3);
    push(&list, 2);
    push(&list, 1);
    remove_value(&list, 2);
    for (struct node *p = list; p != NULL; p = p->next)
        printf("%d ", p->value);
    printf("\n");
    while (list != NULL) {
        struct node *next = list->next;
        free(list);
        list = next;
    }
    return 0;
}
```

In `push`, `head` points to `main`'s `list`, so `*head` *is* `list`: the
new node's `next` takes its old value, and `*head = n` changes `list`
itself.

`remove_value` uses the same idea in a clever way. To unlink a node, we
must change the pointer that leads to it. For the first node, that pointer
is `list`; for any other node, it is the `next` of the node before. Instead
of treating the two cases separately, `link` holds the address of
*whichever pointer leads to the current node*. It starts at `&list`, and
moves on with `link = &(*link)->next`. When the node is found, `*link =
dead->next` makes that pointer skip over it, whichever pointer it is.

Step into `remove_value` and watch `link`. It is an arrow to an arrow: at
first, its arrow leads to `list` in `main`'s frame; after one step, to the
`next` box inside the first node. At line 21, the pointer at the end of
`link`'s arrow swings past the node holding 2, and line 22 frees that node.

### Check yourself

1. What does the program print?
   <details><summary>Answer</summary>

   `1 3`.
   </details>

2. What would go wrong if `push` took `struct node *head`?
   <details><summary>Answer</summary>

   It would change only its own copy of the pointer, and `main`'s `list`
   would stay `NULL`: the problem of lesson 5.3, one level up.
   </details>

### A common mistake

Writing `*link->next` instead of `(*link)->next`. `->` binds more tightly
than `*`, so the first means `*(link->next)`, which treats `link` as a
pointer to a node, which it is not.

## 10.7 How `malloc` works

`malloc` is not magic, and it is not part of the language. It is an
ordinary C function in the library, and it is instructive to write a small
one ourselves. The simplest kind hands out pieces of one big array, one
after the other, like the stack:

```c run
#include <stdio.h>

#define POOL_SIZE 64

static char pool[POOL_SIZE];    /* the memory we hand out */
static char *next_free = pool;  /* the first byte not handed out */

char *alloc(int n) {
    if (pool + POOL_SIZE - next_free >= n) {
        next_free += n;
        return next_free - n;
    }
    return NULL;                /* not enough room */
}

void afree(char *p) {           /* frees p and everything after it */
    if (p >= pool && p < pool + POOL_SIZE)
        next_free = p;
}

int main(void) {
    char *a = alloc(10);
    char *b = alloc(20);
    char *c = alloc(40);
    printf("a at +%ld, b at +%ld, c %s\n", (long) (a - pool), (long) (b - pool),
           c != NULL ? "fits" : "does not fit");
    afree(b);
    char *d = alloc(30);
    printf("d at +%ld\n", (long) (d - pool));
    return 0;
}
```

`pool` is the memory to hand out, and `next_free` points to the first byte
not yet handed out. `alloc` checks whether enough bytes are left, moves
`next_free` on, and returns the old position. `afree` moves `next_free`
back, which frees the given block and everything handed out after it.

Step through it and watch `pool`, a long row of 64 boxes among the
globals, with `next_free`'s arrow moving right at each `alloc` and jumping
back at `afree`. `c` does not fit, since 30 bytes are taken and only 34
remain. After `afree(b)`, `d` is placed where `b` was.

### What a real `malloc` adds

This allocator can only free in the reverse order of allocating. A real
`malloc` must allow blocks to be freed in any order, and it manages that
with a few more ideas:

- In front of each block it keeps a small *header* holding the block's
  size. That is how `free` knows how much to give back, although it is
  given only an address.
- Freed blocks go into a *free list*. The next `malloc` looks there first
  for a block big enough, and neighbouring free blocks are joined into one.
- When nothing on the free list is big enough, `malloc` asks the operating
  system for more memory. On Linux it does so with the system calls `brk`,
  which extends the heap's region, or `mmap`, which maps a fresh area for a
  very large block. Those are slow, so `malloc` asks for a large piece at
  once and then hands it out bit by bit.

This is also why a leak is invisible from inside the program: the memory
is still marked as in use in the allocator's records, and only the
program could have said otherwise.

### Check yourself

1. Why does `c` not fit?
   <details><summary>Answer</summary>

   10 + 20 = 30 bytes are handed out, so 34 are left, and `c` wants 40.
   </details>

2. What goes wrong if the program calls `afree(a)` while `b` is still in
   use?
   <details><summary>Answer</summary>

   `next_free` jumps back to `a`, so the next `alloc` hands out the bytes
   of `b` a second time. This allocator only works if blocks are freed in
   the reverse order of their allocation.
   </details>

### A common mistake

Believing that `free` gives memory back to the operating system. Usually
it only returns the block to the allocator's free list, inside your
program, ready for the next `malloc`.
