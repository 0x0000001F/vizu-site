# Module 10 · Dynamic memory

Read alongside: King chapter 17 (17.1–17.6) · K&R sections 5.4, 6.5 and 8.7

Stack boxes live as long as their frame. When a program needs memory whose
size is known only while it runs, or that must outlive the function that made
it, it asks for it on the *heap*, and gives it back itself. Vizu draws the heap
next to the stack: every block, who points to it, and what happened to it.

## 10.1 The heap: `malloc` and `free`

**Read:** King 17.1 · K&R 5.4
**You will:** allocate a block, use it, and give it back.

`malloc(n)` reserves `n` bytes on the heap and returns their address as a
`void *`, a pointer to no particular type, which converts to any object
pointer. It returns `NULL` when there is no memory. `free(p)` gives the block
back. Between the two, the block is yours; nothing frees it for you.

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

`n * sizeof *a` asks for `n` of whatever `a` points to; if `a`'s type
changes, the line stays right.

**Watch:**

- Line 6: a block of 20 bytes appears on the heap, its cells `?`; `a`'s arrow
  leads to it. It is labelled with the line that made it.
- Line 14: the block fades: freed. `a` still holds the old address until line
  15 sets it to `NULL`.

### Check yourself

1. Where are `n` and `a` themselves, and where is what `a` points to?
   <details><summary>Answer</summary>

   `n` and `a` are on the stack, in `main`'s frame. The five `int`s are on
   the heap.
   </details>

2. Why set `a` to `NULL` after `free`?
   <details><summary>Answer</summary>

   So that a later mistaken use fails clearly (a null pointer) instead of
   quietly using freed memory.
   </details>

### Common mistake

`malloc(n)` when `n` elements were meant: it allocates `n` *bytes*, a quarter
of what five `int`s need.

## 10.2 Growing arrays: `calloc`, `realloc`

**Read:** King 17.3
**You will:** keep an array that grows as data arrives.

`calloc(n, size)` allocates `n` elements and fills them with zeros.
`realloc(p, newsize)` resizes a block: it may grow it where it is, or move it,
copying the contents and freeing the old block. Either way, use the address it
returns. Doubling the size each time it is full keeps copying rare.

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

**Watch:** at each `realloc`, a bigger block appears with the old values
copied in, the old block fades, and `a`'s arrow swings to the new one. This is
exactly what C++'s `std::vector` does inside (C++ lesson 4.2).

### Check yourself

1. How many times does `realloc` run for six numbers?
   <details><summary>Answer</summary>

   Two: the capacity goes 2 → 4 → 8.
   </details>

2. Why not write `a = realloc(a, ...)` directly?
   <details><summary>Answer</summary>

   If `realloc` fails it returns `NULL` and leaves the old block alone.
   Assigning `NULL` to `a` would lose the only pointer to it: a leak.
   </details>

### Common mistake

Keeping a pointer into the old block (`int *third = &a[2];`) across a
`realloc`. If the block moved, `third` points to freed memory.

## 10.3 Leaks, double frees, dangling pointers

**Read:** King 17.4
**You will:** recognize the three classic heap bugs.

- **Use after free**: a pointer still leads to a block that was freed. The
  memory may already belong to something else.
- **Double free**: freeing the same block twice corrupts the allocator.
- **Leak**: the last pointer to a block is lost before it is freed; the block
  can never be given back.

The first is often hidden behind a second name for the same block:

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

**Watch:** after `free(p)` both arrows turn red: `q` is dangling too, although
nobody touched it. Vizu stops at line 9.

A leak does not stop anything; the program just never gives memory back:

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

**Watch:** each round, `p` gets a new block and the previous one has no arrow
left. At the end Vizu marks three leaked blocks.

### Check yourself

1. Is a leak a problem in a program that ends a second later?
   <details><summary>Answer</summary>

   Not much: the operating system takes everything back when a process ends.
   In a server or a game that runs for days, leaks add up until memory runs
   out.
   </details>

