# Module 5 · Functions

Every program so far has been a single function, `main`, and as programs
grow, that stops working. A long `main` is hard to read, because the reader
has to keep all of it in mind at once. It is hard to change, because a
change in one place may break something forty lines further down. And it
repeats itself: if a program needs to raise numbers to a power in three
places, it contains the same loop three times.

The cure is to divide the program into *functions*: small named pieces,
each doing one job, which the rest of the program can *call* whenever it
needs that job done. We have been calling functions from the start,
`printf`, `scanf` and `getchar`, without needing to know how they work
inside. That is exactly the point. A well-made function can be used by
someone who knows only what it does, not how.

This module shows how to write functions, and what happens in memory when
one is called. That second part is the key to the whole module, and to the
modules on pointers that follow. Each time a function is called, it gets a
fresh set of boxes of its own, called a *stack frame*, placed on top of the
caller's. When the function returns, its frame is removed. Once you can see
frames being added and removed, arguments, scope and recursion stop being
mysterious.

## 5.1 Defining and calling a function

Here is a program with a function of our own, `power`, which raises a
number to a whole power: `power(2, 3)` is 2 × 2 × 2, which is 8.

```c run
#include <stdio.h>

/* power: base raised to the n-th power, for n >= 0 */
int power(int base, int n) {
    int p = 1;
    for (int i = 0; i < n; i++)
        p = p * base;
    return p;
}

int main(void) {
    for (int i = 0; i < 4; i++)
        printf("%d %d %d\n", i, power(2, i), power(-3, i));
    return 0;
}
```

### The definition

Lines 4 to 9 are the *definition* of `power`. The first line of it says
three things. The `int` at the front is the type of the value the function
gives back, its *return type*. Then comes the name, `power`. Then, in
parentheses, the *parameters*: the values the function needs in order to do
its job, each with its type and a name. `power` needs a base and an
exponent.

The body, between the braces, does the work. It may declare variables of
its own, as `power` declares `p` and `i`. These are *local* variables:
they belong to this function alone. The `i` in `power` has nothing to do
with the `i` in `main`; they merely share a name.

The statement `return p;` ends the function and hands the value of `p`
back to whoever called it.

### The call

In `main`, `power(2, i)` is a *call*. The values in the parentheses are the
*arguments*. When the call is made, the arguments are worked out first,
and their values become the starting values of the parameters: `base`
starts as 2 and `n` starts as the current value of `main`'s `i`. Then the
body of `power` runs. When it reaches `return`, the call is over, and the
value returned takes the place of the call in `main`'s expression, just as
if `power(2, i)` had been replaced by a number.

### Watching the frames

This is the most important program in the module to step through slowly.

Start in `main`. Its frame is at the bottom of the stack, with its `i`.
When the first call to `power` happens, a new frame appears on top of
`main`'s. It holds `power`'s parameters, `base` and `n`, already filled
with the argument values, and room for its locals, `p` and `i`. Look: there
are now two boxes called `i`, one in each frame, and they are separate
boxes at different addresses.

Step on through `power`'s loop. When it reaches `return p`, the frame
shows the value being returned, and in the next step the frame is gone.
`main`'s frame was below it the whole time, waiting at the line of the
call, and now the program continues there, with the returned value in
hand.

Each call makes a new frame and each return removes it. That is why the
region is called a *stack*: like a stack of plates, the last frame put on
is always the first to be taken off.

### Check yourself

1. How many times is `power` called while the program runs?
   <details><summary>Answer</summary>

   Eight: twice in each of the four rounds of `main`'s loop.
   </details>

2. While `power` is running, how many frames are on the stack?
   <details><summary>Answer</summary>

   Two: `main`'s at the bottom and `power`'s on top. (`printf` gets a frame
   too, when it is called, but Vizu shows only the functions of your own
   program.)
   </details>

### A common mistake

Forgetting the `return` in a function that promises a value. The caller
still receives something, whatever happened to be in the place where
return values are passed, and the program carries on with that garbage.
Compilers warn about it if asked, with the option `-Wall`.

## 5.2 Declarations, `void` and `return`

