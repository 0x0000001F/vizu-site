# Module 13 · Input, output and the library

Read alongside: King chapters 21–24 and 26 · K&R chapter 7, sections 8.1–8.3 and appendix B

C's input and output are not part of the language but of its library, built
on a few calls to the operating system. This module shows the streams every
program starts with, files, and how a program reports what went wrong.

## 13.1 Streams: `stdin`, `stdout`, `stderr` and EOF

**Read:** King 22.1 · K&R 7.1
**You will:** write a filter, and know where its output goes.

Every C program starts with three open *streams*: `stdin` (the keyboard, or a
file given with `<`), `stdout` (the screen, or a file given with `>`) and
`stderr` (the screen, even when `stdout` is redirected: errors stay visible).
A program that reads `stdin` and writes `stdout` is a *filter*, and filters
can be chained with `|` in the shell.

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

Run on a real machine as `./upper < notes.txt > loud.txt`: the file is
converted, and `done` still appears on the screen.

**Watch:** the strip under the memory. The first `getchar` finds stdin's
buffer empty, so the C library calls `read`, and the kernel hands over the
whole line `hello` and its newline; the next `getchar`s take from the buffer
without asking the kernel. Each `putchar` puts one letter into stdout's
buffer, and only the newline sends the line to the screen with `write`.
`stderr` has no buffer: `done` goes out at once.

### Check yourself

1. Why might output written with `printf` not appear on the screen right
   away?
   <details><summary>Answer</summary>

   `stdout` is *buffered*: characters are collected and written in chunks
   (line by line on a terminal). `fflush(stdout)` forces them out; `stderr`
   is not buffered.
   </details>

2. How does a user type EOF at the keyboard?
   <details><summary>Answer</summary>

   Ctrl+D at the start of a line on Linux and macOS; Ctrl+Z then Enter on
   Windows.
   </details>

### Common mistake

Printing error messages to `stdout`. When the output is redirected to a file,
the user never sees them, and the file is spoiled.

## 13.2 Files

**Read:** King 22.2–22.5 · K&R 7.5
**You will:** open, read, write and close a file, checking every step.

`fopen(name, mode)` opens a file and returns a `FILE *`, or `NULL` if it
cannot. The mode says how: `"r"` read, `"w"` write (emptying the file), `"a"`
append, with `"b"` for binary files. The same functions as for the standard
streams work on it with an `f` in front: `fgets`, `fprintf`, `fputs`. `fclose`
writes out what is buffered and frees the stream.

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

Vizu does not open files yet; read this one, and try it with `gcc` on your
machine.

### Check yourself

1. Why check the result of `fclose(out)`?
   <details><summary>Answer</summary>

   Buffered data is written at that moment; a full disk shows up there, not
   at the `fputs`.
   </details>

2. What is the difference between text and binary mode?
   <details><summary>Answer</summary>

   On Windows, text mode turns `\n` into `\r\n` on output and back on input;
   binary mode leaves bytes alone. On Linux and macOS they are the same.
   </details>

### Common mistake

Testing `feof(in)` *before* reading: `while (!feof(in)) { fgets(...); ... }`
processes the last line twice. Test the result of the read itself, as above.

## 13.3 Errors: `errno`, `assert` and `exit`

**Read:** King 24.1–24.2, 9.5 · K&R 7.6
**You will:** report problems clearly and catch your own mistakes early.

- A library function that fails usually returns a special value (`NULL`,
  `EOF`, −1) and sets `errno` to a code; `perror(text)` prints the text and the
  reason in words.
- `exit(status)` ends the program from anywhere; `EXIT_SUCCESS` and
  `EXIT_FAILURE` are the portable codes.
- `assert(condition)` stops the program with the file and line if the
  condition is false. It checks *your* assumptions, not the user's input, and
  disappears when compiled with `-DNDEBUG`.

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

**Watch:** the first call passes the `assert`; the second stops at line 5,
before the division by zero it would have caused.

### Check yourself

1. Should `assert` check that a file opened?
   <details><summary>Answer</summary>

   No: a missing file is something that happens to correct programs. Check it
   with `if` and report it with `perror`. `assert` is for "this cannot happen
   unless my code is wrong".
   </details>

2. What does `return 1;` in `main` have to do with `exit(1)`?
   <details><summary>Answer</summary>

   In `main` they are the same: returning from `main` calls `exit` with that
   value.
   </details>

### Common mistake

Putting work inside an `assert`: `assert(fclose(f) == 0);`. With `-DNDEBUG` the
whole line vanishes, and the file is never closed.
