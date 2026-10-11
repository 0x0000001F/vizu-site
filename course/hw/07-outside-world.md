# Module 7 · The outside world, and software

The computer we have built is complete: RAM and a processor, running a
program of instructions. But if all it could do was rearrange bytes in its
own RAM, nobody would ever know what it was doing. It needs a way to talk to
the world outside it: to take in what a person types, to show results on a
screen, to keep files on a disk that survive when the power goes off.

This module adds that last piece, which turns out to be surprisingly small,
and then steps back to look at the whole: what software is, what an
operating system does, and how a program written in a language like C
becomes the instruction bytes this machine runs. It ends with what our
simple machine leaves out, which is where the last two modules of the track
begin.

## 7.1 The I/O bus

Everything outside the processor and RAM, the keyboard, the screen, the
disk, the network, is called a *peripheral*, and talking to peripherals is
called *input and output*, or I/O. The book's machine does all its I/O
through one extension of its bus.

The *I/O bus* is the processor's ordinary eight-wire bus, carried outside,
plus four more wires: one that says whether the byte moves in or out, one
that says whether it is data or an address, and an enable and a set wire
for the moment of the transfer. Every peripheral is connected to the I/O
bus through a *device adapter*, a small circuit of its own. Each adapter has
its own *I/O address*, a number that has nothing to do with RAM addresses.

Talking to a device takes two steps. First the processor puts the device's
I/O address on the I/O bus, marked as an address. Every adapter compares it
with its own address, and the one that recognizes it switches itself on,
remembering in a memory bit that it has been selected; all the others stay
quiet. Then the processor sends data to it, or takes data from it.

One instruction does all of this, `IN` or `OUT`, in four variations:

| Language | Meaning |
| --- | --- |
| `OUT Addr,RB` | select the device whose I/O address is in RB |
| `OUT Data,RB` | send RB's byte to the selected device |
| `IN Data,RB` | take a byte from the selected device into RB |
| `IN Addr,RB` | take the selected address back into RB |

An output needs only step 4: enable RB, and turn on the I/O bus's set wire.
An input needs only step 5: turn on the I/O bus's enable wire, and set RB.

Vizu's version of the machine has three devices on its I/O bus. The
keyboard is at I/O address 15, as in the book. The other two are simpler
than the book's display adapter, so that a program can show its results
easily: eight *lights* at address 1, which show the bits of the last byte
sent to them, and a text *screen* at address 2, which prints each byte sent to
it as an ASCII character.

```eight run
        DATA R0, 2      ; the screen's I/O address
        OUT Addr, R0    ; select the screen
        DATA R1, 'H'
        OUT Data, R1    ; print H
        DATA R1, 'i'
        OUT Data, R1    ; print i
        DATA R0, 1      ; the lights' I/O address
        OUT Addr, R0    ; select the lights
        DATA R1, 1010 0101
        OUT Data, R1    ; show these bits
stop:   JMP stop
```

`DATA R1, 'H'` puts the ASCII code of the letter H, 72, into R1. Watch the
I/O bus box on the right of the picture. After each `OUT Addr`, it says
which device is selected. After each `OUT Data`, the byte arrives: a letter
appears on the screen, or the lights show the pattern.

### Check yourself

1. Why does the processor send an address before the data?
   <details><summary>Answer</summary>

   Many devices share the I/O bus. The address wakes up exactly one of
   them, so that only that one takes the data that follows.
   </details>

2. What would `OUT Data,R1` do if no device had been selected?
   <details><summary>Answer</summary>

   Nothing. The byte goes out on the I/O bus, but no adapter is switched on
   to take it.
   </details>

## 7.2 The keyboard

A keyboard is the simplest input device. Pressing a key connects the wires
that make the ASCII code of that key, and the keyboard adapter keeps the
code in a register until the processor asks for it. Asking is an `IN Data`
from the keyboard's address. If no key has been pressed since the last time
it asked, the adapter answers 0, and a program knows there is nothing new.

