# Module 9 · Caches, and the limits of speed

The book's machine reads RAM in a single clock step. For the early
computers that was roughly true, and it is the assumption the C track made
too: a variable is read when the program reads it. But over the decades,
processors became faster much more quickly than memory did. Today a
processor can carry out hundreds of instructions in the time it takes to
fetch one value from RAM. If it really had to wait for RAM at every load, all
the cleverness of module 8 would be wasted: the pipeline would stand still
most of the time.

The answer is the *cache*: a small, fast memory, right next to the
processor, that keeps copies of the bytes it is using. This module explains
how caches work, following *Inside the Machine*, and lets you watch one
serve a C program's reads, hit by hit and miss by miss. It shows why the
order in which a program walks through an array can change its speed many
times over. And it ends with the other wall that processors ran into, the
heat they produce, and why today's chips have many cores instead of one
very fast one.

## 9.1 The memory gap

Think of a cook who needs ingredients. The few in use right now are on the
worktop, within arm's reach. Others are in the kitchen cupboard, a few steps
away. Many more are in a storeroom at the back of the building, and almost
anything can be had from a warehouse across town, if you are willing to wait
a day. Nobody runs to the warehouse for every pinch of salt: the cook keeps
what is needed often close at hand, and fetches from further away only what
is missing, and then brings back a whole box at once.

A computer's memory is arranged the same way, as a *memory hierarchy*. Each
level is bigger, slower and cheaper per byte than the one above it. These
are rough figures for a current desktop processor; the exact numbers vary
from chip to chip and change every few years, but the proportions have
stayed much the same:

| Level | Typical size | Time to reach it |
| --- | --- | --- |
| Registers | a few hundred bytes | no delay |
| Level 1 cache (L1) | 32–64 KB per core | about 4 cycles |
| Level 2 cache (L2) | 0.5–2 MB per core | about 12–15 cycles |
| Level 3 cache (L3) | 8–64 MB, shared | about 40–60 cycles |
| Main memory (RAM) | 8–64 GB | about 200–400 cycles |
| SSD | hundreds of GB | about 100 000 cycles |

When the processor asks for a byte, the L1 cache is checked first. If the
byte is there, that is a *hit*, and it arrives in a few cycles. If not, a
*miss*, the request goes on to L2, then L3, then RAM. The byte found is copied
into the caches on its way to the processor, so that the next request for it
is a hit. Registers are managed by the compiler, which decides which values
to keep in them; caches are managed entirely by the hardware, invisibly. A
program never says "put this in the cache"; it just runs faster or slower
depending on how well the caches can guess what it needs.

### Check yourself

1. A program misses the caches and goes to RAM once every 100 instructions,
   and each trip costs 300 cycles. Roughly how much of its time is waiting?
   <details><summary>Answer</summary>

   Most of it. 100 instructions might take around 100 cycles on a
   pipelined processor, plus 300 cycles of waiting: three quarters of the
   time is spent waiting for memory.
   </details>

2. Who decides what is kept in the cache: the program, the compiler, or the
   hardware?
   <details><summary>Answer</summary>

   The hardware, by itself, according to fixed rules. The program only
   influences it through the order in which it uses memory.
   </details>

## 9.2 Locality, and blocks

Why should a small cache help at all, when a program may use gigabytes of
data? Because programs do not use memory at random. They show *locality of
reference*, of two kinds:

- **Temporal locality**: a byte used now is likely to be used again soon.
  A loop counter, a running total, the instructions of a loop body: all are
  used over and over.
- **Spatial locality**: a byte used now is likely to be followed by its
  neighbours. A program that reads `a[0]` usually goes on to `a[1]` and
  `a[2]`; instructions are fetched one after another.

Caches exploit temporal locality simply by keeping what was used recently.
To exploit spatial locality, they never fetch single bytes. Memory is divided
into *blocks* of equal size, typically 64 bytes, called *cache lines*, and a
miss always brings in the whole block containing the byte asked for. The
neighbours then come along for free.

Here is a C program that adds up a table of 4 rows and 8 columns, row after
row, read through a tiny cache. To keep the picture small, this cache has
just 4 frames of 16 bytes each, room for four blocks of four `int`s. Each
step of the lab is one read of the table:

```cache run lines=4 line=16 ways=1 only=reads
#include <stdio.h>

int main(void) {
    int grid[4][8] = {0};
    int sum = 0;
    for (int r = 0; r < 4; r++)
        for (int c = 0; c < 8; c++)
            sum += grid[r][c];
    printf("%d\n", sum);
    return 0;
}
```

