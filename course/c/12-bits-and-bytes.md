# Module 12 · Bits and bytes

Read alongside: King chapters 7 and 20 · K&R sections 2.9 and 6.9

Under every type is a row of bits. This module looks at them directly: how
numbers are written in binary, how to set and test single bits, and in what
order an `int`'s bytes sit in memory. It is the doorway to the assembly and
CPU tracks.

## 12.1 Binary, hex and two's complement

**Read:** King 7.1 (octal and hexadecimal), 20.1 · K&R 2.3
**You will:** read a number in binary and hex, and know how negatives are
stored.

A byte is 8 bits, so it is written as two hex digits (each hex digit is 4
bits): 42 is `0010 1010` in binary, `0x2a` in hex. Negative integers are
stored in *two's complement*: −x is the bits of x inverted, plus one. So −1 is
all ones, `0xffffffff` in 32 bits, and the top bit tells the sign.

```c run
#include <stdio.h>

int main(void) {
    int n = 42;
    printf("%d = 0x%x = 0%o\n", n, n, n);
    for (int bit = 7; bit >= 0; bit--)
        putchar((n >> bit) & 1 ? '1' : '0');
    putchar('\n');
    int m = -1;
    printf("-1 as unsigned: %u = 0x%x\n", (unsigned) m, (unsigned) m);
    return 0;
}
```

`(n >> bit) & 1` moves the bit we want to the far right and keeps only it.

### Check yourself

1. What is `0xff` in decimal? And `0x100`?
   <details><summary>Answer</summary>

   255 and 256.
   </details>

2. In 8 bits, what is −128 in binary, and why is there no +128?
   <details><summary>Answer</summary>

   `1000 0000`. Eight bits have 256 patterns: zero, 127 positives and 128
   negatives.
   </details>

### Common mistake

Writing `010` and meaning ten. A leading 0 makes it octal: `010` is 8.

## 12.2 Bitwise operators and masks

**Read:** K&R 2.9 · King 20.1
**You will:** set, clear, flip and test single bits.

`&` (and), `|` (or), `^` (xor) and `~` (not) work on each bit separately;
`<<` and `>>` shift bits left and right. A *mask* is a number with ones where
the bits you care about are. `x |= m` sets those bits, `x &= ~m` clears them,
`x ^= m` flips them, and `x & m` tests them.

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

`v &= v - 1` clears the lowest set bit (K&R exercise 2-9), so the loop runs
once per bit that is 1.

### Check yourself

1. What is `perms` at the end, in hex?
   <details><summary>Answer</summary>

   `0x5`: READ (1) and EXEC (4).
   </details>

2. Why `1u << 0` and not `1 << 0`?
   <details><summary>Answer</summary>

   Shifting a signed 1 into the sign bit (`1 << 31`) is undefined. Unsigned
   values make bit work predictable.
   </details>

### Common mistake

`&&` for `&`, or `||` for `|`. `perms && WRITE` is 1 whenever both are
non-zero; it does not test the bit.

## 12.3 Looking at an `int` byte by byte

**Read:** King 20.3 (Using Pointers as Addresses) · K&R 6.9
**You will:** see the order of an `int`'s bytes in memory.

An `unsigned char *` can point at any object and read it one byte at a time;
it is the one type C allows to look at the bytes of anything. Doing it to an
`int` shows the machine's *byte order*. On x86 and most ARM machines the
least significant byte comes first: *little-endian*.

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

**Watch:** `bytes` points to the first byte of `n`; `bytes + i` steps one byte
at a time, since its type is a pointer to a 1-byte `char`.

### Check yourself

1. What would a big-endian machine print for byte 0?
   <details><summary>Answer</summary>

   `0x11`: the most significant byte first, the way we write numbers.
   </details>

2. When does byte order matter?
   <details><summary>Answer</summary>

   When bytes leave the program: files, network packets (which use big-endian,
   "network order"), memory shared with hardware. Inside one program it never
   shows unless you look at bytes as here.
   </details>

### Common mistake

Reading an `int` through a `float *` (or any unrelated type) to "see its
bits". That breaks C's aliasing rules and the optimizer may produce nonsense;
copy the bytes with `memcpy` instead, or use `unsigned char *`.
