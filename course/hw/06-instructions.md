# Module 6 · Instructions

The processor of module 5 fetches a byte from RAM into its instruction
register, and the bits of that byte decide what steps 4, 5 and 6 do. This
module is about those bytes: what each pattern of bits makes the processor
do, how the control section is wired to do it, and how people write
programs without having to remember the patterns.

There are only nine kinds of instruction in the book's machine. That turns
out to be enough for everything a computer does, because among them are
three ideas the book calls great inventions: keeping the program in RAM,
*jumping* to another place in it, and jumping *only if* something is true.
With those, a machine can repeat, choose and decide, which is all a program
ever needs.

## 6.1 The instruction code

An instruction is one byte, eight bits, and the 256 possible bytes are
divided into nine kinds. The first bit decides the biggest division: if it
is 1, the instruction is an ALU instruction; if it is 0, the next three bits
choose one of the eight other kinds.

| Bits | Language | Meaning |
| --- | --- | --- |
| `1 op ra rb` | `ADD`, `SHR`, `SHL`, `NOT`, `AND`, `OR`, `XOR`, `CMP RA,RB` | the ALU works on RA and RB; the answer goes to RB |
| `0000 ra rb` | `LD RA,RB` | load RB from the RAM address in RA |
| `0001 ra rb` | `ST RA,RB` | store RB at the RAM address in RA |
| `0010 00 rb` + a byte | `DATA RB,n` | put the next byte, n, into RB |
| `0011 00 rb` | `JMPR RB` | jump to the address in RB |
| `0100 0000` + a byte | `JMP addr` | jump to the address in the next byte |
| `0101 caez` + a byte | `JC`, `JA`, `JE`, `JZ`, … `addr` | jump if a chosen flag is on |
| `0110 0000` | `CLF` | clear all the flags |
| `0111 io da rb` | `IN`/`OUT Data`/`Addr,RB` | input or output on the I/O bus |

Where the table says `ra` or `rb`, two bits name one of the four registers:
`00` is R0, `01` R1, `10` R2 and `11` R3. So `1000 1011` is an ALU
instruction (1), with op 000, ADD, and registers 10, R2, and 11, R3: it adds
R2 and R3 and puts the answer in R3.

### A language for people

Nobody wants to write programs as rows of ones and zeros, or to remember
that `1110` begins an XOR. So along with the instruction code, the book
invents a short way of writing each instruction: `ADD R2,R3` instead of
`1000 1011`. A list of instructions written this way is a program in
*assembly language*, and a program called an *assembler* translates it into
the instruction code, one byte for one line. (The assembly track of this
course does the same for real processors.)

Vizu's labs for this machine show both at once. Beside each line of the
program are the address where its bytes are placed in RAM and the bytes
themselves. The first instruction is placed at address 0; each one follows
the one before. A line can begin with a *label*, a name followed by a colon,
such as `loop:`; the label stands for the address of that line, so that
other instructions can refer to it. Everything after a semicolon is a
comment.

### Check yourself

1. What does the byte `1100 0110` tell the processor to do?
   <details><summary>Answer</summary>

   It begins with 1, so it is an ALU instruction. op 100 is AND; ra 01 is
   R1, rb 10 is R2. So: `AND R1,R2`, R2 gets R1 AND R2.
   </details>

2. How many different ALU instructions are there?
   <details><summary>Answer</summary>

   128: eight operations, times four choices of RA, times four of RB.
   </details>

## 6.2 ALU instructions

When IR holds an ALU instruction, the control section's wires for its first
bit, which is 1, turn on the following in steps 4 to 6:

4. Enable RB, set TMP: RB's byte becomes the ALU's input B.
5. Enable RA, so that its byte is the ALU's input A; send the three op
   bits of IR to the ALU; set ACC and set FLAGS. The answer goes into ACC,
   and the four bits the ALU reports go into the flags register.
6. Enable ACC, set RB: the answer goes into RB.

Which registers are RA and RB? The two pairs of bits at the end of IR go to
two small decoders, which turn the control section's general "enable RA"
and "set RB" wires into the wires of one particular register. For the
one-input operations SHR, SHL and NOT, step 4 is not needed, but it does no
harm, so the book leaves the wiring simple. For CMP, step 6 is switched off
by a gate: a comparison keeps no answer, only the flags.

