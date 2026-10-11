# Module 14 · Program design

You now know the whole of C that most programs use. This last module is
about something different: not new parts of the language, but how to
arrange the parts into a program that stays understandable as it grows.

Small programs can be written straight through, and they work. Large
programs cannot. A program of ten thousand lines written as one tangle,
where any function may touch any variable, becomes impossible to change:
every change breaks something far away. The way out, in C as in every
language, is to divide the program into *modules*, each with a clear job,
and to let each module hide how it does that job behind a small set of
functions. This module shows that idea on a classic example, a stack, and
then builds two complete programs from the pieces of the whole course: a
word counter on a tree, and a table on a hash.

## 14.1 Modules and hiding

A module, in C, is usually a pair of files: a header that says what the
module offers, and a source file that does the work (lesson 5.8). The
header is the module's *interface*: the functions, types and constants
that the rest of the program may use. Everything else in the source file
is the *implementation*, and the rest of the program neither sees it nor
depends on it.

Two qualities make a division into modules a good one. Each module should
have *high cohesion*: everything in it belongs to one job, so that its job
can be described in a sentence. And the modules should have *low coupling*:
they should know as little about each other as possible, communicating only
through their interfaces. Then a module can be understood alone, tested
alone, and rewritten without disturbing the others.

C has one main tool for hiding: the word `static`, which makes a variable
or function outside all functions private to its source file. Here is a
module that keeps a stack of integers in an array, with its data hidden:

```c
/* stack.h: the interface */
#include <stdbool.h>
void make_empty(void);
bool is_empty(void);
void push(int i);
int pop(void);
```

```c
/* stack.c: the implementation */
#include "stack.h"
#define STACK_SIZE 100

static int contents[STACK_SIZE];    /* invisible outside this file */
static int top = 0;

void make_empty(void) { top = 0; }
bool is_empty(void) { return top == 0; }
void push(int i) { contents[top++] = i; }
int pop(void) { return contents[--top]; }
```

A *stack* is a collection where items are added and removed at the same
end, the top, like a pile of plates: the last item pushed is the first one
popped. The rest of the program can only push, pop and ask whether the
stack is empty. It cannot reach `contents` or `top`, so it cannot put them
into an impossible state, and if the array is one day replaced by
something else, no other file has to change.

This module has one limitation: there is exactly one stack, since its data
are single variables. The next lesson removes it.

### Check yourself

1. Why are `contents` and `top` declared `static`?
   <details><summary>Answer</summary>

   So that no other file can use them. The only way to change the stack is
   through the four functions, which keep it consistent.
   </details>

2. What does "low coupling" mean for two modules?
   <details><summary>Answer</summary>

   That each knows as little as possible about the other: only its
   interface, never how it works inside. Changing one then rarely forces a
   change in the other.
   </details>

### A common mistake

Putting a variable in the header "so that other files can use it". That
couples every file that includes the header to the module's inner data.
Offer a function that does what the other files need instead.

## 14.2 An abstract data type

To allow many stacks, each stack must be a value that the program can
create, pass around and destroy, like a `FILE *` from the library. Such a
type, which the rest of the program uses only through its functions,
without seeing what is inside it, is called an *abstract data type*.

C makes this possible with an *incomplete type*. The header says that a
`struct stack` exists, without saying what is in it. The rest of the
program can then hold pointers to stacks and pass them to the functions,
but it cannot look inside, because the compiler does not know the members.
Only the module's own source file completes the type.

Here the interface and one implementation, on a linked list, are put in one
file so that Vizu can run them; in a real program the part above the
second comment would be `stack.h`, and the rest `stack.c`.

```c run
#include <stdio.h>
#include <stdlib.h>
#include <stdbool.h>

/* the interface: all that users of the stack see */
typedef struct stack Stack;
Stack *stack_new(void);
void stack_push(Stack *s, int v);
int stack_pop(Stack *s);
bool stack_empty(const Stack *s);
void stack_free(Stack *s);

/* one implementation: a linked list */
struct cell {
    int value;
    struct cell *below;
};

struct stack {
    struct cell *top;
};

Stack *stack_new(void) {
    Stack *s = malloc(sizeof *s);
    s->top = NULL;
    return s;
}

void stack_push(Stack *s, int v) {
    struct cell *c = malloc(sizeof *c);
    c->value = v;
    c->below = s->top;
    s->top = c;
}

int stack_pop(Stack *s) {
    struct cell *c = s->top;
    int v = c->value;
    s->top = c->below;
    free(c);
    return v;
}

bool stack_empty(const Stack *s) {
    return s->top == NULL;
}

void stack_free(Stack *s) {
    while (!stack_empty(s))
        stack_pop(s);
    free(s);
}

int main(void) {
    Stack *s = stack_new();
    for (int i = 1; i <= 3; i++)
        stack_push(s, i);
    while (!stack_empty(s))
        printf("%d ", stack_pop(s));
    printf("\n");
    stack_free(s);
    return 0;
}
```

`typedef struct stack Stack;` declares the incomplete type and gives it a
short name. `main` uses nothing but `Stack *` and the five functions.

