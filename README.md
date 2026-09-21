# chibicc: A Small C Compiler

(The old master has moved to
[historical/old](https://github.com/rui314/chibicc/tree/historical/old)
branch. This is a new one uploaded in September 2020.)

chibicc is yet another small C compiler that implements most C11
features. Even though it still probably falls into the "toy compilers"
category just like other small compilers do, chibicc can compile
several real-world programs, including [Git](https://git-scm.com/),
[SQLite](https://sqlite.org),
[libpng](http://www.libpng.org/pub/png/libpng.html) and chibicc
itself, without making modifications to the compiled programs.
Generated executables of these programs pass their corresponding test
suites. So, chibicc actually supports a wide variety of C11 features
and is able to compile hundreds of thousands of lines of real-world C
code correctly.

chibicc is developed as the reference implementation for a book I'm
currently writing about the C compiler and the low-level programming.
The book covers the vast topic with an incremental approach; in the first
chapter, readers will implement a "compiler" that accepts just a single
number as a "language", which will then gain one feature at a time in each
section of the book until the language that the compiler accepts matches
what the C11 spec specifies. I took this incremental approach from [the
paper](http://scheme2006.cs.uchicago.edu/11-ghuloum.pdf) by Abdulaziz
Ghuloum.

Each commit of this project corresponds to a section of the book. For this
purpose, not only the final state of the project but each commit was
carefully written with readability in mind. Readers should be able to learn
how a C language feature can be implemented just by reading one or a few
commits of this project. For example, this is how
[while](https://github.com/rui314/chibicc/commit/773115ab2a9c4b96f804311b95b20e9771f0190a),
[[]](https://github.com/rui314/chibicc/commit/75fbd3dd6efde12eac8225d8b5723093836170a5),
[?:](https://github.com/rui314/chibicc/commit/1d0e942fd567a35d296d0f10b7693e98b3dd037c),
and [thread-local
variable](https://github.com/rui314/chibicc/commit/79644e54cc1805e54428cde68b20d6d493b76d34)
are implemented. If you have plenty of spare time, it might be fun to read
it from the [first
commit](https://github.com/rui314/chibicc/commit/0522e2d77e3ab82d3b80a5be8dbbdc8d4180561c).

If you like this project, please consider purchasing a copy of the book
when it becomes available! 😀 I publish the source code here to give people
early access to it, because I was planing to do that anyway with a
permissive open-source license after publishing the book. If I don't charge
for the source code, it doesn't make much sense to me to keep it private. I
hope to publish the book in 2021.
You can sign up [here](https://forms.gle/sgrMWHGeGjeeEJcX7) to receive a
notification when a free chapter is available online or the book is published.

I pronounce chibicc as _chee bee cee cee_. "chibi" means "mini" or
"small" in Japanese. "cc" stands for C compiler.

## Status

chibicc supports almost all mandatory features and most optional
features of C11 as well as a few GCC language extensions.

Features that are often missing in a small compiler but supported by
chibicc include (but not limited to):

- Preprocessor
- float, double and long double (x87 80-bit floating point numbers)
- Bit-fields
- alloca()
- Variable-length arrays
- Compound literals
- Thread-local variables
- Atomic variables
- Common symbols
- Designated initializers
- L, u, U and u8 string literals
- Functions that take or return structs as values, as specified by the
  x86-64 SystemV ABI

chibicc does not support complex numbers, K&R-style function prototypes
and GCC-style inline assembly. Digraphs and trigraphs are intentionally
left out.

chibicc outputs a simple but nice error message when it finds an error in
source code.

There's no optimization pass. chibicc emits terrible code which is probably
twice or more slower than GCC's output. I have a plan to add an
optimization pass once the frontend is done.

I'm using Ubuntu 20.04 for x86-64 as a development platform. I made a
few small changes so that chibicc works on Ubuntu 18.04, Fedora 32 and
Gentoo 2.6, but portability is not my goal at this moment. It may or
may not work on systems other than Ubuntu 20.04.

## Internals

chibicc consists of the following stages:

- Tokenize: A tokenizer takes a string as an input, breaks it into a list
  of tokens and returns them.

- Preprocess: A preprocessor takes as an input a list of tokens and output
  a new list of macro-expanded tokens. It interprets preprocessor
  directives while expanding macros.

- Parse: A recursive descendent parser constructs abstract syntax trees
  from the output of the preprocessor. It also adds a type to each AST
  node.

- Codegen: A code generator emits an assembly text for given AST nodes.

## Contributing

When I find a bug in this compiler, I go back to the original commit that
introduced the bug and rewrite the commit history as if there were no such
bug from the beginning. This is an unusual way of fixing bugs, but as a
part of a book, it is important to keep every commit bug-free.

Thus, I do not take pull requests in this repo. You can send me a pull
request if you find a bug, but it is very likely that I will read your
patch and then apply that to my previous commits by rewriting history. I'll
credit your name somewhere, but your changes will be rewritten by me before
submitted to this repository.

Also, please assume that I will occasionally force-push my local repository
to this public one to rewrite history. If you clone this project and make
local commits on top of it, your changes will have to be rebased by hand
when I force-push new commits.

## Design principles

chibicc's core value is its simplicity and the reability of its source
code. To achieve this goal, I was careful not to be too clever when
writing code. Let me explain what that means.

Oftentimes, as you get used to the code base, you are tempted to
_improve_ the code using more abstractions and clever tricks.
But that kind of _improvements_ don't always improve readability for
first-time readers and can actually hurts it. I tried to avoid the
pitfall as much as possible. I wrote this code not for me but for
first-time readers.

If you take a look at the source code, you'll find a couple of
dumb-looking pieces of code. These are written intentionally that way
(but at some places I might be actually missing something,
though). Here is a few notable examples:

- The recursive descendent parser contains many similar-looking functions
  for similar-looking generative grammar rules. You might be tempted
  to _improve_ it to reduce the duplication using higher-order functions
  or macros, but I thought that that's too complicated. It's better to
  allow small duplications instead.

- chibicc doesn't try too hard to save memory. An entire input source
  file is read to memory first before the tokenizer kicks in, for example.

- Slow algorithms are fine if we know that n isn't too big.
  For example, we use a linked list as a set in the preprocessor, so
  the membership check takes O(n) where n is the size of the set.  But
  that's fine because we know n is usually very small.
  And even if n can be very big, I stick with a simple slow algorithm
  until it is proved by benchmarks that that's a bottleneck.

- Each AST node type uses only a few members of the `Node` struct members.
  Other unused `Node` members are just a waste of memory at runtime.
  We could save memory using unions, but I decided to simply put everything
  in the same struct instead. I believe the inefficiency is negligible.
  Even if it matters, we can always change the code to use unions
  at any time. I wanted to avoid premature optimization.

- chibicc always allocates heap memory using `calloc`, which is a
  variant of `malloc` that clears memory with zero. `calloc` is
  slightly slower than `malloc`, but that should be neligible.

- Last but not least, chibicc allocates memory using `calloc` but never
  calls `free`. Allocated heap memory is not freed until the process exits.
  I'm sure that this memory management policy (or lack thereof) looks
  very odd, but it makes sense for short-lived programs such as compilers.
  DMD, a compiler for the D programming language, uses the same memory
  management scheme for the same reason, for example [1].

## About the Author

I'm Rui Ueyama. I'm the creator of [8cc](https://github.com/rui314/8cc),
which is a hobby C compiler, and also the original creator of the current
version of [LLVM lld](https://lld.llvm.org) linker, which is a
production-quality linker used by various operating systems and large-scale
build systems.

## References

- [tcc](https://bellard.org/tcc/): A small C compiler written by Fabrice
  Bellard. I learned a lot from this compiler, but the design of tcc and
  chibicc are different. In particular, tcc is a one-pass compiler, while
  chibicc is a multi-pass one.

- [lcc](https://github.com/drh/lcc): Another small C compiler. The creators
  wrote a [book](https://sites.google.com/site/lccretargetablecompiler/)
  about the internals of lcc, which I found a good resource to see how a
  compiler is implemented.

- [An Incremental Approach to Compiler
  Construction](http://scheme2006.cs.uchicago.edu/11-ghuloum.pdf)

- [Rob Pike's 5 Rules of Programming](https://users.ece.utexas.edu/~adnan/pike.html)

[1] https://www.drdobbs.com/cpp/increasing-compiler-speed-by-over-75/240158941

> DMD does memory allocation in a bit of a sneaky way. Since compilers
> are short-lived programs, and speed is of the essence, DMD just
> mallocs away, and never frees.

---

If your goal is to **understand chibicc rather than just read the source code**, don't start with `main.c` and read every file from top to bottom. Follow the compiler pipeline.

### Recommended order

For **chibicc**, I recommend this order:

```text
1. main.c
      ↓
2. tokenize.c
      ↓
3. preprocess.c
      ↓
4. parse.c
      ↓
5. type.c
      ↓
6. codegen.c
      ↓
7. main.c / driver-related code
```

**don't study `main.c` in depth first**.

Use it to understand the overall entry point, then spend most of your time in `tokenize.c → preprocess.c → parse.c → type.c → codegen.c`.

---

### 1. Start with `main.c`

Start here to understand **how chibicc starts**.

Look for:

```c
int main(int argc, char **argv)
```

You want to answer:


* How are command-line arguments processed?
* How is the input `.c` file opened?
* How is tokenization invoked?
* How is preprocessing invoked?
* How is parsing invoked?
* How is type checking performed?
* How is assembly generated?
* How is the final executable produced?

Think of `main.c` as the **map of the compiler**, rather than something you need to understand line-by-line initially.

The overall flow you want to discover is approximately:

```text
C source file
     │
     ▼
 tokenize()
     │
     ▼
 preprocess()
     │
     ▼
 parse()
     │
     ▼
 AST
     │
     ▼
 add_type()
     │
     ▼
 codegen()
     │
     ▼
 assembly
     │
     ▼
 assembler / linker
     │
     ▼
 executable
```

Once you understand this pipeline, the other files become much easier.

---

# 2. `tokenize.c` — study this next

This is where I would spend your **first serious study session**.

A compiler cannot parse:

```c
int x = 10 + 20;
```

directly as characters.

It first converts the source into **tokens**:

```text
int
x
=
10
+
20
;
```

Conceptually:

```text
"int x = 10 + 20;"
        │
        ▼
┌────────────────────────┐
│      Tokenizer         │
└────────────────────────┘
        │
        ▼
[int] [x] [=] [10] [+] [20] [;]
```

Study the `Token` structure carefully.

You will encounter concepts such as:

```c
enum TokenKind
```

and:

```c
struct Token
```

Understand what information each token contains.

For example:

```text
Token
├── kind
├── next
├── location
├── length
├── value
└── ...
```

The exact structure changes as you follow the source version, so don't memorize it. Understand **why the information is necessary**.

### Important functions

Pay particular attention to functions like:

```text
tokenize()
new_token()
equal()
starts_with()
```

and the routines handling:

```text
identifiers
numbers
strings
characters
operators
punctuation
comments
```

Once you understand `tokenize.c`, you should be able to explain:

> "How does chibicc turn C source text into a linked list of tokens?"

That is your first milestone.

---

# 3. `preprocess.c`

Next study:

```text
preprocess.c
```

This is more complicated because C has a preprocessor.

For example:

```c
#define SIZE 100

int x = SIZE;
```

The compiler doesn't simply parse that directly.

Conceptually:

```text
source
  │
  ▼
tokenizer
  │
  ▼
tokens
  │
  ▼
preprocessor
  │
  ▼
expanded tokens
```

You'll encounter things such as:

```c
#define
#include
#if
#ifdef
#ifndef
#else
#endif
```

and macros.

Don't try to understand every preprocessor feature immediately.

First understand the basic idea:

```text
C source
   ↓
tokens
   ↓
macro expansion / conditional compilation
   ↓
processed tokens
```

---

# 4. `parse.c` — the most important file

This is probably the **most important file to understand** if you want to understand how a compiler turns source code into a program structure.

Consider:

```c
x = 10 + 20 * 3;
```

The parser needs to understand:

```text
        =
       / \
      x   +
         / \
       10   *
           / \
          20  3
```

This is an **AST — Abstract Syntax Tree**.

So:

```text
tokens
  │
  ▼
parser
  │
  ▼
AST
```

---

## Study the `Node` structure first

In `parse.c`, find the AST node definitions.

You will see concepts corresponding to things like:

```text
ND_ADD
ND_SUB
ND_MUL
ND_DIV

ND_ASSIGN

ND_EQ
ND_NE
ND_LT
ND_LE

ND_IF
ND_FOR
ND_WHILE

ND_RETURN

ND_FUNCALL
ND_VAR

ND_NUM
```

The exact names and organization depend on the current chibicc source, but this is the conceptual structure.

You need to understand:

> **What does each AST node represent?**

For example:

```c
10 + 20
```

becomes approximately:

```text
ND_ADD
├── lhs → 10
└── rhs → 20
```

while:

```c
x = 10;
```

becomes:

```text
ND_ASSIGN
├── lhs → x
└── rhs → 10
```

---

# 5. Learn the parser functions in a specific order

Don't read `parse.c` randomly.

Study the expression parser first.

A useful conceptual order is:

```text
primary
  ↓
unary
  ↓
mul
  ↓
add
  ↓
relational
  ↓
equality
  ↓
assign
  ↓
expr
```

This is how operator precedence gets implemented.

For example:

```c
2 + 3 * 4
```

must mean:

```text
2 + (3 * 4)
```

not:

```text
(2 + 3) * 4
```

The parser hierarchy is what establishes this.

This part of `parse.c` is **extremely worth studying carefully**.

---

# 6. `type.c`

After understanding parsing, move to:

```text
type.c
```

This is where you start seeing the difference between:

> **syntactic structure**

and

> **semantic/type information**

For example:

```c
int x;
int *p;
int a[10];
struct Foo foo;
```

The parser needs to understand these declarations, but the compiler also needs to know what types they represent.

Conceptually:

```text
AST
 │
 ▼
Type information
 │
 ├── int
 ├── pointer
 ├── array
 ├── function
 ├── struct
 └── ...
```

You'll encounter structures representing types and functions that construct and manipulate them.

Understand these concepts:

```text
Type
├── kind
├── size
├── alignment
├── base
├── array length
└── ...
```

For example:

```text
int
```

might conceptually be:

```text
Type
└── TY_INT
    ├── size = 4
    └── alignment = 4
```

while:

```c
int *
```

becomes:

```text
Type
└── TY_PTR
    └── base
         └── TY_INT
```

This becomes very important when you reach pointers and arrays.

---

# 7. `codegen.c`

**Study this after you understand AST and types.**

This is where chibicc turns the AST into machine-level instructions.

For example:

```c
return 2 + 3;
```

becomes conceptually:

```text
AST

   return
      │
      +
     / \
    2   3

      ↓

Assembly

mov $2, %rax
add $3, %rax
ret
```

This is where you start seeing:

```text
AST
 ↓
assembly
```

and things like:

```text
registers
stack
stack frames
function calls
local variables
memory addresses
loads
stores
branches
```

This file can initially look intimidating.

That's normal.

Don't start with complicated constructs like:

```c
struct
```

or:

```c
function calls
```

or:

```c
variadic functions
```

Start with:

```c
return 42;
```

then:

```c
return 2 + 3;
```

then:

```c
int x = 10;
return x;
```

then:

```c
return x + 20;
```

then:

```c
if (x)
    return 1;
return 2;
```

This lets you follow the generated assembly incrementally.

---

# The complete study path

I'd personally use this exact sequence:

```text
                 chibicc
                    │
                    ▼
               ┌─────────┐
               │ main.c  │
               └────┬────┘
                    │
                    ▼
             ┌──────────────┐
             │ tokenize.c   │
             └──────┬───────┘
                    │
                    ▼
             ┌──────────────┐
             │ preprocess.c │
             └──────┬───────┘
                    │
                    ▼
             ┌──────────────┐
             │   parse.c    │
             └──────┬───────┘
                    │
                    ▼
                  AST
                    │
                    ▼
             ┌──────────────┐
             │    type.c    │
             └──────┬───────┘
                    │
                    ▼
              Typed AST
                    │
                    ▼
             ┌──────────────┐
             │  codegen.c   │
             └──────┬───────┘
                    │
                    ▼
                Assembly
                    │
                    ▼
              Executable
```

## One important recommendation

Don't try to understand all of chibicc at once.

Take a tiny C program:

```c
int main() {
    return 2 + 3;
}
```

and follow **that exact program through chibicc**:

```text
main()
  ↓
tokenize()
  ↓
tokens
  ↓
preprocess()
  ↓
parse()
  ↓
AST
  ↓
add_type()
  ↓
codegen()
  ↓
x86-64 assembly
```

Then change it to:

```c
int main() {
    int x = 10;
    return x + 20;
}
```

Then:

```c
int main() {
    int x = 10;

    if (x > 5)
        return 1;

    return 0;
}
```

Then:

```c
int add(int a, int b) {
    return a + b;
}

int main() {
    return add(10, 20);
}
```

**That approach will teach you considerably more than reading the repository from beginning to end.**

If you're going to study chibicc seriously, I would make **`parse.c` your central file**, with `tokenize.c` first and `codegen.c` after you understand the AST.


