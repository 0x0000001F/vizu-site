# Module 8 · Faster processors

The machine of the first seven modules spends six clock cycles on every
instruction, one after another, and for most of those cycles most of its
parts sit idle. Real processors are built on the same ideas, but they have
spent fifty years finding ways to do more in each cycle. This module, which
follows the book *Inside the Machine* by Jon Stokes, explains the most
important of those ways: the pipeline, which overlaps the steps of
successive instructions like an assembly line; superscalar execution, which
runs several instructions side by side; branch prediction, which guesses
where a program is going before it knows; and out-of-order execution, which
lets the processor rearrange the work while keeping the results in order.

The labs of this module use a simpler instruction format than the book's
machine, closer to how Stokes writes them: `r3 = r1 + r2` adds two
registers into a third, `r4 = load r3` reads memory at the address in r3,
and `branch taken` stands for a branch the program takes. The labs step
through clock cycles, and draw the chart Stokes draws: one row per
instruction, one column per cycle, and in each cell the stage the
instruction is in.

## 8.1 Speed: how many instructions per second

What does it mean for one processor to be faster than another? For the
person waiting, it means that a program finishes sooner. A program is a
certain number of instructions, so the time it takes is that number divided
by the rate at which the processor finishes them:

> program time = number of instructions ÷ instructions completed per second

There are two ways to make programs faster, then. One is to need fewer
instructions, which is a matter for compilers and instruction sets. The
other is to complete more instructions per second, and that is what
processor designers work on.

Every instruction goes through the same four phases, which the book's
machine already had in another form:

