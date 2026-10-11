# Module 9 · Structures

An array keeps many values of the same type: thirty marks, a hundred
temperatures. But the things a program describes are rarely like that. A
point on a map has an x and a y. A part in a warehouse has a number, a
name and a quantity in stock, which are an `int`, a string and another
`int`. A date has a day, a month and a year. These values belong together,
and a program that keeps them in separate variables, `part_number`,
`part_name`, `part_stock`, has to pass all three to every function and keep
them in step by hand.

C's answer is the *structure*: a single variable made of several named
parts, called *members*, which may be of different types. In memory, a
structure is one box with smaller boxes inside it, laid out one after the
other. Sometimes, as we will see, there are small gaps between them, and
Vizu draws those too.

This module also covers two relatives of the structure. An *enumeration*
gives names to a set of integer values, such as the days of the week. A
*union* is a box that can hold one of several types, but only one at a time.

## 9.1 A structure is boxes in a box

Before we can make a structure variable, we describe what it looks like:

```c
struct point {
    int x;
    int y;
};
```

This creates no variable. It tells the compiler that from now on, `struct
point` is the name of a type, and that a value of this type has two
members, both `int`s, called `x` and `y`. The word `point` is called the
structure's *tag*. Notice the semicolon after the closing brace.

Now variables of the type can be declared, and given values with an
initializer, just as for an array:

```c
struct point a = {3, 4};
```

The values go to the members in order: `x` gets 3 and `y` gets 4. A member
is reached with the dot operator: `a.x` is the `x` member of `a`, and it can
be used like any variable of type `int`.

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
    struct point c = {.y = 7};
    printf("a = (%d, %d), b = (%d, %d), c = (%d, %d)\n", a.x, a.y, b.x, b.y, c.x, c.y);
    return 0;
}
```

Line 10 shows something that arrays cannot do: a structure can be
assigned as a whole. `b = a` copies every member of `a` into `b`. The two
are separate variables, and changing `b.x` on line 11 leaves `a` alone.

Line 12 uses a *designated initializer*, like the ones for arrays in lesson
4.2: `.y = 7` gives a value to the member `y` by name, and the members
that are not mentioned, here `x`, become 0.

Step through it and look at the memory picture. Each structure is drawn as
one box with its members inside, each member with its own address. `a.y` is
4 bytes after `a.x`, because an `int` takes 4 bytes. When line 11 runs,
only `b.x` turns amber.

### Check yourself

1. Can two structures be compared with `a == b`?
   <details><summary>Answer</summary>

   No. C does not compare structures; the program must compare them member
   by member, and a function is the place to do it if it is needed often.
   </details>

2. What is `sizeof(struct point)`?
   <details><summary>Answer</summary>

   8: two `int`s of 4 bytes, with nothing between them.
   </details>

### A common mistake

Forgetting the semicolon after the closing brace of a structure
declaration. The compiler then takes the next line to be part of the same
declaration, and the error it reports there can be very confusing.

## 9.2 Structures and functions

Since a structure can be assigned, it can also be passed to a function
and returned from one, like an `int`. That makes it easy to write
functions that work with whole points, dates or parts.

Writing `struct point` everywhere becomes tiresome, so it is common to give
the type a shorter name with `typedef`, as we did for `int` in lesson 2.1:

```c run
#include <stdio.h>

typedef struct {
    int x;
    int y;
} Point;

Point make_point(int x, int y) {
    Point p = {x, y};
    return p;
}

Point add_points(Point a, Point b) {
    a.x += b.x;
    a.y += b.y;
    return a;
}

