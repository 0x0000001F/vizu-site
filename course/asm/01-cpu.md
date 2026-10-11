# Module 1 · How a CPU runs a program

Read alongside: Bryant and O'Hallaron, *Computer Systems: A Programmer's
Perspective* (CS:APP), chapter 4 · Patterson and Hennessy, *Computer
Organization and Design, RISC-V Edition* (P&H), chapter 4

The C track drew memory: boxes, addresses, arrows. This track goes inside
the CPU that reads and writes those boxes. A CPU knows nothing about C: it
reads instructions, numbers in memory, one after another, and does what each
says. Three instruction sets are taught here, on the machines people use:
**x86-64** (PCs and most servers), **ARM64** (phones, Apple's M chips, more
and more servers) and **RISC-V** (an open standard, used in teaching and in
new chips). Every program in this track is assembled into real machine code
and runs one instruction at a time, and each instruction one stage at a
time: → goes through the stages, then to the next instruction.

## 1.1 The machine: CPU, memory and the bus

**Read:** CS:APP 1.4 · P&H 1.3–1.4
**You will:** see a program as bytes in memory, and the CPU's few registers.

Memory is a long row of bytes, each with an address. A program is bytes too:
here its instructions are at `0x1000` and up (the *text*), its data at
`0x2000`, and the stack grows down from `0x8000`, as in the C pictures.

The CPU is small next to it. It has a few **registers** (on RISC-V, 32 of
them, 8 bytes each), a **program counter** (PC, the address of the next
instruction), an **ALU** that adds, compares and shifts, and a **decoder**
that works out what an instruction means. A **bus** joins the CPU to memory:
the CPU puts an address on it, and bytes come back (or go out).

```asm run arch="riscv"
    .text
    .global _start
_start:
    li   a0, 2          # a0 = 2
    li   a1, 3          # a1 = 3
    add  a2, a0, a1     # a2 = a0 + a1
    li   a0, 0          # the exit status
    li   a7, 93         # 93: the kernel's exit
    ecall               # ask the kernel to end the program
```

**Watch:**

- In the memory panel, `.text` shows each instruction as the 4 bytes it is.
  The CPU reads them from there; the program is just data it obeys.
- The PC goes `0x1000`, `0x1004`, `0x1008` …: 4 bytes on each time.
- A register changes in the **Write back** stage, lit amber.
- `li` is not a real instruction: the assembler turns it into `addi`, and
  the decoder only ever sees `addi`.

### Check yourself

1. How many bytes does this program take in memory?
   <details><summary>Answer</summary>

   Six instructions of 4 bytes: 24 bytes, from `0x1000` to `0x1017`.
   </details>

2. Where is `a2` stored?
   <details><summary>Answer</summary>

   Inside the CPU, in the register file. It has no address in memory; to
   keep the value in memory, a store instruction would have to write it.
   </details>

### Common mistake

Thinking of registers as variables. There are only a few; a program keeps
its data in memory and loads it into registers to work on it.

## 1.2 Fetch, decode, execute: six stages

**Read:** CS:APP 4.3 · P&H 4.1–4.4
**You will:** follow one instruction through the CPU, stage by stage.

Every instruction goes through the same stages (the sequential processor of
CS:APP, with the same names in P&H):

1. **Fetch**: the PC goes out on the address bus; the instruction's bytes
   come back into the *instruction register*.
2. **Decode**: the decoder splits the bits into fields (which operation,
   which registers) and reads those registers.
3. **Execute**: the ALU computes: a sum, a comparison, or an address.
4. **Memory**: a load reads memory, a store writes it. Others skip this.
5. **Write back**: the result goes into a register.
6. **PC update**: the PC moves to the next instruction, or to a jump target.

An instruction uses only the stages it needs; the strip shows the others
dashed.

```asm run arch="riscv"
    .data
n:  .dword 40
    .text
    .global _start
_start:
    la   t0, n          # t0 = the address of n
    ld   t1, 0(t0)      # load n from memory
    addi t1, t1, 2      # add 2
    sd   t1, 0(t0)      # store it back
    li   a0, 0
    li   a7, 93
    ecall
```