Step through it and watch the heap. `stack_new` makes a small block, the
`Stack` itself, holding only `top`. Each `stack_push` adds a cell whose
`below` arrow points to the previous top, and the `Stack`'s `top` arrow
swings to the new cell. Each `stack_pop` takes the top cell off, frees it,
and moves `top` down. Meanwhile, `main`'s frame holds only `s`, one arrow
into the heap: everything else is the module's business.

The great advantage shows when the implementation changes. The stack could
be rewritten on an array that doubles when it is full (lesson 10.3), or on a
fixed array of 100 elements, and `main` would not change by a single
character, because it never depended on anything but the interface.

### Check yourself

1. What does the program print, and why in that order?
   <details><summary>Answer</summary>

   `3 2 1`. A stack gives items back in the opposite order: the last one
   pushed is the first one popped.
   </details>

2. Could `main` write `s->top`?
   <details><summary>Answer</summary>

   Not in a real program where the definition of `struct stack` is hidden
   in `stack.c`: in `main`'s file the type is incomplete, and the compiler
   refuses to look inside it. (Here, with everything in one file, it would
   compile, but it would break the agreement.)
   </details>

### A common mistake

Calling `stack_pop` on an empty stack. `c` is `NULL`, and `c->value`
fails. Every abstract data type must say what happens in such a case:
either the caller must check `stack_empty` first, or `pop` reports the
error. Whichever it is, it belongs in the header, as a comment.

## 14.3 Counting words with a tree

Now a complete program, bringing together structures, pointers, strings,
the heap and recursion. It counts how often each word appears in its
input, and prints the words in alphabetical order.

The words are kept in a *binary search tree*. Each node holds a word, its
count, and two pointers, `left` and `right`, to two smaller trees. The rule
that makes it a search tree is this: every word in the left subtree of a
node comes before the node's word in alphabetical order, and every word in
the right subtree comes after it. To add a word, start at the root and
compare: if the word is smaller, go left, if larger, go right, and repeat
until the word is found or an empty place is reached, where a new node is
made. To print in order, print the left subtree, then the node, then the
right subtree, which recursion does naturally.

```c run input="the cat and the hat and the bat\n"
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <ctype.h>

struct tnode {
    char *word;
    int count;
    struct tnode *left, *right;
};

struct tnode *add(struct tnode *t, const char *w) {
    if (t == NULL) {
        t = malloc(sizeof *t);
        t->word = malloc(strlen(w) + 1);
        strcpy(t->word, w);
        t->count = 1;
        t->left = t->right = NULL;
    } else {
        int cmp = strcmp(w, t->word);
        if (cmp == 0)
            t->count++;
        else if (cmp < 0)
            t->left = add(t->left, w);
        else
            t->right = add(t->right, w);
    }
    return t;
}

void print_tree(const struct tnode *t) {
    if (t != NULL) {
        print_tree(t->left);
        printf("%4d %s\n", t->count, t->word);
        print_tree(t->right);
    }
}

void free_tree(struct tnode *t) {
    if (t != NULL) {
        free_tree(t->left);
        free_tree(t->right);
        free(t->word);
        free(t);
    }
}

int get_word(char *buf, int max) {
    int c, n = 0;
    while ((c = getchar()) != EOF && !isalpha(c))
        ;
    if (c == EOF)
        return 0;
    do {
        if (n < max - 1)
            buf[n++] = tolower(c);
    } while ((c = getchar()) != EOF && isalpha(c));
    buf[n] = '\0';
    return 1;
}

int main(void) {
    char word[32];
    struct tnode *root = NULL;
    while (get_word(word, sizeof word))
        root = add(root, word);
    print_tree(root);
    free_tree(root);
    return 0;
}
```

The program has four parts, each a function with one job. `get_word` reads
the next word from the input into an array, skipping anything that is not a
letter, and returns 0 when the input runs out. `add` puts a word into the
tree and returns the tree's root. `print_tree` prints the tree in order.
`free_tree` gives all of it back; notice that it frees both subtrees before
the node itself, since after `free(t)` the pointers `t->left` and
`t->right` could no longer be read.

`add` copies each new word into a block of its own, `strlen(w) + 1` bytes,
because the array `word` in `main` is reused for every word read.

This is the longest run in the course, so use the slider. Things to watch:
each new word makes two blocks on the heap, the node and the copy of the
word. The second "the" walks down from the root, finds the word at once,
and only adds one to `count`. While `add` searches, its frames pile up, one
per level of the tree, just like the recursion of lesson 5.6, now following
arrows in the heap. And at the end, `free_tree` makes every block fade, so
that nothing is leaked.

### Check yourself

1. Which word is at the root of the tree, and why?
   <details><summary>Answer</summary>

   `the`, the first word read. Every later word is compared with it first,
   and goes left or right of it.
   </details>

2. What happens to the tree if the input is already in alphabetical order?
   <details><summary>Answer</summary>

   Every word goes to the right of the one before, so the tree becomes one
   long chain, no better than a linked list, and each insertion walks all of
   it. Balanced trees, which rearrange themselves, or the hash table of the
   next lesson avoid this.
   </details>