```eight run
DATA R0, 12
DATA R1, 10
ADD R0, R1      ; R1 = 12 + 10
NOT R0, R2      ; R2 = R0 with every bit turned round
SHL R1          ; R1 = R1 shifted left: doubled
XOR R0, R0      ; R0 = 0, whatever it held
CMP R1, R2      ; only the flags change
```

Skip quickly over the two DATA instructions, which the next lesson
explains, and step slowly through the ALU instructions. In `ADD R0,R1`,
step 4 moves R1 (10) into TMP; step 5 sends R0 (12) into the ALU's input A,
the ALU adds, and ACC and the flags are set; step 6 moves 22 into R1. `SHL R1`
is short for `SHL R1,R1`: R1 is both the input and the place for the answer.
`XOR R0,R0` is a common trick: a byte XOR itself is always zero, so it
clears a register in one instruction. And watch the last instruction: CMP
changes the flags, and nothing else.

### Check yourself

1. After the program, what are R1 and R2?
   <details><summary>Answer</summary>

   R1 is 44 (22 doubled) and R2 is 243, which is `1111 0011`, the bits of 12
   (`0000 1100`) turned round.
   </details>

2. Why does an ALU instruction take all three execute steps, while some
   other instructions need fewer?
   <details><summary>Answer</summary>

   The ALU needs two inputs and gives one answer, and each of those needs a
   move along the bus: one into TMP, one into the ALU with the answer into
   ACC, and one out of ACC.
   </details>

## 6.3 Load, store and data

The ALU instructions work on registers. To bring bytes from RAM into a
register, and to put them back, there are two instructions:

- `LD RA,RB` loads RB from RAM: step 4 enables RA and sets MAR, so RA's
  byte becomes the address; step 5 enables RAM and sets RB.
- `ST RA,RB` stores RB in RAM: step 4 is the same; step 5 enables RB and
  sets RAM.

In both, the register RA holds an *address*, and the instruction reaches the
byte at that address. That is the idea of lesson 3.2 at work: RA is used as
a pointer.

To get a number into a register in the first place, there is a third
instruction, `DATA RB,n`. It is two bytes long: the instruction byte, then
the number n itself, placed in RAM right after it. Recall that after step 3,
IAR already points to the byte after the instruction, which here is the
number. So step 4 enables IAR and sets MAR, and turns on bus 1 to compute
IAR + 1 into ACC as well; step 5 enables RAM and sets RB, which loads the
number; and step 6 moves ACC into IAR, so that the next fetch skips over the
number and does not mistake it for an instruction.

```eight run
DATA R0, 0x80   ; an address: row 8, column 0 of RAM
DATA R1, 65
ST R0, R1       ; RAM[0x80] = 65
DATA R2, 0
LD R0, R2       ; R2 = RAM[0x80]
```

Watch three things. In the listing, each DATA takes two bytes, so the
addresses beside the lines go up by two. In the RAM grid, after `ST R0,R1`
the cell at row 8, column 0 holds `41`, which is 65 in hexadecimal: step 5
of the store turns it amber. And in `LD R0,R2`, step 4 sends R0's byte into
MAR, so the outline in the grid jumps to that cell; step 5 brings its 65
back into R2.

### Check yourself

1. Why does `DATA` need step 6, when `LD` does not?
   <details><summary>Answer</summary>

   DATA's number sits where the next instruction would be. Step 6 moves
   IAR past it; without that, the processor would fetch the number and try
   to carry it out as an instruction.
   </details>

2. How could a program add up the five bytes at addresses 100 to 104?
   <details><summary>Answer</summary>

   Keep the address in a register, say R0, starting at 100: load the byte
   with `LD R0,R1`, add it to a running total, add 1 to R0, and repeat five
   times. Repeating is the subject of the next two lessons.
   </details>

## 6.4 Jumps: the second great invention

So far, the processor goes through RAM in order: fetch, execute, fetch the
next. A program of fifty instructions would run once, from top to bottom,
in a few millionths of a second, and then the processor would carry on
fetching whatever bytes come after it.

The second great invention changes that. A *jump* instruction simply puts a
new address into IAR. The next fetch then takes its instruction from the new
address, and the program carries on from there. With a jump, a program can
skip over instructions, or go back and repeat them.