### Declare before use

The compiler reads a program from top to bottom, once. When it meets a
call, it wants to know already what the function looks like: how many
arguments it takes, of which types, and what it returns. Otherwise it
cannot check the call or convert the arguments correctly. In the last
lesson, `power` was defined above `main`, so that was no problem.

Often it is nicer to put `main` first, so that a reader sees the overall
plan before the details. Then each function must be *declared* before
`main`, with a line that repeats its first line and ends in a semicolon:

```c
int power(int base, int n);
```

Such a declaration is called a *function prototype*. It promises the
compiler that a function with this name and these types exists somewhere,
and the compiler takes the promise on trust. The definition can then come
anywhere later.

### Functions that return nothing, and functions that take nothing

Not every function computes a value. A function that only prints
something, for instance, has nothing to hand back. Its return type is
written `void`, which means "nothing", and in such a function `return;`
without a value ends it early, while reaching the closing brace ends it
normally. In the same way, a function that needs no arguments has `void` in
its parentheses, as `main` does in our programs.

### `return` can come anywhere

A function may contain several `return` statements, and whichever one is
reached first ends it. This is often the clearest way to write a test:

```c run input="91\n"
#include <stdio.h>
#include <stdbool.h>
#include <stdlib.h>

bool is_prime(int n);

int main(void) {
    int n;
    printf("Enter a number: ");
    if (scanf("%d", &n) != 1) {
        printf("That is not a number.\n");
        exit(EXIT_FAILURE);
    }
    if (is_prime(n))
        printf("%d is prime\n", n);
    else
        printf("%d is not prime\n", n);
    return 0;
}

bool is_prime(int n) {
    if (n < 2)
        return false;
    for (int d = 2; d * d <= n; d++)
        if (n % d == 0)
            return false;
    return true;
}
```

`is_prime` tries every possible divisor `d`. As soon as one divides `n`
exactly, the answer is known, and the function returns `false` on the spot,
in the middle of the loop. Only if the loop finishes without finding one
does it reach `return true`. (It stops trying at the square root of `n`,
`d * d <= n`, because if `n` had a divisor larger than that, it would also
have a smaller one, which would already have been found.)

Step through it with the input 91. The loop tries 2, 3, 4, 5 and 6, each
time answering no to `n % d == 0`. At 7 the answer is yes, since 91 is 7 ×
13, and the frame of `is_prime` shows that it returns false.

### Ending the whole program

`return` in `main` ends the program, but sometimes a program must stop from
deep inside some other function. The library function `exit`, declared in
`<stdlib.h>`, ends the program wherever it is called, and its argument
becomes the program's exit status, just like the value returned from
`main`. `EXIT_SUCCESS` and `EXIT_FAILURE` are names for the usual two
values. The program above uses it when the input is not a number, since
there is no sense in going on.

### Check yourself

1. What happens if the prototype of `is_prime` is removed?
   <details><summary>Answer</summary>

   When the compiler meets the call in `main`, it has not seen `is_prime`
   yet. A modern compiler reports an error or a warning; an old one would
   guess that the function returns an `int`, and a wrong guess makes the
   program misbehave.
   </details>

2. How many times does the loop in `is_prime` test a divisor when `n` is
   97, a prime?
   <details><summary>Answer</summary>

   Eight times, for `d` from 2 to 9; at 10, `10 * 10 <= 97` is false and
   the loop ends.
   </details>

### A common mistake

A prototype that does not match the definition, such as `int area(int w,
int h);` above `main` and `double area(double w, double h)` below it. The
compiler sees the mismatch in a single file, but in a program of several
files it cannot, and the call then passes the arguments in the wrong form.

## 5.3 Arguments are copies

What happens if a function changes one of its parameters? Try it:

```c run
#include <stdio.h>

void double_it(int x) {
    x = x * 2;
    printf("inside: %d\n", x);
}

int main(void) {
    int n = 21;
    double_it(n);
    printf("outside: %d\n", n);
    return 0;
}
```

It prints `inside: 42` and then `outside: 21`. The function doubled the
number, but `main`'s `n` did not change.

