# Module 5 · One program, three machines

Read alongside: CS:APP chapter 3 · P&H chapter 2 (its sections comparing
other instruction sets)

The same C function, written by hand for each machine. Compilers do this
mechanically; reading their output (`gcc -S`, or Compiler Explorer) is the
next step after this track.

## 5.1 strlen, three times

**Read:** CS:APP 3.6.7 · P&H 2.9
**You will:** compare one loop on x86-64, ARM64 and RISC-V.

```c
long my_strlen(const char *s) {
    long n = 0;
    while (s[n] != 0)
        n++;
    return n;
}
```

The function's argument arrives in `rdi`, `x0` or `a0`; the result goes
back in `rax`, `x0` or `a0`. Here each `_start` calls it with a string and
exits with the length, 5.

```asm run arch="x86"
section .data
s       db "hello", 0
section .text
global _start
_start:
    lea  rdi, [rel s]
    call my_strlen
    mov  rdi, rax            ; exit status: the length
    mov  rax, 60
    syscall

my_strlen:                   ; rdi = s
    xor  eax, eax            ; n = 0
test:
    cmp  byte [rdi + rax], 0 ; s[n] == 0 ?
    je   done
    inc  rax                 ; n++
    jmp  test
done:
    ret                      ; the result is in rax
```

```asm run arch="arm"
    .data
s:  .asciz "hello"
    .text
    .global _start
_start:
    adr  x0, s
    bl   my_strlen
    mov  x8, #93
    svc  #0                 // exit(x0)

my_strlen:                  // x0 = s
    mov  x1, x0             // x1 walks the string
    mov  x0, #0             // n = 0
test:
    ldrb w2, [x1], #1       // c = *x1++
    cbz  w2, done
    add  x0, x0, #1         // n++
    b    test
done:
    ret                     // the result is in x0
```

```asm run arch="riscv"
    .data
s:  .asciz "hello"
    .text
    .global _start
_start:
    la   a0, s
    call my_strlen
    li   a7, 93
    ecall               # exit(a0)

my_strlen:              # a0 = s
    mv   t0, a0         # t0 walks the string
    li   a0, 0          # n = 0
test:
    lbu  t1, 0(t0)      # c = *t0
    beqz t1, done
    addi a0, a0, 1      # n++
    addi t0, t0, 1
    j    test
done:
    ret                 # the result is in a0
```

**Watch:**

- x86-64 compares memory directly (`cmp byte [rdi + rax], 0`); ARM64 and
  RISC-V load the byte into a register first.
- ARM64's post-index walks the pointer inside the load; RISC-V needs its own
  `addi`.
- Count the bytes in `.text`: x86-64's instructions are 1 to 7 bytes here
  (`ret` is one), the others always 4.

### Check yourself

1. How many instructions run per character on each machine?
   <details><summary>Answer</summary>

   x86-64: 4 (cmp, je, inc, jmp). ARM64: 4 (ldrb, cbz, add, b). RISC-V: 5
   (lbu, beqz, addi, addi, j). A compiler would rotate the loop to save the
   unconditional jump.
   </details>

2. Why does the ARM64 version copy `x0` into `x1` first?
   <details><summary>Answer</summary>

   `x0` is both the argument and the result. The pointer moves to `x1` so
   `x0` can count.
   </details>

### Common mistake

Reading the byte with a 64-bit load (`ld`, `ldr x2`): it reads 8 bytes, and
the test for 0 then looks at eight characters at once.

## 5.2 CISC and RISC, side by side

**Read:** P&H chapter 2, the "Real Stuff" sections · CS:APP 4.1 (the aside on RISC and CISC)
**You will:** know where the three differ, and why.

| | x86-64 | ARM64 | RISC-V (RV64) |
| --- | --- | --- | --- |
| General registers | 16 (`rax` … `r15`) | 31 (`x0` … `x30`) + `sp`, `xzr` | 32 (`x0` = 0 … `x31`) |
| Instruction length | 1 to 15 bytes | 4 bytes | 4 bytes (2 with the C extension) |
| Arithmetic on memory | yes: `add rax, [rbx]` | no: load, then add | no: load, then add |
| Operands | two: `add rax, rbx` | three: `add x0, x1, x2` | three: `add a0, a1, a2` |
| Flags | ZF SF CF OF, set by most arithmetic | N Z C V, only by `…s` instructions | none: branches compare |
| Calls | `call` pushes the return address | `bl` puts it in `x30` | `jal` puts it in `ra` |
| Numbers in an instruction | up to 64 bits (`mov`) | 16 bits at a time (`movz`/`movk`) | 12 bits, 20 with `lui`/`auipc` |
| System call | `syscall`, number in `rax` | `svc #0`, number in `x8` | `ecall`, number in `a7` |

