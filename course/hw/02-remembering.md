# Module 2 · Remembering

The gates of module 1 have no memory. Their output depends only on their
inputs at this very moment: change an input, and the output changes with
it, and nothing is left behind of what was there before. A computer that
could only do that would be like a calculator whose display goes blank the
moment you lift your finger from a key.

So the next step is a part that *remembers*: one that holds a bit even
after the input that set it has gone away. Remarkably, it can be built out
of the very same NAND gates, connected in a loop. From one such memory bit
we then build a byte, a register, and the bus that moves bytes between
registers, which is the main road of the whole computer.

## 2.1 A bit that remembers

Here is the memory bit. It is made of four NAND gates, and the trick that
gives it memory is that two of the gates feed each other: the output of
gate 3 goes back round into gate 4, and the output of gate 4 goes back into
gate 3.

```circuit memory
```

It has two inputs. `i` is the bit we want to remember. `s`, for *set*, says
when to remember it. And it has one output, `o`, the bit it remembers.

Try it in this order, and read the log beside the drawing after each click:

1. Turn `s` on. Now change `i` back and forth: `o` follows it. While set is
   on, the memory bit simply passes its input through.
2. With `i` on, turn `s` off. `o` stays on.
3. Now change `i`. Nothing happens to `o`. The memory bit has kept the
   value `i` had at the moment `s` went off, and it will keep it for as long
   as you like, whatever `i` does.

How does it do that? Follow the wires when `s` is off. Gates 1 and 2 both
have `s` as one input, and a NAND gate with an input off always gives 1. So
`a` and `b` are both 1, whatever `i` is: the input is shut out. Gates 3 and
4 now hold each other in place. If `o` is 1, gate 4 has one input at 1 (b)
and one at 1 (o), so `c` is 0, and gate 3, with `c` at 0, keeps `o` at 1. If
`o` is 0, the same reasoning keeps it at 0. Either way, the loop holds its
state.

When `s` is on, gates 1 and 2 let `i` in: `a` becomes the opposite of `i`,
`b` the opposite of `a`, and those force the loop into the state that
matches `i`.

