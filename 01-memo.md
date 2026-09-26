# 01 memo 

## to understand code of chibicc

[#### what is "-idirafter" flag in GCC with examples](#what-is--idirafter-flag-in-gcc-with-examples)

---

## to understand code of chibicc

[GeminiFlash38]


### 1. Recommended Study Order

#### 1-1. Entry Point and Pipeline Overview

1. Inspect `main.c` first to understand the compiler driver workflow, command-line option parsing, and the top-level compilation pipeline.

2. Follow how input source files are opened, passed to the tokenizer, processed by the parser, and finally emitted as assembly.

```c
// File: main.c
// Typical high-level flow in chibicc's driver
int main(int argc, char **argv) {
    parse_args(argc, argv);
    Token *tok = tokenize_file(base_file);
    Obj *prog = parse(tok);
    codegen(prog, out_file);
    return 0;
}
```

#### 1-2. Data Structures and Common Interfaces

1. Study `chibicc.h` before reading any translation unit to learn the core Abstract Syntax Tree (AST) node types, token definitions, and type representations.

2. Observe the definitions of `struct Node`, `struct Type`, and `struct Token`, which serve as the common contract between compiler stages.

```c
// File: chibicc.h
// Core data structures representing tokens and AST nodes
typedef enum {
    TK_IDENT,
    TK_PUNCT,
    TK_KEYWORD,
    TK_NUM,
    TK_EOF,
} TokenKind;

struct Token {
    TokenKind kind;
    Token *next;
    int64_t val;
    char *loc;
    int len;
};

typedef enum {
    ND_ADD, // +
    ND_SUB, // -
    ND_MUL, // *
    ND_DIV, // /
    ND_NUM, // Integer literal
} NodeKind;

struct Node {
    NodeKind kind;
    Node *next;
    Node *lhs;
    Node *rhs;
    int64_t val;
};
```

#### 1-3. Lexical Analysis
1. Read `tokenize.c` to see how raw source strings are converted into a linked list of tokens with precise source location tracking for diagnostics.
2. Focus on prefix matching, string interning, and basic error reporting mechanisms.

```c
// File: tokenize.c
// Simple token generation routine
static Token *tokenize(char *filename, char *p) {
    current_filename = filename;
    current_input = p;
    Token head = {};
    Token *cur = &head;

    while (*p) {
        if (isspace(*p)) {
            p++;
            continue;
        }
        if (isdigit(*p)) {
            cur = cur->next = new_token(TK_NUM, p, p);
            char *q = p;
            cur->val = strtoul(p, &p, 10);
            cur->len = p - q;
            continue;
        }
        error_at(p, "invalid token");
    }
    cur->next = new_token(TK_EOF, p, p);
    return head.next;
}
```

#### 1-4. Parsing and AST Construction
1. Examine `parse.c` to understand the recursive descent parser implementing standard C grammar productions and operator precedence.
2. Trace function declarations, statement parsing, and expression evaluation down to primary expressions.

```c
// File: parse.c
// Recursive descent handling operator precedence (+, -)
static Node *expr(Token **rest, Token *tok);

static Node *mul(Token **rest, Token *tok) {
    Node *node = primary(&tok, tok);
    for (;;) {
        if (equal(tok, "*")) {
            node = new_binary(ND_MUL, node, primary(&tok, tok->next));
            continue;
        }
        *rest = tok;
        return node;
    }
}

static Node *add(Token **rest, Token *tok) {
    Node *node = mul(&tok, tok);
    for (;;) {
        if (equal(tok, "+")) {
            node = new_binary(ND_ADD, node, mul(&tok, tok->next));
            continue;
        }
        *rest = tok;
        return node;
    }
}
```

#### 1-5. Type System Verification
1. Read `type.c` to see how types are recursively checked, inferred, and assigned to AST nodes prior to code generation.
2. Study the decay of arrays into pointers and the alignment calculations for struct members.

```c
// File: type.c
// Annotating AST nodes with concrete types
void add_type(Node *node) {
    if (!node || node->ty)
        return;

    add_type(node->lhs);
    add_type(node->rhs);

    switch (node->kind) {
    case ND_ADD:
    case ND_SUB:
    case ND_MUL:
    case ND_DIV:
        node->ty = node->lhs->ty;
        return;
    case ND_NUM:
        node->ty = ty_int;
        return;
    default:
        return;
    }
}
```

  #### 1-6. Code Generation
  1. Inspect `codegen.c` to see how the annotated AST is traversed to emit x86-64 assembly instructions compliant with the System V AMD64 ABI.
  2. Track stack frame allocation, register usage, and function call conventions.

  ```c
  // File: codegen.c
  // Stack-based expression code generation for x86-64
  static void gen_expr(Node *node) {
      switch (node->kind) {
      case ND_NUM:
          printf("  mov $%ld, %%rax\n", node->val);
          return;
      case ND_ADD:
          gen_expr(node->rhs);
          push();
          gen_expr(node->lhs);
          pop("%rdi");
          printf("  add %%rdi, %%rax\n");
          return;
      default:
          break;
      }
  }
  ```

### 2. Git History Strategy
  #### 2-1. Reading via Commit Log
  1. Clone the repository and inspect the commit log using `git log --reverse --oneline` rather than reading the master branch directly.
  2. Step through commits sequentially starting from the first commit to observe how Rui Ueyama built the compiler iteratively from a 40-line calculator to a full C11 compiler.

  ```sh
  git clone https://github.com/rui314/chibicc.git
  cd chibicc
  git checkout $(git rev-list --max-parents=0 HEAD)
  ```

# OpenAIgpt56Sol
User:
to understand code of chibicc (in github), which file do I study from?

Assistant:
### 1. Best Starting Point

1. Start with the commit history of [chibicc](https://github.com/rui314/chibicc), rather than immediately reading the latest source tree.  
2. The project is intentionally developed commit by commit, with each commit introducing a small compiler feature; read from the oldest commit forward.

### 2. Recommended File Order

1. **`chibicc.h`**  
   Learn the central data structures—`Token`, `Node`, `Type`, `Obj`, and function declarations—before following control flow.

2. **`main.c`**  
   Trace the compiler driver, command-line processing, compilation stages, assembler invocation, and linker invocation.

3. **`tokenize.c`**  
   Study how source text becomes tokens, including identifiers, punctuators, string literals, numeric literals, and source locations.

4. **`unicode.c`**  
   Read this alongside `tokenize.c` when encountering UTF-8 decoding and Unicode identifier handling; it is not necessary for the first pass.

5. **`preprocess.c`**  
   Study macro expansion, header inclusion, conditional compilation, and preprocessing directives.

6. **`parse.c`**  
   Examine how tokens become an abstract syntax tree and how declarations, statements, expressions, scopes, and initializers are represented.

7. **`type.c`**  
   Follow type assignment, arithmetic conversions, pointer operations, and compatibility rules after understanding the AST.

8. **`codegen.c`**  
   Study last because it assumes familiarity with AST nodes, types, calling conventions, stack layout, registers, and object representation.

9. **`hashmap.c`**  
   Treat this as a supporting utility; read it when investigating symbol tables, macros, or identifier lookup.

### 3. Practical First-Pass Path

1. Follow one minimal input such as:

```c
// File: example.c
int main(void) {
    return 2 + 3;
}
```

2. Trace it through this pipeline:

```text
main.c
  -> tokenize.c
  -> preprocess.c
  -> parse.c
  -> type.c
  -> codegen.c
```

3. Set breakpoints at functions such as `tokenize_file()`, `preprocess()`, `parse()`, `add_type()`, and `codegen()`; exact names can vary across historical commits.

### 4. Most Effective Study Strategy

1. Check out an early commit and compile its smallest supported program.  
2. Read the commit diff and message, run the tests, and then move to the next commit.  
3. After understanding the historical progression, return to the latest `chibicc.h` and `main.c` and trace the complete implementation.

### 5. Files to Use as Documentation

1. Read `test/*.c` after each feature because the tests provide concrete examples of the intended language behavior.  
2. Use `test/driver.sh` to understand command-line behavior and `test.sh` or the project’s build targets to observe integration testing.  
3. Consult `README.md` for the project’s design goals and important limitations.

### 6. Short Answer

1. For the **best learning experience**, begin with the repository’s **oldest commits**.  
2. For the **current codebase**, begin with `chibicc.h`, then read `main.c`, `tokenize.c`, `preprocess.c`, `parse.c`, `type.c`, and finally `codegen.c`.

# AnthropicClaude5Sonnet
User:
to understand code of chibicc (in github), which file do I study from?

Assistant:


---

### 1. Recommended Starting Point

#### 1-1. Begin with `main.c`
Start by reading `main.c` to understand the compiler's overall execution flow, from command-line argument parsing to invoking the tokenizer, parser, and code generator in sequence.

#### 1-2. Why this file first
This file acts as the orchestrator and gives you a high-level map of how each compilation phase connects, which is essential before diving into implementation details.

```c
// Typical structure found in chibicc's main.c
int main(int argc, char **argv) {
  parse_args(argc, argv);

  Token *tok = tokenize_file(input_path);
  tok = preprocess(tok);

  Obj *prog = parse(tok);

  codegen(prog);

  return 0;
}
```

### 2. Core Data Structures

#### 2-1. Study `chibicc.h`
This header file defines all shared structs (`Token`, `Node`, `Type`, `Obj`) and function prototypes used across the codebase.

#### 2-2. Key structs to focus on
Pay particular attention to `Token`, `Node`, and `Type`, as these three structures form the backbone of the tokenizer, parser, and type-checking phases respectively.

```c
// Simplified excerpts typically found in chibicc.h

typedef struct Token Token;
struct Token {
  TokenKind kind;
  Token *next;
  int64_t val;
  char *loc;
  int len;
};

typedef struct Node Node;
struct Node {
  NodeKind kind;
  Node *next;
  Type *ty;
  Node *lhs;
  Node *rhs;
};

typedef struct Type Type;
struct Type {
  TypeKind kind;
  int size;
  int align;
  Type *base;
};
```

### 3. Lexical Analysis

#### 3-1. Read `tokenize.c`
This file converts raw source text into a linked list of `Token` structs, handling identifiers, keywords, numeric literals, and string literals.

#### 3-2. Trace `tokenize()` function
Follow how the function scans character by character and produces tokens, since this pattern recurs (with variations) in most hand-written lexers.

```c
// Simplified tokenizer loop pattern
static Token *tokenize(char *p) {
  Token head = {};
  Token *cur = &head;

  while (*p) {
    if (isspace(*p)) { p++; continue; }
    if (isdigit(*p)) {
      cur = cur->next = new_num_token(p, &p);
      continue;
    }
    if (is_ident_start(*p)) {
      cur = cur->next = new_ident_token(p, &p);
      continue;
    }
    // punctuation handling omitted for brevity
    p++;
  }
  cur->next = new_token(TK_EOF, p, p);
  return head.next;
}
```

### 4. Preprocessing

#### 4-1. Review `preprocess.c`
This module implements macro expansion, `#include`, and conditional compilation directives, operating on the token stream before parsing.

#### 4-2. Focus on macro expansion logic
Understanding how object-like and function-like macros are expanded here will clarify many C language edge cases.

### 5. Parsing

#### 5-1. Study `parse.c`
This is the largest and most important file, implementing a recursive-descent parser that builds an AST (`Node` trees) directly from the token stream.

#### 5-2. Follow the grammar hierarchy
Trace functions in order of precedence, from `program()` down through `expr()`, `assign()`, `equality()`, `add()`, `mul()`, and `unary()`.

```c
// Simplified recursive-descent pattern (precedence climbing)
static Node *expr(Token **rest, Token *tok) {
  return assign(rest, tok);
}

static Node *add(Token **rest, Token *tok) {
  Node *node = mul(&tok, tok);

  for (;;) {
    if (equal(tok, "+")) {
      Node *rhs = mul(&tok, tok->next);
      node = new_binary(ND_ADD, node, rhs);
      continue;
    }
    if (equal(tok, "-")) {
      Node *rhs = mul(&tok, tok->next);
      node = new_binary(ND_SUB, node, rhs);
      continue;
    }
    *rest = tok;
    return node;
  }
}
```

### 6. Type System

#### 6-1. Examine `type.c`
This file defines type creation, type inference, and implicit conversion rules (e.g., integer promotion, pointer arithmetic sizing).

#### 6-2. Connect to `parse.c`
Notice how `add_type()` is called on AST nodes after parsing to annotate each node with a resolved `Type`, which later informs code generation.

### 7. Code Generation

#### 7-1. Finish with `codegen.c`
This file walks the type-annotated AST and emits x86-64 assembly instructions, using a stack-based evaluation model.

#### 7-2. Study the stack machine pattern
Observe how `gen_expr()` pushes intermediate results onto a virtual stack and pops them for binary operations, a common technique in simple compiler backends.

```c
// Simplified codegen pattern for binary expressions
static void gen_expr(Node *node) {
  switch (node->kind) {
  case ND_NUM:
    printf("  mov $%d, %%rax\n", node->val);
    return;
  case ND_ADD:
    gen_expr(node->rhs);
    push();
    gen_expr(node->lhs);
    pop("%rdi");
    printf("  add %%rdi, %%rax\n");
    return;
  }
}
```

### 8. Supplementary Files

#### 8-1. Check `unicode.c`
This handles UTF-8 decoding for identifiers and string literals, relevant if you're interested in character-encoding edge cases.

#### 8-2. Review `hashmap.c`
This provides the hash table implementation used internally for symbol tables and macro definitions, useful as a standalone data-structure reference.

### 9. Suggested Study Order Summary

1. `main.c` — overall pipeline
2. `chibicc.h` — shared data structures
3. `tokenize.c` — lexer
4. `preprocess.c` — macro handling
5. `parse.c` — parser and AST construction
6. `type.c` — type resolution
7. `codegen.c` — assembly generation
8. `unicode.c` / `hashmap.c` — supporting utilities

# XAIGrok46
User:
to understand code of chibicc (in github), which file do I study from?

Assistant:
### 1. Start Here
  #### 1-1. First file
  1. Open `main.c` first and locate `main`.
  2. That function is the compiler driver and shows the full pipeline in call order.
  #### 1-2. Why this file
  1. Every later file is a stage invoked from this driver, so the call graph is visible before any algorithm.
  2. After `main.c`, skim `chibicc.h` only for `Token`, `Node`, `Obj`, and `Type`.

### 2. Compilation Pipeline
  #### 2-1. Stage order
  1. Source text is converted to a token list.
  2. The preprocessor rewrites that list.
  3. The parser builds an AST of `Node` and `Obj`.
  4. The code generator emits x86-64 assembly.
  #### 2-2. Driver shape
  1. The following sketch matches the control flow in `main.c`.
  2. Treat names as the public API of each translation unit.

```c
/* File: main.c */
int main(int argc, char **argv) {
  parse_args(argc, argv);

  Token *tok = tokenize_file(input_path);
  tok = preprocess(tok);
  Obj *prog = parse(tok);

  codegen(prog, output_file);
  return 0;
}
```

### 3. File Reading Order
  #### 3-1. Sequence
  1. Read `tokenize.c` next; a recursive-descent lexer is the smallest complete stage.
  2. Read `parse.c` third; it consumes tokens and builds `Node` trees.
  3. Read `type.c` fourth; parsing and typing are tightly coupled through `add_type`.
  4. Read `codegen.c` fifth; it walks the typed AST and prints assembly.
  5. Read `preprocess.c` last among the core files; macros are orthogonal to the rest of the pipeline.
  #### 3-2. Support files
  1. Defer `hashmap.c` and `unicode.c` until a call site in tokenize or preprocess requires them.
  2. Do not start in `codegen.c`; addresses, types, and ABI details assume the AST shape.

### 4. Core Types to Memorize
  #### 4-1. Token
  1. The lexer allocates one `Token` per lexeme and links them with `next`.
  2. `loc` and `len` are a slice of the original source, not a copied string.

```c
/* File: chibicc.h */
typedef struct Token Token;
struct Token {
  TokenKind kind;
  Token *next;
  int64_t val;
  char *loc;
  int len;
  Type *ty;
  char *str;
  int line_no;
};
```

  #### 4-2. AST node
  1. Binary operators use `lhs` and `rhs`; statements and lists use `next`.
  2. `tok` points back to the source token for diagnostics.

```c
/* File: chibicc.h */
typedef struct Node Node;
struct Node {
  NodeKind kind;
  Node *next;
  Type *ty;
  Token *tok;
  Node *lhs;
  Node *rhs;
  Node *cond;
  Node *then;
  Node *els;
  Node *body;
  Obj *var;
  int64_t val;
};
```

  #### 4-3. Object and type
  1. `Obj` represents a variable, function, or tentative definition.
  2. `Type` is shared, compared by pointer after canonicalization, and carries size and alignment.

```c
/* File: chibicc.h */
typedef struct Obj Obj;
struct Obj {
  Obj *next;
  char *name;
  Type *ty;
  bool is_local;
  bool is_function;
  bool is_definition;
  int offset;
  Obj *params;
  Node *body;
  Obj *locals;
};

typedef struct Type Type;
struct Type {
  TypeKind kind;
  int size;
  int align;
  bool is_unsigned;
  Type *base;
  Token *name;
  Type *return_ty;
  Type *params;
};
```

### 5. What to Extract from Each File
  #### 5-1. tokenize.c
  1. Follow `tokenize` and the `new_token` helper from the first character to `TK_EOF`.
  2. Punctuators are matched longest-first; identifiers and keywords share one scan path.

```c
/* File: tokenize.c */
static Token *new_token(TokenKind kind, char *start, char *end) {
  Token *tok = calloc(1, sizeof(Token));
  tok->kind = kind;
  tok->loc = start;
  tok->len = end - start;
  tok->line_no = current_filename ? current_line : 1;
  return tok;
}

static bool consume_punct(char **p, char *op) {
  int len = strlen(op);
  if (strncmp(*p, op, len) != 0)
    return false;
  *p += len;
  return true;
}
```

  #### 5-2. parse.c
  1. The parser is recursive descent: `program`, `function`, `stmt`, `expr`, `assign`, `equality`, `relational`, `add`, `mul`, `unary`, `postfix`, `primary`.
  2. Each function corresponds to one grammar production and returns a `Node *`.

```c
/* File: parse.c */
static Node *expr(Token **rest, Token *tok);
static Node *assign(Token **rest, Token *tok);

static Node *new_binary(NodeKind kind, Node *lhs, Node *rhs, Token *tok) {
  Node *node = new_node(kind, tok);
  node->lhs = lhs;
  node->rhs = rhs;
  return node;
}

static Node *equality(Token **rest, Token *tok) {
  Node *node = relational(&tok, tok);
  for (;;) {
    Token *start = tok;
    if (equal(tok, "==")) {
      node = new_binary(ND_EQ, node, relational(&tok, tok->next), start);
      continue;
    }
    if (equal(tok, "!=")) {
      node = new_binary(ND_NE, node, relational(&tok, tok->next), start);
      continue;
    }
    *rest = tok;
    return node;
  }
}
```

  #### 5-3. type.c
  1. After a subtree is parsed, `add_type` fills `node->ty` from children.
  2. Usual arithmetic conversions, pointer decay, and array-to-pointer conversion live here.

```c
/* File: type.c */
void add_type(Node *node) {
  if (!node || node->ty)
    return;

  add_type(node->lhs);
  add_type(node->rhs);
  add_type(node->cond);
  add_type(node->then);
  add_type(node->els);
  add_type(node->body);

  switch (node->kind) {
  case ND_ADD:
  case ND_SUB:
  case ND_MUL:
  case ND_DIV:
    node->ty = node->lhs->ty;
    return;
  case ND_ASSIGN:
    node->ty = node->lhs->ty;
    return;
  case ND_VAR:
    node->ty = node->var->ty;
    return;
  case ND_ADDR:
    node->ty = pointer_to(node->lhs->ty);
    return;
  case ND_DEREF:
    node->ty = node->lhs->ty->base;
    return;
  default:
    return;
  }
}
```

  #### 5-4. codegen.c
  1. Functions emit a prologue, walk `node->body`, then emit an epilogue.
  2. Expression nodes leave a value in `rax`; assignments store through an address in `rdi` or a stack slot.

```c
/* File: codegen.c */
static void gen_expr(Node *node) {
  switch (node->kind) {
  case ND_NUM:
    println("  mov $%ld, %%rax", node->val);
    return;
  case ND_VAR:
    gen_addr(node);
    load(node->ty);
    return;
  case ND_ASSIGN:
    gen_addr(node->lhs);
    push();
    gen_expr(node->rhs);
    pop("%rdi");
    store(node->ty);
    return;
  case ND_ADD:
    gen_expr(node->lhs);
    push();
    gen_expr(node->rhs);
    pop("%rdi");
    println("  add %%rdi, %%rax");
    return;
  default:
    error_tok(node->tok, "invalid expression");
  }
}
```

### 6. How to Study in Practice
  #### 6-1. Single-file method
  1. Compile a 10-line C file with `chibicc -S` and keep the assembly next to the corresponding `codegen.c` path.
  2. Set a breakpoint at `tokenize`, `parse`, and `codegen` and inspect one `Token` list and one `Obj` function.
  #### 6-2. Commit-history method
  1. Clone `rui314/chibicc` and read `git log --reverse --oneline`.
  2. The repository was authored as a linear tutorial; each early commit adds one language feature.
  3. Stop at a commit, build it, and feed it a program that uses only features introduced so far.
  #### 6-3. Feature mapping
  1. Locals and `+`/`-` are the first complete path through tokenize, parse, type, and codegen.
  2. Pointers, then arrays, then `struct`, then control flow, then functions, then the preprocessor, is the natural dependency order.
  #### 6-4. Reading discipline
  1. Do not read `parse.c` linearly from line 1; jump from a grammar function to its callees.
  2. Keep `chibicc.h` enumerations `TokenKind` and `NodeKind` open while reading any `.c` file.

---

## what is "-idirafter" flag in GCC with examples.

---

`-idirafter <dir>` adds `<dir>` to the **end** of the include search path, searched only *after* all normal `-I` dirs and the standard system include paths have been checked and failed.

Contrast with `-I` (searched first, in given order) and `-isystem` (searched after `-I` but before standard system dirs).

**Search order:**

```
-I dirs → -isystem dirs → standard system dirs → -idirafter dirs
```

**Use case:** override/supplement system headers without shadowing them — e.g. providing fallback headers only used when the real header is missing.

```bash
gcc -idirafter /opt/fallback/include -c foo1.c
```

Example: cross-compiling and you have stub headers for functions the target libc lacks, but you still want the real system headers to take priority when they exist:

```
/opt/fallback/include/stdint_missing.h   # fallback only
/usr/include/stdint.h                     # real header, found first
```

```bash
gcc -idirafter /opt/fallback/include -I./include -c build.c
```

If `foo.h` exists in both `./include` and `/opt/fallback/include`, the one in `./include` wins because `-I` is searched first; `-idirafter` only kicks in when no earlier directory has the file.

**Quoting `#include` behavior:** applies to both `#include <...>` and `#include "..."` the same as other `-I`-family flags — it only affects search order, not quote-vs-angle-bracket semantics.

---


---

### 1. Using the GCC Compiler

#### 1-1. Overview of GCC

2. The `gcc` command is a compiler driver. It coordinates preprocessing, compilation, assembly, and linking by invoking programs such as the preprocessor, compiler proper, assembler, and linker.

4. A typical C build passes through four major stages:

   1. Preprocessing: source file to preprocessed source.
   2. Compilation: preprocessed source to assembly.
   3. Assembly: assembly to relocatable object code.
   4. Linking: object files and libraries to an executable.

#### 1-2. Installing GCC

1. On Debian and Ubuntu, GCC and common build tools can be installed with:

```bash
sudo apt update
sudo apt install build-essential
```

2. Confirm that GCC is available:

```bash
gcc --version
```

3. Display the target platform and GCC configuration:

```bash
gcc -v
gcc -dumpmachine
```

### 2. Basic Compilation Procedure

#### 2-1. Creating the Source File

1. Create a file named `main.c`:

```c
// File: main.c
#include <stdio.h>

#define MESSAGE "Hello from GCC on Linux!"

int main(void) {
  printf("%s\n", MESSAGE);
  return 0;
}
```

2. Using `int main(void)` explicitly states that `main()` accepts no arguments. In C, this is more precise than writing `int main()`.

#### 2-2. Compiling and Running the Program

1. Compile and link the source file:

```bash
gcc main.c -o main

./main

Hello from GCC on Linux!
```

2. The single `gcc` command normally performs all four build stages automatically:

```text
main.c → main.i → main.s → main.o → main
```

3. GCC usually uses temporary files for intermediate results and removes them after completing the build.

#### 2-3. Saving Intermediate Files

1. The `-save-temps=obj` option retains the preprocessed source, assembly, and object files:

```bash
gcc -save-temps=obj main.c -o main
```

2. The command typically creates the following files:

```text
main.i
main.s
main.o
main
```

### 3. Stage One: Preprocessing

#### 3-1. Purpose of Preprocessing

1. The preprocessor handles directives beginning with `#` before the C compiler performs syntax analysis.

2. Its main responsibilities include:

   1. Processing `#include` directives.
   2. Expanding macros defined with `#define`.
   3. Evaluating conditional-compilation directives such as `#if`, `#ifdef`, and `#ifndef`.
   4. Handling directives such as `#line` and `#pragma`.
   5. Removing comments as part of the earlier translation phases.

3. It is more accurate to say that comments are replaced by whitespace than to say that the preprocessor generally “organizes” or removes unnecessary whitespace.

#### 3-2. Generating Preprocessed Source

1. Run only the preprocessing stage with `-E`:

```bash
gcc -E main.c -o main.i
```

2. `main.i` file contains expanded header declarations and macro replacements.

3. The file is usually much larger than `main.c` because the contents referenced through `<stdio.h>` and related system headers are included.

4. A simplified excerpt may resemble the following:

```c
extern int printf(const char* restrict format, ...);

int main(void) {
  printf("%s\n", "Hello from GCC on Linux!");
  return 0;
}
```

5. This excerpt is illustrative only. Actual preprocessed output contains many declarations, implementation-specific attributes, and line-control markers.

#### 3-3. Suppressing Line Markers

1. GCC normally inserts line markers into preprocessed output so later diagnostics can refer to the original source locations.

2. Use `-P` to suppress those markers when inspecting the output manually:

```bash
gcc -E -P main.c -o main.i
```

3. The `-P` option is useful for readability, but it is generally unnecessary during a normal build.

#### 3-4. Inspecting Macro Definitions

1. The following command displays macros predefined by GCC for the current target:

```bash
gcc -dM -E - < /dev/null
```

2. To inspect predefined macros together with those introduced through a source file, use:

```bash
gcc -dM -E main.c
```

### 4. Stage Two: Compilation to Assembly

#### 4-1. Purpose of the Compilation Stage

1. During this stage, GCC parses the preprocessed C program, performs semantic checks, applies requested optimizations, and emits target-specific assembly code.

2. Assembly language is a textual representation of machine instructions. It is not itself machine code and cannot be executed directly by the CPU.

#### 4-2. Generating Assembly

1. Compile the preprocessed source into assembly with `-S`:

```bash
gcc -S main.i -o main.s
```

2. The same stage can be performed directly from the original source because GCC automatically runs the preprocessor first:

```bash
gcc -S main.c -o main.s
```

3. On an x86-64 GNU/Linux system, a possible result may look like this:

```asm
# File: main.s
	.file	"main.c"
	.text
	.section	.rodata
.LC0:
	.string	"Hello from GCC on Linux!"
	.text
	.globl	main
	.type	main, @function
main:
	pushq	%rbp
	movq	%rsp, %rbp
	leaq	.LC0(%rip), %rax
	movq	%rax, %rdi
	call	puts@PLT
	movl	$0, %eax
	popq	%rbp
	ret
```

#### 4-3. Why `printf()` May Become `puts()`

1. GCC recognizes many std-lib-funcs as built-ins and may transform certain calls even when no explicit optimization level is supplied.

2. The below is similar to `puts(MESSAGE)` when `MESSAGE` is a normal null-terminated string:

```c
printf("%s\n", MESSAGE);
```

3. GCC may emit a call to `puts@PLT` instead of `printf@PLT`.

4. This transformation is not guaranteed. To make examples more predictable, the program can call `puts()` directly:

```c
// File: main.c
#include <stdio.h>

#define MESSAGE "Hello from GCC on Linux!"

int main(void) {
    puts(MESSAGE);
    return 0;
}
```

5. GCC built-in recognition can be disabled globally with `-fno-builtin`, although this is rarely desirable for ordinary programs:

```bash
gcc -S -fno-builtin main.c -o main.s
```

#### 4-4. Generating Readable Assembly

1. Generate optimized assembly with source annotations:

```bash
gcc -O2 -S -fverbose-asm main.c -o main.s
```

2. GCC uses AT&T assembly syntax by default on x86 targets. Intel syntax can be requested with:

```bash
gcc -O2 -S -masm=intel main.c -o main.s
```

3. Assembly is architecture-dependent. An x86-64 build may contain instructions such as `pushq`, `leaq`, and `movq`, whereas an AArch64 build uses a different instruction set and register naming convention.

### 5. Stage Three: Assembly

#### 5-1. Purpose of the Assembler

1. The assembler translates assembly language into machine instructions and stores them in a relocatable object file.

2. An object file can contain:

   1. Machine-code sections such as `.text`.
   2. Read-only data such as string literals.
   3. Symbol tables.
   4. Relocation entries.
   5. Optional debugging information.

3. Although the object file contains machine instructions, it is normally not independently executable because external symbols and final addresses may still be unresolved.

#### 5-2. Generating an Object File

1. Assemble `main.s` into `main.o`:

```bash
gcc -c main.s -o main.o
```

2. GCC can also preprocess, compile, and assemble the original C file without linking:

```bash
gcc -c main.c -o main.o
```

3. The `-c` option means “compile or assemble, but do not link.”

#### 5-3. Inspecting the Object File

1. Identify the object-file format:

```bash
file main.o
```

2. On a typical x86-64 Linux system, the output may resemble:

```text
main.o: ELF 64-bit LSB relocatable, x86-64, version 1 (SYSV), not stripped
```

3. Disassemble its executable sections:

```bash
objdump -d main.o
```

4. A possible abbreviated result is:

```text
main.o: file format elf64-x86-64

Disassembly of section .text:

0000000000000000 <main>:
   0:  55                    push   %rbp
   1:  48 89 e5              mov    %rsp,%rbp
   4:  48 8d 05 00 00 00 00 lea    0x0(%rip),%rax
   b:  48 89 c7              mov    %rax,%rdi
   e:  e8 00 00 00 00       call   13 <main+0x13>
  13:  b8 00 00 00 00       mov    $0x0,%eax
  18:  5d                    pop    %rbp
  19:  c3                    ret
```

5. The hexadecimal bytes represent encoded machine instructions, but placeholder values can remain where the linker must later apply relocations.

#### 5-4. Inspecting Symbols and Relocations

1. Display the object file’s symbol table:

```bash
nm main.o
```

2. If GCC transformed `printf()` into `puts()`, the output may contain:

```text
0000000000000000 T main
                 U puts
```

3. `T` indicates a symbol defined in the text section, while `U` indicates an undefined external symbol that must be resolved during linking.

4. If the transformation did not occur, the undefined symbol may instead be `printf`.

5. Display relocation entries with:

```bash
readelf -r main.o
```

6. Display section headers with:

```bash
readelf -S main.o
```

### 6. Stage Four: Linking

#### 6-1. Purpose of Linking

1. The linker combines object files, startup code, runtime support, and libraries to produce an executable or shared library.

2. During linking, symbol references are resolved and relocation entries are applied where possible.

3. When GCC invokes the linker, it also supplies platform-specific startup files and libraries that would otherwise need to be specified manually.

#### 6-2. Creating the Executable

1. Link the object file with:

```bash
gcc main.o -o main
```

2. Using `gcc` for the final link is normally preferable to invoking `ld` directly because GCC automatically selects the required startup files, runtime support, library paths, and linker options.

#### 6-3. Inspecting the Executable

1. Inspect the output format:

```bash
file main
```

2. On a modern Linux distribution where position-independent executables are enabled by default, the result may resemble:

```text
main: ELF 64-bit LSB pie executable, x86-64, dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, not stripped
```

3. The exact output is platform-dependent. Some toolchains produce a non-PIE executable unless `-pie` is explicitly requested.

4. Inspect the ELF header directly with:

```bash
readelf -h main
```

#### 6-4. Dynamic Linking

1. Most Linux toolchains dynamically link against the system C library by default.

2. Inspect the program’s dynamic dependencies with:

```bash
ldd ./main
```

3. A possible result is:

```text
linux-vdso.so.1
libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6
/lib64/ld-linux-x86-64.so.2
```

4. For security reasons, do not use `ldd` on untrusted executables. A safer static inspection method is:

```bash
readelf -d main
```

5. The dynamic linker loads required shared libraries when the program starts. Depending on the relocation mode, some function addresses may be resolved lazily when they are first called.

6. Saying that `printf()` is fully inserted into the executable during ordinary dynamic linking is inaccurate. The executable normally records a dependency on the relevant shared library and contains the metadata required for runtime symbol resolution.

#### 6-5. Static Linking

1. Request static linking with:

```bash
gcc -static main.c -o main-static
```

2. Static linking copies required library code into the executable, producing a larger file with fewer runtime shared-library dependencies.

3. The command can fail if static versions of the required libraries are not installed.

4. Fully static programs that use glibc can still have deployment considerations involving name resolution, locales, and other system facilities. Static linking should therefore be chosen for a specific deployment reason rather than assumed to guarantee complete portability.

### 7. Building Multiple Source Files

#### 7-1. Example Program Structure

1. Create a header file named `message.h`:

```c
// File: message.h
#ifndef MESSAGE_H
#define MESSAGE_H

void print_message(void);

#endif
```

2. Create an implementation file named `message.c`:

```c
// File: message.c
#include "message.h"

#include <stdio.h>

void print_message(void)
{
    puts("Hello from multiple source files!");
}
```

3. Create `main.c`:

```c
// File: main.c
#include "message.h"

int main(void)
{
    print_message();
    return 0;
}
```

#### 7-2. Separate Compilation and Linking

1. Compile each translation unit separately:

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -c main.c -o main.o
gcc -Wall -Wextra -Wpedantic -std=c17 -c message.c -o message.o
```

2. Link the object files:

```bash
gcc main.o message.o -o main
```

3. Run the executable:

```bash
./main
```

4. The expected output is:

```text
Hello from multiple source files!
```

5. Separate compilation avoids recompiling every source file when only one implementation file changes.

### 8. Recommended GCC Options

#### 8-1. Language Standard

1. Select a specific C standard to make language assumptions explicit:

```bash
gcc -std=c17 main.c -o main
```

2. GCC also supports newer modes such as `-std=c23`, depending on the installed GCC version.

3. GNU dialects such as `-std=gnu17` enable the corresponding ISO C standard together with GNU extensions.

#### 8-2. Warning Options

1. A practical warning set is:

```bash
gcc -Wall -Wextra -Wpedantic main.c -o main
```

2. Despite its name, `-Wall` does not enable every available warning. It enables a useful collection of commonly recommended diagnostics.

3. During strict development or continuous integration, warnings can be treated as errors:

```bash
gcc -Wall -Wextra -Wpedantic -Werror main.c -o main
```

4. `-Werror` should be used carefully when compiling across different GCC versions because newly introduced warnings can break an otherwise unchanged build.

#### 8-3. Debug Information

1. Add debugging information for GDB with:

```bash
gcc -g main.c -o main
```

2. For debugging, low optimization is often easier to understand:

```bash
gcc -g -O0 main.c -o main
```

3. `-Og` enables optimizations intended to preserve a reasonable debugging experience:

```bash
gcc -g -Og main.c -o main
```

#### 8-4. Optimization Levels

1. GCC provides several optimization levels:

| Option   | Purpose                                                                                     |
|----------|---------------------------------------------------------------------------------------------|
| `-O0`    | Disables most optimizations; default and convenient for basic debugging                     |
| `-Og`    | Enables debugging-friendly optimizations                                                    |
| `-O1`    | Enables moderate optimization                                                               |
| `-O2`    | Enables broadly useful optimizations without the most aggressive size-speed tradeoffs       |
| `-O3`    | Enables more aggressive optimization, which may increase code size                          |
| `-Os`    | Optimizes for size while retaining many `-O2` optimizations                                 |
| `-Oz`    | Optimizes more aggressively for size where supported                                        |
| `-Ofast` | Enables aggressive optimizations that can relax strict language or floating-point semantics |

2. `-O2` does not guarantee that every program will run faster. Performance should be measured with a representative workload.

3. A common release command is:

```bash
gcc -std=c17 -O2 -Wall -Wextra -Wpedantic main.c -o main
```

#### 8-5. Runtime Error Detection

1. AddressSanitizer detects many invalid memory accesses:

```bash
gcc -g -O1 -fsanitize=address -fno-omit-frame-pointer main.c -o main
```

2. UndefinedBehaviorSanitizer detects several classes of undefined behavior:

```bash
gcc -g -O1 -fsanitize=undefined main.c -o main
```

3. The sanitizers can be combined:

```bash
gcc -g -O1 \
    -fsanitize=address,undefined \
    -fno-omit-frame-pointer \
    main.c \
    -o main
```

4. Sanitized builds are intended primarily for testing and debugging because they add runtime overhead.

### 9. Diagnosing Build Errors

#### 9-1. Preprocessing Errors

1. Missing header files commonly produce errors such as:

```text
fatal error: example.h: No such file or directory
```

2. Add a project include directory with `-I`:

```bash
gcc -Iinclude main.c -o main
```

#### 9-2. Compilation Errors

1. Syntax errors, type errors, and invalid declarations are normally detected during compilation:

```text
error: expected ';' before 'return'
```

2. Enable comprehensive warnings and fix the earliest diagnostic first:

```bash
gcc -std=c17 -Wall -Wextra -Wpedantic main.c -o main
```

#### 9-3. Linking Errors

1. An undefined-reference error means that a symbol was declared or referenced but no matching definition was found by the linker:

```text
undefined reference to `print_message'
```

2. Ensure that the object file or library containing the definition is included:

```bash
gcc main.o message.o -o main
```

3. Library order can matter with static libraries. A library should generally appear after the object files that reference it:

```bash
gcc main.o -lm -o main
```

4. A multiple-definition error means that the same external symbol was defined in more than one linked object file.

#### 9-4. Displaying GCC’s Internal Commands

1. Use `-v` to display detailed compiler-driver activity:

```bash
gcc -v main.c -o main
```

2. Use `-###` to print the commands GCC would execute without actually executing them:

```bash
gcc -### main.c -o main
```

3. These options are useful for identifying the assembler, linker, startup files, library paths, and target-specific options selected by GCC.

### 10. Practical Command Sets

#### 10-1. Development Build

1. Use strict diagnostics, debugging information, and sanitizers during development:

```bash
gcc -std=c17 \
    -Wall \
    -Wextra \
    -Wpedantic \
    -g \
    -Og \
    -fsanitize=address,undefined \
    -fno-omit-frame-pointer \
    main.c \
    -o main
```

#### 10-2. Release Build

1. Use measured optimization and retain useful warnings:

```bash
gcc -std=c17 \
    -Wall \
    -Wextra \
    -Wpedantic \
    -O2 \
    main.c \
    -o main
```

#### 10-3. Inspecting Every Build Stage

1. Run each stage separately:

```bash
gcc -E main.c -o main.i
gcc -S main.i -o main.s
gcc -c main.s -o main.o
gcc main.o -o main
```

2. Inspect the generated files:

```bash
file main.o
nm main.o
readelf -r main.o
objdump -d main.o
file main
readelf -h main
readelf -d main
```

### 11. Summary

1. GCC’s build pipeline consists of preprocessing, compilation, assembly, and linking.

2. Preprocessing expands directives and macros, compilation translates C into target-specific assembly, assembly creates relocatable machine code, and linking combines object files with runtime components and libraries.

3. Intermediate files such as `main.i`, `main.s`, and `main.o` are valuable for diagnosing preprocessing problems, compiler behavior, unresolved symbols, and relocation details.

4. Options such as `-std=c17`, `-Wall`, `-Wextra`, `-Wpedantic`, `-g`, `-Og`, `-O2`, and the sanitizers improve portability, diagnostics, debugging, and code quality.

5. Generated assembly, ELF metadata, dynamic dependencies, and optimization behavior are environment-dependent and should not be assumed to be identical across GCC versions or target systems.


### 2. The Linker: Role and Customization of `ld`

#### 2-1. Role of the Linker

1. On GNU/Linux systems, GCC commonly uses the GNU linker, `ld`, from GNU Binutils. The linker is not built into GCC; the `gcc` driver invokes it indirectly, often through GCC components such as `collect2`.

2. The linker performs the following operations:

   1. Combines relocatable object files and libraries.
   2. Resolves references to functions and global variables.
   3. Assigns final or load-time addresses to sections and symbols.
   4. Applies relocation information.
   5. Produces an executable or shared object, usually in ELF format on Linux.

3. In ordinary C development, use `gcc` rather than invoking `ld` directly. GCC automatically supplies startup objects, runtime libraries, library search paths, and platform-specific linker options.

#### 2-2. Building a Program from Multiple Object Files

1. Create `add.h` to declare the public interface:

```c
// File: add.h
#ifndef ADD_H
#define ADD_H

int add(int a, int b);

#endif
```

2. Create `add.c` to define the function:

```c
// File: add.c
#include "add.h"

int add(int a, int b)
{
    return a + b;
}
```

3. Create `main.c` to use the function:

```c
// File: main.c
#include "add.h"

#include <stdio.h>

int main(void)
{
    printf("Sum: %d\n", add(3, 4));
    return 0;
}
```

4. Placing the declaration in `add.h` allows both translation units to use the same declaration and reduces the risk of mismatched function types.

#### 2-3. Compiling and Linking Separately

1. Compile each source file into a relocatable object file:

```bash
gcc -std=c17 -Wall -Wextra -Wpedantic -c main.c -o main.o
gcc -std=c17 -Wall -Wextra -Wpedantic -c add.c -o add.o
```

2. Link the object files into an executable:

```bash
gcc main.o add.o -o main
```

3. Run the executable:

```bash
./main
```

4. The expected output is:

```text
Sum: 7
```

5. During the final command, the GCC driver invokes the linker and supplies the required startup code and system libraries.

#### 2-4. Inspecting the Link Command

1. Use `-v` to display detailed information about the programs invoked by GCC:

```bash
gcc -v main.o add.o -o main
```

2. Use `-###` to print the commands that GCC would execute without running them:

```bash
gcc -### main.o add.o -o main
```

3. The output varies by GCC version and platform, but it may show `collect2`, `ld`, startup objects such as `crt1.o`, and libraries such as `libc`.

4. To identify the linker selected by GCC, run:

```bash
gcc -print-prog-name=ld
```

#### 2-5. Symbol Resolution

1. The compiler records `add` as an undefined external symbol in `main.o` because its implementation is located in another translation unit:

```bash
nm main.o
```

2. The relevant output may resemble:

```text
                 U add
0000000000000000 T main
                 U printf
```

3. The `U` symbol type means “undefined in this object file,” while `T` means that the symbol is defined in the text section.

4. The `add.o` file provides the definition:

```bash
nm add.o
```

5. The output should contain an entry similar to:

```text
0000000000000000 T add
```

6. When `main.o` and `add.o` are linked, the linker resolves the reference from `main.o` to the definition in `add.o`.

#### 2-6. Relocations

1. Object files usually contain placeholders for addresses that are unknown during separate compilation.

2. Inspect relocation entries with:

```bash
readelf -r main.o
```

3. The relocation table identifies locations that the linker must adjust when sections and symbols receive their final addresses.

4. A linked position-independent executable can still contain dynamic relocations that are processed by the dynamic linker when the program is loaded:

```bash
readelf -r main
```

#### 2-7. Undefined References

1. If `add.o` is omitted from the link command, the linker cannot resolve `add`:

```bash
gcc main.o -o main
```

2. The linker will report an error similar to:

```text
undefined reference to `add'
collect2: error: ld returned 1 exit status
```

3. Fix the error by including the object file or library that defines the missing symbol:

```bash
gcc main.o add.o -o main
```

4. An `undefined reference` is a link-time error, whereas an undeclared function or incompatible declaration is normally a compile-time diagnostic.

5. Common causes of undefined references include:

   1. Omitting an object file from the link command.
   2. Omitting a required library.
   3. Misspelling a function or variable name.
   4. Declaring a symbol but never defining it.
   5. Defining a symbol with internal linkage by using `static`.
   6. Linking libraries in the wrong order.
   7. Mixing C and C++ without compatible language linkage.

#### 2-8. Multiple Definitions

1. A global function or variable with external linkage must normally have exactly one definition in the linked program.

2. If the same function is defined in both `first.c` and `second.c`, the linker may report:

```text
multiple definition of `add'
```

3. Header files should normally contain declarations rather than ordinary external definitions.

4. For example, declare a shared global variable in a header with `extern`:

```c
// File: counter.h
#ifndef COUNTER_H
#define COUNTER_H

extern int counter;

#endif
```

5. Provide exactly one definition in a source file:

```c
// File: counter.c
#include "counter.h"

int counter = 0;
```

#### 2-9. Linking Libraries

1. Libraries are typically specified using `-l`, and additional library directories are specified using `-L`.

2. For example, link the mathematical library with:

```bash
gcc main.o -lm -o main
```

3. The option `-lm` asks the linker to search for an appropriate library named `libm`, such as `libm.so` or `libm.a`.

4. With traditional static-library processing, order matters because the linker normally scans archives from left to right:

```bash
gcc main.o -lm -o main
```

5. The following ordering can fail if `main.o` requires symbols from `libm`:

```bash
gcc -lm main.o -o main
```

6. Place libraries after the object files or libraries that reference their symbols.

#### 2-10. Creating and Linking a Static Library

1. Compile `add.c` into an object file:

```bash
gcc -std=c17 -Wall -Wextra -Wpedantic -c add.c -o add.o
```

2. Create the static library `libadd.a`:

```bash
ar rcs libadd.a add.o
```

3. Compile `main.c`:

```bash
gcc -std=c17 -Wall -Wextra -Wpedantic -c main.c -o main.o
```

4. Link against `libadd.a` by specifying the current directory with `-L.` and the library name with `-ladd`:

```bash
gcc main.o -L. -ladd -o main
```

5. The linker translates `-ladd` into a search for a suitable file such as `libadd.so` or `libadd.a`.

6. To explicitly link the archive by path, use:

```bash
gcc main.o ./libadd.a -o main
```

7. Inspect the archive contents with:

```bash
ar t libadd.a
nm libadd.a
```

#### 2-11. Creating and Linking a Shared Library

1. Compile position-independent object code:

```bash
gcc -std=c17 \
    -Wall \
    -Wextra \
    -Wpedantic \
    -fPIC \
    -c add.c \
    -o add.o
```

2. Create the shared library:

```bash
gcc -shared add.o -o libadd.so
```

3. Link the application against it and record the executable’s directory as a runtime search location:

```bash
gcc main.c \
    -L. \
    -ladd \
    -Wl,-rpath,'$ORIGIN' \
    -o main
```

4. `$ORIGIN` represents the directory containing the executable. Quoting prevents the shell from expanding it before it reaches the linker.

5. Inspect the executable’s dynamic dependencies with:

```bash
readelf -d main
```

6. Run the program:

```bash
./main
```

7. For production deployment, shared-library naming, ABI compatibility, `SONAME` values, and installation paths should be managed explicitly.

#### 2-12. Passing Options to the Linker

1. GCC’s `-Wl,` option passes comma-separated arguments directly to the linker:

```bash
gcc main.o add.o -Wl,-Map=main.map -o main
```

2. This command creates the link map `main.map`, which describes section placement, loaded object files, and symbol addresses.

3. The `-Xlinker` option passes one linker argument at a time:

```bash
gcc main.o add.o \
    -Xlinker -Map \
    -Xlinker main.map \
    -o main
```

4. Common GCC options used during linking include:

| Option           | Purpose                                      |
|------------------|----------------------------------------------|
| `-Ldirectory`    | Adds a library search directory              |
| `-lname`         | Links a library named `libname`              |
| `-shared`        | Creates a shared library                     |
| `-static`        | Requests static linking                      |
| `-pie`           | Creates a position-independent executable    |
| `-no-pie`        | Creates a non-PIE executable where supported |
| `-Wl,option`     | Passes an option to the linker               |
| `-nostartfiles`  | Omits standard startup files                 |
| `-nodefaultlibs` | Omits default system libraries               |
| `-nostdlib`      | Omits startup files and default libraries    |

5. Options such as `-nostdlib` are primarily intended for specialized environments such as operating-system kernels, bootloaders, and bare-metal firmware.

#### 2-13. Linker Scripts

1. A linker script controls the memory layout of sections and symbols in the generated image.

2. The system’s default linker script can be displayed with:

```bash
ld --verbose
```

3. Custom linker scripts are especially important in embedded systems, where code, read-only data, initialized data, stacks, interrupt vectors, and memory-mapped regions must be placed at specific addresses.

4. A simplified embedded linker script may look like this:

```ld
/* File: firmware.ld */
ENTRY(_start)

MEMORY
{
    FLASH (rx)  : ORIGIN = 0x08000000, LENGTH = 256K
    RAM   (rwx) : ORIGIN = 0x20000000, LENGTH = 64K
}

SECTIONS
{
    .text :
    {
        KEEP(*(.isr_vector))
        *(.text*)
        *(.rodata*)
    } > FLASH

    .data :
    {
        _sdata = .;
        *(.data*)
        _edata = .;
    } > RAM AT > FLASH

    .bss (NOLOAD) :
    {
        _sbss = .;
        *(.bss*)
        *(COMMON)
        _ebss = .;
    } > RAM
}
```

5. The `MEMORY` block describes available memory regions, while the `SECTIONS` block maps input sections to output sections.

6. `> RAM AT > FLASH` means that `.data` executes from RAM but its initial values are stored in the firmware image in flash memory.

7. Startup code must copy initialized data from flash to RAM and clear `.bss` before calling `main()`.

8. Supply a linker script through GCC with `-T`:

```bash
arm-none-eabi-gcc \
    startup.o \
    main.o \
    -T firmware.ld \
    -Wl,-Map=firmware.map \
    -nostartfiles \
    -o firmware.elf
```

9. Memory addresses, startup files, architecture flags, runtime libraries, and section names are target-specific and must match the microcontroller and its toolchain.

---

### 3. Build Automation with Make

#### 3-1. Purpose of `make`

1. `make` reads build rules from a file normally named `Makefile` and rebuilds targets whose prerequisites are newer or missing.

2. A Makefile describes dependencies and commands; it does not inherently understand C unless the required rules are explicitly written or supplied by built-in rules.

3. For a multi-file project, a typical structure is:

```text
project/
├── Makefile
├── add.c
├── add.h
└── main.c
```

#### 3-2. Basic Multi-File Makefile

1. The following Makefile compiles each source file separately and links the resulting object files:

```makefile
# File: Makefile
CC := gcc
CPPFLAGS :=
CFLAGS := -std=c17 -Wall -Wextra -Wpedantic -g
LDFLAGS :=
LDLIBS :=

TARGET := main
SOURCES := main.c add.c
OBJECTS := $(SOURCES:.c=.o)

.PHONY: all clean

all: $(TARGET)

$(TARGET): $(OBJECTS)
	$(CC) $(LDFLAGS) $(OBJECTS) $(LDLIBS) -o $@

%.o: %.c
	$(CC) $(CPPFLAGS) $(CFLAGS) -c $< -o $@

clean:
	$(RM) $(OBJECTS) $(TARGET)
```

2. Recipe lines under targets must begin with a tab character unless `.RECIPEPREFIX` is deliberately changed.

#### 3-3. Meaning of the Variables

1. `CC` specifies the C compiler driver:

```makefile
CC := gcc
```

2. `CPPFLAGS` stores preprocessor-related options such as include paths and macro definitions:

```makefile
CPPFLAGS := -Iinclude -DDEBUG
```

3. `CFLAGS` stores C compilation options such as warning, debugging, optimization, and language-standard options:

```makefile
CFLAGS := -std=c17 -Wall -Wextra -Wpedantic -g
```

4. `LDFLAGS` stores linker options such as library search paths and linker configuration:

```makefile
LDFLAGS := -Llib
```

5. `LDLIBS` stores library dependencies, generally after the object files in the link command:

```makefile
LDLIBS := -lm
```

6. Keeping these categories separate allows users and build environments to override them more predictably.

#### 3-4. Automatic Variables

1. The pattern rule uses Make’s automatic variables:

```makefile
%.o: %.c
	$(CC) $(CPPFLAGS) $(CFLAGS) -c $< -o $@
```

2. `$@` is the current target name, such as `main.o`.

3. `$<` is the first prerequisite, such as `main.c`.

4. `$^` expands to all prerequisites with duplicate entries removed.

5. The link recipe can therefore be written more generally as:

```makefile
$(TARGET): $(OBJECTS)
	$(CC) $(LDFLAGS) $^ $(LDLIBS) -o $@
```

#### 3-5. Running the Build

1. Run the default target:

```bash
make
```

2. A first build may produce output similar to:

```text
gcc -std=c17 -Wall -Wextra -Wpedantic -g -c main.c -o main.o
gcc -std=c17 -Wall -Wextra -Wpedantic -g -c add.c -o add.o
gcc main.o add.o -o main
```

3. Run the resulting program:

```bash
./main
```

4. If no relevant files have changed, Make may report:

```text
make: Nothing to be done for 'all'.
```

5. If only `add.c` is modified, Make recompiles `add.o` and then relinks `main`; it does not need to recompile `main.c`.

#### 3-6. Cleaning Generated Files

1. Delete generated object files and the executable with:

```bash
make clean
```

2. The `.PHONY` declaration ensures that `clean` and `all` are treated as target names representing actions rather than ordinary files:

```makefile
.PHONY: all clean
```

3. GNU Make’s built-in `RM` variable normally expands to `rm -f`, so the following recipe is portable across typical Unix-like Make environments:

```makefile
clean:
	$(RM) $(OBJECTS) $(TARGET)
```

#### 3-7. Header Dependencies

1. The basic pattern rule only states that `main.o` depends on `main.c`, so it does not automatically rebuild `main.o` when `add.h` changes.

2. Dependencies can be written manually:

```makefile
main.o: main.c add.h
add.o: add.c add.h
```

3. Manual dependency lists become difficult to maintain as a project grows.

4. GCC can generate dependency files automatically using `-MMD` and `-MP`.

#### 3-8. Makefile with Automatic Dependency Generation

1. A more practical Makefile generates one `.d` dependency file for each object file:

```makefile
# File: Makefile
CC := gcc

CPPFLAGS :=
CFLAGS := -std=c17 -Wall -Wextra -Wpedantic -g -MMD -MP
LDFLAGS :=
LDLIBS :=

TARGET := main
SOURCES := main.c add.c
OBJECTS := $(SOURCES:.c=.o)
DEPS := $(OBJECTS:.o=.d)

.PHONY: all clean rebuild

all: $(TARGET)

$(TARGET): $(OBJECTS)
	$(CC) $(LDFLAGS) $^ $(LDLIBS) -o $@

%.o: %.c
	$(CC) $(CPPFLAGS) $(CFLAGS) -c $< -o $@

clean:
	$(RM) $(OBJECTS) $(DEPS) $(TARGET)

rebuild: clean all

-include $(DEPS)
```

2. `-MMD` generates dependency information for user headers while omitting most system headers.

3. `-MP` adds dummy targets for headers, preventing certain errors when a header is renamed or removed.

4. The leading hyphen in `-include $(DEPS)` tells Make not to fail when dependency files do not yet exist during the first build.

5. If `add.h` changes, both `main.o` and `add.o` are rebuilt because both source files include that header.

#### 3-9. Out-of-Tree Build Directory

1. Keeping generated files in a separate directory makes the source tree easier to manage:

```makefile
# File: Makefile
CC := gcc

CPPFLAGS :=
CFLAGS := -std=c17 -Wall -Wextra -Wpedantic -g -MMD -MP
LDFLAGS :=
LDLIBS :=

TARGET := build/main
SOURCES := main.c add.c
OBJECTS := $(SOURCES:%.c=build/%.o)
DEPS := $(OBJECTS:.o=.d)

.PHONY: all clean run rebuild

all: $(TARGET)

$(TARGET): $(OBJECTS)
	$(CC) $(LDFLAGS) $^ $(LDLIBS) -o $@

build/%.o: %.c
	@mkdir -p $(dir $@)
	$(CC) $(CPPFLAGS) $(CFLAGS) -c $< -o $@

run: $(TARGET)
	./$(TARGET)

clean:
	$(RM) -r build

rebuild: clean all

-include $(DEPS)
```

2. Build the program with:

```bash
make
```

3. Run it through Make with:

```bash
make run
```

4. Remove the entire build directory with:

```bash
make clean
```

#### 3-10. Debug and Release Configurations

1. Build modes can be selected through a Make variable:

```makefile
# File: Makefile
CC := gcc

BUILD ?= debug

CPPFLAGS :=
CFLAGS_COMMON := -std=c17 -Wall -Wextra -Wpedantic -MMD -MP
LDFLAGS :=
LDLIBS :=

ifeq ($(BUILD),release)
    CFLAGS := $(CFLAGS_COMMON) -O2 -DNDEBUG
else ifeq ($(BUILD),debug)
    CFLAGS := $(CFLAGS_COMMON) -Og -g
else
    $(error Unsupported BUILD value: $(BUILD))
endif

TARGET := build/$(BUILD)/main
SOURCES := main.c add.c
OBJECTS := $(SOURCES:%.c=build/$(BUILD)/%.o)
DEPS := $(OBJECTS:.o=.d)

.PHONY: all clean run

all: $(TARGET)

$(TARGET): $(OBJECTS)
	$(CC) $(LDFLAGS) $^ $(LDLIBS) -o $@

build/$(BUILD)/%.o: %.c
	@mkdir -p $(dir $@)
	$(CC) $(CPPFLAGS) $(CFLAGS) -c $< -o $@

run: $(TARGET)
	./$(TARGET)

clean:
	$(RM) -r build

-include $(DEPS)
```

2. Create a debugging build with:

```bash
make BUILD=debug
```

3. Create an optimized release build with:

```bash
make BUILD=release
```

4. Separate build directories prevent object files produced with different compiler options from being mixed accidentally.

#### 3-11. Overriding Make Variables

1. Variables can be overridden from the command line:

```bash
make CC=clang
```

2. Additional compiler options can be supplied when the Makefile is designed to preserve them:

```bash
make CFLAGS="-std=c17 -O2 -Wall -Wextra -Wpedantic"
```

3. Overriding `CFLAGS` replaces the Makefile’s value, so additive project-specific flags are often kept in a separate variable:

```makefile
CFLAGS ?= -O0
CFLAGS += -std=c17 -Wall -Wextra -Wpedantic
```

4. `?=` assigns a value only when the variable has not already been defined, while `+=` appends additional text.

#### 3-12. Parallel Builds

1. GNU Make can execute independent recipes in parallel:

```bash
make -j
```

2. A bounded number of parallel jobs can be specified:

```bash
make -j4
```

3. Parallel builds work correctly only when all actual dependencies are declared accurately.

4. Missing dependencies may remain unnoticed in serial builds but cause intermittent failures during parallel execution.

#### 3-13. Useful Make Commands

1. Build the first target:

```bash
make
```

2. Build a specific target:

```bash
make main
```

3. Display commands without executing them:

```bash
make -n
```

4. Force targets to be rebuilt regardless of timestamps:

```bash
make -B
```

5. Run parallel jobs:

```bash
make -j4
```

6. Delete generated files:

```bash
make clean
```

7. Rebuild from a clean state when a `rebuild` target exists:

```bash
make rebuild
```

#### 3-14. Benefits and Limitations of Makefiles

1. A correctly written Makefile provides the following benefits:

   1. Recompiles only files affected by a change.
   2. Separates compilation from linking.
   3. Records compiler and linker options.
   4. Supports repeatable development and release builds.
   5. Automates cleaning, testing, execution, and other project tasks.
   6. Supports parallel compilation.

2. Make decides whether to rebuild primarily by comparing file timestamps; it does not automatically detect changes to compiler flags or environment variables.

3. If build options change, use separate configuration directories or perform a clean rebuild:

```bash
make clean
make
```

4. Automatic `.d` files track included headers, but accurate rules are still required for generated source files, code-generation tools, external libraries, and other non-C dependencies. 

---

what output is the following?

```bash
ichi@DESKTOP-769NCVU:~/ub_c$ file main
main: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=6e7b3c9643d7778d6d80282551eafc4932faa9c6, for GNU/Linux 3.2.0, not stripped
```

---


That's the actual output of `file main` — not something to predict, it's already showing you the result. Breaking down each field:

| Field                                     | Meaning                                                                          |
|-------------------------------------------|----------------------------------------------------------------------------------|
| `ELF 64-bit LSB`                          | ELF binary, 64-bit, Little-Endian                                                |
| `pie executable`                          | Position-Independent Executable (ASLR-friendly `-fPIE`, default on modern GCC)   |
| `x86-64`                                  | Target architecture                                                              |
| `version 1 (SYSV)`                        | ELF version, System V ABI                                                        |
| `dynamically linked`                      | Needs shared libs (`.so`) at runtime, not statically linked                      |
| `interpreter /lib64/ld-linux-x86-64.so.2` | Dynamic linker/loader that resolves shared libs before `main()` runs             |
| `BuildID[sha1]=...`                       | Unique hash embedded by the linker, useful for matching debug symbols            |
| `for GNU/Linux 3.2.0`                     | Minimum kernel ABI version required                                              |
| `not stripped`                            | Symbol table still present (no `strip` applied) — debugging symbols/names intact |

If you compiled with plain `gcc main.c -o main`, `pie executable` + `dynamically linked` + `not stripped` is exactly the expected default output on a modern Ubuntu/WSL toolchain.

---

explain the following in detail with examples.

```markdown
ELF binary, 64-bit, Little-Endian
```

---

## ELF binary, 64-bit, Little-Endian

**ELF** = Executable and Linkable Format — the standard binary format for executables, shared libraries (`.so`), object files (`.o`), and core dumps on Linux/Unix systems.

### 1. ELF binary

Structure:

```
+------------------+
|   ELF Header     |  <- magic number, arch, entry point
+------------------+
| Program Headers  |  <- how to load segments into memory (used at runtime)
+------------------+
|  .text (code)    |
|  .data (init'd)  |
|  .bss (uninit'd) |
|  .rodata         |
|  ...             |
+------------------+
| Section Headers  |  <- for linking/debugging (used by linker, objdump, etc.)
+------------------+
```

Check the magic number yourself:

```bash
xxd -l 20 main
# 00000000: 7f45 4c46 0201 0100 ...
#           ^^-- 0x7F 'E' 'L' 'F'
```

`0x7F 45 4C 46` is the ELF magic — how `file` recognizes it in the first place.

### 2. 64-bit

Refers to the **address/word size** the binary targets — pointers are 8 bytes, registers used are 64-bit (rax, rbx, ...).

Byte 5 of the ELF header (`EI_CLASS`) encodes this:
- `01` = ELFCLASS32 (32-bit)
- `02` = ELFCLASS64 (64-bit) ← matches the `02` above

```bash
readelf -h main | grep Class
# Class:                             ELF64
```

Compare with a 32-bit build:

```bash
gcc -m32 main.c -o main32
file main32
# main32: ELF 32-bit LSB pie executable, Intel 80386, ...
```

### 3. Little-Endian (LSB)

Byte order for multi-byte values. **LSB** = Least Significant Byte stored first (at the lowest memory address).

Example: the 32-bit integer `0x12345678` in memory:

| Endianness   | Byte order in memory (low→high addr) |
|--------------|--------------------------------------|
| Little (LSB) | `78 56 34 12`                        |
| Big (MSB)    | `12 34 56 78`                        |

Verify with a small C program:

```c
#include <stdio.h>

int main() {
  int x = 0x12345678;
  unsigned char* p = (unsigned char*)&x;
  for (int i = 0; i < 4; i++) printf("%02x ", p[i]);
}
// Output on x86-64: 78 56 34 12
```

x86/x86-64 is natively little-endian, so this is expected on virtually all desktop/server Linux. ARM can run either mode, but Linux ARM distros are little-endian by convention (`aarch64`, not `aarch64_be`).

Byte 6 of the ELF header (`EI_DATA`) encodes this:
- `01` = ELFDATA2LSB ← the `01` above
- `02` = ELFDATA2MSB

```bash
readelf -h main | grep -i data
# Data:                              2's complement, little endian
```