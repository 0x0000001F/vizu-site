# Module 13 · Input, output and the library

The C language itself is small. It has no way to print, to read a file,
to take a square root or to ask what time it is. All of that comes from
the *standard library*, the collection of functions that every C system
provides, declared in headers such as `<stdio.h>`, `<stdlib.h>` and
`<math.h>`. We have used the library since the first lesson; this module
looks at it more closely.

Most of the module is about input and output. They deserve the attention,
because they are where a program meets the world outside it, and because
there is more going on than meets the eye. `printf` does not draw letters on
a screen. It asks the operating system to do so, through a narrow doorway
called a *system call*, and the library works hard to use that doorway as
rarely as possible. The strip at the bottom of every lab has been showing
this since lesson 1.1; here we explain it properly.

After input and output come the functions for numbers, random numbers and
time, and finally the ways a C program reports that something has gone
wrong.

## 13.1 Streams: `stdin`, `stdout` and `stderr`

A C program does not read "the keyboard" or write "the screen". It reads
and writes *streams*: sources and destinations of characters, which the
library manages. Every program starts with three of them already open:

- `stdin`, the standard input, where `scanf` and `getchar` read from;
- `stdout`, the standard output, where `printf` and `putchar` write to;
- `stderr`, the standard error, for error messages.

Normally all three are connected to the terminal: input comes from the
keyboard, and both outputs go to the screen. But the person who starts the
program can connect them elsewhere, without the program knowing. On the
command line, `./upper < notes.txt` makes `stdin` read from the file
`notes.txt`, and `./upper > loud.txt` sends `stdout` into the file `loud.txt`.
`stderr` stays on the screen, which is exactly why it exists: when the
normal output is going into a file, error messages should still reach the
person.

A program that reads `stdin` and writes `stdout` is called a *filter*.
Filters can be chained, the output of one feeding the input of the next,
with `|` in the shell: `./upper < notes.txt | sort` converts the file and
sorts its lines. Small filters combined like this are a central idea of
Unix.

```c run input="hello\nworld\n"
#include <stdio.h>
#include <ctype.h>

int main(void) {
    int c;
    while ((c = getchar()) != EOF)
        putchar(toupper(c));
    fprintf(stderr, "done\n");
    return 0;
}
```

`fprintf` is `printf` with a stream as its first argument; `fprintf(stdout,
…)` would be the same as `printf(…)`, and `fprintf(stderr, …)` writes to the
standard error.

The end of the input, `EOF`, reaches the program when there is nothing
more to read: at the end of a file, or when the person at the keyboard
types Ctrl+D (on Linux and macOS) or Ctrl+Z and Enter (on Windows) at the
start of a line.

### Buffers

Step through the program and watch the strip under the memory. It shows
something important. The first `getchar` finds the input buffer empty, so
the library asks the operating system for input, and the kernel hands over
the whole line, `hello` and its newline, at once. The next five `getchar`s
take their characters from the buffer, without asking the kernel at all.
On the output side, each `putchar` puts one letter into `stdout`'s buffer,
and the letters wait there; only the newline sends the whole line to the
screen. `stderr`, on the other hand, has no buffer, and `done` goes out
the moment it is printed.

Why all this collecting? Because asking the operating system for anything
is slow compared with copying a byte inside the program, perhaps a
hundred times slower. A program that called the kernel once per character
would spend most of its time on calls. So the library gathers characters
and passes them on in batches. The rule for `stdout` depends on where it
goes: to a terminal it is *line buffered*, sent at each newline; to a file
it is *fully buffered*, sent when the buffer is full, typically every 4096
bytes. The switch on the strip, "Terminal" or "File", lets you see the
difference. `fflush(stdout)` sends whatever is waiting at once.

### Check yourself

1. Why might the output of `printf("Working...")` not appear on the screen
   right away?
   <details><summary>Answer</summary>

   It has no newline, so on a terminal it waits in `stdout`'s buffer until a
   newline comes, the buffer fills, or the program ends. `fflush(stdout)`
   forces it out.
   </details>

2. The program is run as `./upper < in.txt > out.txt`. Where does `done`
   appear?
   <details><summary>Answer</summary>

   On the screen. Only `stdout` was sent into the file; `stderr` still goes
   to the terminal.
   </details>

### A common mistake

Printing error messages to `stdout`. When the output is redirected into a
file, the person never sees the message, and the file contains it in the
middle of the real output.

## 13.2 Below the library: `read` and `write`

What does the library do when it finally passes text to the operating
system? It makes a *system call*: it switches the processor from the
program into the kernel, the core of the operating system, which alone may
talk to the hardware. On Linux and macOS the two calls for input and output
are `read` and `write`. They know nothing about formats, lines or `%d`; they
move bytes.

