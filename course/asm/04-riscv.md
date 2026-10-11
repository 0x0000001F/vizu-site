# Module 4 · RISC-V

Read alongside: P&H (RISC-V Edition) chapter 2 · *The RISC-V Instruction Set
Manual, Volume I* (unprivileged ISA)

RISC-V is an open instruction set: anyone may build a chip for it without a
licence. It was designed at Berkeley for teaching and research, and is now
in microcontrollers, storage chips and new processors. It is the cleanest of
the three: a small base (RV64I here, with M for multiply and divide),
instructions of 4 bytes in six formats, no flags, and a register that is
always zero. Programs use GNU as syntax, with `#` for comments.

## 4.1 Registers, x0 and the ABI names

**Read:** P&H 2.3 · ISA manual, "Programmers' Model"
**You will:** use the 32 registers by their ABI names and meet x0.

There are 32 registers of 64 bits, `x0` to `x31`. **`x0` is always 0**:
reading it gives 0, writing it does nothing. People write the other names,
from the calling convention (the ABI): `ra` (return address), `sp` (stack
pointer), `a0`–`a7` (arguments and results), `t0`–`t6` (temporaries a
function may change), `s0`–`s11` (saved: a function must give them back).

Assemblers add **pseudo-instructions** that are really others: `mv t1, t0`
is `addi t1, t0, 0`; `li` loads a number, as `addi` from `x0` when it fits
in 12 bits, as `lui` + `addiw` when it does not.

```asm run arch="riscv"
    .text
    .global _start
_start:
    li   t0, 5          # addi t0, zero, 5
    add  zero, t0, t0   # writes x0: thrown away
    mv   t1, t0         # addi t1, t0, 0
    li   t2, 100000     # too big for 12 bits: lui + addiw
    li   a0, 0
    li   a7, 93
    ecall
```

**Watch:**

- `add zero, t0, t0`: the ALU computes 10, and Write back says it is thrown
  away.
- `li t2, 100000` is two instructions: `lui t2, 0x18` (0x18 × 4096 =
  98304), then `addiw t2, t2, 1696`.
- The caption says what each pseudo-instruction became.

### Check yourself

1. Why is a zero register useful?
   <details><summary>Answer</summary>

   Many instructions get a second meaning for free: `addi rd, zero, n` loads
   a number, `sub rd, zero, rs` negates, `beq rs, zero, L` tests for zero,
   and writing `zero` discards a result.
   </details>

2. Which registers may a function change without saving them?
   <details><summary>Answer</summary>

   `t0`–`t6` and `a0`–`a7` (and `ra`, if it saves it for its own return).
   `s0`–`s11` and `sp` must come back as they were.
   </details>

### Common mistake

Expecting `li` to be one instruction always. With a big number it is two,
which matters when counting bytes or jumping over it.

## 4.2 Arithmetic without flags

**Read:** P&H 2.2, 2.6, 3.3–3.4 · ISA manual, "Integer Computational
Instructions"
**You will:** compute, compare with `slt`, and divide.

RISC-V's arithmetic sets no flags, ever. Overflow is silent: the result just
wraps. To compare, `slt rd, a, b` ("set if less than") writes 1 or 0 into a
register. The M extension adds `mul`, `div` and `rem`; dividing by zero does
not trap but gives a defined result.

```asm run arch="riscv"
    .text
    .global _start
_start:
    li   t0, 7
    li   t1, 10
    sub  t2, t0, t1     # -3, and no flags anywhere
    slt  t3, t0, t1     # 1: 7 < 10
    li   t4, 100
    li   t5, 7
    div  t6, t4, t5     # 14
    rem  a0, t4, t5     # 2
    li   a7, 93
    ecall               # exit(2)
```

**Watch:**

- The Flags box says *none*: there is nothing to set.
- `slt` writes 1 into `t3`: the comparison's answer is a number in a
  register.
- `div` and `rem` are two instructions: one keeps the quotient, the other
  the remainder.

### Check yourself

1. What is `slt t3, t1, t0` here?
   <details><summary>Answer</summary>

   0: 10 < 7 is false.
   </details>

