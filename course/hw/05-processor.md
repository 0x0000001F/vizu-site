# Module 5 · The processor

We have all the parts: registers that hold bytes, RAM that holds many, a bus
that carries a byte from any part to any other, and an ALU that computes.
What is missing is something that makes them work together, turning the
right *enable* and *set* wires on and off, in the right order, at the right
moments. That something is the *control section*, and together with the
registers and the ALU it makes the *processor*, or CPU, the central
processing unit.

This module assembles the processor of the book: its registers around the
bus, the clock that gives it a heartbeat, the stepper that counts the steps
of each piece of work, and finally the idea the book calls the first great
invention: a program, kept in RAM, whose instructions the processor fetches
one after another.

## 5.1 Around the bus

The processor has four registers for the program to use, called R0, R1, R2
and R3, each connected to the bus with its own *enable* and *set* wire. Then
there are the parts that serve the ALU:

- **TMP**, a temporary register. The ALU needs two inputs at the same time,
  but the bus can carry only one byte at a time. So the first byte is
  parked in TMP, which feeds the ALU's input B; the second byte comes
  straight from the bus into input A.
- **bus 1**, a small device between TMP and the ALU. Normally it passes
  TMP's byte through, but when its control wire is on it gives the number 1
  instead, `0000 0001`. That is useful whenever something must be counted up
  by one.
- **ACC**, the accumulator, a register that catches the ALU's output. The
  ALU's answer cannot go straight back onto the bus, because the bus is
  already carrying input A while the ALU works; so it waits in ACC, and is
  put on the bus in a later step.
- **FLAGS**, a small register with the four bits the ALU reports: carry,
  A larger, equal and zero.

Now suppose we want to add R1 to R0, and keep the answer in R0. With these
parts it takes three moves along the bus:

1. Enable R1, and set TMP: R1's byte goes into TMP, the ALU's input B.
2. Enable R0, so its byte is on the bus, the ALU's input A; tell the ALU to
   ADD, and set ACC: the sum goes into ACC.
3. Enable ACC, and set R0: the sum goes into R0.

Here is the processor of the book doing exactly that. The first two lines
of the program put two numbers into the registers, and the third is the
addition; module 6 explains how such lines are written. For now, step
forward with the right arrow and stop at steps 4, 5 and 6 of the last
instruction.

```eight run
DATA R0, 7
DATA R1, 5
ADD R1, R0
```

The picture is the whole computer. The thick line across the middle is the
bus. Above it on the left is RAM, with MAR; on the right are the control
section and the four registers. Below it are IAR and IR, which we meet in
lesson 5.4, then TMP, bus 1, the ALU with ACC under it, FLAGS, and on the far
right the devices on the I/O bus.

In each step, the part that is *enabled* has a blue outline and a blue
wire to the bus, and the byte it puts there is written on the bus in a blue
label. The part that is *set* turns amber, the colour this course always
uses for "just written". Watch the three moves of the addition: R1 into TMP,
R0 through the ALU into ACC, ACC into R0. The caption above the picture says
each move in words.

### Check yourself

1. Why does the ALU need TMP, when the bus can carry its input?
   <details><summary>Answer</summary>

   The ALU needs two inputs at once, but the bus carries one byte at a
   time. TMP holds the first input while the bus brings the second.
   </details>

2. Why does the answer go into ACC, and not straight onto the bus?
   <details><summary>Answer</summary>

   At that moment the bus is busy carrying input A. The answer waits in
   ACC and is put on the bus in the next step.
   </details>

## 5.2 The clock

The three moves of the addition must happen one after another, and each
move has an order of its own: the "from" register must be enabled first,
its byte must have time to travel along the bus, then the "to" register must
be set, and only after the set is over may the "from" register be disabled.
If the order went wrong, the "to" register might catch the bus while it was
still changing.

The timing comes from the *clock*. A computer's clock has nothing to do with
the time of day: it is a single bit that turns on and off, on and off,
regularly and very fast, billions of times a second in a modern computer.
That speed, measured in hertz (cycles per second), is the "3 GHz" in the
description of a processor. The clock bit is made in the simplest possible
way: the output of a NOT gate is connected back to its own input, so that it
keeps turning itself round, slowed down to the speed wanted.

The book derives two more bits from the clock. A copy of the clock delayed
by a quarter of a cycle, *clk d*, is combined with the clock itself: their
OR gives *clk e*, which is on for three quarters of each cycle, and their
AND gives *clk s*, which is on for one quarter in the middle of clk e. Step
through one cycle a quarter at a time:

```circuit clock
```

That is exactly the timing a move along the bus needs. The enable wire of
the "from" register is connected to clk e, so it comes on first and goes off
last. The set wire of the "to" register is connected to clk s, so it comes
on only after the byte is on the bus, and goes off while the byte is still
there. When set goes off, the register keeps the byte (lesson 2.1). One
clock cycle, one move along the bus.

In the lab of the last lesson, the control section showed both clock
signals: *clk e* lights blue when something is enabled, and *clk s* amber
when something is set.

