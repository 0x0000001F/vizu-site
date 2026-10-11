# Module 8 · Strings

Read alongside: King chapter 13 · K&R sections 1.9, 5.5–5.6 and 5.10

C has no string type. A string is a row of `char`s that ends with the
character `'\0'`, and every string function finds the end by looking for it.
Vizu draws strings cell by cell, the `'\0'` included, so you can see where a
string really ends.

## 8.1 A string ends in `'\0'`

**Read:** King 13.1–13.2 · K&R 5.5
**You will:** tell a `char` array from a pointer to a string literal.

A string literal such as `"cat"` is stored by the compiler as four characters:
`'c' 'a' 't' '\0'`. It lives in memory the program must not change.

- `char word[8] = "cat";` makes an array of 8 `char`s and *copies* the literal
  into it; the rest is filled with `'\0'`. The array is yours to change.
- `char *lit = "dog";` makes a pointer to the literal itself. You may read it,
  not change it.

```c run
#include <stdio.h>

int main(void) {
    char word[8] = "cat";
    char *lit = "dog";
    word[0] = 'b';
    printf("%s %s %zu\n", word, lit, sizeof word);
    return 0;
}
```

`%s` prints characters from the given address until it meets `'\0'`.

**Watch:** `word` is eight cells: `'c' 'a' 't'` and five `'\0'`s. `lit` is an
8-byte box with an arrow into the string area, where `"dog"` sits with its
`'\0'`. Line 6 changes `word[0]` amber.

### Check yourself

1. How many bytes does `"hello"` take?
   <details><summary>Answer</summary>

   Six: five letters and the `'\0'`.
   </details>

2. What does `printf("%s", word + 1)` print?
   <details><summary>Answer</summary>

   `at`: printing starts at the second cell.
   </details>

### Common mistake

Changing a string literal through a pointer:

```c run expect="error"
#include <stdio.h>

int main(void) {
    char *name = "lisa";
    name[0] = 'L';
    printf("%s\n", name);
    return 0;
}
```

It compiles, and on most systems it crashes, because literals are in
read-only memory. If you need to change it, make it an array:
`char name[] = "lisa";`.

## 8.2 Reading strings safely

**Read:** King 13.3
**You will:** read a line without overflowing the array.

`scanf("%s", name)` reads one word and writes as many characters as it finds,
whatever the array's size. `fgets(buf, size, stdin)` reads a line but never
more than `size - 1` characters, then adds `'\0'`. It keeps the newline,
which you usually remove.

```c run input="Ada Lovelace\n"
#include <stdio.h>
#include <string.h>

int main(void) {
    char name[16];
    printf("Name: ");
    fgets(name, sizeof name, stdin);
    char *newline = strchr(name, '\n');
    if (newline != NULL)
        *newline = '\0';        /* drop the newline */
    printf("Hello, %s!\n", name);
    return 0;
}
```

**Watch:** after line 7 the array holds `Ada Lovelace\n\0` and garbage after
it; line 10 turns the `'\n'` into `'\0'`.

### Check yourself

1. What happens with fgets if the line is 30 characters long?
   <details><summary>Answer</summary>

   The first 15 characters are stored with a `'\0'`; the rest stay in the
   input for the next read. There is no `'\n'` in `name` then, which is how
   you can tell the line was cut.
   </details>

2. Why `sizeof name` and not 16?
   <details><summary>Answer</summary>

   If the array's size changes, the call stays right.
   </details>

### Common mistake

`scanf("%s")` into a small array:

```c run input="Bartholomew\n" expect="error"
#include <stdio.h>

int main(void) {
    char name[6];
    scanf("%s", name);
    printf("%s\n", name);
    return 0;
}
```

Eleven letters and a `'\0'` go into six cells. On a real machine this
overwrites the stack: the classic *buffer overflow*. Vizu stops at line 5.
(`scanf("%5s", name)` limits it; `gets` cannot be limited at all and was
removed from C.)

## 8.3 `strlen` and `strcpy` by hand

**Read:** K&R 5.5 · King 13.6
**You will:** write the two most basic string functions with pointers.

K&R show `strcpy` in four versions, each shorter than the last: with an index,
with two pointers, with the increment folded into the test, and finally with
the comparison with `'\0'` left out (`'\0'` is 0, which is false). The last is
`while ((*dst++ = *src++))`, an idiom you will meet in old code.

```c run
#include <stdio.h>

size_t my_strlen(const char *s) {
    const char *p = s;
    while (*p != '\0')
        p++;
    return p - s;
}

void my_strcpy(char *dst, const char *src) {
    while ((*dst++ = *src++) != '\0')
        ;
}

int main(void) {
    char buf[16];
    my_strcpy(buf, "pointer");
    printf("%s has %zu characters\n", buf, my_strlen(buf));
    return 0;
}
```