There are two kinds. `JMPR RB` jumps to the address held in a register: step
4 enables RB and sets IAR. `JMP addr` jumps to the address written in the
byte after the instruction: step 4 enables IAR and sets MAR, and step 5
enables RAM and sets IAR. `JMP` is the one to use when the address is known
when the program is written; `JMPR` when the program works the address out
as it runs.

```eight run
        DATA R0, 1
        JMP skip
        DATA R0, 99     ; jumped over: never runs
skip:   DATA R1, add_one
        JMPR R1         ; jump to the address in R1
        DATA R0, 50     ; never runs either
add_one: DATA R2, 1
        CLF
        ADD R2, R0
stop:   JMP stop
```

Follow the highlighted line in the listing. After `JMP skip`, it does not
move to the next line but leaps to `skip:`: step 5 of the jump has put that
label's address into IAR, as you can see in the picture. Then the program
puts the address of `add_one` into R1 as a number, and `JMPR R1` jumps
there.

The last line, `stop: JMP stop`, jumps to itself. A processor never really
stops; this is how a program ends: by going round and round in a loop of one
instruction. Vizu notices a jump to itself and stops the run there.

### Check yourself

1. What is in R0 at the end?
   <details><summary>Answer</summary>

   2. It was set to 1, and both lines that would have changed it were
   jumped over; then `ADD R2,R0` added 1.
   </details>

2. What would `JMP` back to the first line do, at the end instead of `JMP
   stop`?
   <details><summary>Answer</summary>

   Run the whole program again, for ever: a loop that never ends. To stop a
   loop, a program needs a jump that is taken only sometimes, which is the
   next lesson.
   </details>

## 6.5 Flags and Jump If: the third great invention

A jump that is always taken can repeat something for ever. To repeat
something *ten times*, or to do one thing if a number is large and another
if it is small, the program needs a jump that is taken *only if* something
is true. That is the third great invention, and with it the machine can
make decisions.

The decision is based on the four bits the ALU reports. They only exist
while the ALU is working, during step 5 of an ALU instruction, so they are
caught in the *flags register*, set at the same moment as ACC. They stay
there until the next ALU instruction. The four flags are:

- **C**, carry: the last addition or shift had a carry out;
- **A**, A larger: input A was larger than input B;
- **E**, equal: the two inputs were equal;
- **Z**, zero: the answer was all zeros.

The *Jump If* instruction is `0101` followed by four bits, one for each flag,
then a byte with the address to jump to. A 1 in a flag's bit means "jump if
this flag is on"; if several are 1, the jump is taken if any of them is on.
In the language, the instruction is written with the letters of the flags
it tests: `JC`, `JA`, `JE`, `JZ`, or combinations such as `JAE`, jump if A
is larger or equal.

The wiring is neat. Step 4 sends IAR to MAR, ready to read the jump
address, and uses bus 1 to compute the address after it into ACC. Step 5
moves ACC into IAR, so that if nothing else happens the program goes on
after this instruction. Step 6 enables RAM and sets IAR, but the set wire
passes through gates that compare the instruction's four bits with the four
flags: only if a chosen flag is on does IAR take the jump address.

Here is a loop that counts to five:

```eight run
        DATA R0, 0      ; the counter
        DATA R1, 1      ; what to add each time
        DATA R2, 5      ; where to stop
loop:   CLF
        ADD R1, R0      ; R0 = R0 + 1
        CMP R0, R2      ; compare the counter with 5
        JE done         ; equal: finished
        JMP loop        ; not yet: round again
done:   JMP done
```

Step through two rounds of the loop, then drag the slider to the end. Each
`CMP` sets the flags: watch E in the FLAGS box stay off while R0 is below 5.
At each `JE`, step 6 is where the decision happens, and the caption says
whether the jump was taken. Only in the fifth round is E on, and `JE` leaps
to `done`.

### Choosing between two

Decisions also choose between two paths. This program puts the larger of
two numbers into R2:

```eight run
        DATA R0, 23
        DATA R1, 42
        XOR R2, R2      ; R2 = 0
        CMP R0, R1      ; is R0 larger than R1?
        JA first
        OR R1, R2       ; no: copy R1 into R2
        JMP stop
first:  OR R0, R2       ; yes: copy R0 into R2
stop:   JMP stop
```