1. **Fetch** the instruction from memory, at the address in the program
   counter (the book's IAR), and move the program counter on.
2. **Decode** it: work out what it says.
3. **Execute** it: read its registers and let the ALU compute.
4. **Write** the result back into a register.

The simplest design does all four for one instruction, then all four for the
next. Here are four instructions on such a processor, in which each phase
takes one clock cycle:

```pipeline mode=single
r1 = r2 + r3
r4 = r5 + r6
r7 = r8 - r9
r10 = r11 * r12
```

Step through it, a cycle at a time. In each cycle only one of the four stages
at the top is busy, and the other three wait. The four instructions take 16
cycles, and the processor completes one instruction every four cycles: a
*throughput* of 0.25 instructions per cycle.

### Check yourself

1. A processor completes 2 billion instructions per second. How long does a
   program of 10 billion instructions take?
   <details><summary>Answer</summary>

   5 seconds: 10 billion divided by 2 billion per second.
   </details>

2. In the lab, for what fraction of the time is the Execute stage busy?
   <details><summary>Answer</summary>

   One cycle in four, 25%. The other three cycles it waits while the same
   instruction is fetched, decoded and written.
   </details>

## 8.2 The pipeline

Stokes explains the cure with a factory. Suppose a car is built in five
stages, each by its own crew, and each taking an hour. If the factory waits
until one car is completely finished before starting the next, it produces
one car every five hours, and at any moment four of its five crews are
idle. But nothing stops the first crew from starting the next car as soon as
it has passed the first car on to the second crew. Once the line is full,
every crew works all the time, each on a different car, and a finished car
rolls out every hour. Each car still spends five hours on the line; but the
factory completes five times as many cars.

A processor can do exactly the same with the phases of its instructions.
Each phase gets its own part of the hardware, a *pipeline stage*, and in
every clock cycle each stage passes its instruction on to the next stage and
takes a new one from the stage before:

```pipeline
r1 = r2 + r3
r4 = r5 + r6
r7 = r8 - r9
r10 = r11 * r12
```

Step through this one. In the first cycle, only Fetch is busy; in the
second, Fetch and Decode; by the fourth, all four stages are working, each
on a different instruction. From then on, one instruction is written back in
every cycle. The same four instructions now take 7 cycles instead of 16, and
once the pipeline is full the throughput is one instruction per cycle,
four times as much.

Two words are worth keeping apart. The *latency* of an instruction, the
time from its fetch to its write, is still four cycles: pipelining does not
make any single instruction faster. What it improves is the *throughput*,
the number of instructions completed per cycle, and for a long program that
is what counts.

### Deeper pipelines and faster clocks

If four stages give four times the throughput, why not more? Splitting each
stage into two halves gives eight shorter stages, and since each stage now
does less work, the clock can tick twice as fast. Processor makers did
exactly this for years, and pipelines of 10, 14 and even 31 stages appeared,
with clock speeds rising from tens of megahertz to several gigahertz. But a
deep pipeline has costs: every time the flow of instructions is interrupted,
more stages run empty, and refilling them takes longer. The next lessons are
about those interruptions.

### Check yourself

1. How many cycles would 100 instructions take on the four-stage pipeline,
   if nothing ever held it up?
   <details><summary>Answer</summary>

   103: four cycles for the first instruction to come through, then one
   more for each of the other 99.
   </details>

2. Does pipelining make an instruction finish sooner after it is fetched?
   <details><summary>Answer</summary>

   No. Its latency stays four cycles (or gets slightly longer, because of
   the extra registers between stages). Pipelining raises throughput, not
   the speed of a single instruction.
   </details>

## 8.3 Hazards and stalls

A pipeline works at full speed only if every instruction can move on in
every cycle. Situations that prevent this are called *hazards*, and when one
occurs, the instruction affected must wait, leaving an empty slot, a
*bubble*, travelling down the pipeline behind it. Stokes distinguishes three
kinds: *data hazards*, *structural hazards* and *control hazards*. This
lesson is about the first two; branches, the cause of control hazards, get a
lesson of their own.

### Data hazards

Consider two instructions where the second uses the result of the first:

```pipeline forwarding=off
r1 = r2 + r3
r4 = r1 + r5
r6 = r4 & r7
r8 = r9 + r10
```

The second instruction needs r1, but when it reaches the Execute stage, the
first instruction has only just executed: its result will not be written to
r1 until the next cycle. So the second instruction must wait in Decode until
r1 has been written. The third instruction depends on the second in the same
way. Step through it and look for the `·d` cells in the chart: those are
cycles in which an instruction waits in Decode, and the caption says why.
Four instructions now take 9 cycles instead of 7.

Most pipelined processors solve this with a trick called *forwarding*. The
result of the first instruction exists at the output of the ALU at the end
of its Execute cycle, a cycle before it is written to the register. Extra
wires carry it straight from the ALU's output back to its input, so the next
instruction can use it at once:

```pipeline
r1 = r2 + r3
r4 = r1 + r5
r6 = r4 & r7
r8 = r9 + r10
```

With forwarding, the same four instructions flow through in 7 cycles, with
no waiting at all.

Forwarding cannot help everywhere. A load reads memory during its Execute
stage, which takes an extra cycle, so its value is not available one cycle
after it starts executing but two. An instruction right after a load that
uses the loaded value must wait one cycle even with forwarding:

```pipeline
r1 = load r2
r3 = r1 + r4
r5 = r6 + r7
```

Compilers know this, and when they can, they place an unrelated instruction
between a load and the instruction that uses its result, so that the wait
does useful work.

### Structural hazards

A *structural hazard* is a lack of hardware: two instructions need the same
part at the same time. In the load example, the load occupies the Execute
stage for two cycles, and the instruction behind it cannot enter until it is
free. A processor with one multiplier, given two multiplications in a row,
faces the same problem. The cure is more hardware, which the next lesson
looks at.

### Check yourself

1. Why does forwarding remove the wait after an addition but not after a
   load?
   <details><summary>Answer</summary>

   An addition's result exists at the end of its Execute cycle, in time for
   the next instruction's Execute. A load's value arrives from memory only
   a cycle later, so the next instruction is one cycle too early.
   </details>

2. How could a compiler avoid the wait in the load example?
   <details><summary>Answer</summary>

   By moving `r5 = r6 + r7`, which does not use r1, between the load and
   `r3 = r1 + r4`. The pipeline then has something useful to do in the
   cycle it would otherwise lose.
   </details>

## 8.4 Superscalar: more than one at a time

A pipeline can complete at most one instruction per cycle. To do better, a
processor must work on several instructions in each stage at once: fetch
two, decode two, execute two and write two in every cycle. Such a processor
is called *superscalar*; one that handles two instructions per stage is
*two-way* superscalar. It needs more of everything: wider fetch and decode,
and, above all, more than one execution unit.

```pipeline width=2
r1 = r2 + r3
r4 = r1 + r5    ; needs r1
r6 = r7 + r8
r9 = r6 - r10   ; needs r6
r11 = r12 + r13
r14 = r15 | r16
```

Step through it. Instructions enter in pairs, two letters in the same
column. But the second instruction of the first pair needs r1, which the
first is computing in that very cycle; even with forwarding, it cannot
execute beside it. It waits a cycle in Decode, and the next instruction,
which is independent, moves up and executes beside it instead. The same
happens with the second pair. The six instructions take 7 cycles, where a
pipeline one instruction wide would need 9:

```pipeline
r1 = r2 + r3
r4 = r1 + r5    ; needs r1
r6 = r7 + r8
r9 = r6 - r10   ; needs r6
r11 = r12 + r13
r14 = r15 | r16
```

The pattern is general: a superscalar processor can only run instructions
at the same time if they do not depend on each other. How many such
independent instructions a program offers, its *instruction-level
parallelism*, sets the real limit on the speed-up.

Modern processors go much further than two. Their execution units are
specialized: several integer units for arithmetic and logic, units for
floating-point numbers, units for loads and stores, a unit for branches,
and *vector* units that do the same operation on many numbers at once. A
current desktop processor can start six or eight instructions in a single
cycle, if it can find that many independent ones.

### Check yourself

1. Six instructions on a two-way superscalar pipeline, all independent: how
   many cycles?
   <details><summary>Answer</summary>

   6: three pairs, each pair one cycle behind the one before, plus three
   cycles for the last pair to pass through the remaining stages.
   </details>

2. Why would doubling the width from two to four rarely double the speed?
   <details><summary>Answer</summary>

   Most stretches of real programs do not contain four independent
   instructions in a row; dependencies force some of them to wait, and the
   extra units stand idle.
   </details>

## 8.5 Branches and prediction

Every program is full of branches: about one instruction in six or seven
is a jump of some kind, for loops and `if` statements. And a branch is a
problem for a pipeline, because the address of the next instruction is not
known until the branch has executed. Meanwhile the Fetch stage must fetch
*something* every cycle, or the pipeline runs dry. This is the third kind
of hazard, a *control hazard*.

The front end therefore makes a guess, a *prediction*, and goes on fetching
from the guessed address. If the guess was right, nothing is lost. If it was
wrong, everything fetched after the branch belongs to the wrong path and must
be thrown away, and fetching starts again from the right address:

```pipeline
r1 = r1 - r2
branch taken
r3 = r4 + r5
r6 = r3 + r7
```

This front end guesses "not taken", that the program will simply go on with
the next instruction in memory. But the branch is taken. Step to cycle 4,
when the branch executes and its outcome is known: the two instructions
fetched after it are crossed out, thrown away, and the right instruction is
fetched only in cycle 5. Two cycles are lost. With a perfect guess, nothing
would be:

```pipeline predict=perfect
r1 = r1 - r2
branch taken
r3 = r4 + r5
r6 = r3 + r7
```

The deeper the pipeline, the more stages lie between Fetch and the stage
where a branch is decided, and the more work a wrong guess throws away. On a
pipeline of twenty stages, a wrong guess can cost twenty cycles. Good
prediction is therefore vital.

### How processors guess

*Static prediction* uses a fixed rule. The best simple rule is that a branch
pointing backwards is taken, because it is probably the end of a loop that
goes round again, and a branch pointing forwards is not.

*Dynamic prediction* learns from the program as it runs. The processor keeps
a *branch history table*, which records for each recently seen branch
whether it was taken the last few times, and predicts that it will do the
same again. A *branch target buffer* remembers where each taken branch went,
so that the front end knows the target address straight away. Stokes
describes how the Pentium Pro raised the share of correct guesses from
about 75 percent to more than 90; today's predictors, which look at patterns
across many branches, get well over 95 percent right on typical programs.

### Speculative execution

Instructions fetched after a predicted branch are executed before anyone
knows whether they should be. This is called *speculative execution*. Their
results are kept apart and marked as speculative until the branch is
decided; if the guess was right, they become real, and if it was wrong, they
are discarded without a trace in the registers or memory. (Almost without a
trace: in 2018, attacks known as Spectre showed that speculative instructions
can leave footprints in the cache, which a careful attacker can measure.)

### Check yourself

1. Why does a deeper pipeline make wrong guesses more expensive?
   <details><summary>Answer</summary>

   More instructions have entered the pipeline behind the branch by the time
   it is decided, and all of them are thrown away; the pipeline then needs as
   many cycles to fill again.
   </details>

2. A loop runs 100 times. Its branch at the end is taken 99 times and not
   taken once. How often does "backward branches are taken" guess right?
   <details><summary>Answer</summary>

   99 times out of 100: it is wrong only at the very end, when the loop
   finishes.
   </details>

## 8.6 Out of order

The pipelines in the labs are *in order*: an instruction that must wait holds
up every instruction behind it, even ones that have nothing to do with it.
In the load example of lesson 8.3, `r5 = r6 + r7` could have executed during
the cycle the addition spent waiting, but the strict order did not allow it.

Since the mid-1990s, most fast processors execute instructions *out of
order*. Stokes describes the design of the Pentium Pro, whose ideas are
still the basis of today's processors:

- After decoding, instructions enter a waiting area, the *reservation
  station*. An instruction leaves it for an execution unit as soon as its
  input values are ready, whatever its place in the program. So
  independent instructions overtake those that wait.
- Results must still appear in program order, or an interrupt or a wrong
  branch guess would leave the registers in a state no program could
  explain. So every instruction is also entered, in order, in a *reorder
  buffer*. When it finishes executing, it waits there until all the
  instructions before it have finished too, and only then are its results
  *committed* to the real registers.
- Together, these two structures form the *instruction window*: the
  stretch of the program the processor can look across for work to do. The
  Pentium Pro's window held 40 instructions; today's hold several hundred.

### Register renaming

Out-of-order execution meets a new kind of conflict. Suppose an instruction
writes r1, and a later, independent instruction also writes r1 for a
different purpose. The second must not finish first and be overwritten,
nor may it overwrite a value that an earlier instruction still needs to read.
Programs reuse register names all the time simply because there are few of
them. The processor therefore has many more physical registers than the
program can name, and it *renames*: each new value written to r1 gets a
fresh physical register of its own, so that two unrelated uses of r1 no
longer conflict. The program still sees its sixteen registers; underneath
there may be two hundred.

### Translating instructions

x86 instructions are complicated and of uneven length. Since the Pentium
Pro, x86 processors translate each of them, in the decode stage, into one
or more simple, fixed-format internal instructions, often called
*micro-operations*, and it is those that flow through the out-of-order
machinery. Outside, the processor speaks the old instruction set; inside,
it works much like a simple RISC processor (the assembly track compares the
two kinds of instruction set).

### Check yourself

1. Why must results be committed in program order, if instructions execute
   out of order?
   <details><summary>Answer</summary>

   So that at any moment the registers look as if the program had run
   instruction by instruction. If a branch was mispredicted or an interrupt
   arrives, the processor can then throw away everything after a certain
   point and be in a correct state.
   </details>

2. What problem does register renaming solve?
   <details><summary>Answer</summary>

   Conflicts that come only from reusing a register name for unrelated
   values. With a fresh physical register for each new value, such
   instructions become independent and can run in any order.
   </details>

## 8.7 64 bits

The book's machine was an 8-bit machine: its registers, bus and addresses
held 8 bits. The processors of the 1980s were 16- and then 32-bit, and
today's are 64-bit. What does that mean, and why did it matter?

A processor's *width* is the size of its general registers and of the
addresses it uses. It brings two things. The first is *dynamic range*: the
size of the numbers it can handle in one step. A 32-bit register holds
integers up to about 4 billion; a 64-bit one, up to about 18 billion billion.
The second, and the reason the change happened, is *address space*. With
32-bit addresses, a program can name only 4 GB of memory, and by the early
2000s computers had that much RAM and programs wanted more. 64-bit
addresses can name far more memory than will ever be built.

The 64-bit extension of the x86 instruction set, designed by AMD and now
used by every PC, also took the chance to fix an old weakness: x86 had only
eight general registers, and x86-64 has sixteen, each 64 bits wide. More
registers mean fewer values kept in memory, and so fewer loads and stores.
The assembly track's x86-64 module shows these registers, `rax` to `r15`.

There is a cost too. Pointers become 8 bytes instead of 4, so data
structures full of pointers, like the linked lists of the C track, take more
memory, and fewer of them fit in the caches of the next module.

### Check yourself

1. Why could a 32-bit program not use more than 4 GB of memory directly?
   <details><summary>Answer</summary>

   Its addresses have 32 bits, which can name 2³² different bytes, about 4.3
   billion: 4 GB.
   </details>

2. Why does `sizeof(int *)` give 8 in the C track's programs?
   <details><summary>Answer</summary>

   Vizu runs C with the sizes of a 64-bit computer, where an address, and
   so every pointer, takes 64 bits: 8 bytes.
   </details>
