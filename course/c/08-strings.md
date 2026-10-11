# Module 8 · Strings

Text is everywhere in programs: names, messages, commands, the lines of a
file. Most languages have a type for it, a string type that knows its own
length and grows when you add to it. C has no such type. In C, a string is
simply a row of characters in memory, an array of `char`, with one special
character at the end to say "the string stops here". That end marker is
the character whose code is 0, written `'\0'` and called the *null
character*.

That is the whole idea, and everything in this module follows from it.
Since a string does not know its length, every function that works on a
string must walk along it until it finds the `'\0'`. Since a string is an
array, it has a fixed size, and nothing stops a program from writing past
the end of it. And since a string is reached through the address of its
first character, all the pointer techniques of the last two modules apply
to strings directly.

Vizu draws strings character by character, with the `'\0'` as a box of its
own, so you can always see where a string really ends, and where the
array that holds it ends, which is not always the same place.

## 8.1 A string ends in `'\0'`

When you write a *string literal* such as `"cat"` in a program, the
compiler stores its characters in memory, one per byte, and adds a `'\0'`
after them. So `"cat"` takes four bytes: `'c'`, `'a'`, `'t'` and `'\0'`.
The literal itself, used in an expression, stands for the address of its
first character, like the name of an array.

There are two quite different ways to keep a string in a variable, and
the difference matters.

```c
char word[8] = "cat";
```

declares an array of eight `char`s and *copies* the characters of the
literal into it, `'\0'` included. The remaining elements are filled with
`'\0'` too. The array belongs to the program, and its characters may be
changed freely.

```c
char *lit = "dog";
```

declares a pointer and makes it point to the literal itself, wherever the
compiler stored it. Nothing is copied. The literal is stored in memory
that the program is not supposed to change, so `lit` may be used to read
the string, but not to change its characters.

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

The conversion `%s` makes `printf` print a string: it is given an address
and prints characters from there until it meets a `'\0'`.

Step through the program and compare the two variables in the memory
picture. `word` is a row of eight boxes on the stack: `'c'`, `'a'`, `'t'`,
and then five `'\0'`s. `lit` is a single 8-byte box with an arrow leading
out of the stack into a separate area of memory where the literals live,
and there `"dog"` sits, with its own `'\0'`. Line 6 changes `word[0]`,
which turns amber and becomes a `'b'`.

### Check yourself

1. How many bytes does `"hello"` take in memory?
   <details><summary>Answer</summary>

   Six: five letters and the `'\0'`.
   </details>

2. What does `printf("%s\n", word + 1)` print?
   <details><summary>Answer</summary>

   `at`. `word + 1` is the address of the second character, and `printf`
   starts printing from there.
   </details>

3. What is the difference between `'a'` and `"a"`?
   <details><summary>Answer</summary>

   `'a'` is a single character, the number 97. `"a"` is a string: two bytes
   in memory, `'a'` and `'\0'`, and the expression stands for the address of
   the first of them.
   </details>

### A common mistake

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

The compiler accepts this, but on most systems the program crashes at line
5, because the literal is in memory that may not be written. Vizu stops
there too. If the string must change, keep it in an array: `char name[] =
"lisa";` makes a copy that is yours.

## 8.2 Writing and reading strings

### Writing

Printing a string is easy. `printf("%s", s)` prints it, and `puts(s)`
prints it followed by a newline. Both stop at the `'\0'`.

### Reading a word

Reading is harder, because the program must have an array ready to receive
the characters, and the array must be large enough. `scanf("%s", name)`
reads one word: it skips white space, then stores characters until the
next space, tab or newline, and adds a `'\0'`. Notice that there is no `&`
before `name`: `name` is an array, and its name already stands for the
address of its first element.

The trouble is that `scanf` has no idea how large the array is. It stores
as many characters as the word has, and if the word is longer than the
array, it simply writes on past the end. This is the famous *buffer
overflow*, and it has caused countless security holes.

### Reading a line, safely

`fgets(buf, size, stdin)` reads a whole line, spaces and all, and it is
told the size of the array. It never stores more than `size - 1`
characters, which leaves room for the `'\0'` it adds. It keeps the newline
at the end of the line, if there was room for it, and that newline usually
has to be removed.

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

