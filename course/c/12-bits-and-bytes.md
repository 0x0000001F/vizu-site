# Module 12 · Bits and bytes

Every value in a program, whatever its type, is in the end a row of bits in
memory. Most of the time C lets us forget that: we add `int`s and compare
strings without thinking about the bits underneath. But some programs have
to work with the bits directly. A program that talks to hardware reads and
sets single bits in a device's registers. A program that stores many
yes-or-no settings packs them into the bits of one number. Compression,
encryption, graphics and network protocols all work at the level of bits.

C was designed for writing operating systems, and it gives full access to
bits through a small set of *bitwise operators*. This module starts with
how numbers, including negative ones, are written in binary, then
introduces the operators, and ends by looking at the bytes of a number in
memory one by one. It is also the doorway to the tracks on assembly and the
machine, where everything is bits.

## 12.1 Binary, hexadecimal and negative numbers

We write numbers in decimal, base 10: each digit is worth ten times the
digit to its right. In 345, the 3 is worth 300, the 4 is worth 40, and the
5 is worth 5. Binary, base 2, works the same way with only two digits, 0
and 1, and each place is worth twice the place to its right: 1, 2, 4, 8,
16, 32, 64, 128 and so on. So `0010 1010` in binary is 32 + 8 + 2 = 42.

Binary numbers are long and hard to read, so programmers usually write
them in hexadecimal, base 16, as lesson 2.2 introduced. Each hexadecimal
digit stands for exactly four bits, so converting is a matter of splitting
the bits into groups of four: `0010 1010` is `2` and `A`, that is `0x2A`.

### Negative numbers: two's complement

How can bits, which are only 0 or 1, store a minus sign? Nearly every
computer today uses a scheme called *two's complement*. Think of the
numbers an 8-bit box can hold as arranged on a wheel, like the odometer of
lesson 2.3. Counting up from 0 goes 1, 2, 3 … 127. Counting *down* from 0
must reach the pattern just before `0000 0000` on the wheel, which is `1111
1111`; so that pattern stands for −1. The one before it, `1111 1110`, is −2,
and so on down to `1000 0000`, which is −128.

Two useful facts follow. The leftmost bit tells the sign: it is 1 for every
negative number. And to negate a number, invert all its bits and add one:
42 is `0010 1010`, the inverted bits are `1101 0101`, plus one is `1101
0110`, which is −42. The beauty of the scheme is that the processor does
not need separate circuits for negative numbers: adding −42 is the very
same binary addition as adding any other pattern, and the carry that falls
off the left end is simply forgotten.

```c run
#include <stdio.h>

int main(void) {
    int n = 42;
    printf("%d = 0x%x = 0%o\n", n, n, n);
    for (int bit = 7; bit >= 0; bit--)
        putchar((n >> bit) & 1 ? '1' : '0');
    putchar('\n');
    int m = -n;
    for (int bit = 7; bit >= 0; bit--)
        putchar((m >> bit) & 1 ? '1' : '0');
    putchar('\n');
    printf("-1 as unsigned: %u = 0x%x\n", (unsigned) -1, (unsigned) -1);
    return 0;
}
```

The loops print the lowest eight bits of a number, from the highest of
them to the lowest. The expression `(n >> bit) & 1` uses two operators from
the next lesson: it moves the wanted bit to the far right and keeps only
that bit. The last line shows −1 read as an unsigned number: all 32 bits
are 1, which is `0xffffffff`.

In the memory picture, rest the pointer on the box of `n` or `m`: Vizu
shows the bits of each byte. You will see `0010 1010` in the first byte of
`n`, and `1101 0110` in the first byte of `m`, followed by bytes of all ones.

### Check yourself

1. What is `0xFF` in decimal? And `0x100`?
   <details><summary>Answer</summary>

   255 and 256. `0xFF` is eight ones; `0x100` is a one followed by eight
   zeros.
   </details>

2. In 8 bits, why is there a −128 but no +128?
   <details><summary>Answer</summary>

   Eight bits have 256 patterns. One is zero, the 128 with the top bit set
   are negative, and that leaves only 127 for the positive numbers.
   </details>

### A common mistake

Writing `010` and meaning ten. A leading 0 makes a constant octal, so
`010` is 8.

## 12.2 The bitwise operators

C has six operators that work on the individual bits of integers. Four of
them combine bits in the same position:

| Operator | Name | A result bit is 1 when |
| --- | --- | --- |
| `a & b` | and | the bits of `a` and `b` are both 1 |
| `a \| b` | or | at least one of them is 1 |
| `a ^ b` | exclusive or | exactly one of them is 1 |
| `~a` | complement | the bit of `a` is 0 (every bit is flipped) |

Do not confuse `&` and `|` with `&&` and `||`. The logical operators treat
a whole number as true or false and give 1 or 0; the bitwise operators work
on each of the 32 bits separately and give a whole new number.

The other two move all the bits of a number sideways. `a << n` shifts the
bits of `a` left by `n` places, filling with zeros on the right; each place
doubles the value, so `5 << 3` is 5 × 8 = 40. `a >> n` shifts them right,
dropping the bits that fall off, which halves the value for each place.