**Watch:**

- `ld` uses all six stages: Execute computes the address (`t0 + 0`), Memory
  reads 8 bytes over the data bus.
- `addi` skips Memory; `sd` skips Write back but its Memory stage turns the
  bytes of `n` amber in `.data`.
- `la` became two instructions, `auipc` and `addi`: the address of `n` is
  built from the PC.

### Check yourself

1. Which stages does `addi` go through?
   <details><summary>Answer</summary>

   Fetch, Decode, Execute, Write back and PC update; no Memory.
   </details>

2. What does the Execute stage of `ld t1, 0(t0)` compute?
   <details><summary>Answer</summary>

   The address: `t0 + 0` (`0x2000`). The ALU works on addresses as on any
   number.
   </details>

### Common mistake

Expecting an instruction to read memory by itself. On RISC-V (and ARM64)
only loads and stores touch memory; `add` works on registers only.

## 1.3 Instructions are bits

**Read:** CS:APP 3.2.2 · P&H 2.5
**You will:** read the fields of an instruction, on three machines.

An instruction is a number, and its bits are cut into **fields**. A RISC-V
`addi` is 32 bits: 12 for the number added, 5 for the source register, 3
that say "add", 5 for the destination, and 7 for the *opcode*, the kind of
instruction. ARM64 also uses 32 bits with fields of its own. x86-64 is
different: an instruction is 1 to 15 bytes, a prefix, an opcode, a *ModRM*
byte naming registers, then numbers. Here is the same 3 + 4 on all three.

```asm run arch="riscv"
    .text
    .global _start
_start:
    li   a0, 3
    addi a0, a0, 4      # a0 = 7
    li   a7, 93         # exit(a0): the exit status is 7
    ecall
```

```asm run arch="arm"
    .text
    .global _start
_start:
    mov  x0, #3
    add  x0, x0, #4     // x0 = 7
    mov  x8, #93        // exit
    svc  #0
```

```asm run arch="x86"
section .text
global _start
_start:
    mov  rdi, 3
    add  rdi, 4         ; rdi = 7
    mov  rax, 60        ; exit
    syscall
```

**Watch:**

- In **Decode**, the fields: RISC-V's `opcode`, `rd`, `funct3`, `rs1`,
  `imm`; ARM64's `sf`, `op`, `imm12`, `Rn`, `Rd`; x86-64's `REX`, `opcode`,
  `ModRM` and `imm`.
- The sizes in `.text`: always 4 bytes on RISC-V and ARM64; 7, 4, 7 and 2
  on x86-64.
- Each machine asks the kernel differently: `ecall`, `svc #0`, `syscall`,
  with the request's number in `a7`, `x8` or `rax` (and it is 93 on two of
  them, 60 on x86-64).

### Check yourself

1. Why must an x86-64 CPU decode the first bytes before it knows where the
   next instruction starts?
   <details><summary>Answer</summary>

   Instructions have different lengths; the prefix, opcode and ModRM say how
   many bytes follow. With a fixed 4 bytes, ARM64 and RISC-V know at once.
   </details>

2. What is in `rd` of `addi a0, a0, 4`?
   <details><summary>Answer</summary>

   `01010`: register 10, which is `a0`.
   </details>

### Common mistake

Talking of "assembly" as one language. Each CPU family has its own
instructions and encodings; a program written for one does not run on
another.

## 1.4 Deciding: flags and branches

**Read:** CS:APP 3.6 · P&H 2.7
**You will:** see how a CPU decides, with flags or by comparing.

An `if` or a loop becomes a **conditional jump**. On x86-64 and ARM64 a
compare instruction (`cmp`) subtracts and keeps only the **flags**, single
bits about the result: zero? negative? a carry? an overflow? The jump that
follows reads them. RISC-V has no flags: its branch compares two registers
itself (module 4).