int main(void) {
    Point start = make_point(1, 2);
    Point step = make_point(3, 1);
    Point end = add_points(start, step);
    printf("(%d, %d) + (%d, %d) = (%d, %d)\n", start.x, start.y, step.x, step.y, end.x, end.y);
    return 0;
}
```

Here the structure has no tag at all. `typedef` gives the type the name
`Point`, and from then on `Point` is used as a type name, without the word
`struct`.

`make_point` builds a structure from two numbers and returns it.
`add_points` receives two structures. Like every argument, they are
copies (lesson 5.3), so `add_points` may change its `a` freely: `start` in
`main` is not affected. It returns the changed copy.

Step into `add_points` and look at its frame: it holds two whole
structures, copies of `start` and `step`, each with its own `x` and `y`.
When `a.x` and `a.y` turn amber, look down at `main`'s `start`: it has not
changed. Then the returned structure is copied into `end`.

Copying a small structure such as a point costs nothing worth mentioning.
Copying a large one, with an array of a thousand elements inside it, at
every call would be wasteful, and that is one reason for the next lesson.

### Check yourself

1. Why does `start` still hold (1, 2) after the call to `add_points`?
   <details><summary>Answer</summary>

   `add_points` received a copy of `start` in its parameter `a`, and
   changed only that copy.
   </details>

2. Can a function return an array the way `make_point` returns a
   structure?
   <details><summary>Answer</summary>

   No, C does not allow an array as a return type. Curiously, an array
   inside a structure can be returned, because the whole structure is
   copied.
   </details>

### A common mistake

Expecting a function to change a structure that is passed to it, as in
`void move_right(Point p) { p.x++; }`. It changes its own copy, which
vanishes. To change the caller's structure, pass its address, as the next
lesson shows.

## 9.3 Pointers to structures, and `->`

When a function must change a structure, or when the structure is large,
we pass its address instead. The parameter is then a pointer to the
structure, and the members are reached through it.

Through a pointer `p`, the member `balance` is `(*p).balance`: first `*p`,
the structure the pointer points to, then its member. The parentheses are
needed, because `.` binds more tightly than `*`. This happens so often that
C has a shorter way to write it: `p->balance` means exactly `(*p).balance`.
The arrow is meant to suggest following the pointer to the member.

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

Step into `deposit`. Its frame holds `acc`, a single pointer, 8 bytes,
with an arrow down into `main`'s frame to `mine`. Nothing was copied but
the address. When line 9 runs, the box that turns amber is `mine.balance`,
in `main`'s frame, at the end of the arrow.

### Check yourself

1. What would `*acc.balance` mean, without the parentheses?
   <details><summary>Answer</summary>

   `*(acc.balance)`: take the member `balance` of `acc`, which is not
   possible since `acc` is a pointer, and then follow it as a pointer. The
   compiler rejects it. That is why `->` exists.
   </details>

2. When is it fine to pass a structure by value instead?
   <details><summary>Answer</summary>

   When it is small, such as a point or a date, and the function only
   needs to read it.
   </details>

### A common mistake

Using `.` on a pointer, as in `acc.balance`, or `->` on a structure, as in
`mine->balance`. The compiler catches both. Read its message as "the
wrong one of the two operators": `.` for a structure, `->` for a pointer to
one.

## 9.4 Structures inside structures, and tables

A member of a structure can be of any type, including an array or another
structure. A rectangle, for instance, can be described by two corners,
each of them a point:

```c
struct rect {
    struct point top_left;
    struct point bottom_right;
};
```

If `r` is a `struct rect`, then `r.top_left` is a `struct point`, and
`r.top_left.x` is an `int`. The dots simply chain.

### Arrays of structures

An array whose elements are structures is a table: one structure per row,
one member per column. It is the natural way to keep records, such as the
parts in a small warehouse:

```c run
#include <stdio.h>

struct part {
    int number;
    char name[12];
    int on_hand;
};

struct part *find_part(struct part table[], int n, int number);

int main(void) {
    struct part inventory[3] = {
        {528, "disk drive", 10},
        {914, "printer", 5},
        {37, "cable", 120},
    };
    int total = 0;
    for (int i = 0; i < 3; i++)
        total += inventory[i].on_hand;
    printf("%d items in stock\n", total);

    struct part *p = find_part(inventory, 3, 914);
    if (p != NULL) {
        p->on_hand -= 2;
        printf("%s: %d left\n", p->name, p->on_hand);
    }
    return 0;
}