`strchr(s, c)`, from `<string.h>`, looks for the character `c` in the
string `s` and returns a pointer to it, or `NULL` if it is not there. If
the line contained a newline, we replace it with `'\0'`, and the string now
ends one character earlier.

Step through it. After line 7, the array holds the characters of `Ada
Lovelace`, a `'\n'` and a `'\0'`, followed by boxes that were never
written. Line 10 turns the `'\n'` into a `'\0'`, and that is all it takes to
shorten a string.

### Reading a line yourself

Reading character by character with `getchar` gives complete control.
This function reads one line into an array, keeps at most `n`
characters, and quietly skips the rest of a line that is too long:

```c run input="Grace Hopper wrote compilers\n"
#include <stdio.h>

int read_line(char str[], int n);

int main(void) {
    char line[13];
    int len = read_line(line, 12);
    printf("kept %d characters: \"%s\"\n", len, line);
    return 0;
}

int read_line(char str[], int n) {
    int ch, i = 0;
    while ((ch = getchar()) != '\n' && ch != EOF)
        if (i < n)
            str[i++] = ch;
    str[i] = '\0';
    return i;
}
```

The loop reads until the end of the line, but stores a character only
while there is room. The `'\0'` goes in last, after the characters that
were kept. The array has 13 elements so that 12 characters and the `'\0'`
fit. To print the quotes around the string, the format uses `\"`.

Watch the array fill one box at a time. When `i` reaches 12, the
characters still go on being read, the strip at the bottom shows them
being taken from the input, but they are no longer stored.

### Check yourself

1. With `fgets(name, 16, stdin)`, what happens if the line is 30
   characters long?
   <details><summary>Answer</summary>

   The first 15 characters are stored, followed by `'\0'`. The rest of the
   line stays in the input for the next read. There is no `'\n'` in `name`
   then, which is how a program can tell that the line was cut.
   </details>

2. Why `sizeof name` and not 16 in the call to `fgets`?
   <details><summary>Answer</summary>

   So that the call stays right if the size of the array is ever changed.
   </details>

### A common mistake

Reading with `scanf("%s")` into an array that is too small:

```c run input="Bartholomew\n" expect="error"
#include <stdio.h>

int main(void) {
    char name[6];
    scanf("%s", name);
    printf("%s\n", name);
    return 0;
}
```

Eleven letters and a `'\0'` are written into six boxes. On a real computer
the extra bytes overwrite whatever lies after the array on the stack. Vizu
stops at line 5. `scanf("%5s", name)` would limit the word to five
characters. (There was once a function `gets` that read a line with no
limit at all; it was so dangerous that it has been removed from C.)

## 8.3 Going through a string

Since a string is an array ending in `'\0'`, going through its characters
is a loop that stops at the `'\0'`. It can be written with an index or with
a pointer, and C programmers use both.

```c run
#include <stdio.h>
#include <ctype.h>

int count_spaces(const char s[]);
int count_vowels(const char *s);

int main(void) {
    char text[] = "Hello, big world";
    printf("%d spaces, %d vowels\n", count_spaces(text), count_vowels(text));
    for (int i = 0; text[i] != '\0'; i++)
        text[i] = toupper(text[i]);
    printf("%s\n", text);
    return 0;
}

int count_spaces(const char s[]) {
    int count = 0;
    for (int i = 0; s[i] != '\0'; i++)
        if (s[i] == ' ')
            count++;
    return count;
}

int count_vowels(const char *s) {
    int count = 0;
    for (; *s != '\0'; s++)
        if (*s == 'a' || *s == 'e' || *s == 'i' || *s == 'o' || *s == 'u')
            count++;
    return count;
}
```

`count_spaces` uses an index: `s[i]` is the character, and the loop stops
when it is `'\0'`. `count_vowels` does the same with a pointer: `s` itself
moves along the string, `*s` is the current character, and `s++` moves on.
Since `s` is a parameter, a private copy, moving it does no harm to the
caller.