```c run
#include <stdio.h>

void show(const char *label, unsigned v) {
    printf("%-8s", label);
    for (int bit = 7; bit >= 0; bit--)
        putchar((v >> bit) & 1 ? '1' : '0');
    printf("  %3u\n", v);
}

int main(void) {
    unsigned a = 0x5C, b = 0x36;
    show("a", a);
    show("b", b);
    show("a & b", a & b);
    show("a | b", a | b);
    show("a ^ b", a ^ b);
    show("~a", ~a & 0xFF);
    show("a << 1", (a << 1) & 0xFF);
    show("a >> 2", a >> 2);
    return 0;
}
```

The function `show` prints the lowest eight bits of a number and its value,
so the effect of each operator can be read column by column. Compare the
rows: under `a & b` there is a 1 only where both rows above have one; under
`a ^ b`, only where they differ. For `~a` and `a << 1`, the `& 0xFF` keeps
just the lowest eight bits, since `~` flips all 32 and a shift can push a
bit past the eighth.

### Shifting negative numbers

Shifting an unsigned number right always brings in zeros. For a negative
signed number, most compilers copy the sign bit in from the left, so that
`-16 >> 2` is −4, but C does not promise it. For this reason, and because
shifting a 1 into the sign bit is undefined, bit manipulation is done with
unsigned types.

### Check yourself

1. What are `6 & 3`, `6 | 3` and `6 ^ 3`?
   <details><summary>Answer</summary>

   2, 7 and 5. In binary, 6 is `110` and 3 is `011`: they share only the
   middle bit (2), together they cover all three (7), and they differ in the
   outer two (5).
   </details>

2. What is `1u << 10`?
   <details><summary>Answer</summary>

   1024: one shifted left ten places is 2¹⁰.
   </details>

### A common mistake

Forgetting the precedence of the bitwise operators, which is lower than
that of `==`. `if (flags & MASK == 0)` means `flags & (MASK == 0)`. Write
`if ((flags & MASK) == 0)`.

## 12.3 Masks: setting, clearing and testing bits

A common use of bits is to pack several yes-or-no settings into one
number, each setting in a bit of its own. The permissions of a file, read,
write and execute, are kept like this. To work with one bit, we use a
*mask*: a number that has 1s exactly in the bits we care about. A mask with
a single 1 in position `n` is written `1u << n`.

With a mask `m`, four operations cover everything:

- `x | m` sets the bits of `m` to 1, leaving the others alone.
- `x & ~m` clears them to 0: `~m` has 1s everywhere except those bits.
- `x ^ m` flips them.
- `x & m` keeps only them, so it is non-zero exactly when one of them is set.

```c run
#include <stdio.h>

#define READ  (1u << 0)
#define WRITE (1u << 1)
#define EXEC  (1u << 2)

int main(void) {
    unsigned perms = 0;
    perms |= READ | WRITE;      /* set two bits */
    perms &= ~WRITE;            /* clear one */
    perms ^= EXEC;              /* flip one */
    printf("read %d, write %d, exec %d (0x%x)\n",
           (perms & READ) != 0, (perms & WRITE) != 0, (perms & EXEC) != 0, perms);
    int count = 0;
    for (unsigned v = 0xF0; v != 0; v &= v - 1)
        count++;
    printf("0xF0 has %d bits set\n", count);
    return 0;
}
```

Step through it and watch `perms` in the memory picture. After line 9 it
holds 3 (`011`), after line 10 it holds 1 (`001`), and after line 11 it
holds 5 (`101`).

The loop at the end counts the bits that are 1, with a well-known trick:
`v & (v - 1)` clears the lowest 1 bit of `v`. Subtracting one turns the
lowest 1 into a 0 and every 0 below it into a 1; the `&` with the original
then clears all of those. So the loop runs exactly once for each bit that
was set. Watch `v` lose one bit per round: `0xF0`, `0xE0`, `0xC0`, `0x80`,
and 0.

### Fields of several bits

A mask can also cover a group of bits that together hold a small number.
Here is a function that extracts `n` bits of `x`, the highest of which is
bit `p`:

```c run
#include <stdio.h>

unsigned getbits(unsigned x, int p, int n) {
    return (x >> (p + 1 - n)) & ~(~0u << n);
}

int main(void) {
    unsigned short date = (26u << 9) | (10u << 5) | 11u;
    printf("packed: 0x%04x\n", date);
    printf("year %u, month %u, day %u\n",
           getbits(date, 15, 7), getbits(date, 8, 4), getbits(date, 4, 5));
    return 0;
}
```

The program packs a date into 16 bits: 5 bits for the day (up to 31), 4
for the month (up to 15) and 7 for the year (up to 127). The day goes into
the lowest five bits, the month is shifted above it, and the year above
that, and `|` joins them. `getbits` takes them apart again. It shifts the
wanted field down to the right end, then masks off everything above it.
The mask `~(~0u << n)` is worth reading carefully: `~0u` is all ones;
shifted left by `n`, it has `n` zeros at the right end; inverted, it has
`n` ones at the right end and zeros above, which is exactly the mask
needed.