### Check yourself

1. Why must the "to" register be set *after* the "from" register is enabled?
   <details><summary>Answer</summary>

   The byte needs a moment to arrive on the bus. If the set came first, the
   register would catch whatever was on the bus before.
   </details>

2. A processor runs at 2 GHz. How long is one clock cycle?
   <details><summary>Answer</summary>

   Half a billionth of a second, 0.5 nanoseconds: two billion cycles fit in
   one second.
   </details>

## 5.3 The stepper and the control section

The clock gives a beat, but work like the addition needs several beats in a
row: the first move, then the second, then the third. Something must count
the beats and say which move comes now. That is the *stepper*.

The stepper is a chain of memory bits, connected so that at each clock
cycle the "on" moves one place further down the chain. It has outputs
numbered 1 to 7: at the first cycle, output 1 is on; at the second, output 2;
and so on. When output 7 comes on, it resets the whole chain, and output 1
comes on again. So the stepper goes round and round, steps 1 to 6, one step
per clock cycle.

Now the control section can be wired. For the addition of lesson 5.1, we
connect:

- step 4 to R1's enable and TMP's set;
- step 5 to R0's enable, the ALU's op wires (000 for ADD) and ACC's set;
- step 6 to ACC's enable and R0's set.

Each connection goes through AND gates with clk e or clk s, so that the
enables come on with clk e and the sets with clk s. And that is all the
control section is: wires from the stepper's steps, through gates, to the
enable and set wires of the parts. When step 4 comes round, R1 goes to TMP;
at step 5, R0 and TMP go through the ALU into ACC; at step 6, ACC goes to R0.

In the picture of the machine, in the labs of lessons 5.1 and 5.4, the
stepper is the row of six boxes in the control section: the current step is
filled in, steps 1 to 3 are marked *fetch*, and 4 to 6 *execute*. Under them, the control section lists which wires are on in
this step. (Our addition used steps 4 to 6; lesson 5.4 explains what steps 1
to 3 are for.)

### Check yourself

1. Why does the stepper reset at step 7 instead of going on for ever?
   <details><summary>Answer</summary>

   So that the same steps come round again, and the next piece of work
   can begin at step 1.
   </details>

2. Wired this way, the control section always adds R1 to R0. What would be
   needed to make it do different things?
   <details><summary>Answer</summary>

   Something that tells it, each time round, *what* to do: which registers,
   which operation. That is the instruction, the subject of the next
   lesson.
   </details>

## 5.4 The first great invention: a program in RAM

A control section wired for one addition can only ever do that addition.
The idea that changes everything, which the book calls the first great
invention, is this: keep the orders in RAM, as bytes, and let the processor
fetch them one by one. Each order is an *instruction*, and the bytes that
encode it are its *instruction code*. A row of instructions in RAM is a
*program*.

Two more registers make this possible. The *instruction register*, IR,
holds the instruction currently being carried out; its bits are wired into
the control section, and they decide which enable and set wires steps 4 to 6
turn on. The *instruction address register*, IAR, holds the address in RAM
of the next instruction.

So now the stepper's six steps have two halves. Steps 1 to 3 *fetch* the next
instruction from RAM, and they are the same for every instruction:

1. Enable IAR, and set MAR: the address of the instruction goes to RAM.
   At the same time, turn bus 1 on, so that the ALU adds 1 to IAR, and set
   ACC: ACC now holds the address of the byte after this one.
2. Enable RAM, and set IR: the instruction goes into the instruction
   register.
3. Enable ACC, and set IAR: IAR now points to the next instruction.

Steps 4 to 6 then *execute* the instruction in IR, whatever it says. When
step 6 is over, the stepper starts again at step 1, which fetches the next
instruction, from the address that IAR now holds.

Here is a two-instruction program to watch the fetch. Step through the
first three steps of each instruction:

```eight run
DATA R1, 5
ADD R1, R1      ; R1 = 5 + 5
```

In step 1, IAR's address goes along the bus into MAR, and the
cell it selects in RAM is outlined; bus 1 is on, and the ALU adds 1. In step
2, that cell's byte goes into IR, and the control section reads it, for
example "IR says: ADD R1,R1". In step 3, ACC goes into IAR. Notice that step 1 uses the
ALU to count, with bus 1: there is no separate counting circuit.

This is how every processor works, from the book's little machine to the
one in your computer: fetch an instruction, carry it out, fetch the next.
The *fetch-execute cycle* repeats, billions of times a second, for as long as
the computer is switched on.

### Check yourself

1. Why is ACC, and not IAR directly, set in step 1?
   <details><summary>Answer</summary>

   In step 1 IAR is the one on the bus. It cannot also take the ALU's
   answer at the same moment, so the answer waits in ACC and is moved into
   IAR in step 3.
   </details>

2. Where does the processor find out what to do in steps 4 to 6?
   <details><summary>Answer</summary>

   In the bits of IR. They are wired into the control section and decide
   which enable and set wires those steps turn on. Module 6 shows the code
   they follow.
   </details>
