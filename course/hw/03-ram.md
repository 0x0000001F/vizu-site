# Module 3 · RAM

A register holds one byte. A computer needs to hold a great many: the
program it is running, the data that program works on, the text being
edited, the picture on the screen. The part of the computer that holds them
is called *RAM*, short for "random access memory". "Random access" means
that any byte can be reached directly, in the same short time, wherever it
is: there is no need to go through the others first, as there is on a tape.

In this module we build RAM from the parts we already have: registers, a
special register that holds the number of the byte we want, and two
decoders. Then we look carefully at that number, the *address*, which is
one of the most important ideas in all of computing, and the one the C
track was built on.

## 3.1 Choosing one byte out of 256

Our RAM will hold 256 bytes. That is tiny, a modern computer has billions,
but the idea is exactly the same, and 256 is the number of different values
one byte can have. That is not a coincidence: it means that a single byte is
enough to say which of the 256 bytes we want.

So RAM has a register of its own, called the *memory address register*, or
MAR. A byte put into MAR selects one of the 256 storage registers. The other
255 are left alone. Then, through the RAM's own *set* and *enable* wires, the
bus can store a byte in the selected register or read the byte it holds.

How does the number in MAR select one register out of 256? With decoders,
from lesson 1.5. One decoder with eight inputs and 256 outputs would work,
but there is a neater way, the one the book uses. The 256 registers are
arranged in a grid of 16 rows and 16 columns. The first four bits of MAR go
into a 4 × 16 decoder, which turns on exactly one of 16 wires running
across the rows. The last four bits go into a second 4 × 16 decoder, which
turns on exactly one of 16 wires running down the columns. At every
crossing of a row wire and a column wire sits an AND gate, and only at one
crossing in the whole grid are both wires on. That AND gate's output is the
selection signal for the register at that crossing: it allows the RAM's set
and enable wires to reach that register, and no other.

```circuit ram
```

Try it. The MAR row holds the address: its first four bits (underlined in
blue) choose the row, and its last four (underlined in amber) the column.
Click its bits and watch the chosen row and column move across the grid,
with the selected register where they cross. Then put a byte on the bus
row and press *set*: it is stored in the selected register, and only there.
Turn *enable* on, and the selected register's byte appears on the bus.

The grid shows each byte in hexadecimal, two digits of four bits each
(lesson 2.2 of the C track explains the notation). The address works the
same way: `0x2A` means row 2, column A. Notice how neatly the hexadecimal
digits of the address match the rows and columns: that is because each
hexadecimal digit is exactly four bits.

### Check yourself

1. MAR holds `0011 0101`. Which row and column are selected?
   <details><summary>Answer</summary>

   Row 3 and column 5: address `0x35`, which is 53.
   </details>

2. How many AND gates does it take to select the registers of a 16 × 16
   grid?
   <details><summary>Answer</summary>

   256, one at each crossing, plus the gates inside the two decoders. That
   is far fewer than one decoder with 256 outputs, each an AND gate with
   eight inputs, would need.
   </details>

## 3.2 Addresses and contents

Every byte in RAM has two numbers connected with it, and it is essential
not to confuse them. Its *address* is where it is: which register in the
grid, the number that MAR must hold to select it. Its *content* is what it
holds: the byte stored in that register. The address of a byte never
changes, since the registers do not move. Its content changes whenever
something is stored there.

An analogy helps. A street of houses has house numbers, and each house has
people living in it. The house number is the address; the people are the
content. Number 12 is always number 12, whoever lives there.

Since both the address and the content are bytes, they look exactly alike.
`0010 1010` might be an address, the house numbered 42, or the content of
some house, the number 42 stored there. The computer only knows which it is
by where the byte goes: a byte that is put into MAR is used as an address,
and a byte that is put into a register of the grid is content.

And here is the idea on which much of computing rests: since an address
is just a byte, it can be stored as the content of another byte. RAM can
hold, in one place, the address of another place. This is precisely what a
*pointer* is in C. In lesson 6.1 of the C track, every variable had an
address on its box; a pointer was a variable whose content was such an
address. Now you can see what that means in the machine: the content of
one register of the grid is a number that, put into MAR, selects another
register.

### Counting in binary

Since addresses are numbers, it is worth being able to count in binary.
Counting works just as in decimal: add one to the rightmost digit, and when
a digit goes past its largest value, set it back to zero and carry one to
the left. In binary, the largest digit is 1, so carries come very often:

| decimal | binary |
| --- | --- |
| 0 | 0000 0000 |
| 1 | 0000 0001 |
| 2 | 0000 0010 |
| 3 | 0000 0011 |
| 4 | 0000 0100 |
| 5 | 0000 0101 |
| … | … |
| 254 | 1111 1110 |
| 255 | 1111 1111 |

One more than 255 would be `1 0000 0000`, a ninth bit that does not fit in a
byte. A machine whose addresses are one byte long can therefore reach only
256 bytes. Real computers use longer addresses, 32 or 64 bits, for exactly
this reason: so that MAR can name more places.

### Check yourself

1. RAM address 7 holds the byte 20, and RAM address 20 holds the byte 99.
   You put 7 into MAR and read. What do you get? And if you then put what you
   read into MAR and read again?
   <details><summary>Answer</summary>

   20, then 99. The first read gives the content of address 7, which is
   itself an address; following it leads to 99. That is what `*p` does in C
   when `p` holds 20.
   </details>

2. How many bytes could a machine with 16-bit addresses reach?
   <details><summary>Answer</summary>

   2¹⁶ = 65 536. Every extra bit of address doubles the number of places
   it can name.
   </details>