x86-64 is a **CISC** (complex instruction set): its instructions do more
each, and a program takes fewer bytes, but decoding them is hard. Since the
1990s, x86 chips translate them inside into simpler operations, much like
RISC instructions. ARM64 and RISC-V are **RISC** (reduced instruction set):
simple, regular instructions that are easy to decode and pipeline. In
practice, speed now depends far more on the design of the core (its caches,
its prediction, how many instructions it runs at once) than on the
instruction set.

### Check yourself

1. Which machine needs the most instructions to load the number 0x12345678
   into a register?
   <details><summary>Answer</summary>

   ARM64: `movz` and `movk` (two). RISC-V also takes two (`lui` + `addi`).
   x86-64 does it in one `mov`, 5 bytes long with a 32-bit register.
   </details>

2. Why can a RISC-V decoder find the next instruction without decoding the
   current one?
   <details><summary>Answer</summary>

   Every base instruction is 4 bytes (with the C extension, the low two bits
   say whether it is 2 or 4). On x86-64 the length depends on prefixes,
   opcode and ModRM.
   </details>

### Common mistake

Taking "RISC" to mean "fewer instructions". It means simpler ones: ARM64 has
hundreds. What is reduced is what each instruction does.

## 5.3 From C to assembly

**Read:** CS:APP 3.7–3.8 · P&H 2.13–2.14
**You will:** translate a C function the way a compiler does.

A compiler turns each C construct into a fixed shape: arguments into the
convention's registers, a loop into a label and a conditional jump, an
array access into a scaled address. Here is `sum`, by hand, in each:

```c
long sum(const long *a, long n) {
    long s = 0;
    for (long i = 0; i < n; i++)
        s += a[i];
    return s;
}
```

```asm run arch="x86"
section .data
nums    dq 10, 20, 30, 40
section .text
global _start
_start:
    lea  rdi, [rel nums]     ; a
    mov  rsi, 4              ; n
    call sum
    mov  rdi, rax            ; exit(100)
    mov  rax, 60
    syscall

sum:                         ; rdi = a, rsi = n
    xor  eax, eax            ; s = 0
    xor  ecx, ecx            ; i = 0
    jmp  check
body:
    add  rax, [rdi + rcx*8]  ; s += a[i]
    inc  rcx                 ; i++
check:
    cmp  rcx, rsi            ; i < n ?
    jl   body
    ret
```

```asm run arch="arm"
    .data
nums:   .quad 10, 20, 30, 40
    .text
    .global _start
_start:
    adr  x0, nums           // a
    mov  x1, #4             // n
    bl   sum
    mov  x8, #93
    svc  #0                 // exit(100)

sum:                        // x0 = a, x1 = n
    mov  x2, #0             // s
    mov  x3, #0             // i
    b    check
body:
    ldr  x4, [x0], #8       // a[i], and a moves on
    add  x2, x2, x4
    add  x3, x3, #1
check:
    cmp  x3, x1
    b.lt body
    mov  x0, x2             // return s
    ret
```

```asm run arch="riscv"
    .data
nums:   .dword 10, 20, 30, 40
    .text
    .global _start
_start:
    la   a0, nums       # a
    li   a1, 4          # n
    call sum
    li   a7, 93
    ecall               # exit(100)

sum:                    # a0 = a, a1 = n
    li   t0, 0          # s
    li   t1, 0          # i
    j    check
body:
    ld   t2, 0(a0)      # a[i]
    add  t0, t0, t2
    addi a0, a0, 8      # the next element
    addi t1, t1, 1
check:
    blt  t1, a1, body   # i < n ?
    mv   a0, t0         # return s
    ret
```

**Watch:**

- All three jump to the test first, then loop with the test at the bottom:
  the shape compilers use for a `for` loop.
- x86-64 indexes (`[rdi + rcx*8]`); the others step the pointer by 8.
- The result ends where the calling convention says: `rax`, `x0`, `a0`.

### Check yourself

1. Why jump to `check` before the first round?
   <details><summary>Answer</summary>

   A `for` loop may run zero times (n = 0). Testing at the bottom, after
   one jump in, needs only one conditional jump per round.
   </details>

2. What would `gcc -O2` do differently?
   <details><summary>Answer</summary>

   Similar shapes, often fewer instructions: it might unroll the loop,
   use the pointer as the counter, or (on x86-64 and ARM64) add several
   elements at once with vector instructions.
   </details>

### Common mistake

Expecting a compiler's assembly to follow the C line by line. With
optimisation on, variables live in registers, loops change shape, and some
code disappears; the meaning stays, the form does not.
