# Module 3 · Control flow

Read alongside: King chapters 5–6 · K&R chapter 3 and section 1.5

Programs choose and repeat. Vizu shows every condition as a step of its own:
the question in words, the values that went into it, and the answer. This
module is about reading those steps.

## 3.1 `if` and truth

**Read:** King 5.1–5.2 · K&R 3.1–3.3
**You will:** know what C counts as true, and follow an `else if` chain.

C has no special "true" and "false" in conditions: a condition is a number,
and 0 means false, anything else true. `<`, `==`, `!=` and friends give 1 or
0. `&&` (and) and `||` (or) stop as soon as the answer is known: in
`n != 0 && total / n > 10`, the division is never done when `n` is 0.

```c run
#include <stdio.h>

int main(void) {
    int score = 72;
    char grade;
    if (score >= 90)
        grade = 'A';
    else if (score >= 80)
        grade = 'B';
    else if (score >= 70)
        grade = 'C';
    else
        grade = 'F';
    printf("%d -> %c\n", score, grade);
    return 0;
}
```

**Watch:** three condition steps. The first two answer "no" and the program
moves to the next `else if`; the third answers "yes", and the rest of the
chain is skipped.

### Check yourself

1. With `score = 95`, how many conditions are checked?
   <details><summary>Answer</summary>

   One. The first is true, and an `else` is never looked at once its `if` was
   true.
   </details>

2. In `if (n != 0 && total / n > 10)` with `n` equal to 0, what does Vizu show
   for the second part?
   <details><summary>Answer</summary>

   That it was not checked: `&&` already knew the answer was false.
   </details>

### Common mistake

`=` instead of `==`:

```c run
#include <stdio.h>

int main(void) {
    int x = 3;
    if (x = 5)
        printf("x is five\n");
    printf("x is now %d\n", x);
    return 0;
}
```

`x = 5` is an assignment, and its value is 5, which is true. The condition
always holds, and `x` is changed on the way. Watch `x` turn amber at line 5.

## 3.2 `switch`

**Read:** King 5.3 · K&R 3.4
**You will:** choose among many values, and know why `break` is there.

`switch` jumps to the `case` that matches the value, and runs from there
*until a `break`*, straight through the following cases if there is none.
That is useful for grouping several values; it is also the source of a famous
bug.

```c run
#include <stdio.h>

int main(void) {
    int month = 4;
    int days;
    switch (month) {
    case 2:
        days = 28;
        break;
    case 4: case 6: case 9: case 11:
        days = 30;
        break;
    default:
        days = 31;
        break;
    }
    printf("month %d has %d days\n", month, days);
    return 0;
}
```

**Watch:** the `switch` step shows the value it matched, and the next step is
already inside `case 4`.

### Check yourself

1. Which months get 31 days here?
   <details><summary>Answer</summary>

   Every month not listed: `default` catches all the others.
   </details>

2. Can a `case` label be a variable?
   <details><summary>Answer</summary>

   No, only a constant expression: `case 2:` or `case MAX:` with `#define MAX 2`.
   </details>

### Common mistake

A missing `break`:

```c run
#include <stdio.h>

int main(void) {
    int level = 2;
    switch (level) {
    case 1:
        printf("one ");
    case 2:
        printf("two ");
    case 3:
        printf("three ");
    }
    printf("\n");
    return 0;
}
```

It prints `two three`: after `case 2` the program falls into `case 3`. When
you mean to fall through, say so in a comment.

## 3.3 Loops and their counts

**Read:** King 6.1–6.3 · K&R 3.5–3.6
**You will:** pick the right loop and count its rounds before running it.

- `while (c) body`: check, then run; may run zero times.
- `do body while (c);`: run, then check; runs at least once.
- `for (start; c; next) body`: a `while` with its counter in one place.

Counting digits needs at least one round (0 has one digit), so it is a
`do … while`:

```c run
#include <stdio.h>

int main(void) {
    int n = 4096;
    int digits = 0;
    do {
        n /= 10;
        digits++;
    } while (n > 0);
    printf("%d digits\n", digits);
    return 0;
}
```