```asm run arch="x86"
section .text
global _start
_start:
    xor  ebx, ebx       ; sum = 0
    mov  rcx, 1         ; i = 1
again:
    add  rbx, rcx       ; sum += i
    inc  rcx            ; i++
    cmp  rcx, 5
    jle  again          ; jump back while i <= 5
    mov  rdi, rbx       ; exit status: the sum
    mov  rax, 60
    syscall
```

**Watch:**

- `cmp rcx, 5` sets ZF, SF, CF and OF in **Execute**; `jle` reads them in
  its own Execute, and **PC update** shows where it goes.
- The last `cmp` compares 6 with 5: 6 − 5 = 1, so ZF = 0 and SF = OF, and
  `jle` falls through.
- The exit status is 15.

### Check yourself

1. Which flags does `cmp rcx, 5` set when `rcx` is 5?
   <details><summary>Answer</summary>

   5 − 5 = 0: ZF = 1 (zero), SF = 0, CF = 0, OF = 0. `jle` jumps (ZF = 1).
   </details>

2. Why `xor ebx, ebx` rather than `mov rbx, 0`?
   <details><summary>Answer</summary>

   Both give 0. `xor` with itself is the usual idiom: it is shorter (2 bytes
   here), and writing `ebx` clears all of `rbx`.
   </details>

### Common mistake

Putting an instruction that changes the flags between `cmp` and the jump.
`add` or `inc` would replace the flags, and the jump would decide on the
wrong ones.

## 1.5 Memory, the stack and calls

**Read:** CS:APP 3.7 · P&H 2.8
**You will:** watch a call go and come back, and a frame on the stack.

A function call must remember where to come back. x86-64's `call` pushes
the return address on the stack; ARM64's `bl` and RISC-V's `jal` (written
`call`) put it in a register instead: `x30` or `ra`. A function that calls
another must save that register first, on the **stack**: memory below
`0x8000` that the stack pointer (`sp`) walks down into.

```asm run arch="riscv"
    .text
    .global _start
_start:
    li   a0, 5
    call square         # ra = the next address; jump to square
    li   a7, 93         # exit(a0): 25
    ecall

square:
    addi sp, sp, -16    # make room on the stack
    sd   ra, 8(sp)      # save the return address
    mul  a0, a0, a0
    ld   ra, 8(sp)      # and take it back
    addi sp, sp, 16
    ret                 # jump to ra
```

**Watch:**

- `call square` is `jal ra, …`: Write back puts `0x1008` in `ra`.
- `sd ra, 8(sp)` writes 8 bytes on the stack (amber), and the `◀ sp`
  marker shows where the stack ends.
- `ret` is `jalr zero, 0(ra)`: PC update jumps back to `0x1008`.
- `square` calls nothing, so saving `ra` is not needed here; a function
  that calls another must do it, or its own `ret` goes to the wrong place.

### Check yourself

1. Why does the stack grow downwards?
   <details><summary>Answer</summary>

   It is a convention: the stack starts at the top of its region and each
   frame takes the addresses below. `addi sp, sp, -16` takes 16 bytes.
   </details>

2. What would happen if `square` forgot `addi sp, sp, 16`?
   <details><summary>Answer</summary>

   It would return with `sp` 16 bytes too low: the caller's frame would be
   off, and its own saved values read from the wrong place.
   </details>

### Common mistake

Overwriting the return register before saving it: a second `call` replaces
`ra` (or `x30`), and the first function can no longer return.

## 1.6 Asking the kernel: system calls

**Read:** CS:APP 8.1 · the Linux manual page syscall(2)
**You will:** print a line with no C library at all.

A program cannot reach the screen by itself: only the kernel can. It asks
with a **system call**: the request's number and its arguments in agreed
registers, then one instruction that switches the CPU from *user mode* into
*kernel mode* and jumps into the kernel. Linux's numbers:

| | instruction | number in | write | exit | arguments in |
| --- | --- | --- | --- | --- | --- |
| x86-64 | `syscall` | `rax` | 1 | 60 | `rdi`, `rsi`, `rdx` |
| ARM64 | `svc #0` | `x8` | 64 | 93 | `x0`, `x1`, `x2` |
| RISC-V | `ecall` | `a7` | 64 | 93 | `a0`, `a1`, `a2` |

```asm run arch="riscv"
    .data
msg:    .ascii "Hello, world\n"
    .equ len, . - msg
    .text
    .global _start
_start:
    li   a0, 1          # fd 1: stdout
    la   a1, msg        # where the bytes are
    li   a2, len        # how many: 13
    li   a7, 64         # 64: write
    ecall
    li   a0, 0
    li   a7, 93         # 93: exit
    ecall
```

**Watch:**

- `ecall` adds a **Kernel** stage: the CPU says *kernel mode*, the kernel's
  `write` copies the 13 bytes to the screen, and `a0` gets 13 back.
- There is no stdout buffer here (the C track's printf had one): `write` is
  the system call itself, and the text goes out at once.
- `.equ len, . - msg`: `.` is the current address, so `len` is the string's
  length, worked out by the assembler.

### Check yourself

1. What does `write` return in `a0`?
   <details><summary>Answer</summary>

   The number of bytes written, 13; a negative number on an error (−9 for a
   bad file descriptor).
   </details>

2. Why can the program not just write into the screen's memory?
   <details><summary>Answer</summary>

   It runs in user mode and owns only its own memory. Devices belong to the
   kernel; the system call is the one door.
   </details>

### Common mistake

Forgetting the `exit` call. The CPU then fetches whatever bytes follow the
last instruction; here that stops the program with an error:

```asm run arch="riscv" expect="error"
    .text
    .global _start
_start:
    li   a0, 1
    li   a1, 2
    add  a0, a0, a1
```

## 1.7 Faster: pipelines, prediction and caches

**Read:** CS:APP 4.4–4.5, 6.2–6.4 · P&H chapters 4 (pipelining) and 5 (caches)
**You will:** know what a real CPU does beyond the six stages.

The CPU in these labs does one stage at a time, one instruction at a time.
Real CPUs overlap them:

- **Pipelining.** While one instruction is in Execute, the next is in
  Decode and the one after in Fetch, like a laundry with a washer and a
  dryer. With five stages, up to five instructions are in flight, and one
  can finish every clock cycle.
- **Hazards.** An instruction that needs the result of the one just before
  must wait for it, or get it *forwarded* from the ALU early. A branch is
  worse: until it is decided, the CPU does not know what to fetch next.
- **Branch prediction.** So the CPU guesses, from what the branch did
  before, and keeps going. A wrong guess throws the started work away and
  costs many cycles; a loop branch that is taken a thousand times and then
  falls through is guessed right almost every time.
- **Out of order, several at once.** Modern cores fetch several
  instructions per cycle, run them when their inputs are ready rather than
  in order, and put the results back in order.
- **Caches.** Main memory takes on the order of a hundred nanoseconds; the
  CPU keeps recent bytes in small fast caches (a level-1 cache answers in a
  few cycles). Memory arrives in *cache lines* of 64 bytes, so walking an
  array in order is much faster than jumping around.

None of this changes what a program means: the instructions still behave as
if they ran one after another, as they do in these labs.

### Check yourself

1. Why does a sorted array make a loop with an `if` on its values faster?
   <details><summary>Answer</summary>

   With sorted values the `if` goes the same way for long runs, so the
   branch predictor guesses right; with random values it guesses wrong
   about half the time, and each wrong guess costs many cycles.
   </details>

2. Two nested loops visit a 1000 × 1000 array, row by row or column by
   column. Which is faster, and why?
   <details><summary>Answer</summary>

   Row by row: C stores rows one after another, so each 64-byte cache line
   brought in is used fully. Column by column touches a new line on almost
   every access.
   </details>

### Common mistake

Counting instructions to guess speed. On a modern CPU, cache misses and
wrong branch guesses often matter more than how many instructions run.