The reason is the way C passes arguments, which is called *passing by
value*. When `double_it(n)` is called, the parameter `x` is a new box in
`double_it`'s frame, and it starts with a *copy* of the value of `n`. From
then on, the two boxes have nothing to do with each other. Changing `x`
changes only the copy.

Step through it and you can see this plainly. At line 4, `x` in the top
frame turns amber with 42. Look down at `main`'s frame: `n` still holds 21,
and it never changes.

This rule has a good side. Since a parameter is the function's private
copy, the function may use it freely as a working variable, without any
risk to the caller. `power` from lesson 5.1 could have counted `n` down
instead of using `i`, and the caller's exponent would have been safe.

The copy is also converted to the parameter's type, as in an assignment.
Calling `power(2.9, 3)` passes 2, not 2.9, because `base` is an `int`.

### Check yourself

1. How can `main` get the doubled value?
   <details><summary>Answer</summary>

   By having the function return it, `int doubled(int x) { return x * 2;
   }`, and writing `n = doubled(n);`. Or by passing the address of `n`,
   which module 6 explains.
   </details>

2. `scanf("%d", &n)` does change `n`. How, if arguments are copies?
   <details><summary>Answer</summary>

   `scanf` receives a copy of the *address* of `n`. A copy of an address
   still leads to the same box, and `scanf` writes there.
   </details>

### A common mistake

Writing a function that tries to hand back a result through a parameter:
`void read_age(int age) { scanf("%d", &age); }`. It reads the number into
its own copy, which disappears with its frame, and the caller's variable is
untouched.

## 5.4 Arrays as arguments

Arrays are passed to functions differently, and the difference matters.
An array is never copied into a function. Copying a large array at every
call would be slow, so instead the function receives the address of the
array's first element. The parameter is written `int a[]`, but what the
function really holds is an address: through it, `a[i]` reaches the
caller's own elements.

This has two consequences. First, a function *can* change the elements of
an array that is passed to it, and the caller will see the change. Second,
the function has no way of knowing how long the array is, since all it has
is the address of the start. The length must therefore be passed as a
separate argument.

```c run
#include <stdio.h>

int sum(int a[], int n) {
    int s = 0;
    for (int i = 0; i < n; i++)
        s += a[i];
    return s;
}

void zero_first(int a[]) {
    a[0] = 0;
}

int main(void) {
    int nums[5] = {3, 1, 4, 1, 5};
    printf("%d\n", sum(nums, 5));
    zero_first(nums);
    printf("%d %d\n", nums[0], sum(nums, 5));
    return 0;
}
```

Step into `sum` and look at its frame. `a` is not drawn as a row of five
boxes. It is a single box with an arrow leading out of it, down into
`main`'s frame, to `nums`. That arrow is the address it holds. Then step
into `zero_first`: when line 11 writes `a[0] = 0`, the box that turns amber
is `main`'s `nums[0]`, at the other end of the arrow.

### Check yourself

1. What does the program print?
   <details><summary>Answer</summary>

   `14`, then `0 11`. `zero_first` changed `main`'s array.
   </details>

2. How could `sum` promise not to change the array?
   <details><summary>Answer</summary>

   By declaring the parameter `const int a[]`. The compiler then refuses
   any assignment to an element. Lesson 6.6 explains this use of `const`.
   </details>

### A common mistake

Working out the length inside the function:

```c run
#include <stdio.h>

int count(int a[]) {
    return sizeof a / sizeof a[0];
}

int main(void) {
    int nums[5] = {3, 1, 4, 1, 5};
    printf("%zu %d\n", sizeof nums / sizeof nums[0], count(nums));
    return 0;
}
```

In `main`, `sizeof nums` is the size of the whole array, 20 bytes, and the
length comes out as 5. In `count`, `a` is only an address, and an address
takes 8 bytes, so the "length" comes out as 2. The `sizeof` trick of lesson
4.2 works only where the array itself is declared.

## 5.5 Scope and lifetime