2. How can a program notice that an addition overflowed, with no flags?
   <details><summary>Answer</summary>

   Compare: for unsigned `a + b`, the sum is smaller than `a` exactly when
   it wrapped (`sltu` tests it). Signed checks take a few more instructions.
   </details>

### Common mistake

Looking for `cmp`: there is none. Compare inside the branch (4.3), or with
`slt` into a register.

## 4.3 Branches that compare

**Read:** P&H 2.7 · ISA manual, "Control Transfer Instructions"
**You will:** loop with a branch that compares two registers itself.

A RISC-V branch names two registers and a condition: `beq`, `bne`, `blt`,
`bge`, and the unsigned `bltu`, `bgeu`. It compares and decides in one
instruction. The missing ones are pseudo-instructions with the registers
swapped (`bgt a, b` is `blt b, a`; `ble a, b` is `bge b, a`) or with `zero`
(`beqz`, `bnez`).

```asm run arch="riscv"
    .text
    .global _start
_start:
    li   a0, 0          # sum
    li   t0, 1          # i
    li   t1, 5
again:
    add  a0, a0, t0     # sum += i
    addi t0, t0, 1      # i++
    ble  t0, t1, again  # really bge t1, t0, again
    li   a7, 93
    ecall               # exit(15)
```

**Watch:**

- `ble t0, t1, again` decodes as `bge t1, t0`: the B-type fields name
  `rs1` = `t1`, `rs2` = `t0`.
- Execute: the ALU compares the two registers and says *true* or *false*;
  PC update jumps back or goes on.
- The offset is split into pieces across the instruction (`imm[12]`,
  `imm[10:5]`, `imm[4:1]`, `imm[11]`): a hardware trick that keeps the
  register fields in the same place in every format.

### Check yourself

1. How many instructions does this loop run per round, and on x86-64?
   <details><summary>Answer</summary>

   Three here (add, addi, ble); x86-64 needs four (add, inc, cmp, jle),
   since its compare and jump are two instructions.
   </details>

2. Why might RISC-V's designers have left out flags?
   <details><summary>Answer</summary>

   Flags are state every instruction may change, which makes pipelines and
   out-of-order execution harder: each later instruction depends on them.
   A compare-and-branch carries everything it needs.
   </details>

### Common mistake

Using `blt` on addresses or sizes: they are unsigned, so use `bltu`.

## 4.4 Load and store

**Read:** P&H 2.3, 2.9 · ISA manual, "Load and Store Instructions"
**You will:** walk an array of 8-byte numbers with `ld`.

Loads and stores take `offset(register)`: `ld t2, 0(t0)` reads 8 bytes at
`t0 + 0`. The size is in the name: `lb`/`lbu` 1 byte (signed or not), `lh`/
`lhu` 2, `lw`/`lwu` 4, `ld` 8; `sb`, `sh`, `sw`, `sd` store. `la` loads a
label's address as two instructions: `auipc` (PC + upper bits) and `addi`.

```asm run arch="riscv"
    .data
nums:   .dword 3, 1, 4, 1, 5
    .text
    .global _start
_start:
    la   t0, nums       # auipc + addi
    li   t1, 5          # count
    li   a0, 0          # sum
loop:
    ld   t2, 0(t0)      # 8 bytes from memory
    add  a0, a0, t2
    addi t0, t0, 8      # the next element
    addi t1, t1, -1
    bnez t1, loop
    li   a7, 93
    ecall               # exit(14)
```

**Watch:**

- `auipc t0, 0x1` adds `0x1000` to the PC; `addi` fixes the rest: `t0`
  becomes `0x2000`, wherever the program is loaded.
- Each `ld`: Execute adds the offset, Memory reads 8 bytes (blue in
  `.data`), Write back fills `t2`.
- `addi t0, t0, 8` walks a pointer; there is no scaled index as on x86-64.

### Check yourself

1. A byte in memory is `0xFF`. What do `lb` and `lbu` load?
   <details><summary>Answer</summary>

   `lb`: −1 (sign-extended to 64 bits). `lbu`: 255 (zero-extended).
   </details>

2. Why does `la` take two instructions?
   <details><summary>Answer</summary>

   An instruction holds at most 20 bits of a number; an address needs more,
   so `auipc` supplies the upper 20 bits (from the PC) and `addi` the lower 12.
   </details>