Step through it. The first read, `grid[0][0]`, is a miss: the cache is empty.
The block containing it, `grid[0][0]` to `grid[0][3]`, is brought in, and it
turns amber in its frame. The next three reads are hits, since those
elements arrived with the first; their frame is outlined in blue. Then
`grid[0][4]` misses and brings in the next block, and so on. In the strip of
squares under the cache, the pattern is plain: one amber miss, then three
dark hits, over and over. Three reads in four come from the cache, a hit rate
of 75 percent, although each block is used only once.

These misses are called *compulsory misses*: the first use of a block must
miss, whatever the cache does. With real 64-byte lines, sixteen `int`s come
in at a time, and a row-by-row walk hits fifteen times out of sixteen.

### Check yourself

1. With blocks of 64 bytes, what fraction of reads miss when a program
   walks through an array of `double`s from the start to the end?
   <details><summary>Answer</summary>

   One in eight: a 64-byte block holds eight 8-byte `double`s, and only the
   first read of each block misses.
   </details>

2. Which variable in the program has the best temporal locality?
   <details><summary>Answer</summary>

   `sum` (and the counters `r` and `c`): they are used in every round of the
   loop. In a real program the compiler would keep them in registers.
   </details>

## 9.3 Where a block may go

When a block is brought in, which frame of the cache does it go into? The
rule is called the *placement policy*, and there are three kinds.

- In a **fully associative** cache, a block may go into any frame. That
  makes the best use of the space, but to find a block, the cache must
  compare its address with the *tag* of every frame at once, which is
  expensive in hardware and slow for a large cache.
- In a **direct-mapped** cache, each block has exactly one frame it may go
  into: block number modulo the number of frames. Finding a block is quick,
  since only one frame needs checking. But two blocks that map to the same
  frame cannot both be in the cache, even if every other frame is empty.
- An **n-way set associative** cache is the compromise. The frames are
  grouped into *sets* of n frames; a block may go into any frame of one set,
  its block number modulo the number of sets. Only n tags need comparing.

The address of a byte tells the cache everything it needs. Its lowest bits
are the *offset*, the position of the byte within its block; the next bits
are the *set index*, which set to look in; the remaining high bits are the
*tag*, kept with each frame to say which block it holds. The lab shows this
split for every access: the tag bits on grey, the set bits on blue, the
offset bits outlined.

Now the same table, but added up column by column: `grid[0][0]`, then
`grid[1][0]`, `grid[2][0]`, `grid[3][0]`, then the next column:

```cache run lines=4 line=16 ways=1 only=reads
#include <stdio.h>

int main(void) {
    int grid[4][8] = {0};
    int sum = 0;
    for (int c = 0; c < 8; c++)
        for (int r = 0; r < 4; r++)
            sum += grid[r][c];
    printf("%d\n", sum);
    return 0;
}
```

Step through it, and watch the strip fill with amber. Every single read is a
miss. A row of the table is 32 bytes, two blocks, so going down a column
jumps two blocks each time: the blocks of rows 0 and 2 map to the same frame,
and so do those of rows 1 and 3. Each read throws out the block that the read
two steps later will need again. These are *conflict misses*: the data would
fit in the cache, but the placement rule forces blocks into the same frames.

Give the same cache four ways, one set of four frames in which any block may
go anywhere:

```cache run lines=4 line=16 ways=4 only=reads
#include <stdio.h>

int main(void) {
    int grid[4][8] = {0};
    int sum = 0;
    for (int c = 0; c < 8; c++)
        for (int r = 0; r < 4; r++)
            sum += grid[r][c];
    printf("%d\n", sum);
    return 0;
}
```

Now the four blocks of a column's top half fit together, and stay for the
next three columns: 75 percent hits again. Real L1 caches are typically 8- or
12-way set associative, which avoids most conflicts while keeping the search
small.

But the lesson for programmers is the first one. Walk through memory in the
order it is laid out. The C track showed (lesson 4.5) that a two-dimensional
array is stored row after row; a loop that goes along the rows uses every
byte of each block it brings in, and a loop that goes down the columns may
use only a few bytes of each block before throwing it away. On large tables,
swapping the two loops can make a program several times faster or slower.

### Check yourself

1. In a direct-mapped cache of 4 frames, which frame does block 2042 go
   into?
   <details><summary>Answer</summary>

   Frame 2: 2042 modulo 4 is 2.
   </details>

2. Why does a 2-way cache of 4 frames still miss on every read of the
   column walk?
   <details><summary>Answer</summary>

   With 2 sets, the blocks of all four rows' first halves have even numbers
   and fall into the same set, which has room for only two of them. Each
   read throws out a block that is needed again soon.
   </details>