The loop in `main` changes each letter to a capital, with `toupper` from
`<ctype.h>`. That header has a family of small functions for characters:
`isdigit(c)` tests for a digit, `isalpha(c)` for a letter, `isspace(c)` for
white space, `isupper` and `islower` for the case, and `toupper` and
`tolower` change it.

Step through `count_vowels` and watch its arrow walk along `text` in
`main`'s frame, one box per round, until it reaches the `'\0'`. Then watch
the loop in `main` turn the boxes of `text` amber, one capital at a time.

### Check yourself

1. What does the program print?
   <details><summary>Answer</summary>

   `2 spaces, 4 vowels`, then `HELLO, BIG WORLD`.
   </details>

2. Why can `count_vowels` change `s` when it is declared `const char *`?
   <details><summary>Answer</summary>

   The `const` protects the characters, not the pointer. The function may
   move `s` along; it may not change the characters it points to.
   </details>

### A common mistake

Stopping a string loop at the size of the array instead of at the
`'\0'`, as in `for (i = 0; i < 100; i++)`. The characters after the `'\0'`
are not part of the string; they may be leftovers from earlier, or never
written at all.

## 8.4 `strlen` and `strcpy` by hand

The standard library has functions for the common jobs on strings, and we
will use them in the next lesson. First, though, it is worth writing two
of them ourselves, because they are short, and because their pointer
loops are idioms you will meet everywhere.

`strlen(s)` returns the length of a string, the number of characters
before the `'\0'`. `strcpy(dst, src)` copies the string `src`, `'\0'`
included, into the array `dst`.

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

`my_strlen` moves a pointer `p` from the start of the string to its
`'\0'`, and the length is the distance between `p` and the start, pointer
subtraction from lesson 7.1.

`my_strcpy` does all its work in the condition of its loop. `*dst++ =
*src++` copies one character from where `src` points to where `dst` points,
and moves both pointers on. The value of the assignment is the character
copied, and the loop stops when that character was the `'\0'`. The body is
an empty statement, since there is nothing left to do. Notice that the
`'\0'` is copied before the test sees it, which is exactly right: the copy
needs its own `'\0'`.

Step into `my_strcpy` and watch the two arrows walk side by side, `src`
along the literal and `dst` along `buf`, while the boxes of `buf` turn
amber one after another, the `'\0'` last.

In older code you will see this loop written even more briefly, as
`while ((*dst++ = *src++))`, leaving out the comparison with `'\0'`: since
`'\0'` is 0, and 0 is false, the loop stops by itself.

### Check yourself

1. What does `my_strlen("")` return?
   <details><summary>Answer</summary>

   0. The very first character is the `'\0'`, so `p` never moves.
   </details>

2. What does `my_strcpy` do if `buf` is too small?
   <details><summary>Answer</summary>

   It writes past the end of `buf`. It has no way of knowing how large
   `dst` is; only the caller knows. That is the price of this simple
   design.
   </details>

### A common mistake

Calling `strlen` in the condition of a loop, as in `for (i = 0; i <
strlen(s); i++)`. Each call walks the whole string again, so the loop does
far more work than it seems to. Work out the length once, before the loop.

## 8.5 The string library

The header `<string.h>` declares the standard string functions. The most
used are these:

| Function | Does |
| --- | --- |
| `strlen(s)` | the length of `s`, not counting the `'\0'` |
| `strcpy(dst, src)` | copies `src` into `dst` |
| `strcat(dst, src)` | adds `src` to the end of the string in `dst` |
| `strcmp(a, b)` | compares two strings |
| `strchr(s, c)` | finds the first `c` in `s` |
| `strstr(s, t)` | finds the first place where the string `t` occurs in `s` |

None of them checks sizes. Whenever a function writes into an array, the
array must be large enough for the result, `'\0'` included, and making sure
of that is the programmer's job.

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

`strcat` finds the `'\0'` at the end of the string already in
`greeting`, and writes the new characters from there, followed by a new
`'\0'`. Step through the two calls and watch: the old `'\0'` is overwritten
by the first new character, and a new `'\0'` appears at the new end.