C can also do this packing automatically, with *bit-fields*: members of a
structure declared with a width in bits, such as `unsigned int day: 5;`.
The compiler then does the shifting and masking. Vizu does not run
bit-fields yet, so the explicit version is shown here; it is also what the
compiler does underneath.

### Check yourself

1. What does the first program print?
   <details><summary>Answer</summary>

   `read 1, write 0, exec 1 (0x5)`, then `0xF0 has 4 bits set`.
   </details>

2. How do you test whether bit 3 of `x` is set?
   <details><summary>Answer</summary>

   `(x & (1u << 3)) != 0`, or `(x >> 3) & 1`.
   </details>

### A common mistake

Writing `&&` where `&` was meant, as in `if (perms && WRITE)`. That is
true whenever `perms` is not zero at all, because `WRITE` is not zero; it
does not test the write bit.

## 12.4 Exclusive or: a tiny cipher

Exclusive or has a property that the other operators lack: applying it
twice with the same value gives back what you started with. `(x ^ k) ^ k`
is `x`, because every bit flipped by `k` the first time is flipped back the
second time. That makes it the simplest possible cipher: to hide a
message, `^` each character with a key; to read it again, do exactly the
same thing.

```c run
#include <stdio.h>

int main(void) {
    char msg[] = "Attack at dawn";
    char key = '&';
    for (int i = 0; msg[i] != '\0'; i++)
        msg[i] ^= key;
    printf("hidden:");
    for (int i = 0; msg[i] != '\0'; i++)
        printf(" %02x", (unsigned char) msg[i]);
    printf("\n");
    for (int i = 0; msg[i] != '\0'; i++)
        msg[i] ^= key;
    printf("again:  %s\n", msg);
    return 0;
}
```

Some of the hidden characters are not printable, so the program shows them
in hexadecimal. Step through the first loop and watch the letters in `msg`
turn into other characters one by one; then watch the third loop turn them
back.

This cipher is far too weak for real secrets, since a single repeated key
is easy to find. But the same operation, with a long and unpredictable
key, is at the heart of real stream ciphers.

### Check yourself

1. What is `'A' ^ ' '`, and why is that interesting?
   <details><summary>Answer</summary>

   `'a'`. A space is 32, a single bit, and in ASCII that bit is the only
   difference between a capital and a small letter. `^ ' '` flips the case
   of a letter.
   </details>

2. What happens if a character in the message equals the key?
   <details><summary>Answer</summary>

   It becomes 0, the `'\0'` that ends a string. The loops that stop at
   `'\0'` would then stop early, which is why real programs keep such data
   as bytes with a separate length, not as a string.
   </details>

### A common mistake

Printing the hidden bytes with `%c` or `%s`. Some of them are control
characters that move the cursor or do nothing visible, so the output is
confusing. Print bytes you do not trust in hexadecimal.

## 12.5 Looking at an `int` byte by byte

An `int` takes four bytes. In which order are they stored? You might
expect the most significant byte first, the way we write numbers, and
some computers do it that way. But the processors in nearly every PC and
phone today store the *least* significant byte first, at the lowest
address. This is called *little-endian* order; the other is *big-endian*.

To see it, we need to look at the bytes of an `int` one by one. C allows a
pointer to `unsigned char` to point at any object and read it a byte at a
time; it is the one type with that permission.

```c run
#include <stdio.h>

int main(void) {
    unsigned int n = 0x11223344;
    unsigned char *bytes = (unsigned char *) &n;
    for (int i = 0; i < 4; i++)
        printf("byte %d at %p: 0x%02x\n", i, (void *) (bytes + i), bytes[i]);
    printf(bytes[0] == 0x44 ? "little-endian\n" : "big-endian\n");
    return 0;
}
```

`bytes` points to the first byte of `n`, and since it is a pointer to a
one-byte type, `bytes + i` moves one byte at a time (lesson 7.1). The
program prints the bytes in the order they lie in memory: `0x44`, `0x33`,
`0x22`, `0x11`. The lowest byte of the number comes first.

Step through it and look at the arrow from `bytes`: it points into the
middle of `n`'s box, at single bytes, which no `int *` could do. Then rest
the pointer on `n` to see its four bytes laid out in binary.

### Check yourself

1. What would a big-endian computer print as byte 0?
   <details><summary>Answer</summary>

   `0x11`: the most significant byte first, the way we write numbers.
   </details>

2. When does the order of bytes matter?
   <details><summary>Answer</summary>

   When bytes leave the program: in files, in network packets (which use
   big-endian, called network order), or in memory shared with hardware.
   Inside one program it never shows, unless the program looks at the
   bytes, as this one does.
   </details>

### A common mistake

Reading an `int` through a pointer to some unrelated type, such as a
`float *`, to see its bits. C's rules allow the compiler to assume that
this never happens, and the optimized program may produce nonsense. Use
`unsigned char *`, `memcpy`, or a union (lesson 9.7) instead.
