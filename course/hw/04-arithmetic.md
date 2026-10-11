# Module 4 · Arithmetic and logic

We have a way to remember bytes, in registers and in RAM, and a way to
move them, along the bus. The computer is half built. What it cannot do yet
is *change* a byte: add two numbers, compare them, turn their bits round.
This module builds those abilities, once more from nothing but gates, and
gathers them into one part, the *arithmetic and logic unit*, or ALU, which
sits at the heart of the processor.

The ALU is less mysterious than its name. It is a box with two bytes going
in, one byte coming out, and a few extra bits that say what it should do and
report what happened. Inside are a handful of simple devices, each working
on all eight bits of a byte at once.

## 4.1 Working on whole bytes

The gates of module 1 work on single bits. To work on a byte, we simply
use eight of them side by side, one for each bit.

Eight NOT gates, each connected to one bit of an input byte, make a
*NOTter*: every bit of the output is the opposite of the matching bit of
the input. Eight AND gates, each taking one bit from byte A and the bit in
the same place from byte B, make an *ANDer*: each output bit is on only
where both bytes have a 1. Eight OR gates make an *ORer*, and eight XOR
gates an *XORer*. These are exactly C's `~`, `&`, `|` and `^` (module 12 of
the C track), and this is how a processor carries them out: all eight bits
at the same moment.

### Shifting

Two more devices need no gates at all, only clever wiring. A *right
shifter* connects each bit of the input to the output bit one place to the
right. The rightmost input bit has nowhere to go, so it comes out on a
separate wire, *shift out*; the leftmost output bit has no input above it,
so it takes a separate wire, *shift in*. The result is every bit moved one
place right: `0001 0110` (22) becomes `0000 1011` (11). Moving every bit of a
binary number one place to the right halves it, just as moving the digits
of a decimal number one place to the right divides it by ten. A *left
shifter* does the opposite, and doubles.

Here is an ALU to experiment with, built from all of these devices and the
ones of the next lessons. Choose an operation with the buttons in the
middle; click the bits of A and B to change them. For now, try NOT, AND, OR,
XOR, SHR and SHL.

```circuit alu
```

Look at the bits rather than the numbers. With AND, a bit of the output is
on only where both A and B have one; with XOR, only where they differ. With
SHR and SHL, the whole pattern slides along, and the bit that falls off the
end lights *carry out*.

### Check yourself

1. What is `0101 1100` shifted left once?
   <details><summary>Answer</summary>

   `1011 1000`. The value goes from 92 to 184: doubled.
   </details>

2. Which operation turns every bit of a byte off, whatever the byte is,
   when the byte is used as both inputs?
   <details><summary>Answer</summary>

   XOR: a byte XOR itself has no bits that differ, so the output is all
   zeros. The book's programs use `XOR R2, R2` exactly for this.
   </details>

## 4.2 The adder

Adding is the most important thing a computer does with numbers, and it
too is built from gates. Let us start with how we add by hand.

To add two decimal numbers, we add the digits of one column at a time,
starting from the right. If the sum of a column is 10 or more, we write down
its last digit and carry one to the next column on the left. Binary addition
works exactly the same way, only with two digits. In one column there are
just three possibilities: 0 + 0 is 0; 0 + 1 is 1; and 1 + 1 is 2, which in
binary is written `10`: write down 0 and carry 1. And if a carry is coming in
from the column to the right, 1 + 1 + 1 is 3, `11`: write down 1, carry 1.

So the part that adds one column needs three inputs, the bit of A, the bit
of B, and the carry coming in, and two outputs, the sum bit to write down,
and the carry going out to the next column. Here it is, built from two XOR
gates, two AND gates and an OR gate:

```circuit adder
```

Check it against the rules. The sum bit is on when an odd number of the
three inputs are on, which is what two XOR gates in a row compute. The
carry out is on when at least two of the inputs are on: either both a and b
(the lower AND gate), or one of them together with the carry in (the upper
AND gate). Click through the table: it is the addition table of binary
digits, written out completely.

### Eight columns

To add two bytes, we place eight of these one-bit adders side by side,
one for each column, and connect the carry out of each column to the carry
in of the column to its left, just as we carry by hand. The carry in of the
rightmost column is a separate input of the whole adder, and the carry out
of the leftmost column is a separate output.

Here is the ALU of lesson 4.1 again. Choose ADD, and try 23 + 19. Then try
two numbers whose sum is more than 255, such as 200 + 100.

```circuit alu
```

 The output cannot hold 300,
