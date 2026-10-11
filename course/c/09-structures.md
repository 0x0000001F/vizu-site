# Module 9 · Structures

Read alongside: King chapter 16 · K&R chapter 6

A structure groups values of different types under one name: a point's `x`
and `y`, a part's number, name and stock. In memory it is one box holding
smaller boxes, sometimes with gaps between them.

## 9.1 A struct is boxes in a box

**Read:** King 16.1–16.2 · K&R 6.1
**You will:** declare a struct type, make variables of it, and copy them.

`struct point { int x; int y; };` declares a new type; `struct point a;` makes
a variable of it, with a box for each *member*. The `.` operator reaches a
member: `a.x`. Unlike arrays, structs can be assigned: `b = a` copies every
member.

```c run
#include <stdio.h>

struct point {
    int x;
    int y;
};

int main(void) {
    struct point a = {3, 4};
    struct point b = a;     /* copies every member */
    b.x = 10;
    printf("a = (%d, %d), b = (%d, %d)\n", a.x, a.y, b.x, b.y);
    return 0;
}
```

**Watch:** `a` and `b` are two separate boxes, each holding `x` and `y`.
Line 11 changes only `b.x`.

### Check yourself

1. Can you compare two structs with `a == b`?
   <details><summary>Answer</summary>

   No. Compare member by member, in a function if you do it often.
   </details>

2. What is `sizeof(struct point)`?
   <details><summary>Answer</summary>

   8: two 4-byte `int`s, no gap.
   </details>

### Common mistake

Forgetting the `;` after the closing `}` of a struct declaration. The error
then appears on the next line, often in a confusing way.

## 9.2 Pointers to structs and `->`

**Read:** K&R 6.2, 6.4 · King 16.2
**You will:** pass a struct to a function by address, and use `->`.

Passing a struct copies all of it, and the function changes only its copy.
Pass its address instead: it is cheaper and lets the function change the
original. `p->balance` is short for `(*p).balance`.

```c run
#include <stdio.h>

struct account {
    int id;
    double balance;
};

void deposit(struct account *acc, double amount) {
    acc->balance += amount;     /* the same as (*acc).balance */
}

int main(void) {
    struct account mine = {1, 50.0};
    deposit(&mine, 25.5);
    printf("account %d: %.2f\n", mine.id, mine.balance);
    return 0;
}
```

**Watch:** `acc` is an arrow from `deposit`'s frame into `main`'s `mine`; the
write at line 9 lights `mine.balance` amber.

### Check yourself

1. Why are the parentheses needed in `(*acc).balance`?
   <details><summary>Answer</summary>

   `.` binds tighter than `*`: `*acc.balance` would mean `*(acc.balance)`.
   `->` exists to avoid writing them.
   </details>

2. When is passing a struct by value fine?
   <details><summary>Answer</summary>

   When it is small (a point, a complex number) and the function only reads
   it.
   </details>

### Common mistake

Using `.` on a pointer (`acc.balance`) or `->` on a struct (`mine->balance`).
The compiler catches both; read the error as "wrong operator".

## 9.3 Arrays of structs

**Read:** King 16.3 · K&R 6.3
**You will:** keep a table of records.

An array of structs is a table: one struct per row. King builds an inventory
of parts this way; here is a small one.

```c run
#include <stdio.h>

struct part {
    int number;
    char name[12];
    int on_hand;
};

int main(void) {
    struct part inventory[3] = {
        {528, "disk drive", 10},
        {914, "printer", 5},
        {37, "cable", 120},
    };
    int total = 0;
    for (int i = 0; i < 3; i++)
        total += inventory[i].on_hand;
    struct part *last = &inventory[2];
    printf("%d items in stock; %s is part %d\n", total, last->name, last->number);
    return 0;
}
```

**Watch:** the three structs lie next to each other, each with its `name` as a
row of characters inside it.

### Check yourself

1. What is `sizeof inventory`?
   <details><summary>Answer</summary>

   60: three structs of 4 + 12 + 4 = 20 bytes.
   </details>