## 9.4 Making room, and writing

### Which block to throw out

When a block must come in and its set is full, one of the blocks in the set
has to go. The *replacement policy* chooses which. The usual choice is the
block used *least recently*, LRU: if temporal locality holds, the block not
touched for the longest time is the least likely to be needed soon. (Real
caches often use a cheaper approximation of LRU, or even a random choice,
which works surprisingly well.) The labs' caches use LRU, and the caption
says which block was thrown out to make room.

### Three kinds of miss

Misses come in three kinds, and each has its own cure:

- **Compulsory** misses: the first use of a block. Larger blocks help, and so
  does *prefetching*: hardware that notices a program walking through memory
  and fetches the next blocks before they are asked for.
- **Capacity** misses: the program's data simply does not fit in the cache,
  so blocks are thrown out before they are used again. Only a bigger cache,
  or a program that works on less data at a time, helps.
- **Conflict** misses: the data would fit, but the placement rule forces it
  into the same sets, as in the column walk. More ways help, and so does a
  different order of access.

### Writing

When the processor writes a byte, what happens to the copy in RAM? A
*write-through* cache writes the byte to the cache and to the next level at
once, which keeps memory up to date but costs a trip for every write. A
*write-back* cache only changes its own copy, marks the block as *dirty*, and
writes the whole block back to memory only when the block is thrown out.
Most processors today use write-back caches, because programs often write
the same bytes many times.

With several cores, each with its own cache, a new problem appears: two
caches may hold copies of the same block, and a write in one must be noticed
by the other. The hardware solves this with a *coherence protocol*: a write
to a shared block makes the other copies invalid, so that no core ever reads
a stale value.

### Check yourself

1. A program scans through a 1 GB file once, from start to end. What kind of
   misses does it have, and what helps?
   <details><summary>Answer</summary>

   Compulsory misses: every block is new. Prefetching helps most, since the
   pattern is easy to spot; a bigger cache would not help at all, as nothing
   is used twice.
   </details>

2. Why can a linked list be much slower to walk than an array, even when
   both hold the same numbers?
   <details><summary>Answer</summary>

   The array's elements lie next to each other, so each block brought in
   serves several of them, and the prefetcher can see the pattern. The
   list's nodes may lie anywhere on the heap, so almost every step to the
   next node can be a miss.
   </details>

## 9.5 Power, heat and many cores

For thirty years, processors became faster in two ways at once: more
transistors on each chip, and a faster clock. Clock speeds rose from a few
megahertz in the late 1970s to more than three gigahertz by the early
2000s. Then they stopped rising. A processor of today runs at roughly the
same clock speed as one of 2005. What happened?

Heat happened. Every time a transistor switches, it uses a little energy,
which turns into heat. The power used by switching grows with the number of
transistors, with the clock speed, and with the square of the voltage. As
chips packed in more transistors and ran them faster, the *power density*,
the heat produced on each square millimetre of silicon, rose until it
approached that of a hot plate, and then of a nuclear reactor's core. Stokes
describes how Intel called this the *power wall*. On top of that, very small
transistors leak current even when they are not switching, so a chip uses
power just by being on. A chip that gets too hot must slow down, or it will
damage itself.

The way out, around 2005, was a change of direction. Instead of one core
running ever faster, chips got several *cores*, each a complete processor,
running at a moderate clock speed. Intel's Core Duo and Core 2 Duo, which
Stokes describes, grew out of the power-efficient Pentium M, designed for
laptops; two cores at a sensible speed gave more work per watt than one core
pushed to its limit. Today a phone has eight cores, often of two kinds,
fast ones for heavy work and small, frugal ones for everything else, and a
desktop processor may have sixteen or more.

More cores only help a program that is written to use them, by dividing its
work into *threads* that run at the same time, which is a subject of its own.
A program written as one sequence of instructions runs on one core, and
speeds up only as much as that one core's pipeline, prediction and caches can
make it.

### Check yourself

1. Why did clock speeds stop rising around 2005, although transistors kept
   getting smaller?
   <details><summary>Answer</summary>

   Because of heat. Faster switching of more transistors produced more heat
   per square millimetre than could be carried away, and small transistors
   leak power even when idle. Designers used the extra transistors for more
   cores and bigger caches instead.
   </details>

2. A program takes 10 seconds on one core. Will it take 2.5 seconds on four?
   <details><summary>Answer</summary>

   Only if it is written to split its work into four parts that run at the
   same time, and only if those parts do not wait for each other or for
   memory. A program that is not divided this way still takes 10 seconds.
   </details>