struct part *find_part(struct part table[], int n, int number) {
    for (int i = 0; i < n; i++)
        if (table[i].number == number)
            return &table[i];
    return NULL;
}
```

The initializer has one pair of braces per structure, and the string
`"disk drive"` is copied into the member `name`, which is an array of
twelve characters inside the structure.

`find_part` searches the table for a part number. If it finds it, it
returns a pointer to that row of the table; if not, it returns `NULL`, the
pointer that points nowhere (lesson 6.2). `main` checks for `NULL` before
using the pointer, and then changes the stock through it.

Step through it and look at how the table lies in memory: three
structures one after the other, each with its `name` drawn as a row of
characters inside it. When `find_part` returns, `p`'s arrow points to the
second row, and the subtraction on line 24 turns `inventory[1].on_hand`
amber.

### Check yourself

1. What is `sizeof inventory`?
   <details><summary>Answer</summary>

   60: three structures of 4 + 12 + 4 = 20 bytes each.
   </details>

2. Why does `find_part` return a pointer instead of a copy of the part?
   <details><summary>Answer</summary>

   So that the caller can change the part in the table, as `main` does,
   and so that "not found" can be said with `NULL`.
   </details>

### A common mistake

Assigning a string to an array member: `inventory[i].name = "mouse";`.
An array cannot be assigned, not even inside a structure. Copy the
characters with `strcpy` from lesson 8.5, after checking that they fit.

## 9.5 Padding and alignment

How big is a structure? The obvious answer, the sum of the sizes of its
members, is often wrong, and the reason lies in the hardware.

A processor does not fetch memory one byte at a time. It reads it in
pieces of 4 or 8 bytes, starting at addresses that are multiples of 4 or
8. An `int` that starts at an address divisible by 4 can be read in one
go; one that straddles the boundary between two pieces needs two reads,
or, on some processors, cannot be read at all. So compilers place each
value at an address that is a multiple of its size: an `int` at a multiple
of 4, a `double` at a multiple of 8. This is called *alignment*.

Inside a structure, the compiler keeps every member aligned too. Where a
member would start at an unsuitable address, it leaves a few unused bytes
before it, called *padding*. It also pads the end of the structure, so
that in an array of structures each one starts properly aligned. As a
result, the order of the members can change the size of the structure.

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
    struct loose a = {'a', 1, 'b'};
    struct tight b = {1, 'a', 'b'};
    printf("loose: %zu bytes, tight: %zu bytes\n",
           sizeof(struct loose), sizeof(struct tight));
    return 0;
}
```

In `struct loose`, `tag` takes one byte at offset 0. `value` must start
at a multiple of 4, so three bytes of padding come first, and `value`
lies at offsets 4 to 7. `flag` takes offset 8, and three more bytes of
padding round the size up to 12, a multiple of 4. In `struct tight`,
`value` comes first, the two `char`s follow at offsets 4 and 5, and two
bytes of padding make the size 8.

Step through it and look at `a` and `b` in the memory picture. Vizu draws
the padding bytes as boxes of their own, marked "padding", so you can see
exactly where the gaps are.

### Check yourself

1. What is the size of `struct { char c; double d; }`?
   <details><summary>Answer</summary>

   16: `c` at offset 0, seven bytes of padding, and `d` at offset 8.
   </details>

2. Does the padding matter in practice?
   <details><summary>Answer</summary>

   An array of a million `struct loose` takes 12 MB, of `struct tight` 8
   MB. And when structures are written to a file or sent over a network,
   the padding goes with them.
   </details>

### A common mistake

Writing structures to a file with one program and reading them with
another, compiled for a different computer. Sizes, padding and even the
order of the bytes inside a number (lesson 12.3) may differ, and the data
comes out scrambled.

## 9.6 Enumerations

Many values in programs are one of a small, fixed set: a day of the week,
a direction, a suit of cards, the state of a traffic light. We could
number them, 0 for Monday, 1 for Tuesday, but then the program is full of
numbers whose meaning the reader has to remember. An *enumeration* gives
the values names.

```c run
#include <stdio.h>

enum weekday { MON, TUE, WED, THU, FRI, SAT, SUN };

const char *day_name(enum weekday d) {
    switch (d) {
    case SAT: case SUN:
        return "weekend";
    default:
        return "working day";
    }
}

int main(void) {
    enum weekday today = FRI;
    int left = SAT - today;
    printf("day %d is a %s; %d day(s) to the weekend\n", today, day_name(today), left);
    return 0;
}
```