This is the first moment in this track where a computer part has a past.
Its output does not depend only on what its inputs are now, but on what
they were. Every memory in every computer, the registers inside the
processor and the billions of bytes of RAM, rests on this idea of a loop of
gates that holds itself in place. (Real RAM chips use a cheaper trick, a
tiny charge stored on a capacitor that must be refreshed many times a
second, but the processor's own registers work like this.)

### Check yourself

1. `s` is off and `o` is 0. Then `i` goes to 1. What is `o`?
   <details><summary>Answer</summary>

   Still 0. While `s` is off, the input is shut out.
   </details>

2. What does the memory bit remember when the power is switched off?
   <details><summary>Answer</summary>

   Nothing. The loop holds its state only while electricity flows through
   the gates. That is why a computer's memory is empty when it is switched
   on, and why programs and files must be kept on a disk.
   </details>

## 2.2 Eight bits: the byte

One bit holds very little. Put eight memory bits side by side, connect all
eight of their `s` inputs to one wire, and we have something that can hold
eight bits at once, set all together. Eight bits is called a *byte*, and the
byte is the unit almost every computer uses to store and move information.

Why eight? Partly history, partly convenience. Eight bits have 256
combinations, which is enough for the letters, digits and punctuation of
English, with room to spare. And eight is a power of two, which makes the
wiring tidy.

### What a byte can mean

A byte is a row of eight bits, and as module 1 said, the bits do not know
what they mean. The meaning comes from a code.

The most natural code is the *binary number* code. It works like our
ordinary numbers, but with two digits instead of ten. In a decimal number,
each place is worth ten times the place to its right: ones, tens, hundreds.
In a binary number, each place is worth *two* times the place to its right:

| bit | 8th | 7th | 6th | 5th | 4th | 3rd | 2nd | 1st |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| worth | 128 | 64 | 32 | 16 | 8 | 4 | 2 | 1 |

To read a byte as a number, add up the worth of every place that holds a
1. So `0000 0101` is 4 + 1 = 5, and `0100 0001` is 64 + 1 = 65. The smallest
byte is `0000 0000`, which is 0, and the largest is `1111 1111`, which is
255. (Bytes are usually written in two groups of four, only to make them
easier for people to read.)

The other everyday code is for text. The *ASCII* code gives each letter,
digit and punctuation mark a number from 0 to 127: `A` is 65, `B` is 66, `a`
is 97, the digit `0` is 48, and a space is 32. There is nothing natural about
these numbers; a committee chose them in the 1960s, and everyone agreed to
use them. A byte holding `0100 0001` is the number 65 to a program that
treats it as a number, and the letter A to a program that treats it as
text. It is the same byte either way. (This is exactly what the C track
showed in its lesson on characters: a `char` is a small number.)

Other codes give bytes other meanings: the colour of a dot on the screen,
part of a sound, or, as we will see in module 6, an instruction for the
processor.

### Check yourself

1. What number is `0010 1010`?
   <details><summary>Answer</summary>

   42: 32 + 8 + 2.
   </details>

2. Which byte stands for the letter `C` in ASCII?
   <details><summary>Answer</summary>

   67, which is `0100 0011`: 64 + 2 + 1. The capital letters follow each
   other from 65 for A.
   </details>

## 2.3 Registers and the bus

A computer is mostly busy moving bytes from one place to another: from
memory to the processor, from one register to another, through the
arithmetic unit and back. To move bytes, it needs a road, and the road is
called the *bus*. A bus is nothing more than eight wires running side by
side, one for each bit of a byte, that pass by every part that needs to
send or receive bytes.

If many parts are connected to the same eight wires, there must be a rule
about who may put a byte on them, or several bytes would mix into nonsense.
The rule is simple: at any moment, only one part may put its byte on the
bus, and any number of parts may take a copy of what is there.

To let a byte out onto the bus only when it is allowed, each storage place
gets an *enabler*: eight AND gates, one per bit, all controlled by one wire
called *enable*, written *e*. When enable is off, every AND gate gives 0, and
nothing reaches the bus. When enable is on, the AND gates let the eight
stored bits through. (This is the use of AND gates that lesson 1.3 promised:
one bit allowing other bits through.)

A byte of memory bits together with an enabler is called a *register*. It
has two control wires: *set*, which makes its memory bits take whatever byte
is on the bus, and *enable*, which puts its own byte onto the bus.

```circuit register
```

The top row is the bus coming in: click its bits to put a different byte
on it. The middle row is the register's eight memory bits. The bottom row is
what the register puts out onto the bus. Try this: put a byte on the bus,
press *set*, then change the bus again. The register keeps the byte it took
at the moment set went on and off. Then turn *enable* on, and the kept byte
appears on the bus.

### Moving a byte

Now imagine several registers connected to the same bus. Moving a byte
from register R1 to register R2 takes just two control wires: turn on R1's
*enable*, so that R1's byte is on the bus, and while it is there, turn R2's
*set* on and off, so that R2 takes a copy. Then turn R1's enable off again.
R1 still has its byte, and now R2 has the same one.

Everything a processor does, every instruction of every program, is made
of moves like this one: one part enabled onto the bus, another part set
from it. The rest of this track is about which enable and set wires to turn
on, and when.

### Check yourself

1. What does a register put on the bus while its enable wire is off?
   <details><summary>Answer</summary>

   Nothing: all eight AND gates of its enabler give 0, so it does not
   disturb the bus at all.
   </details>

2. Why may only one register be enabled onto the bus at a time?
   <details><summary>Answer</summary>

   The bus is one set of eight wires. Two bytes on it at once would mix:
   every wire that either byte has on would be on, and the result would be
   neither byte.
   </details>