```c
#include <unistd.h>

/* cat: copy the input to the output, with no library buffering */
int main(void) {
    char buf[64];
    long n;
    while ((n = read(0, buf, sizeof buf)) > 0)
        write(1, buf, n);
    return 0;
}
```

`read(0, buf, 64)` asks the kernel for up to 64 bytes from *file
descriptor* 0, and returns how many it got, or 0 at the end of the input.
A file descriptor is the kernel's number for an open file or device: 0 is
the standard input, 1 the standard output, and 2 the standard error.
`write(1, buf, n)` gives `n` bytes to descriptor 1.

That is all `stdin` and `stdout` are, underneath: descriptors 0 and 1, plus
a buffer in the library. `getchar` takes the next byte from its buffer, and
when the buffer is empty it calls `read(0, …)` to fill it. `printf` formats
the text, puts it into `stdout`'s buffer, and when the buffer must be
emptied, calls `write(1, …)`.

Vizu does not run `read` and `write` themselves, but every lab shows them:
in the strip, the stage marked `write()` is this system call, with the bytes
it carries, and the kernel stage after it is the operating system doing its
part. "Open in detail" draws the same journey in layers: your program, the
library, the system call, the kernel, the hardware.

### Check yourself

1. How many system calls does `printf("a"); printf("b"); printf("c\n");`
   make on a terminal?
   <details><summary>Answer</summary>

   One `write`, for the three characters and the newline together, made when
   the newline arrives.
   </details>

2. Why does the `cat` above not need `<stdio.h>`?
   <details><summary>Answer</summary>

   It uses no library streams at all, only the system calls `read` and
   `write`, which are declared in `<unistd.h>`.
   </details>

### A common mistake

Mixing `printf` with `write(1, …)` in one program and expecting the output
in program order. `printf`'s text may still be waiting in the buffer when
`write` goes straight to the kernel, so it can appear later.

## 13.3 Strings as streams: `sprintf` and `sscanf`

`printf` and `scanf` have relatives that work on strings instead of
streams. `sprintf(buf, format, …)` formats exactly as `printf` would, but
writes the result into the array `buf`, with a `'\0'` at the end.
`snprintf` does the same and is told the size of the array, which makes it
safe (lesson 8.5). `sscanf(text, format, …)` reads from a string as `scanf`
would from the input.

These are the standard way to turn numbers into text and text into
numbers. For the second direction, `<stdlib.h>` also has `atoi`, which
reads an `int` from the start of a string, and the more careful `strtol`
and `strtod`, which also report where they stopped reading.

```c run
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    char date[16];
    snprintf(date, sizeof date, "%d-%02d-%02d", 2026, 10, 11);
    printf("%s\n", date);

    int y, m, d;
    int got = sscanf("1999/12/31", "%d/%d/%d", &y, &m, &d);
    printf("read %d numbers: %d %d %d\n", got, y, m, d);

    char *end;
    long weight = strtol("42 kg", &end, 10);
    printf("%ld, and the rest is \"%s\"\n", weight, end);
    printf("%ld %ld\n", strtol("ff", NULL, 16), strtol("101", NULL, 2));
    return 0;
}
```

`%02d` pads the month and the day with a zero to two digits, so the date
comes out as `2026-10-11`. `sscanf` returns the number of values it filled,
like `scanf`.

`strtol(text, &end, base)` reads a `long` in the given base, and stores in
`end` a pointer to the first character it did not use. That lets a program
check that a string really was a number, `*end == '\0'`, or go on reading
after it. Step past line 14 and look at the arrow from `end`: it points into
the string literal, just after the 42.

### Check yourself

1. What does `atoi("12abc")` return? And `atoi("abc")`?
   <details><summary>Answer</summary>

   12, and 0. `atoi` reads digits for as long as there are any. It cannot
   tell "0" from "not a number", which is why `strtol` is preferred.
   </details>

2. How would you check that a string `s` holds nothing but a number?
   <details><summary>Answer</summary>

   `char *end; long v = strtol(s, &end, 10);` and then require `end != s`
   (something was read) and `*end == '\0'` (nothing is left over).
   </details>

### A common mistake

Using `sprintf` with an array that is too small. Like `strcpy`, it has no
idea how big the array is. Use `snprintf` and give it the size.

## 13.4 Files

To read or write a file other than the standard streams, a program opens
it with `fopen(name, mode)`, which returns a `FILE *`: a pointer to the
library's record of the open stream, with its buffer. If the file cannot
be opened, because it does not exist or the program may not read it,
`fopen` returns `NULL`. The mode is a short string: `"r"` to read, `"w"` to
write (creating the file, or emptying it if it exists), `"a"` to append to
the end. Adding `b`, as in `"rb"`, opens it as a binary file.