`enum weekday { MON, TUE, … }` declares a type and seven constants. The
constants are simply integers: `MON` is 0, `TUE` is 1, and so on, each one
more than the one before. A different value can be given to any of them,
as in `enum { LOW = 10, MID, HIGH }`, where `MID` then is 11 and `HIGH` 12.

Since they are integers, they can be compared, subtracted and used as
`case` labels, as `day_name` does. And the program reads far better than
with bare numbers: `today = FRI` says what it means.

In the memory picture, `today` is an ordinary 4-byte box holding 4. The
names exist only in the program text, like the names made with
`#define`.

### Check yourself

1. What does the program print?
   <details><summary>Answer</summary>

   `day 4 is a working day; 1 day(s) to the weekend`.
   </details>

2. What are the values in `enum { A = 1, B, C = 10, D }`?
   <details><summary>Answer</summary>

   1, 2, 10 and 11. Each constant without a value is one more than the one
   before it.
   </details>

### A common mistake

Assuming that C checks enumeration values. It does not: `enum weekday d =
42;` compiles without complaint. An enumeration documents the intention,
but it does not enforce it.

## 9.7 Unions

A *union* is declared like a structure, but its members do not lie one
after the other. They all start at the same address and share the same
bytes. A union is therefore only as big as its largest member, and it can
hold only one of its members at a time: writing one member overwrites the
others.

Why would anyone want that? To save space, when a value can be one of
several kinds but never more than one at once. A spreadsheet cell, for
instance, holds either a number or a piece of text. A value in a
calculator might be either an `int` or a `double`. To know which member a
union currently holds, it is usually put inside a structure together with
a *tag*, an enumeration that says which member is in use. This
combination is called a *tagged union*.

```c run
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
    printf("union: %zu bytes, struct number: %zu bytes\n", sizeof a.u, sizeof a);
    return 0;
}
```

The union `u` has two members, an `int` and a `double`. Its size is 8,
the size of the `double`, not 12. Every function that uses a `struct
number` first looks at `kind` to know which member to read, as
`print_number` does.

Step through it and watch the union inside `a` and `b`. Vizu draws a union
with the member that was written last. After line 23, `a.u` shows `.i`
holding 42, followed by a dashed box of four bytes labelled "rest of the
union": those bytes belong to the union, but only the `double` would use
them. After line 25, `b.u` shows `.d` holding 3.14, filling all eight
bytes. Notice also the four bytes of padding after each `.kind`: the union
contains a `double`, so it must start at a multiple of 8. The sizes printed
at the end confirm all of it: 8 bytes for the union, and 16 for the whole
structure.

### One value, two readings

Because the members share their bytes, a union can also show how a value
is stored. Writing a `float` and reading the same four bytes as an
`unsigned int` reveals the bits of the floating-point number:

```c run
#include <stdio.h>

union bits {
    float f;
    unsigned int n;
};

int main(void) {
    union bits x;
    x.f = 1.0f;
    printf("1.0 is stored as %08x\n", x.n);
    x.f = -1.0f;
    printf("-1.0 is stored as %08x\n", x.n);
    return 0;
}
```

The two lines differ only in their first hexadecimal digit, 3 against B,
that is `0011` against `1011` in binary: the very first bit is the sign.
Module 12 looks at bits like these in detail.

### Check yourself

1. What is `sizeof a.u`, and why?
   <details><summary>Answer</summary>

   8, the size of its largest member, the `double`. All its members start
   at the same address, so it never needs more.
   </details>

2. What is the tag `kind` for?
   <details><summary>Answer</summary>

   The union itself does not remember which member was written last. The
   tag records it, so that the program reads the member that really holds
   the value.
   </details>

### A common mistake

Writing one member of a union and reading another by accident, such as
`a.u.i = 42;` followed by `printf("%f", a.u.d)`. The program reads the
four bytes of the `int` plus four bytes that were never written, as if
they were a `double`, and prints nonsense. Vizu warns when the bytes read
were never written. Keeping a tag, and checking it, prevents the mistake.