**Watch:** in `my_strcpy`, two arrows walk side by side: `src` over the
literal, `dst` over `buf`, and each step writes one cell amber, the `'\0'`
last. In `my_strlen`, the answer is the distance between two arrows.

### Check yourself

1. Why does `my_strcpy` copy the `'\0'` too?
   <details><summary>Answer</summary>

   The assignment happens before the test: the `'\0'` is copied, and then the
   loop sees it and stops.
   </details>

2. What does `my_strcpy` do if `buf` is too small?
   <details><summary>Answer</summary>

   It writes past the end: it has no idea how big `dst` is. That is why safer
   functions take a size.
   </details>

### Common mistake

Calling `strlen` in a loop condition, `for (i = 0; i < strlen(s); i++)`: it
walks the whole string on every round.

## 8.4 The string library

**Read:** King 13.5, 23.6
**You will:** use `strcpy`, `strcat`, `strcmp` and `strlen`, and know their
traps.

`<string.h>` has the functions you just wrote and more. None of them checks
sizes: the destination must be big enough, `'\0'` included.

```c run
#include <stdio.h>
#include <string.h>

int main(void) {
    char greeting[32] = "Hello";
    strcat(greeting, ", ");
    strcat(greeting, "world");
    printf("%s (%zu)\n", greeting, strlen(greeting));
    if (strcmp("apple", "banana") < 0)
        printf("apple comes first\n");
    char copy[32];
    strcpy(copy, greeting);
    copy[0] = 'J';
    printf("%s / %s\n", greeting, copy);
    return 0;
}
```

`strcmp` returns a negative number, zero or a positive number, as its first
string comes before, equals or comes after the second.

**Watch:** each `strcat` finds `greeting`'s `'\0'` and writes from there; the
old `'\0'` is overwritten and a new one is placed at the end.

### Check yourself

1. Predict the output.
   <details><summary>Answer</summary>

   `Hello, world (12)`, `apple comes first`, `Hello, world / Jello, world`.
   </details>

2. Why is `strncpy(dst, src, sizeof dst)` not fully safe?
   <details><summary>Answer</summary>

   If `src` is too long, `strncpy` fills `dst` and adds no `'\0'`. Set the last
   cell yourself, or use `snprintf(dst, sizeof dst, "%s", src)`.
   </details>

### Common mistake

Comparing strings with `==`:

```c run
#include <stdio.h>

int main(void) {
    char a[] = "yes";
    char b[] = "yes";
    if (a == b)
        printf("same\n");
    else
        printf("different addresses\n");
    return 0;
}
```

`a == b` compares two addresses, and two arrays are never at the same one.
Use `strcmp(a, b) == 0`.

## 8.5 Arrays of strings and `argv`

**Read:** King 13.7 · K&R 5.6, 5.10
**You will:** choose between an array of pointers and a 2-D `char` array, and
read the command line.

Two ways to keep several strings:

- `const char *planets[] = {...}`: an array of pointers, each to a literal
  stored elsewhere (a *ragged* array: each string takes what it needs).
- `char grid[4][8] = {...}`: one block, four rows of 8 `char`s each, whatever
  the strings' lengths.

```c run
#include <stdio.h>

int main(void) {
    const char *planets[] = {"Mercury", "Venus", "Earth", "Mars"};
    char grid[4][8] = {"Mercury", "Venus", "Earth", "Mars"};
    int n = sizeof planets / sizeof planets[0];
    for (int i = 0; i < n; i++)
        printf("%-8s %c\n", planets[i], grid[i][0]);
    printf("%zu bytes vs %zu bytes\n", sizeof planets, sizeof grid);
    return 0;
}
```

`%-8s` prints a string left-aligned in 8 columns.

**Watch:** `planets` is four arrows into the string area; `grid` is four rows
of cells, with unused cells after the shorter names.

The command line reaches a program the first way. `main` can be declared
`int main(int argc, char *argv[])`: `argc` is the number of words typed,
`argv` an array of pointers to them, `argv[0]` being the program's name and
`argv[argc]` a null pointer.

```c
/* echo: print the command-line arguments */
#include <stdio.h>

int main(int argc, char *argv[]) {
    for (int i = 1; i < argc; i++)
        printf("%s%s", argv[i], i < argc - 1 ? " " : "\n");
    return 0;
}
```

Run as `./echo hello world`, it prints `hello world`. (Vizu runs programs
with no arguments, so this one is for reading.)

### Check yourself

1. Why is `sizeof planets` 32?
   <details><summary>Answer</summary>

   Four pointers of 8 bytes. The strings are not inside `planets`.
   </details>

2. Can `"Mercury"` fit in `grid[i][7]`… sorry, in a row of `grid`?
   <details><summary>Answer</summary>

   Exactly: 7 letters and the `'\0'` make 8. A ninth character would not fit.
   </details>

### Common mistake

Declaring `char *planets[]` without `const` and then changing a letter: the
pointers lead to literals, which are read-only (lesson 8.1).