2. How does setting pointers to `NULL` after `free` help with double frees?
   <details><summary>Answer</summary>

   `free(NULL)` does nothing, so a second `free` of a nulled pointer is
   harmless. It does not help with other copies of the pointer, like `q`.
   </details>

### Common mistake

Freeing memory that did not come from `malloc` (a local array, a string
literal) or freeing a pointer into the middle of a block. Only the exact
address `malloc` returned may be freed.

## 10.4 Linked lists

**Read:** King 17.5 · K&R 6.5 (self-referential structures)
**You will:** build, walk and free a linked list.

A *node* is a struct holding a value and a pointer to the next node. A list is
a pointer to the first node, and the last node's `next` is `NULL`. Unlike an
array, a list grows one node at a time, and inserting at the front costs the
same whatever its length.

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

**Watch:**

- Each `push` makes a node on the heap whose `next` arrow leads to the old
  first node; then `list` swings to the new one.
- The printing loop's `p` hops from node to node along the arrows.
- The freeing loop saves `next` *before* freeing: after `free(list)`,
  `list->next` may not be read.

### Check yourself

1. In which order are the values printed?
   <details><summary>Answer</summary>

   `30 -> 20 -> 10 -> NULL`: each push goes in front.
   </details>

2. Write the search: return the first node holding `v`, or `NULL`.
   <details><summary>Answer</summary>

   ```c
   struct node *find(struct node *p, int v) {
       while (p != NULL && p->value != v)
           p = p->next;
       return p;
   }
   ```
   The order of the `&&` matters: `p->value` is read only when `p` is not
   `NULL`.
   </details>

### Common mistake

Freeing in the loop header: `for (p = list; p; p = p->next) free(p);` reads
`p->next` from a node that was just freed.

## 10.5 Pointers to pointers

**Read:** King 17.6 · K&R 5.6
**You will:** let a function change a caller's pointer, and remove a node
cleanly.

To let a function change an `int`, pass an `int *`. To let it change a
`struct node *`, such as the head of a list, pass a `struct node **`. The same
idea gives a short way to remove a node: walk with a pointer to the *link*
that leads to the current node, so the first node needs no special case.

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

**Watch:** in `remove_value`, `link` is an arrow to an arrow: first to `list`
in `main`, then to the first node's `next`. At line 20 the arrow it points to
swings past the dead node.

### Check yourself

1. What is printed?
   <details><summary>Answer</summary>

   `1 3`.
   </details>

2. What would go wrong if `push` took `struct node *head`?
   <details><summary>Answer</summary>

   It would change its own copy of the head; `main`'s `list` would stay `NULL`
   (lesson 5.2, one level up).
   </details>

### Common mistake

`*link->next` instead of `(*link)->next`. `->` binds tighter than `*`, so it
means `*(link->next)`, and `link` is not a pointer to a node.

## 10.6 Writing a tiny allocator

**Read:** K&R 5.4 (a rudimentary storage allocator), 8.7
**You will:** see that `malloc` is just C code handing out pieces of a big
array.

K&R write an allocator twice: a simple one in 5.4 that hands out pieces of a
big `char` array like a stack, and a real one in 8.7 that keeps a list of free
blocks. The simple one fits on a screen:

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

**Watch:** `pool` is one long row of 64 cells in the globals area; `next_free`
is an arrow that moves right with each `alloc` and jumps back at `afree`.

A real `malloc` cannot free things in any order with a single pointer. It puts
a small *header* before each block (its size, so `free` needs no size), keeps
freed blocks in a list, and asks the operating system for more memory when
the list has nothing big enough.

### Check yourself

1. Why does `c` not fit?
   <details><summary>Answer</summary>

   10 + 20 = 30 bytes are taken, 34 are left, and `c` wants 40.
   </details>

2. What is wrong with calling `afree(a)` while `b` is still in use?
   <details><summary>Answer</summary>

   `next_free` jumps back to `a`, so the next `alloc` hands out `b`'s bytes
   again. This allocator only works last-in, first-out.
   </details>

### Common mistake

Thinking `free` returns memory to the operating system. Usually it returns it
to the allocator's free list, inside your program, ready for the next
`malloc`.