`strcmp` compares two strings character by character, the way words are
ordered in a dictionary, by their character codes. It returns a negative
number if the first string comes first, zero if the strings are equal, and
a positive number if the first comes after the second. So `strcmp(a, b) <
0` reads "a comes before b".

`strcpy(copy, greeting)` makes a real copy in a second array. Changing the
copy, as line 13 does, leaves the original alone.

### Writing into a string with `snprintf`

The safest way to build a string is `snprintf(buf, size, format, ...)`. It
works like `printf`, but writes into `buf` instead of to the screen, and it
never writes more than `size` bytes, `'\0'` included. If the result is
too long, it is cut short, but it is always properly ended.

### Check yourself

1. What does the program print?
   <details><summary>Answer</summary>

   `Hello, world (12)`, `apple comes first`, and `Hello, world / Jello,
   world`.
   </details>

2. What is the result of `strcmp("Zebra", "apple")`, and why?
   <details><summary>Answer</summary>

   A negative number: `'Z'` has the code 90 and `'a'` has 97, so "Zebra"
   comes first. `strcmp` compares codes, and all the capitals come before
   all the small letters.
   </details>

### A common mistake

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

`a` and `b` are two arrays, and in the comparison each name turns into the
address of its first element. Two arrays are never at the same address, so
`a == b` is false, even though they hold the same characters. Step to the
comparison and Vizu shows two addresses being compared. To compare the
contents, write `strcmp(a, b) == 0`.

## 8.6 Arrays of strings, and the command line

A program often needs a list of strings: the names of the months, the
words of a sentence, the lines of a file. There are two ways to keep one,
and they look alike in the program text but are quite different in
memory.

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

`planets` is an array of pointers. Each element is an address, 8 bytes,
pointing to a string literal stored elsewhere, and each string takes
exactly the room it needs. Such an array is sometimes called *ragged*,
because its strings have different lengths.

`grid` is a two-dimensional array of characters: four rows of eight
`char`s each, in one block. Every row is eight bytes long, whatever the
length of the name in it, so the shorter names leave unused boxes at the
end of their rows. The longest name, "Mercury", needs exactly eight: seven
letters and the `'\0'`.

Step through it and compare the two pictures. `planets` is four boxes with
four arrows leading out into the area of literals. `grid` is four rows of
boxes, with a row of `'\0'`s after each shorter name. The last line of
output gives their sizes: 32 bytes for four pointers, and 32 bytes for the
grid, but the pointer version also uses the memory of the literals.

`%-8s` prints a string left-aligned in a field eight characters wide, so
that the second column lines up.

### The command line

The array of pointers is also the way a program receives the words typed
after its name on the command line. `main` may be declared with two
parameters:

```c
int main(int argc, char *argv[])
```

`argc` is the number of words, and `argv` is an array of pointers to them,
as strings. `argv[0]` is the name of the program itself, `argv[1]` the
first word after it, and so on, and `argv[argc]` is a null pointer that
marks the end. Here is a program that prints its arguments back:

```c
/* echo: print the command-line arguments */
#include <stdio.h>

int main(int argc, char *argv[]) {
    for (int i = 1; i < argc; i++)
        printf("%s%s", argv[i], i < argc - 1 ? " " : "\n");
    return 0;
}
```

Run as `./echo hello world`, it prints `hello world`. Vizu runs programs
without a command line, so this one is for reading.

### Check yourself

1. Why is `sizeof planets` 32?
   <details><summary>Answer</summary>

   It holds four pointers of 8 bytes each. The strings themselves are not
   inside `planets`; they are stored with the other literals.
   </details>

2. Would `"Jupiter"` fit in a row of `grid`? And `"Neptune!"`?
   <details><summary>Answer</summary>

   `"Jupiter"` fits exactly: seven letters and the `'\0'`. `"Neptune!"` has
   eight characters and needs nine bytes, so it does not.
   </details>

### A common mistake

Declaring the array as `char *planets[]`, without `const`, and then
changing a letter through it. The pointers lead to string literals, which
may not be changed (lesson 8.1). The `const` makes the compiler catch the
attempt.