So a program that wants to read what is typed must keep asking. This
program reads keys and prints each on the screen, until no key is waiting:

```eight run input="hello"
        DATA R2, 15     ; the keyboard's I/O address
        DATA R3, 2      ; the screen's I/O address
        XOR R1, R1      ; R1 = 0, to compare with
loop:   OUT Addr, R2    ; select the keyboard
        IN Data, R0     ; the next key, or 0 if there is none
        CMP R0, R1      ; was it 0?
        JE stop         ; yes: nothing more was typed
        OUT Addr, R3    ; select the screen
        OUT Data, R0    ; print the key
        JMP loop
stop:   JMP stop
```

The keys typed for this lab are shown under the program. Step to the first
`IN Data`: in step 5 the keyboard puts the code of `h`, 104, on the bus, and
R0 takes it. The next lines select the screen and print it, and the loop goes
round for the next key. In the I/O bus box, "waiting" shows the keys not yet
read.

This way of reading, asking the device again and again whether it has
something, is called *polling*. It works, but it keeps the processor busy
asking, even when nothing happens for minutes on end. The next lesson
describes the better way that most computers use.

### Check yourself

1. Why does the program compare the key with 0?
   <details><summary>Answer</summary>

   0 is the keyboard adapter's way of saying "no key". A real key always
   has a code above 0.
   </details>

2. Why must the program select the keyboard again at the top of every round?
   <details><summary>Answer</summary>

   Because in the middle of the round it selects the screen to print.
   Only one device is selected at a time.
   </details>

## 7.3 Screens, disks and interrupts

### The screen

A real display adapter is more interesting than our text screen. The
picture on a screen is made of tiny dots, *pixels*, arranged in rows, and
the adapter has its own RAM, the *display RAM*, with a few bits for every
pixel saying how bright or what colour it is. Many times a second, the
adapter runs through its display RAM from the first byte to the last, and
lights the pixels of the screen one after another to match. The processor
never draws anything itself: it only writes bytes into the display RAM,
through the I/O bus, and the adapter makes them visible. To show a letter, a
program writes the bit pattern of that letter's shape, taken from a table
of shapes called a *font*, into the right part of the display RAM.

### The disk

RAM forgets everything when the power goes off, and there is never enough
of it. A *disk* is a peripheral that remembers bytes without power, and
holds far more of them. To the processor it is just another device on the
I/O bus: a program sends it an address that names a block of bytes on the
disk, and then either sends it bytes to store there or takes stored bytes
back. The processor can only run programs and work on data that are in RAM,
so everything the computer does starts with copying bytes from the disk into
RAM. A *file* is a collection of such blocks that belong together, given a
name.

### Interrupts

Polling wastes time. Most computers therefore have one more wire on the I/O
bus, the *interrupt* wire, which a device adapter turns on when it has
something for the processor, such as a key that was just pressed. When the
interrupt wire is on and the stepper comes back to step 1, the processor
does not fetch the next instruction as usual. Instead, its control section
runs a special sequence of steps that saves IAR and the flags in two fixed
bytes of RAM, and loads IAR and the flags from two other fixed bytes. The
next fetch therefore comes from a different place: a short program, the
*interrupt handler*, whose address was stored there in advance. The
handler deals with the device, reading the key, for instance. At its end, it
puts the saved IAR and flags back, and the interrupted program carries on as
if nothing had happened. It is like being called away in the middle of a
task, dealing with the request, and returning to exactly where you left
off.

### Check yourself

1. Does the processor draw letters on the screen?
   <details><summary>Answer</summary>

   No. It writes bytes into the display adapter's RAM. The adapter turns
   those bytes into lit pixels, over and over, without the processor's
   help.
   </details>

2. Why must an interrupt save the flags, not only IAR?
   <details><summary>Answer</summary>

   The interrupted program may have been between an ALU instruction and the
   Jump If that tests its flags. The handler's own ALU instructions would
   change the flags, and the program would decide wrongly after it resumed.
   </details>

## 7.4 Hardware and software