### Common mistake

Adding 1 to a pointer to reach the next 8-byte element: the step is the
element's size, 8.

## 4.5 Functions and the stack

**Read:** P&H 2.8 · the RISC-V calling convention (psABI)
**You will:** write a recursive factorial with a real stack frame.

`call f` (written for `jal ra, f`) puts the return address in `ra` and
jumps; `ret` (`jalr zero, 0(ra)`) jumps back. A function that calls another
saves `ra` on the stack, and any `s` register it uses. The stack pointer
moves by multiples of 16.

```asm run arch="riscv"
    .text
    .global _start
_start:
    li   a0, 5
    call fact           # jal ra, fact
    li   a7, 93
    ecall               # exit(120)

fact:                   # long fact(long n)
    addi sp, sp, -16
    sd   ra, 8(sp)      # save the return address
    sd   s0, 0(sp)      # and s0, which this function uses
    mv   s0, a0         # s0 = n
    li   t0, 1
    ble  a0, t0, base   # n <= 1: the answer is 1
    addi a0, a0, -1
    call fact           # a0 = fact(n - 1)
    mul  a0, a0, s0     # n * fact(n - 1)
    j    out
base:
    li   a0, 1
out:
    ld   s0, 0(sp)
    ld   ra, 8(sp)
    addi sp, sp, 16
    ret
```

**Watch:**

- Five frames build up on the stack, 16 bytes each: the saved `ra` and `s0`
  of each call, amber as they are written.
- Each `ret` reads its own `ra` back; the stack shrinks frame by frame.
- `mul a0, a0, s0` uses the `s0` saved for this call: 2, 3, 4, 5.

### Check yourself

1. Why save `s0` and not `t0`?
   <details><summary>Answer</summary>

   `s0` must survive the inner `call`, and a called function may change
   any `t` register. The rule: values needed after a call go in `s`
   registers (saved by the callee) or on the stack.
   </details>

2. How deep does the stack go for `fact(5)`?
   <details><summary>Answer</summary>

   Five frames of 16 bytes: `sp` goes down to `0x8000 − 80 = 0x7fb0`.
   </details>

### Common mistake

Restoring `ra` too early or not at all in a function that calls another:
its `ret` then jumps to wherever the inner call left `ra`.

## 4.6 System calls: read and write

**Read:** ISA manual, "Environment Call" · Linux's `asm-generic/unistd.h`
**You will:** read a name and greet it.

`ecall` traps into the kernel (from user mode to supervisor mode). Linux
takes the number in `a7`, the arguments in `a0`–`a5`, and returns in `a0`:
63 `read`, 64 `write`, 93 `exit`, the same as ARM64.

```asm run arch="riscv" input="Grace\n"
    .data
hi:     .ascii "Hi, "
    .bss
name:   .space 32
    .text
    .global _start
_start:
    li   a0, 0          # read(0, name, 32)
    la   a1, name
    li   a2, 32
    li   a7, 63
    ecall               # a0 = how many bytes came: 6
    mv   s1, a0
    li   a0, 1          # write(1, hi, 4)
    la   a1, hi
    li   a2, 4
    li   a7, 64
    ecall
    li   a0, 1          # write(1, name, s1)
    la   a1, name
    mv   a2, s1
    li   a7, 64
    ecall
    li   a0, 0
    li   a7, 93
    ecall
```

**Watch:**

- The first `ecall`: the kernel waits for a line, the terminal echoes it,
  and the 6 bytes `Grace\n` land in `name` (amber in the memory panel).
- `mv s1, a0` keeps the count: the next `ecall` returns its own result in
  `a0`.
- The screen shows the typed line, then `Hi, Grace`.

### Check yourself

1. What does `read` return when there is no more input?
   <details><summary>Answer</summary>

   0: the end of the input. A negative number would be an error.
   </details>

2. What would `write(1, name, 32)` print?
   <details><summary>Answer</summary>

   The 6 bytes read and then 26 zero bytes: `write` sends exactly as many
   bytes as it is told, whatever they are.
   </details>

### Common mistake

Passing the string's length wrong: `write` does not look for a 0 at the
end, as C's `puts` does; it trusts the count.