2. How would you find a part by its number?
   <details><summary>Answer</summary>

   Loop over the array and compare `inventory[i].number`; return the index or
   a pointer to the struct, or −1 / `NULL` if it is not there. Keeping the
   array sorted by number allows a binary search.
   </details>

### Common mistake

Writing `inventory[i].name = "mouse";`. An array member cannot be assigned;
copy into it with `strcpy` (checking the length).

## 9.4 Padding and alignment

**Read:** King 16 Q&A (holes in structures) · K&R 6.4 (sizeof structures)
**You will:** predict a struct's size, and shrink it.

The CPU reads an `int` fastest from an address that is a multiple of 4, a
`double` from a multiple of 8. So the compiler places each member at such an
address, leaving *padding* bytes where needed, and pads the end so that
structs in an array line up too. The order of members changes the size.

```c run
#include <stdio.h>

struct loose {
    char tag;
    int value;
    char flag;
};

struct tight {
    int value;
    char tag;
    char flag;
};

int main(void) {
    printf("loose: %zu bytes, tight: %zu bytes\n",
           sizeof(struct loose), sizeof(struct tight));
    return 0;
}
```

In `loose`, `tag` is followed by 3 padding bytes so that `value` starts at
offset 4; after `flag` come 3 more, to make the size a multiple of 4. `tight`
needs only 2 at the end.

### Check yourself

1. What is the size of `struct { char c; double d; }`?
   <details><summary>Answer</summary>

   16: `c`, 7 bytes of padding, then `d` at offset 8.
   </details>

2. Why does this matter?
   <details><summary>Answer</summary>

   An array of a million `loose` structs takes 12 MB; of `tight`, 8 MB. And
   when structs are written to files or sent over a network, the padding
   travels too.
   </details>

### Common mistake

Writing a struct to a file and reading it on another machine or compiler. The
padding, the sizes and the byte order (lesson 12.3) may differ.

## 9.5 Unions and enums

**Read:** King 16.4–16.5 · K&R 2.3 (enum), 6.8
**You will:** store one of several types in the same bytes, and name a set of
values.

A *union* is like a struct whose members all start at the same address: it is
as big as its largest member, and holds one of them at a time. An *enum*
gives names to integer constants. Together they make a *tagged union*: an enum
says which member of the union is in use.

An enum on its own first:

```c run
#include <stdio.h>

enum weekday { MON, TUE, WED, THU, FRI, SAT, SUN };

int main(void) {
    enum weekday today = FRI;
    int left = SAT - today;
    printf("day %d, %d day(s) to the weekend\n", today, left);
    return 0;
}
```

Then the tagged union (Vizu does not run unions yet; read this one):

```c
#include <stdio.h>

enum kind { INT_VALUE, DOUBLE_VALUE };

struct number {
    enum kind kind;
    union {
        int i;
        double d;
    } u;
};

void print_number(struct number n) {
    if (n.kind == INT_VALUE)
        printf("int %d\n", n.u.i);
    else
        printf("double %.2f\n", n.u.d);
}

int main(void) {
    struct number a, b;
    a.kind = INT_VALUE;
    a.u.i = 42;
    b.kind = DOUBLE_VALUE;
    b.u.d = 3.14;
    print_number(a);
    print_number(b);
    printf("union size: %zu\n", sizeof a.u);
    return 0;
}
```

In memory, `a.u.i` and `a.u.d` start at the same address: writing one
overwrites the first bytes of the other.

### Check yourself

1. What is `sizeof a.u`, and why?
   <details><summary>Answer</summary>

   8, the size of its largest member, the `double`.
   </details>

2. What values do `INT_VALUE` and `DOUBLE_VALUE` have?
   <details><summary>Answer</summary>

   0 and 1: an enum counts from 0 unless told otherwise
   (`enum { A = 1, B }` makes `B` 2).
   </details>

### Common mistake

Writing one member of a union and reading another (`a.u.i = 42;` then
`a.u.d`). The bytes of the `int` are read as a `double`: nonsense. The tag is
there to prevent it.