Everything so far has been *hardware*: gates, wires, registers, things you
could in principle hold and look at. A program is different. It is a
pattern of bytes in RAM, and the same machine runs a word processor, a game
or a spreadsheet depending only on which bytes are there. That changeable
pattern is called *software*.

### The operating system

When a computer is switched on, its RAM is empty. A small program kept in a
special memory that does not forget, the *firmware* or ROM, runs first: it
tests the hardware and loads a bigger program from the disk into RAM. This
is called *booting*, from the old phrase about pulling yourself up by your
own bootstraps. The program it loads is the *operating system*, the program
that runs the computer: it reads the disk's files, starts other programs by
copying them from the disk into RAM and jumping to their first instruction,
handles the interrupts from the devices, and gives programs a tidy way to
use the devices without knowing how each adapter works. When a C program
calls `printf`, the library ends up asking the operating system to show the
text, through a *system call*; that is the `write` you saw in the strip of
every C lab.

### Languages

Writing every program in the instruction code, or even in assembly
language, would be slow and tiring. So programs are written in languages
like C, closer to how people think: `total = total + price;` instead of
loads, adds and stores. A *compiler* translates such a program into the
instruction code of a particular processor. Each line of C becomes a few
instructions, such as load the two values into registers, add them, store
the result. The C track's programs became instruction bytes in exactly this
way; the assembly track shows the instructions that real compilers produce
for real processors.

A compiler is itself just a program, and so is the operating system, and so
is the assembler. All of them, in the end, are bytes in RAM fetched and
executed by a processor doing nothing but its six steps, over and over.

### Check yourself

1. What is the first program a computer runs after it is switched on, and
   where does it come from?
   <details><summary>Answer</summary>

   The firmware, kept in a memory that does not forget when the power is
   off. It loads the operating system from the disk.
   </details>

2. A C program contains `x = y + 5;`. Roughly which of the book's
   instructions could it become?
   <details><summary>Answer</summary>

   Something like: `DATA` the address of y into a register and `LD` y;
   `DATA` 5 into another register; `CLF` and `ADD`; `DATA` the address of x
   and `ST` the sum there.
   </details>

## 7.5 What the simple machine leaves out

The book's machine is a real computer: it can run any program, given
enough RAM and time. But a processor in a phone or a laptop differs from it
in many ways, and it is worth knowing which, before the last two modules
look inside a modern processor.

- **Bigger numbers.** Its registers, bus and addresses have 8 bits. Today's
  processors have 64, which is what "64-bit" means: numbers up to about 18
  billion billion, and RAM addresses for more memory than will ever be
  built.
- **More registers, more instructions.** It has 4 registers and 9 kinds of
  instruction. A modern processor has 16 or 32 general registers and
  hundreds of instructions, including ones that multiply, divide, work with
  numbers with fractions, and handle many numbers at once.
- **Fewer steps per instruction.** It spends six clock steps on every
  instruction, even when some of them do nothing. Real processors overlap
  the steps of several instructions, like an assembly line, and finish
  several instructions in every clock cycle. Module 8 shows how.
- **Memory that is faster than RAM.** It reads RAM in a single step. Real
  RAM is much slower than a processor, hundreds of clock cycles away, and
  processors keep copies of recently used bytes in small, fast memories
  called *caches*. Module 9 explains them.

None of this changes the ideas. A modern processor still fetches
instructions from memory, decodes them, moves bytes between registers and
through an ALU, keeps flags, and jumps. It only does all of that much
faster, and many times at once.

### Check yourself

1. Why does a 64-bit processor need 64-bit addresses?
   <details><summary>Answer</summary>

   With 8-bit addresses, a processor can name only 256 bytes. Every extra
   bit doubles the number of addresses, and modern programs need billions
   of bytes.
   </details>

2. Is the book's machine able to run any program a modern computer can?
   <details><summary>Answer</summary>

   In principle yes, if it had enough memory and time: it can load, store,
   compute, and jump on conditions, which is all any program needs. It
   would just be enormously slower.
   </details>