Two separate questions can be asked about any variable. Where in the
program can its name be used? That is its *scope*. And when does its box
exist? That is its *lifetime*. For the variables we have used so far, the
two questions had the same answer, but they need not.

### Local variables

A variable declared inside a block, between braces, is *local* to that
block. Its scope is the block: from the declaration to the closing brace,
and nowhere else. Its lifetime is also the block: the box is made when the
program enters the block and removed when it leaves. Such variables are
called *automatic*, because this happens by itself. Parameters are local
variables of the same kind, whose first values come from the arguments.

### Static local variables

The word `static` in front of a local variable changes its lifetime but
not its scope. The variable is still known only inside its block, but its
box is made once, before the program starts, and lives until the program
ends. Its initializer is applied only that once. So the variable keeps its
value from one call of the function to the next.

### External variables

A variable declared outside all functions is *external*, often called
*global*. Its scope is the rest of the file from the declaration onwards,
so every function below it can use it, and it lives for the whole run.

```c run
#include <stdio.h>

int calls = 0;              /* external: lives for the whole run */

int next_id(void) {
    static int id = 100;    /* static: made once, keeps its value */
    int step = 1;           /* automatic: made at each call */
    calls++;
    id += step;
    return id;
}

int main(void) {
    int a = next_id();
    int b = next_id();
    int c = next_id();
    printf("%d %d %d after %d calls\n", a, b, c, calls);
    return 0;
}
```

Step through it and look at where each variable lives. `calls` and `id`
are not on the stack at all. They are drawn in a separate area, the
globals, above the stack, where they stay for the whole run and keep their
values between the calls. `step`, on the other hand, appears in each new
frame of `next_id`, starts at 1 every time, and vanishes with the frame.

### When to use which

Local variables should be your first choice. A function whose data is all
in its parameters and locals can be understood by reading that function
alone. An external variable can be changed by any function in the file, so
when its value is wrong, every function is a suspect. They are useful for
data that truly belongs to the whole program, but each one makes a large
program a little harder to follow.

### Check yourself

1. What does the program print?
   <details><summary>Answer</summary>

   `101 102 103 after 3 calls`.
   </details>

2. Can `main` use the name `id`?
   <details><summary>Answer</summary>

   No. The box of `id` exists for the whole run, but its name is known only
   inside `next_id`. Lifetime and scope are different things.
   </details>

### A common mistake

Giving a local variable the same name as an external one. Inside the
block, the local name *hides* the external one, and the function quietly
works on the local box while the author thinks it is changing the global.

## 5.6 Recursion

A function may call itself. That sounds like a circle that could never
end, but it is a natural way to solve a problem that contains a smaller
copy of itself. The factorial of a number, written n!, is the product of
all whole numbers from 1 to n: 5! = 5 × 4 × 3 × 2 × 1. Notice that 5! is
simply 5 × 4!, and 4! is 4 × 3!, and so on down to 1! = 1. Written in C,
that observation is a complete function:

```c run
#include <stdio.h>

int fact(int n) {
    if (n <= 1)
        return 1;
    return n * fact(n - 1);
}

int main(void) {
    printf("%d\n", fact(5));
    return 0;
}
```

A function that calls itself is *recursive*. It works because of the frames.
Each call to `fact` gets a frame of its own, with its own `n`, so the calls
do not disturb each other. `fact(5)` needs `fact(4)`, so it calls it and
waits; `fact(4)` calls `fact(3)` and waits; and so on. Every recursive
function needs at least one case that does not call itself, here `n <= 1`,
which is called the *base case*. Without it, the calls would never stop.

Step through it and watch the stack. Five frames of `fact` pile up on top
of `main`'s, with `n` equal to 5, 4, 3, 2 and 1. The top one reaches the
base case and returns 1. Then the frames come off one at a time, and each
shows the value it returns: 1, then 2, then 6, then 24, and finally 120.

### Work on the way back

A recursive function can do its work after the call returns as well as
before. This is useful whenever things come out in the wrong order. To
print the digits of a number, it is easy to get the last digit, `n % 10`,
but we want to print the first digit first. So we first print all the
digits but the last, which is the same problem on a smaller number, `n /
10`, and then print the last one:

```c run
#include <stdio.h>

void print_digits(int n) {
    if (n >= 10)
        print_digits(n / 10);
    putchar(n % 10 + '0');
}

int main(void) {
    print_digits(4096);
    putchar('\n');
    return 0;
}
```

Watch which call reaches `putchar` first. It is the deepest one, where `n`
is 4. Its 4 is printed, its frame is removed, and the call below it, with
`n` equal to 40, prints its 0, and so on back down the stack.

### Check yourself

1. At most, how many frames are on the stack while `print_digits(4096)`
   runs?
   <details><summary>Answer</summary>

   Five: `main`'s and four of `print_digits`, one for each digit.
   </details>

2. Why does `n % 10 + '0'` give a printable digit?
   <details><summary>Answer</summary>

   `n % 10` is a number from 0 to 9, and adding the code of `'0'` gives the
   code of the matching digit character, as in lesson 2.4.
   </details>

### A common mistake

Forgetting the base case:

```c run expect="error"
int fact(int n) {
    return n * fact(n - 1);
}

int main(void) {
    return fact(3);
}
```

Every call makes another call, and every call needs a new frame, until the
stack, which is not large, is full. A real program then dies with a *stack
overflow*. Vizu stops it the same way, reports the overflow, and asks
whether the recursion ever reaches its base case.

## 5.7 Sorting with recursion: quicksort

Recursion is not only for small mathematical examples. One of the fastest
ways to sort an array, *quicksort*, is recursive, and it is a good example
of the main idea behind recursion: split a big problem into smaller
problems of the same kind.

Quicksort works like this. Choose one element of the array, called the
*pivot*. Then rearrange the array so that every element smaller than the
pivot comes before it and every larger element after it. After that, the
pivot is in its final place: it never needs to move again. What remains
are two smaller arrays, the part before the pivot and the part after it,
and each of them is sorted in exactly the same way, by calling quicksort on
it. A part with one element or none is already sorted, which is the base
case.

```c run
#include <stdio.h>

void quicksort(int a[], int low, int high);
int split(int a[], int low, int high);

int main(void) {
    int a[7] = {12, 3, 6, 18, 7, 15, 10};
    quicksort(a, 0, 6);
    for (int i = 0; i < 7; i++)
        printf("%d ", a[i]);
    printf("\n");
    return 0;
}

void quicksort(int a[], int low, int high) {
    if (low >= high)
        return;
    int middle = split(a, low, high);
    quicksort(a, low, middle - 1);
    quicksort(a, middle + 1, high);
}

/* Moves a[low], the pivot, to its final place: smaller values
   before it, larger ones after it. Returns where it ended up. */
int split(int a[], int low, int high) {
    int pivot = a[low];
    for (;;) {
        while (low < high && pivot <= a[high])
            high--;
        if (low >= high)
            break;
        a[low++] = a[high];
        while (low < high && a[low] <= pivot)
            low++;
        if (low >= high)
            break;
        a[high--] = a[low];
    }
    a[high] = pivot;
    return high;
}
```

`quicksort` sorts the part of the array from index `low` to index `high`.
It is short, because `split` does the real work. `split` takes the first
element as the pivot and keeps a copy of it, which leaves a "hole" at
`a[low]`. It then works inwards from both ends. From the right, it looks
for an element smaller than the pivot and moves it into the hole on the
left, which leaves a hole on the right. From the left, it looks for an
element larger than the pivot and moves it into the hole on the right.
When the two ends meet, the hole is exactly where the pivot belongs, and
the pivot goes into it.

This is a longer run, so use the slider as well as the buttons. Things to
watch: the array lives in `main`'s frame, and every frame of `quicksort` and
`split` has an arrow pointing to it, so all of them work on the same seven
elements. During the first `split`, values jump from one end to the other,
each turning amber, until 12 lands in position 4 with the smaller values on
its left and the larger ones on its right. Then the stack grows as
`quicksort` calls itself on the left part, and shrinks and grows again for
the right part. Each frame has its own `low` and `high`, so each knows which
part of the array it is responsible for.

### Check yourself