All the stream functions have versions that take a `FILE *`: `fgetc`,
`fgets`, `fscanf` for reading, `fputc`, `fputs`, `fprintf` for writing.
When the program has finished with a file, `fclose` empties its buffer and
releases it.

```c
/* copy: copy one file to another, line by line */
#include <stdio.h>

int main(int argc, char *argv[]) {
    if (argc != 3) {
        fprintf(stderr, "usage: %s from to\n", argv[0]);
        return 1;
    }
    FILE *in = fopen(argv[1], "r");
    if (in == NULL) {
        perror(argv[1]);            /* prints the name and the reason */
        return 1;
    }
    FILE *out = fopen(argv[2], "w");
    if (out == NULL) {
        perror(argv[2]);
        fclose(in);
        return 1;
    }
    char line[256];
    while (fgets(line, sizeof line, in) != NULL)
        fputs(line, out);
    fclose(in);
    if (fclose(out) != 0) {         /* the last write can fail here */
        perror(argv[2]);
        return 1;
    }
    return 0;
}
```

The program takes the two file names from its command line (lesson 8.6),
checks each `fopen`, copies line by line, and checks even the final
`fclose`, because that is when the last part of the buffer is actually
written, and a full disk shows up there. Underneath, `fopen` calls the
system call `open`, which returns a file descriptor, and the library
attaches a buffer to it, exactly as for the standard streams.

Vizu does not open files yet, so this program is for reading. With `gcc` on
your own computer it runs as it is.

### Check yourself

1. Why check the result of `fclose(out)`?
   <details><summary>Answer</summary>

   Buffered data is written out when the file is closed. If the disk is
   full, the failure is reported there, not at the `fputs` that put the
   data into the buffer.
   </details>

2. What is the difference between text and binary mode?
   <details><summary>Answer</summary>

   On Windows, text mode turns `\n` into the two characters `\r\n` when
   writing, and back when reading; binary mode leaves every byte alone. On
   Linux and macOS there is no difference.
   </details>

### A common mistake

Testing for the end of the file *before* reading: `while (!feof(in)) {
fgets(line, …); … }`. `feof` only becomes true after a read has failed, so
the loop handles the last line twice. Test the result of the read itself,
as the program above does.

## 13.5 Numbers, chance and time

### Mathematics

`<math.h>` declares the mathematical functions, all working with `double`s:
`sqrt` (square root), `pow(x, y)` (x to the power y), `fabs` (absolute
value), `floor` and `ceil` (round down and up), `round`, the trigonometric
functions `sin`, `cos` and `tan`, `exp` and `log`, and more. On Linux, a
program that uses them is linked with the option `-lm`.

### Random numbers

`rand()`, from `<stdlib.h>`, returns a number between 0 and `RAND_MAX`
that looks random. It is not truly random: it is computed from the
previous number by a fixed formula, starting from a *seed*. The same seed
always gives the same sequence. `srand(seed)` sets the seed; programs that
want a different sequence each run use the current time, `srand(time(NULL))`.
To get a number in a small range, such as a die from 1 to 6, the usual way
is `rand() % 6 + 1`.

### Time

`time(NULL)`, from `<time.h>`, returns the current time as a number of
seconds since the start of 1970. `clock()` returns how much processor time
the program has used, which is useful for measuring how long something
takes.

```c run
#include <stdio.h>
#include <stdlib.h>
#include <math.h>

int main(void) {
    printf("sqrt(2) = %.4f, 2^10 = %.0f\n", sqrt(2.0), pow(2, 10));
    printf("floor(-2.5) = %.1f, ceil(-2.5) = %.1f\n", floor(-2.5), ceil(-2.5));

    srand(42);              /* the same seed: the same rolls every run */
    int counts[7] = {0};
    for (int i = 0; i < 60; i++)
        counts[rand() % 6 + 1]++;
    for (int face = 1; face <= 6; face++)
        printf("%d: %d\n", face, counts[face]);
    return 0;
}
```

The program rolls a die sixty times and counts each face, using the counting
technique of lesson 4.4. With a fair die, each face should come up about ten
times. Because the seed is fixed at 42, the lab gives the same counts every
time you run it; with `srand(time(NULL))`, they would change.

### Check yourself

1. Why is `floor(-2.5)` −3 and not −2?
   <details><summary>Answer</summary>

   `floor` rounds down, towards minus infinity, and −3 is below −2.5.
   Converting to an `int` would cut towards zero instead and give −2.
   </details>