The machine has no instruction to copy one register into another, so the
program makes one from two it does have: clear R2 with XOR, then `OR R1,R2`,
which gives R1's bits OR nothing, that is, R1 itself. This kind of
cleverness, building what you need from what the machine offers, is a large
part of programming in assembly language.

One detail of the book's machine is worth knowing: CMP selects no device in
the ALU, so its output is all zeros, and after a CMP the zero flag is always
on. After a comparison, test A and E, not Z.

### Check yourself

1. How many times is the `ADD` in the counting loop carried out?
   <details><summary>Answer</summary>

   Five: once for each value from 1 to 5. After the fifth, the comparison
   finds R0 equal to 5 and the loop ends.
   </details>

2. Swap the numbers in the second program, 42 into R0 and 23 into R1. Which
   path is taken now?
   <details><summary>Answer</summary>

   R0 is now the larger, so A is on, `JA` jumps to `first`, and R2 gets 42
   from R0. Either way R2 ends with the larger number.
   </details>

## 6.6 Arithmetic with the machine

One instruction is still missing from our list: `CLF`, clear flags. It is
needed because of the carry. The ALU's carry in is connected to the carry
flag, so that a carry left over from one addition is added into the next,
which is how the machine adds numbers longer than one byte. But when you
just want to add two single bytes, an old carry left in the flag would be
added in too, and 2 + 2 could come out as 5. So, before an addition or a
shift where an old carry must not count, a program clears the flags.

### Subtraction

The machine has no subtraction instruction, and it does not need one.
To subtract 21 from 37, turn every bit of 21 round with NOT, add 1, and add
the result to 37:

```eight run
DATA R0, 37
DATA R1, 21
DATA R2, 1
NOT R1, R1      ; R1 = 21 with its bits turned round: 234
CLF
ADD R2, R1      ; R1 = 234 + 1 = 235
CLF
ADD R0, R1      ; R1 = 37 + 235, which leaves 16 in the byte
```

37 + 235 is 272, which does not fit in a byte: the byte keeps 16, and the
carry comes on. 16 is exactly 37 − 21. This works because "turn the bits
round and add 1" is how negative numbers are stored, the two's complement
of lesson 12.1 in the C track: 235 is the byte that stands for −21, and
adding −21 is subtracting 21. The carry that comes out is simply ignored.

### Multiplication

Multiplication is built from shifts and additions, the way you multiply on
paper, only in binary, where it is easier: each digit of the multiplier is
either 0 or 1, so for each digit you either add the top number or you do not.
Going through the multiplier from the right, for each 1 add the top number
to the answer, and before each next digit, shift the top number one place
left. Here is the book's program for it, multiplying 5 by 5:

```eight run
        DATA R0, 5      ; the bottom number
        DATA R1, 5      ; the top number
        DATA R3, 1      ; a single 1 that counts the eight rounds
        XOR R2, R2      ; the answer starts at 0
loop:   CLF
        SHR R0          ; the lowest bit of R0 goes into carry
        JC add
        JMP next
add:    CLF
        ADD R1, R2      ; add the top number to the answer
next:   CLF
        SHL R1          ; double the top number
        SHL R3          ; move the counter's 1 to the left
        JC done         ; after eight shifts the 1 falls out
        JMP loop
done:   JMP done
```

`SHR R0` moves the lowest bit of the bottom number into the carry flag,
where `JC` can test it. R3 counts the rounds in a clever way: it starts as
`0000 0001`, and each round shifts its single 1 one place left; after eight
rounds the 1 falls off the end into the carry, and `JC done` ends the loop.

This is the longest run in the track, almost 500 clock steps, so use the
slider. Watch R2 grow: after the first round it is 5, after the third it is
25, and it stays 25 while the remaining rounds find only zeros in R0.

### Check yourself

1. What does the subtraction program leave in R1 if R1 starts at 37 and R0
   at 21, the other way round?
   <details><summary>Answer</summary>

   `21 − 37` is −16, so the byte holds 240, which is −16 in two's
   complement, and there is no carry. The machine does not know whether you
   meant 240 or −16; that is up to the program.
   </details>

2. Why does the multiplication program need `CLF` before `SHR R0`?
   <details><summary>Answer</summary>

   A shift takes the carry flag in at its other end. A carry left over
   from an earlier instruction would be shifted into R0 and spoil the
   number.
   </details>
