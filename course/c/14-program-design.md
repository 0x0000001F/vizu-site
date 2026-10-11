# Module 14 · Program design

Read alongside: King chapter 19 · K&R sections 6.5–6.6

The last module puts the pieces together: an abstract data type that hides
how it works, and a complete program built from a tree, strings and the
heap.

## 14.1 A stack, three ways

**Read:** King 19.1–19.4
**You will:** separate what a data structure does from how it does it.

King builds a stack (push, pop, is it empty) three times: on a fixed array, on
an array that grows, and on a linked list. The users of the stack never
change, because they only see its *interface*: a few functions and an
*incomplete type*, `struct stack`, whose members they cannot see. In a real
program the interface is `stack.h`, and the struct's definition hides in
`stack.c`.

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

**Watch:** the heap holds the `Stack` and a chain of cells, each pointing to
the one below. `main` only ever holds `s`.

### Check yourself

1. What is printed, and why in that order?
   <details><summary>Answer</summary>

   `3 2 1`: last in, first out.
   </details>

2. Replace the list with an array that doubles (lesson 10.2). What changes in
   `main`?
   <details><summary>Answer</summary>

   Nothing. Only `struct stack` and the five functions change. That is the
   point of the interface.
   </details>

### Common mistake

`stack_pop` on an empty stack: `c` is `NULL` and `c->value` fails. Decide
what the contract is (the caller must check `stack_empty`, or `pop` reports an
error) and write it in the header.

## 14.2 Capstone: a word-frequency counter

**Read:** K&R 6.5 · King 17.5
**You will:** build a complete program from a binary tree, strings and the
heap.

K&R end their structures chapter with a program that counts how often each
word appears, keeping the words in a *binary search tree*: every node's left
subtree holds smaller words, its right subtree larger ones. Inserting walks
down from the root comparing words; printing visits the left subtree, the
node, then the right, which gives alphabetical order. Here is a version of
that program, with the tree freed at the end.

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

**Watch:**

- Each new word makes two heap blocks: the node, and a copy of the word
  (`strlen(w) + 1` bytes, for the `'\0'`).
- A repeated word walks down the tree and only bumps `count`.
- `print_tree`'s frames pile up as it goes left, the same recursion as in
  lesson 5.5, now over a tree.

### Check yourself

1. Which word is the root of the tree, and why?
   <details><summary>Answer</summary>

   `the`, the first word inserted. Every later word is compared with it first.
   </details>

2. What does the tree look like if the input is already in alphabetical
   order?
   <details><summary>Answer</summary>

   Every word goes to the right of the previous one: the tree is a linked
   list, and each insert walks all of it. Balanced trees (or a hash table, K&R
   6.6) avoid this.
   </details>

### Your project

Extend the program, one step at a time, running it in Vizu after each:

1. Print the words by count, most frequent first: copy the node pointers into
   an array and sort it with `qsort` (lesson 11.1).
2. Ignore a list of common words ("the", "and", "a").
3. Replace the tree with a hash table of 101 linked lists (K&R 6.6), and
   compare the number of steps Vizu takes on a long input.

### Common mistake

`t->word = w;` instead of copying. Every node would point into `main`'s single
`word` buffer, which the next `get_word` overwrites: all nodes end up showing
the last word read.