**Watch:** `n` goes 4096, 409, 40, 4, 0. Vizu counts the loop's checks; the
fourth one says "no".

### Check yourself

1. How many times does `for (int i = 10; i > 0; i -= 3)` run its body?
   <details><summary>Answer</summary>

   Four: `i` is 10, 7, 4, 1; then −2 fails the check.
   </details>

2. `for (int i = 0; i < n; i++)` and `for (int i = 1; i <= n; i++)`: how many
   rounds each?
   <details><summary>Answer</summary>

   Both `n`. The first is C's usual form, because arrays start at index 0.
   </details>

### Common mistake

`<=` where `<` was meant: one round too many. Lesson 4.2 shows what that does
to an array.

## 3.4 `break`, `continue` and the empty statement

**Read:** King 6.4–6.5 · K&R 3.7
**You will:** leave a loop early, skip a round, and spot a stray `;`.

`break` leaves the innermost loop (or `switch`). `continue` skips the rest of
this round and goes to the next check (in a `for`, after running `next`).
`for (;;)` is a loop with no condition: it runs until a `break`.

```c run input="5 -3 8 0 4\n"
#include <stdio.h>

int main(void) {
    int n, sum = 0;
    for (;;) {
        scanf("%d", &n);
        if (n == 0)
            break;
        if (n < 0)
            continue;
        sum += n;
    }
    printf("sum of the positives: %d\n", sum);
    return 0;
}
```

**Watch:** at −3, `continue` jumps back without touching `sum`. At 0, `break`
leaves the loop, and the last number in the input, 4, is never read.

### Check yourself

1. What is printed?
   <details><summary>Answer</summary>

   `sum of the positives: 13` (5 + 8).
   </details>

2. Inside two nested loops, what does `break` leave?
   <details><summary>Answer</summary>

   Only the inner loop. To leave both, use a flag, a `return`, or (rarely) a
   `goto`.
   </details>

### Common mistake

A semicolon after the loop's parentheses is an *empty statement*: it is the
whole body.

```c run
#include <stdio.h>

int main(void) {
    int i;
    for (i = 0; i < 3; i++);
    {
        printf("i is %d\n", i);
    }
    return 0;
}
```

The loop counts to 3 doing nothing, then the block runs once: `i is 3`.

## 3.5 Counting characters, lines and words

**Read:** K&R 1.5 · King 7.3 (getchar)
**You will:** write a small real tool, and use the `(c = getchar()) != EOF`
idiom.

`getchar()` returns the next input character, or `EOF` (a negative number) when
there is no more input. That is why `c` is an `int`, not a `char`: it must hold
every character *and* `EOF`. K&R build a word counter around it; a word is a
run of characters that are not blanks, and a variable remembers whether we
are inside one.

```c run input="the quick brown\nfox jumps\n"
#include <stdio.h>

#define IN  1   /* inside a word */
#define OUT 0   /* outside a word */

int main(void) {
    int c, lines = 0, words = 0, chars = 0;
    int state = OUT;
    while ((c = getchar()) != EOF) {
        chars++;
        if (c == '\n')
            lines++;
        if (c == ' ' || c == '\n' || c == '\t')
            state = OUT;
        else if (state == OUT) {
            state = IN;
            words++;
        }
    }
    printf("%d %d %d\n", lines, words, chars);
    return 0;
}
```

`(c = getchar()) != EOF` reads a character, stores it in `c`, and compares it,
in one condition. The parentheses matter: without them `!=` would be done
first, and `c` would get 0 or 1.

**Watch:** `state` flips between 1 and 0 at each word's edge; `words` grows
only on the step where it flips to 1.

### Check yourself

1. Predict the three numbers before running.
   <details><summary>Answer</summary>

   `2 5 26`: two newlines, five words, 16 + 10 characters.
   </details>

2. What would two spaces between words change?
   <details><summary>Answer</summary>

   Only `chars`. The second space finds `state` already `OUT`.
   </details>

### Common mistake

`char c = getchar();` On machines where `char` is unsigned, `c` can never equal
`EOF` and the loop never ends; where it is signed, a real character with code
255 looks like `EOF`. Keep `c` an `int`.
