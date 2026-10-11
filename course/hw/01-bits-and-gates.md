# Module 1 · Bits and gates

Every program in the C track ended up as instructions for a processor, and
every variable as bytes in memory. But what *is* a processor? What is a
byte, physically? How can a lump of silicon add two numbers, or remember
one? This track answers those questions from the very bottom, and it does
so the way the book *But How Do It Know?* does: by building a whole
computer out of one simple part, step by step, until nothing about it is
mysterious.

The surprising thing about computers is how little there is inside them.
There are no letters in there, no pictures, no numbers. There is only one
kind of thing, repeated billions of times, and it can do only one thing: be
on or off. Everything else is the way those on-and-off things are
connected to each other.

This first module introduces that thing, the *bit*, and the part that
switches bits, the *gate*. By its end you will have built, from one kind of
gate, every other gate a computer needs.

## 1.1 What a computer is made of

Think about an ordinary light switch on a wall. It can be in one of two
states: on or off. There is no "half on", no "a little bit on"; it is one or
the other, and you can change it from one to the other whenever you like.
Anything that can be in exactly two states, and can be changed between
them, can store one *bit* of information. The word is short for "binary
digit", and we usually write the two states as 1 (on) and 0 (off).

A computer is made of bits. Not mechanical switches, of course, but tiny
places on a chip where there either is electricity or there is not. A wire
is a bit: it carries electricity or it does not. That is the whole story of
what is inside: a very large number of such places, each either on or off,
and connections between them.

How can on-and-off places do anything interesting? The answer comes in two
parts, and this track is about both.

The first part is *codes*. A single bit can only say yes or no, but a group
of bits can say much more. Two bits have four combinations, 00, 01, 10 and
11; eight bits have 256. If we agree that a certain pattern of eight bits
stands for the letter A, and another for the number 65, then a row of bits
can hold letters and numbers. The bits themselves do not know what they
mean. Their meaning is an agreement, a code, and module 2 looks at the codes
computers use.

The second part is *gates*: parts that look at some bits and switch another
bit on or off depending on what they see. With gates, bits can control
other bits, and from that comes everything: remembering, adding, choosing,
and finally running a program.

### How fast?

The other secret of computers is speed. A computer does only very simple
things, but it does them unimaginably fast. A bit changes from off to on in
less than a billionth of a second, because nothing moves except
electricity. A program that seems to do something clever, like showing a
page of text, is in fact millions of tiny steps, each of them trivial, done
one after another faster than any person could follow. Vizu's job in this
track is to slow those steps down until you can watch every one of them.

### Check yourself

1. How many different patterns can three bits make?
   <details><summary>Answer</summary>

   Eight: 000, 001, 010, 011, 100, 101, 110 and 111. Each extra bit
   doubles the number.
   </details>

2. Does the pattern `0100 0001` mean 65 or the letter A?
   <details><summary>Answer</summary>

   Either, or something else entirely. The bits are only bits. What they
   mean depends on the code that the program reading them agrees to use.
   </details>

## 1.2 The NAND gate

Now we need a bit that is controlled by other bits. Imagine a switch that
is not flipped by a finger but by electricity: when electricity reaches its
control wire, it closes, and when the electricity goes away, it opens. Early
computers used relays for this, switches pulled shut by a small magnet;
today's computers use *transistors*, which do the same job with no moving
parts at all, and are so small that billions fit on one chip.

From a few such switches, engineers build a part called a *gate*. A gate
has inputs, wires coming in that are on or off, and an output, a wire
going out whose state depends on the inputs. The gate that everything else
in this track is built from is called the *NAND gate*. It has two inputs
and one output, and it follows one simple rule:

> The output is off only when both inputs are on. In every other case, the
> output is on.

The name comes from "not and": the output is on when it is *not* the case
that input a *and* input b are both on.

Here is a NAND gate to try. The two switches on the left are its inputs;
click them to turn them on and off. The light on the right is its output.
Wires that are on are drawn dark and thick.

```circuit nand
```

Try all four combinations and watch the table beside the gate. It is
called a *truth table*, and it lists every combination of the inputs with
the output that goes with it. The row that matches the switches right now
is highlighted; you can also click a row to set the switches to it. Notice
that only one row, the one where both inputs are 1, has an output of 0.

That is all a NAND gate does. It may seem a strange and useless rule to
build a computer on, but the next lessons show that every other gate, and
then memory, arithmetic and decisions, can be made from NAND gates alone.

### Check yourself

1. A NAND gate has its inputs at 1 and 0. What is its output?
   <details><summary>Answer</summary>

   1. The output is 0 only when both inputs are 1.
   </details>

2. Why does the truth table of a gate with two inputs have four rows?
   <details><summary>Answer</summary>

   Two inputs, each 0 or 1, make four combinations, and the table lists
   them all.
   </details>

## 1.3 NOT and AND

Let us build our first new gate. What happens if both inputs of a NAND
gate are connected to the same wire? Then the two inputs are always equal:
both on or both off. When the wire is on, both inputs are on, and the
output is off. When the wire is off, both inputs are off, and the output is
on. The output is always the opposite of the input.

```circuit not
```