### A common mistake

Writing `t->word = w;` instead of copying the word. Every node would then
point into `main`'s single array `word`, which the next call of `get_word`
overwrites, and in the end every node would show the last word read.

## 14.4 Finding things fast: a hash table

A tree finds a word in a number of steps that grows with the height of the
tree. A *hash table* can usually find it in a few steps, however many
words there are. The idea is to compute, from the word itself, a number
called its *hash*, and use that number as an index into an array of
*buckets*. To find a word, compute its hash and look only in that bucket.

Different words can have the same hash; this is called a *collision*. So
each bucket is a linked list of the entries whose words hash to it. As long
as the lists stay short, which a good hash function and enough buckets
ensure, a lookup only compares the word with a handful of others.

Here is a small table that maps names to numbers, like the table a
preprocessor keeps of its `#define`s:

```c run
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define BUCKETS 7

struct entry {
    char *name;
    int value;
    struct entry *next;     /* the next entry in the same bucket */
};

static struct entry *table[BUCKETS];

unsigned hash(const char *s) {
    unsigned h = 0;
    for (; *s != '\0'; s++)
        h = *s + 31 * h;
    return h % BUCKETS;
}

struct entry *lookup(const char *name) {
    for (struct entry *e = table[hash(name)]; e != NULL; e = e->next)
        if (strcmp(name, e->name) == 0)
            return e;
    return NULL;
}

struct entry *install(const char *name, int value) {
    struct entry *e = lookup(name);
    if (e == NULL) {
        e = malloc(sizeof *e);
        e->name = malloc(strlen(name) + 1);
        strcpy(e->name, name);
        unsigned b = hash(name);
        e->next = table[b];
        table[b] = e;
    }
    e->value = value;
    return e;
}

int main(void) {
    install("width", 80);
    install("height", 24);
    install("depth", 3);
    install("size", 12);
    install("speed", 5);
    install("width", 100);
    const char *names[] = {"width", "size", "colour"};
    for (int i = 0; i < 3; i++) {
        struct entry *e = lookup(names[i]);
        if (e != NULL)
            printf("%s = %d (bucket %u)\n", names[i], e->value, hash(names[i]));
        else
            printf("%s is not in the table\n", names[i]);
    }
    return 0;
}
```

`hash` mixes all the characters of a name into one number: each step
multiplies what it has so far by 31 and adds the next character's code.
The remainder after dividing by the number of buckets is the bucket.
`lookup` walks the list in that bucket, comparing names. `install` looks the
name up first; if it is new, it makes an entry, with its own copy of the
name, and puts it at the front of its bucket's list; either way it sets the
value.

The table itself, `table`, is an array of seven pointers among the globals,
all `NULL` at the start, since global variables begin as zero. Step through
the installs and watch it fill. "depth" and "size" happen to hash to the
same bucket, 2, and so do "width" and "speed", in bucket 1: watch the second
one of each pair go in front, with its `next` arrow pointing to the first.
The second `install("width", 100)` finds the existing entry and only
changes its value.

The table has only seven buckets so that the picture stays small. A real
table chooses many more, perhaps a few thousand, and some grow the array
when the lists get long.

### Check yourself

1. How many names does `lookup("speed")` compare with?
   <details><summary>Answer</summary>

   Only one: "speed" is at the front of bucket 1, so the first comparison
   finds it. "width" would need two, since it is second in the same bucket.
   </details>

2. What would happen with `BUCKETS` set to 1?
   <details><summary>Answer</summary>

   Every name would land in the same bucket, and the table would become a
   single linked list, searched from the front every time.
   </details>

### A common mistake

A hash function that uses only part of the name, such as its first letter.
Then all names beginning with the same letter collide, the lists grow long,
and the table loses its speed. A good hash lets every character change the
result.

## 14.5 Your project

You have reached the end of the C track. The best way to make it stick is
to change the programs of this module, one small step at a time, running
each step in the tool and watching what it does. Here are some steps,
roughly in order of difficulty:

1. In the word counter, ignore a list of common words such as "the",
   "and" and "a". Where is the best place to check: in `get_word`, in `add`,
   or in `main`?
2. Print the words with their counts sorted by count, the most frequent
   first. Copy the pointers to the nodes into an array, and sort it with
   `qsort` (lesson 11.2) and a comparison that looks at `count`.
3. Count the words with the hash table instead of the tree: store the
   count as the value, adding one when the word is already there. Then sort
   the entries for printing, as in step 2.
4. Give the hash table a function that removes an entry, using a pointer
   to a pointer as in lesson 10.6, and make sure Vizu reports no leak at the
   end.
5. Rewrite the stack of lesson 14.2 on an array that doubles, and check
   that `main` runs unchanged.

Every one of these uses the same few ideas you have now seen dozens of
times: boxes, addresses and arrows, frames that come and go, and blocks on
the heap that live until they are freed. That picture is what C is.