2. Why might a program print the same "random" numbers every time?
   <details><summary>Answer</summary>

   It never calls `srand`, or calls it with the same seed each time. Without
   `srand`, the sequence starts from the seed 1.
   </details>

### A common mistake

Calling `srand(time(NULL))` before every `rand()`. Within one second the
time does not change, so every call restarts the same sequence and returns
the same number. Seed once, at the start of the program.

## 13.6 When things go wrong: `errno`, `assert` and `exit`

A program has to deal with two very different kinds of trouble, and C has
different tools for each.

The first kind comes from outside: a file that does not exist, a disk that
is full, input that is not a number. A correct program must expect these.
Library functions report such failures through their return value, `NULL`,
`EOF` or −1, and many of them also store a code describing the reason in a
global variable called `errno`, from `<errno.h>`. `perror(text)` prints the
text followed by that reason in words, such as `notes.txt: No such file or
directory`. The program then decides what to do: try again, ask the user,
or give up with `exit(EXIT_FAILURE)`, which ends the program from anywhere
(lesson 5.2).

The second kind comes from inside: a mistake in the program itself, such
as a function called with a value it was never meant to receive. For
these, C has `assert`, from `<assert.h>`. `assert(condition)` does nothing
if the condition is true. If it is false, it prints the condition, the
file and the line, and stops the program at once. It states an assumption
the programmer has made, and checks it every time.

```c run expect="error"
#include <stdio.h>
#include <assert.h>

int average(const int *a, int n) {
    assert(n > 0);
    int s = 0;
    for (int i = 0; i < n; i++)
        s += a[i];
    return s / n;
}

int main(void) {
    int nums[3] = {3, 6, 9};
    printf("%d\n", average(nums, 3));
    printf("%d\n", average(nums, 0));
    return 0;
}
```

The first call passes the check. The second would divide by zero, but it
never gets that far: step to it and watch the program stop at line 5, with
a message that names the condition that failed. An `assert` turns a crash
somewhere later into a clear message at the place where the assumption was
broken.

Assertions are meant for the programmer, not for the user. When a program
is compiled with `-DNDEBUG`, every `assert` disappears (lesson 11.6), so
they must never be the only check for something that can really happen.

### Check yourself

1. Should a program use `assert` to check that a file opened?
   <details><summary>Answer</summary>

   No. A missing file is something that happens to correct programs. Check
   it with `if`, report it with `perror`, and carry on or stop. `assert` is
   for things that cannot happen unless the program itself is wrong.
   </details>

2. What does `return 1;` in `main` have to do with `exit(1)`?
   <details><summary>Answer</summary>

   In `main` they mean the same: returning from `main` ends the program
   with that value as its exit status, exactly as `exit` would.
   </details>

### A common mistake

Putting work inside an `assert`, as in `assert(fclose(f) == 0);`. With
`-DNDEBUG` the whole line vanishes, and the file is never closed. Do the
work first, then assert about its result.

## 13.7 Functions with any number of arguments

`printf` takes a format and then any number of further arguments. How can
a function do that? Its prototype ends in three dots, `int printf(const
char *format, ...);`, and inside, it reads the extra arguments one by one
with the macros of `<stdarg.h>`:

```c
#include <stdarg.h>

/* sum: adds up count ints that follow it */
int sum(int count, ...) {
    va_list ap;                 /* where we are among the extra arguments */
    va_start(ap, count);        /* they start after count */
    int total = 0;
    for (int i = 0; i < count; i++)
        total += va_arg(ap, int);   /* the next one, read as an int */
    va_end(ap);
    return total;
}
```

`sum(4, 10, 20, 30, 40)` returns 100. The function must learn from its
fixed arguments how many extra ones there are and what their types are:
`sum` is told the count, and `printf` works it out from the conversions in
its format. That is also why a `printf` whose format does not match its
arguments goes wrong: it reads the arguments that the format promises,
whatever was really passed.

Vizu does not run functions with `...` of your own yet, so this one is for
reading.

### Check yourself

1. What happens with `sum(3, 1, 2)`?
   <details><summary>Answer</summary>

   `sum` reads three arguments, but only two were given. The third read
   takes whatever happens to be there, and the result is garbage. Nothing
   checks.
   </details>

2. Why does `printf` need the format string to know its arguments?
   <details><summary>Answer</summary>

   The extra arguments carry no information about their types or their
   number. The format is the only description `printf` has.
   </details>

### A common mistake

Passing a `float` and reading it with `va_arg(ap, float)`. Extra arguments
undergo the default promotions: a `float` is passed as a `double`, and a
`char` or a `short` as an `int`. Read them as `double` and `int`.