A gate whose output is the opposite of its input is called a *NOT gate*,
or an inverter. It has only one input, and its truth table has only two
rows. It is drawn as a small triangle with a circle at its tip; the circle,
here and on the NAND gate's symbol, is the sign for "turned the other way
round".

Now put a NOT gate after a NAND gate. The NAND's output is the opposite of
"both inputs on", and the NOT turns it round again, so the final output is
simply "both inputs on":

```circuit and
```

This is the *AND gate*: its output is on only when both inputs are on. The
column `c` in the table shows the wire between the two gates, the NAND's
output, so you can follow the signal through. Compare it with the final
output `d`: every bit is turned round.

With NAND, NOT and AND we already have something useful. An AND gate is a
way for one bit to control whether another gets through: if input a is
off, the output is off whatever b does; if a is on, the output follows b. Bits
that allow or block other bits are exactly what a computer needs, and AND
gates will be everywhere in the machine we build.

### Check yourself

1. How many NAND gates are inside an AND gate built this way?
   <details><summary>Answer</summary>

   Two: one NAND, and one more NAND with its inputs joined, acting as the
   NOT.
   </details>

2. An AND gate has its input a on. How does its output behave?
   <details><summary>Answer</summary>

   It copies input b: on when b is on, off when b is off. With a off, the
   output stays off whatever b does.
   </details>

## 1.4 OR and XOR

Two more combinations complete the set of gates the computer needs.

The first turns both inputs of a NAND gate round before they arrive: a NOT
gate on each input. Think through what that does. The NAND's output is off
only when both of its inputs are on, and its inputs are on only when the
original inputs are off. So the final output is off only when both original
inputs are off, and on in every other case: when a is on, or b is on, or
both.

```circuit or
```

This is the *OR gate*. It is drawn with a curved back and a pointed front.
The columns `c` and `d` show the two turned-round inputs, so you can check
the reasoning row by row.

The second combination takes five gates, but its rule is easy to say: the
output is on when the inputs are *different*, and off when they are the
same. It is called the *exclusive OR* gate, or *XOR*, because its output is
on when one input or the other is on, but not both.

```circuit xor
```

Follow it through. `c` and `d` are the two inputs turned round. The upper
NAND gives `e`, which is off only when a is on and b is off; the lower NAND
gives `f`, which is off only when a is off and b is on. The last NAND
turns its output on when either `e` or `f` is off, that is, in exactly those
two cases where the inputs differ.

We now have four gates with two inputs, and it is worth looking at their
truth tables side by side:

| a | b | NAND | AND | OR | XOR |
| --- | --- | --- | --- | --- | --- |
| 0 | 0 | 1 | 0 | 0 | 0 |
| 0 | 1 | 1 | 0 | 1 | 1 |
| 1 | 0 | 1 | 0 | 1 | 1 |
| 1 | 1 | 0 | 1 | 1 | 0 |

Every one of them, remember, is made of NAND gates and nothing else. These
are the same operations C offers on whole numbers as `&`, `|`, `^` and `~`
(module 12 of the C track): a processor carries them out with exactly these
gates, one per bit.

### Check yourself

1. What is the output of an OR gate whose inputs are both 1? And of an XOR
   gate?
   <details><summary>Answer</summary>

   1 for OR, 0 for XOR. XOR is on only when the inputs differ.
   </details>

2. How could you make an AND gate with three inputs?
   <details><summary>Answer</summary>

   Feed the output of one AND gate, with inputs a and b, into a second AND
   gate together with c. The final output is on only when all three are
   on. In the same way, AND and OR gates can have as many inputs as needed.
   </details>

## 1.5 The decoder

The last combination of this module has more outputs than inputs. It takes
two input bits, a and b, and has four outputs, one for each of the four
combinations that a and b can be in. Exactly one output is on at any time:
the one that matches the inputs.

```circuit decoder
```

Inside, each input is split into two wires: the input itself, and the
input turned round by a NOT gate. That makes four wires running down the
middle of the drawing: a, not a, b and not b. Each of the four AND gates on
the right is connected to a different pair of those wires. The top one
looks at "not a" and "not b", so it is on only when both inputs are 0; the
next looks at "not a" and "b", and so on. Whatever the inputs, exactly one
AND gate has both of its inputs on.

This part is called a *decoder*, because it turns a code, the two input
bits, into a choice of one wire out of four. With three inputs and eight
AND gates of three inputs each, a decoder chooses one of eight; with four
inputs, one of sixteen. Decoders are named by their inputs and outputs: 2 ×
4, 3 × 8, 4 × 16.

Why does a computer need such a thing? Because it constantly has to choose
one thing out of many by a number: one byte out of all the bytes in memory,
one register out of four, one operation out of eight. A decoder turns the
number into a single wire that selects the one. The next modules use
decoders for all three.

### Check yourself

1. In the 2 × 4 decoder, which output is on when a is 1 and b is 0?
   <details><summary>Answer</summary>

   The one labelled `1/0`, the third. Its AND gate is connected to "a" and
   "not b".
   </details>

2. How many outputs does a decoder with 8 inputs have?
   <details><summary>Answer</summary>

   256: one for each combination of eight bits. Module 3 needs choices of
   that size, and builds them from two smaller decoders.
   </details>
