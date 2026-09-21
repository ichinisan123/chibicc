# 01 memo 

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