which needs nine bits; it keeps the lowest eight bits, 44, and the ninth
bit comes out as *carry out*. That carry is not lost: it can be added into
the next byte of a larger number, which is how an 8-bit machine adds
numbers bigger than a byte. Turn *carry in* on to see it added in.

### Check yourself

1. What is `0110` + `0011` in binary?
   <details><summary>Answer</summary>

   `1001`: 6 + 3 = 9. In the second column from the right, 1 + 1 makes 0
   carry 1; the carries ripple to the left.
   </details>

2. Why is there a carry in on the rightmost column, where nothing comes
   from the right?
   <details><summary>Answer</summary>

   So that a carry left over from adding a previous byte can be added in.
   Two bytes added with the carry joined make one 16-bit addition. The
   processor of the next modules uses this.
   </details>

## 4.3 Comparing, and testing for zero

Programs constantly need to know how two numbers relate: is the counter
equal to ten yet, is this score larger than the best one? That calls for a
*comparator*.

Comparing two bytes is done one bit at a time too, starting from the left,
the most significant bit, which is how you compare two numbers of equal
length by eye. As long as the bits are equal, nothing is decided yet. At the
first place where they differ, the byte with the 1 is the larger, and the
rest of the bits no longer matter. The comparator is a chain of eight small
circuits, one per bit, each made of an XOR gate (to see whether the two
bits differ) and a few AND and OR gates (to say which byte has the 1, and to
pass "equal so far" or "already decided" down the chain). At its end come
two answers: *equal*, on when every pair of bits matched, and *A larger*, on
when the first difference had the 1 in A.

One more small device watches the ALU's output: a *zero detector*, an OR
gate with eight inputs followed by a NOT gate. The OR is on if any output
bit is on; turned round, it says *zero*: on only when every bit of the
output is 0.

In the ALU, the comparator and the zero detector work all the time,
whatever operation is chosen. Watch *A larger* and *equal* as you change
A and B, and *zero* as the output becomes 0.

```circuit alu
```

 These answers will matter a
great deal in module 6, because they are how a program makes decisions.

### Check yourself

1. Compare `1001 0000` with `0111 1111`. Which is larger, and which bit
   decides it?
   <details><summary>Answer</summary>

   The first, 144 against 127. The leftmost bit already differs, and the 1
   is in the first byte; the seven bits after it do not matter.
   </details>

2. When is *zero* on after A XOR B?
   <details><summary>Answer</summary>

   When A and B are equal: no bit differs, so the output is all zeros.
   </details>

## 4.4 The ALU

Now we have seven devices that work on bytes: the adder, the two shifters,
the NOTter, the ANDer, the ORer and the XORer. The ALU puts them all in one
box. All of them are connected to the two inputs all the time, and all of
them compute their answer all the time. What decides which answer comes
out is a 3 × 8 decoder, driven by three input bits called *op*, for
operation. The decoder turns on exactly one of eight wires, and each of the
seven devices has an enabler on its output, opened by one of those wires.
So only the chosen device's answer reaches the ALU's output.

The three op bits are a small code of their own:

| op | operation |
| --- | --- |
| 000 | ADD: A + B |
| 001 | SHR: A shifted right |
| 010 | SHL: A shifted left |
| 011 | NOT: A turned round |
| 100 | AND: A and B |
| 101 | OR: A or B |
| 110 | XOR: A exclusive or B |
| 111 | CMP: compare only |

The eighth code, 111, selects no device at all. The output is then empty,
all zeros, but the comparator still works, so this is the code to use when
only the comparison matters. That is why it is called CMP, compare.

The three carry wires, of the adder and the two shifters, are joined into
one *carry in* and one *carry out* for the whole ALU. Together with the
comparator's two answers and the zero detector, the ALU therefore reports
four bits about every operation: *carry*, *A larger*, *equal* and *zero*.

The ALU below shows all of it at once: the two inputs, the op code on each
button, carry in, the output, and the four bits it reports. It is the same
ALU, wire for wire, that sits inside the processor of the next module.

```circuit alu
```

### Check yourself

1. With op set to 101, A to `1100 0000` and B to `0000 0011`, what comes out?
   <details><summary>Answer</summary>

   `1100 0011`: 101 is OR, so every bit that is on in either byte is on in
   the output.
   </details>

2. Why can the ALU leave all seven devices computing at once, and still give
   only one answer?
   <details><summary>Answer</summary>

   Each device's output passes through an enabler, and the op decoder opens
   only one of them. The other answers are computed and thrown away, which
   costs nothing, since the gates are there anyway.
   </details>