1. After the first call to `split`, what does the array look like, and
   what does `split` return?
   <details><summary>Answer</summary>

   `10 3 6 7 12 15 18`, and it returns 4: the pivot 12 is at index 4, with
   the four smaller values before it and the two larger ones after it.
   </details>

2. Why is quicksort fast?
   <details><summary>Answer</summary>

   Each `split` looks at every element of its part once, and if the pivot
   lands near the middle, each level of recursion halves the parts. An
   array of a million elements is then sorted in about twenty levels.
   (If the pivot is always the smallest element, as for an array that is
   already sorted, the parts shrink by only one each time, and quicksort
   becomes slow.)
   </details>

### A common mistake

Passing `middle` instead of `middle - 1` and `middle + 1` to the two
recursive calls. The pivot is then included in a part again, and for some
arrays a part never gets smaller, so the recursion never ends.

## 5.8 Programs in many files

Real programs are much too large for a single file. A program is split
into several *source files*, each holding functions that belong together,
and each file can be compiled on its own. Here is a tiny example, a counter
kept in a file of its own.

```c
/* counter.h */
#ifndef COUNTER_H
#define COUNTER_H

void counter_add(int n);
int counter_value(void);

#endif
```

```c
/* counter.c */
#include "counter.h"

static int total = 0;     /* static: invisible outside this file */

void counter_add(int n) { total += n; }
int counter_value(void) { return total; }
```

```c
/* main.c */
#include <stdio.h>
#include "counter.h"

int main(void) {
    counter_add(5);
    counter_add(7);
    printf("%d\n", counter_value());
    return 0;
}
```

### Headers

`main.c` calls functions that are defined in `counter.c`, so, as lesson
5.2 explained, it needs their prototypes. We could copy the prototypes
into every file that uses them, but then there would be several copies to
keep in step. Instead, the prototypes go into a *header file*, `counter.h`,
and every file that needs them includes it. A header written in double
quotes, `#include "counter.h"`, is looked for first among the program's own
files; one in angle brackets, like `<stdio.h>`, among the system's.

The lines `#ifndef COUNTER_H`, `#define COUNTER_H` and `#endif` are an
*include guard*. In a large program the same header can easily be
included twice, through other headers, and the guard makes sure that its
contents are seen only once: the first time, `COUNTER_H` is not yet
defined, so the contents are read and `COUNTER_H` is defined; any later
time, everything up to `#endif` is skipped.

The word `static` in front of `total` has a new meaning here. On a
variable outside all functions, it does not change the lifetime, which is
the whole run anyway; it makes the name private to its file. No other file
can use `total`, so the only way to change the counter is through the
functions. This is how C hides the inner workings of one part of a program
from the others.

### Building

Each source file is compiled separately into an *object file*, which
contains its machine instructions and a list of the names it defines and
the names it needs from elsewhere. The *linker* then joins the object files
and the library, connecting every use of a name with its definition:

```text
gcc -c counter.c
gcc -c main.c
gcc counter.o main.o -o counter
```

or all at once, `gcc counter.c main.c -o counter`. When only one file
changes, only that file needs compiling again, which saves a great deal of
time in a large program. A *makefile* records which files depend on which,
so that the program `make` can rebuild exactly what has changed.

Vizu runs programs of a single file, so this lesson has no lab.

### Check yourself

1. What does `static` mean on `total`?
   <details><summary>Answer</summary>

   That the variable is private to `counter.c`: other files cannot use its
   name. On a variable outside all functions, `static` is about who may see
   it, not about how long it lives.
   </details>

2. A program compiles, but the linker reports "undefined reference to
   `counter_add`". What is missing?
   <details><summary>Answer</summary>

   `counter.c`, or `counter.o`, in the command that links the program. The
   header promised that the function exists; the linker could not find its
   code.
   </details>

### A common mistake

Defining a variable in a header, as in `int total = 0;` in `counter.h`.
Every file that includes the header then defines its own `total`, and the
linker complains that the name is defined more than once. A header only
*declares* a shared variable, `extern int total;`, and exactly one source
file defines it.
