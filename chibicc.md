# chibicc.h

## code explaination

[OpenAIgpt56Sol]

### 1. Purpose of `chibicc.h`

### 2. POSIX feature selection

```c
#define _POSIX_C_SOURCE 200809L
```

1. `_POSIX_C_SOURCE` requests interfaces defined by POSIX.1-2008 from the system headers. It must be defined before including headers such as `<unistd.h>` and `<sys/wait.h>`.

2. This makes POSIX functions available while helping the compiler detect accidental dependencies on non-POSIX extensions.

3. The value `200809L` corresponds to the 2008 edition of POSIX, including later corrections recognized by the host C library.

### 3. Included system headers

#### 3-1. Standard C headers

1. Headers such as `<stdint.h>`, `<stdbool.h>`, and `<stdlib.h>` provide fixed-width integers, Boolean values, memory allocation, and process-termination utilities.

2. `<stdnoreturn.h>` provides `noreturn`, which marks functions that never return normally.

```c
noreturn void error(char *fmt, ...);
```

3. `<stdarg.h>` is needed by variadic functions such as `error()` and `format()`.

```c
char *format(char *fmt, ...);
```

#### 3-2. POSIX headers

1. `<unistd.h>` provides POSIX interfaces such as `read()`, `write()`, `close()`, and `fork()`.

2. `<sys/wait.h>` provides process-waiting facilities such as `waitpid()` and macros such as `WIFEXITED()`.

3. `<glob.h>` provides pathname pattern expansion, while `<libgen.h>` provides pathname utilities such as `dirname()` and `basename()`.

### 4. `MAX` and `MIN` macros

```c
#define MAX(x, y) ((x) < (y) ? (y) : (x))
#define MIN(x, y) ((x) < (y) ? (x) : (y))
```

1. `MAX(x, y)` evaluates to the greater argument, while `MIN(x, y)` evaluates to the smaller argument.

```c
int upper = MAX(10, 20);  // 20
int lower = MIN(10, 20);  // 10
```

2. The parentheses preserve operator precedence when expressions are passed as arguments.

```c
int value = MAX(a + 1, b * 2);
```

3. These macros may evaluate an argument more than once, so expressions with side effects are unsafe.

```c
int x = 1;
int y = MAX(x++, 10);  // x may be incremented once or twice
```

4. In this macro, the condition evaluates `x++`, and one selected branch can evaluate it again. `chibicc` can safely use the macro when its arguments are ordinary values without side effects.

🟢 When Double Evaluation Occurs

1. If `x` starts at `10`, the condition is false:

```c
int x = 10;
int y = MAX(x++, 10);
```

2. The expansion evaluates `x++` in both the condition and selected branch:

```c
int y = ((x++) < (10) ? (10) : (x++));
```

3. The result is `x == 12` and `y == 11`. This is not undefined behavior because evaluation of the condition is sequenced before evaluation of the selected conditional branch.

🟢 Better Explanation

1. The example should avoid claiming that the same fixed execution may arbitrarily increment `x` once or twice. The number of evaluations depends deterministically on the condition.

```c
int x = 1;
int y = MAX(x++, 10);  // x++ is evaluated once; x becomes 2

int a = 10;
int z = MAX(a++, 10);  // a++ is evaluated twice; a becomes 12
```

2. The precise rule is:

- `MAX(x, y)` evaluates `x` twice when `x < y` is false.
- `MAX(x, y)` evaluates `y` twice when `x < y` is true.
- `MIN(x, y)` evaluates `x` twice when `x < y` is true.
- `MIN(x, y)` evaluates `y` twice when `x < y` is false.

🟢 Corrected Text

1. A more accurate version is:

> These macros can evaluate one argument twice. Therefore, arguments containing increments, assignments, function calls, volatile accesses, or other side effects are unsafe. Which argument is evaluated twice depends on the comparison result.

```c
int x = 10;
int y = MAX(x++, 10);  // x++ is evaluated twice
```

2. For `chibicc`, the macros are safe only when each argument can be evaluated repeatedly without changing program behavior. “Ordinary values without side effects” is broadly correct, but “side-effect-free expressions” is more precise.

### 5. Compatibility definition for `__attribute__`

```c
// File: chibicc.h
#ifndef __GNUC__
# define __attribute__(x)
#endif
```

1. GCC and compatible compilers support `__attribute__((...))` to attach additional semantic information to declarations.

2. If the host compiler does not define `__GNUC__`, this code replaces `__attribute__(x)` with nothing so the declarations remain syntactically valid.

3. For GCC or Clang, this declaration enables compile-time checking of `printf`-style arguments:

```c
char *format(char *fmt, ...)
  __attribute__((format(printf, 1, 2)));
```

4. The values `1` and `2` mean that the format string is argument 1 and the variadic arguments begin at argument 2.

```c
format("%d", 42);       // Correct
format("%d", "text");   // Compiler can diagnose a type mismatch
```

5. Defining reserved identifiers such as `__attribute__` is generally not recommended in application code, but it is used here as a small portability mechanism for the compiler project.

🟢 What this code does

```c
#ifndef __GNUC__
# define __attribute__(x)
#endif
```

This is a portability shim. It lets code sprinkled with GCC's `__attribute__((...))` syntax compile cleanly on non-GCC-compatible compilers, by making the attribute a no-op there.

🟢 Why it's needed

`__attribute__` is a GCC/Clang extension, not standard C. If you compile chibicc's source with, say, MSVC, the parser would choke on `__attribute__((format(printf, 1, 2)))` since MSVC doesn't know that syntax.

`__GNUC__` is a predefined macro that GCC (and GCC-compatible compilers like Clang) always define. So:

```c
#ifndef __GNUC__
# define __attribute__(x)
#endif
```

reads as: "if we're **not** on a GCC-like compiler, redefine `__attribute__(x)` as an empty macro that swallows its argument and expands to nothing."

🟢 Expansion trace

On GCC/Clang (`__GNUC__` defined): the macro is *not* defined, so `__attribute__((...))` passes through untouched to the compiler's native handling.

On MSVC (`__GNUC__` undefined):

```c
char *format(char *fmt, ...)
  __attribute__((format(printf, 1, 2)));
```

expands to:

```c
char *format(char *fmt, ...)
  ;
```

The attribute vanishes entirely — the declaration survives, just without the extra semantic annotation.

🟢 The `format(printf, 1, 2)` attribute specifically

This tells GCC/Clang: "this function behaves like `printf` — validate the call site against the format string."

- `1` = the position of the format-string parameter (`fmt` is argument 1; note `self`/implicit receiver isn't counted for free functions, but *is* counted as 1 for non-static member-like cases — for a plain function, counting starts at 1).
- `2` = the position where variadic arguments begin.

```c
char *format(char *fmt, ...)
  __attribute__((format(printf, 1, 2)));
```

Given this, the compiler statically checks every call site:

```c
format("%d", 42);      // OK — %d expects int, got int
format("%d", "text");  // warning: format '%d' expects 'int', but argument has type 'char *'
format("%s %d", "x");  // warning: too few arguments for format
```

This is purely a diagnostic aid — it costs nothing at runtime and catches a common class of bugs (format string / argument mismatches) at compile time, the same way it does for real `printf`.

🟢 Why chibicc does this at all

chibicc is a C compiler implemented in C, and it wants to:

1. Use GCC-specific diagnostics (like `format`) to keep its own source bug-free when built with GCC/Clang during development.

2. Still be buildable with compilers lacking that extension, without `#ifdef`-guarding every single attribute usage at every call site.

Centralizing the fallback in one macro definition in `chibicc.h` means the rest of the codebase can use `__attribute__(...)` freely without scattering compiler-detection logic everywhere.

🟢 Note on the reserved-identifier caveat (point 5)

Identifiers starting with `__` (double underscore) or `_` followed by a capital letter are **reserved for the implementation** per the C standard (C11 §7.1.3). Application code redefining them is technically undefined behavior / not portable.

```c
#define __attribute__(x)  // reserved identifier — implementation territory
```

It's acceptable here specifically because:
- chibicc *is* a compiler implementation, operating close to that boundary.
- The macro is a narrowly scoped fallback (empty-expansion only), not a redefinition that changes meaning on compilers where it's already defined.
- It's guarded by `#ifndef __GNUC__`, so it never conflicts with the real GCC/Clang definition.

In ordinary application code, you'd avoid this and instead do something like:

```c
#if defined(__GNUC__) || defined(__clang__)
# define PRINTF_LIKE(fmt_idx, va_idx) __attribute__((format(printf, fmt_idx, va_idx)))
#else
# define PRINTF_LIKE(fmt_idx, va_idx)
#endif

char *format(char *fmt, ...) PRINTF_LIKE(1, 2);
```

— defining your *own* macro name instead of shadowing the reserved `__attribute__` identifier itself.

---

### 6. Forward declarations

```c
// File: chibicc.h
typedef struct Type Type;
typedef struct Node Node;
typedef struct Member Member;
typedef struct Relocation Relocation;
typedef struct Hideset Hideset;
```

1. These declarations introduce structure types without defining their fields. Code can declare pointers to incomplete types without knowing the full structure layout.

```c
Type *ty;
Node *node;
Hideset *hideset;
```

2. Forward declarations reduce ordering constraints and circular dependencies among compiler data structures.

3. The complete definitions appear later in the header or in another relevant section.

4. This form also allows `Type` to be used instead of repeatedly writing `struct Type`.

### 7. Dynamic string array

```c
// File: chibicc.h
typedef struct {
  char **data;
  int capacity;
  int len;
} StringArray;
```

1. `StringArray` is a dynamically growable array of string pointers.

2. `data` points to the allocated array, `len` is the number of occupied entries, and `capacity` is the number of currently allocated entries.

```text
data
  |
  v
+---------+---------+---------+---------+
| char *  | char *  | unused  | unused  |
+---------+---------+---------+---------+
     len = 2, capacity = 4
```

3. The associated insertion function is:

```c
void strarray_push(StringArray *arr, char *s);
```

4. A typical implementation expands the allocation when `len == capacity`, then stores the new pointer and increments `len`.

```c
StringArray arr = {};
strarray_push(&arr, "input.c");
strarray_push(&arr, "output.o");
```

5. The structure stores pointers rather than copies of the strings, so ownership and lifetime depend on how `strarray_push()` is implemented and how its callers allocate strings.

### 8. Formatted string creation

```c
char* format(char* fmt, ...) __attribute__((format(printf, 1, 2)));
```

This is a **function declaration** with a **GCC attribute** attached, telling the compiler to type-check calls to `format()` as if it were `printf`.

| Part                                    | Meaning                                                                              |
|-----------------------------------------|--------------------------------------------------------------------------------------|
| `char* format(char* fmt, ...)`          | A variadic function — `fmt` is the format string, `...` is the rest of the arguments |
| `__attribute__((format(printf, 1, 2)))` | GCC extension: validate this function's calls like `printf`                          |
| `printf`                                | Which format-checking convention to use (`printf`, `scanf`, `strftime`, etc.)        |
| `1`                                     | Position of the format-string argument (`fmt` is argument #1)                        |
| `2`                                     | Position where the variadic arguments start (argument #2 onward)                     |

🟢 Why the `1, 2`

Argument positions are 1-indexed, counting from the first parameter:

```c
char* format(char* fmt, ...);
//           ^1         ^2 (variadic args start here)
```

Compare with a member-like function taking an extra leading argument:

```c
void log_msg(int level, char* fmt, ...) __attribute__((format(printf, 2, 3)));
//                      ^2  fmt is arg 2, varargs start at arg 3
```

🟢 What it catches

```c
#include <stdarg.h>
#include <stdio.h>

char* format(char* fmt, ...) __attribute__((format(printf, 1, 2)));

char* format(char* fmt, ...) {
  static char buf[256];
  va_list ap;
  va_start(ap, fmt);
  vsnprintf(buf, sizeof(buf), fmt, ap);
  va_end(ap);
  return buf;
}

int main(void) {
  format("%d apples", 5);  // OK
  format("%s", "hello");   // OK
  format("%d", "oops");    // warning: format '%d' expects int, got char*
  format("%d and %d", 1);  // warning: too few arguments for format
  format("%d", 1, 2);
  // warning: too many arguments for format (some compilers)
}
```

Compile with warnings on:

```sh
gcc -Wall -Wformat main.c
```

```
main.c: In function 'main':
main.c:16:12: warning: format '%d' expects argument of type 'int', but argument 2 has type 'char *' [-Wformat=]
main.c:17:5: warning: too few arguments for format [-Wformat=]
```

🟢 Why this matters

Without the attribute, `format()` is just an ordinary variadic function

the compiler has no idea it's `printf`-like, so mismatched types slip through silently until runtime (often a crash or garbage output).

With the attribute, GCC/Clang statically catches these bugs at compile time, same as it does for the real `printf`.

🟢 Portability note

Since `__attribute__` is GCC/Clang-only, real-world code guards it (as seen in chibicc's `chibicc.h`):

```c
#ifndef __GNUC__
# define __attribute__(x)
#endif
```

so the declaration still compiles (minus the checking) on compilers like MSVC.

🟢 What `__attribute__` is

A GCC extension for attaching compiler metadata to declarations — functions, variables, types.

Not standard C, but widely used in systems code (Linux kernel, glibc, chibicc, etc.).

Syntax: `__attribute__((attr1, attr2, ...))` — double parens, comma-separated list.

🟢 Common attributes

🟠 `unused` — suppress unused warnings

```c
void handler(int sig __attribute__((unused))) {
    // sig intentionally unused
}
```

```
-Wunused-parameter
```

Useful for callback signatures where you don't need every parameter.

🟠 `noreturn` — function never returns

```c
void die(const char *msg) __attribute__((noreturn));

void die(const char *msg) {
    fprintf(stderr, "%s\n", msg);
    exit(1);
}
```

Lets the compiler skip "missing return" warnings and enables better dead-code elimination after calls to `die()`.

🟠 `packed` — remove struct padding

```c
struct __attribute__((packed)) Header {
    uint8_t  type;
    uint32_t length;
};
// Without packed: sizeof == 8 (padding after 'type')
// With packed:    sizeof == 5
```

Critical for binary protocols / file formats where layout must match a spec exactly.

🟠 `aligned` — force alignment

```c
int buf[4] __attribute__((aligned(16)));
```

Common for SIMD (SSE/AVX needs 16/32-byte aligned data).

🟠 `always_inline` — force inlining

```c
static inline int add(int a, int b) __attribute__((always_inline));
```

Overrides the compiler's inlining heuristics (`inline` is only a hint; this is closer to a mandate).

🟠 `constructor` / `destructor` — run before/after `main()`

```c
__attribute__((constructor))
void init(void) {
    printf("runs before main\n");
}

__attribute__((destructor))
void cleanup(void) {
    printf("runs after main returns\n");
}

int main(void) {
    printf("main\n");
    return 0;
}
```

```
runs before main
main
runs after main returns
```

Used for library init (e.g., registering plugins at load time).

🟠 `deprecated` — compile-time warning on use

```c
int old_api(void) __attribute__((deprecated("use new_api() instead")));

int main(void) {
    old_api();  // warning: 'old_api' is deprecated: use new_api() instead
}
```

🟠 `warn_unused_result` — force callers to check return value

```c
int must_check(void) __attribute__((warn_unused_result));

int main(void) {
    must_check();  // warning: ignoring return value
}
```

glibc uses this on `malloc`-adjacent functions and `write()` to catch silently-dropped errors.

🟠 `weak` — allow symbol override at link time

```c
void hook(void) __attribute__((weak));

void hook(void) {
    // default no-op implementation
}
```

Another translation unit can define a strong `hook()` that overrides this one — used for optional plugin hooks.

🟠 `section` — place symbol in a custom linker section

```c
const char version[] __attribute__((section(".version_info"))) = "1.0.0";
```

Used in embedded/kernel code to control memory layout precisely.

🟠 `cleanup` — RAII-like automatic cleanup (GCC/Clang)

```c
void free_str(char **p) { free(*p); }

void example(void) {
    char *buf __attribute__((cleanup(free_str))) = strdup("hello");
    // buf automatically freed when it goes out of scope
}
```

🟢 Combining multiple attributes

```c
int fast_add(int a, int b) __attribute__((const, always_inline));
```

`const` here means "pure function with no side effects, return depends only on args" — enables aggressive optimization/CSE.

🟢 Placement matters

```c
__attribute__((noreturn)) void die(void);   // before declaration — applies to function
void die(void) __attribute__((noreturn));   // after declaration — also valid, more common
```

Both are accepted; placement after the declarator is the conventional style (matches the `format` example from chibicc).

🟢 Portability pattern (recap)

```c
#if defined(__GNUC__) || defined(__clang__)
# define NORETURN __attribute__((noreturn))
#else
# define NORETURN
#endif

NORETURN void die(const char *msg);
```

Wrapping each attribute in its own macro (rather than blanket no-op'ing `__attribute__` itself) is the more disciplined approach for portable libraries — chibicc's simpler `#define __attribute__(x)` works because chibicc doesn't need MSVC support to *behave* correctly, only to *compile*.


---

### 9. Token kinds

```c
// File: chibicc.h
typedef enum {
  TK_IDENT,
  TK_PUNCT,
  TK_KEYWORD,
  TK_STR,
  TK_NUM,
  TK_PP_NUM,
  TK_EOF,
} TokenKind;
```

1. `TokenKind` classifies each lexical token generated from C source code.

#### 9-1. `TK_IDENT`

1. `TK_IDENT` represents identifiers such as variable names, function names, typedef names, and macro names.

```c
counter
printf
MyType
```

2. Whether an identifier is a typedef name is usually determined later by the parser’s scope information.

#### 9-2. `TK_PUNCT`

1. `TK_PUNCT` represents punctuators and operators.

```c
+
->
==
{
;
```

2. The exact punctuator is identified by the source text referenced by `loc` and `len`.

#### 9-3. `TK_KEYWORD`

1. `TK_KEYWORD` represents reserved C keywords.

```c
int
return
struct
sizeof
```

2. A tokenizer may initially recognize these as identifier-shaped tokens and later reclassify them as keywords.

#### 9-4. `TK_STR`

1. `TK_STR` represents string literals.

```c
"hello"
"line\n"
u8"text"
```

2. Its decoded value is stored in `str`, while its array type is stored in `ty`.

#### 9-5. `TK_NUM`

1. `TK_NUM` represents a numeric constant whose value and type have already been determined.

```c
123
0xffU
3.14
1.0L
```

2. Integer values use `val`, floating-point values use `fval`, and the selected C type is stored in `ty`.

#### 9-6. `TK_PP_NUM`

1. `TK_PP_NUM` represents a preprocessing number, which follows the preprocessor’s more permissive lexical grammar.

```c
123
1e+10
0x1p+2
```

2. Preprocessing numbers are converted to ordinary numeric tokens only after macro expansion and preprocessing.

#### 9-7. `TK_EOF`

1. `TK_EOF` marks the end of a token stream.

2. It allows loops to stop without using a null pointer as the end marker.

```c
for (Token *tok = first; tok->kind != TK_EOF; tok = tok->next) {
  /* Process tok. */
}
```

### 10. Source-file representation

```c
// File: chibicc.h
typedef struct {
  char *name;
  int file_no;
  char *contents;

  char *display_name;
  int line_delta;
} File;
```

1. `File` represents one source file loaded by the compiler.

#### 10-1. `name`

1. `name` stores the file’s actual pathname.

```text
src/main.c
/usr/include/stdio.h
```

2. It is used for locating and identifying the physical input file.

#### 10-2. `file_no`

1. `file_no` is a compiler-assigned numeric identifier.

2. It can be emitted into assembler debugging directives so external tools can associate generated instructions with source files.

#### 10-3. `contents`

1. `contents` points to the complete source text held in memory.

2. Token locations can point directly into this buffer, avoiding a separate allocation for every token’s original spelling.

```text
contents: "int answer = 42;\n"
           ^^^
           loc
```

#### 10-4. `display_name`

1. `display_name` is the filename shown in diagnostics and generated source-location metadata.

2. It may differ from `name` because the C `#line` directive can replace the logical filename.

```c
#line 100 "generated.c"
```

#### 10-5. `line_delta`

1. `line_delta` adjusts physical source line numbers to logical line numbers specified by a `#line` directive.

2. Conceptually, the displayed line is calculated using an adjustment similar to:

[
\text{displayed line} = \text{physical line} + \text{line delta}
]

### 11. Token structure

```c
// File: chibicc.h
typedef struct Token Token;

struct Token {
  TokenKind kind;
  Token *next;
  int64_t val;
  long double fval;
  char *loc;
  int len;
  Type *ty;
  char *str;

  File *file;
  char *filename;
  int line_no;
  int line_delta;
  bool at_bol;
  bool has_space;
  Hideset *hideset;
  Token *origin;
};
```

1. `Token` stores both lexical information and preprocessing metadata.

#### 11-1. `kind`

1. `kind` specifies how the token should be interpreted.

```c
if (tok->kind == TK_IDENT) {
  /* Parse an identifier. */
}
```

#### 11-2. `next`

1. `next` links tokens into a singly linked list.

```text
Token A -> Token B -> Token C -> TK_EOF
```

2. This representation makes macro expansion and token insertion easier than modifying a contiguous array.

#### 11-3. `val`

1. `val` stores an integer constant’s value.

```c
if (tok->kind == TK_NUM) {
  printf("%lld\n", (long long)tok->val);
}
```

2. The exact signedness and width are represented separately by `tok->ty`.

#### 11-4. `fval`

1. `fval` stores the value of a floating-point constant.

2. `long double` provides enough host-side precision to represent the supported floating constant categories during compilation.

#### 11-5. `loc` and `len`

1. `loc` points to the beginning of the token spelling, and `len` specifies its length.

2. The spelling does not need to be null-terminated.

```c
printf("%.*s\n", tok->len, tok->loc);
```

3. For the source below, the token `count` has `loc` pointing at `c` and `len == 5`.

```c
int count = 10;
```

4. This pointer-and-length representation avoids allocating a separate string for every identifier and punctuator.

#### 11-6. `ty`

1. `ty` stores the token’s resolved C type when the token is a number or string literal.

2. Example mappings include:

```text
42       -> int
42UL     -> unsigned long
3.14     -> double
3.14f    -> float
"abc"    -> char[4]
L"abc"   -> wchar_t[4]
```

#### 11-7. `str`

1. `str` stores decoded string-literal contents and includes the terminating null character.

2. For `"A\n"`, the source spelling and decoded memory contents differ:

```text
Source spelling:  "A\n"
Stored bytes:     'A', '\n', '\0'
```

### 12. Token source-location fields

#### 12-1. `file`

1. `file` points to the `File` object containing the token.

2. It provides access to the source buffer and physical file information.

#### 12-2. `filename`

1. `filename` stores the filename associated with the token for diagnostics.

2. It can reflect logical source-location changes introduced by preprocessing directives.

#### 12-3. `line_no`

1. `line_no` records the token’s physical or initially calculated source line number.

2. It allows diagnostics such as the following:

```text
example.c:15: invalid expression
```

#### 12-4. `line_delta`

1. `line_delta` carries the line adjustment applicable to this particular token.

2. Storing it in the token preserves the correct logical location even when different parts of one token stream have different `#line` adjustments.

### 13. Whitespace and preprocessing metadata

#### 13-1. `at_bol`

1. `at_bol` means “at beginning of line.” It is essential because a preprocessing directive begins with `#` at the start of a logical line.

```c
#define SIZE 100
```

2. A `#` that is not in the required beginning-of-line position must not be interpreted as the start of a directive.

#### 13-2. `has_space`

1. `has_space` indicates that whitespace appeared before the token.

2. This is important when reconstructing preprocessed output and handling function-like macro invocation rules.

```c
MACRO(arg)
MACRO (arg)
```

3. It also helps prevent accidentally concatenating tokens while printing them.

#### 13-3. `hideset`

1. `hideset` records macros that must not be expanded again for a token.

2. It prevents infinite recursive expansion.

```c
#define A B
#define B A
```

3. Without a hideset or equivalent mechanism, expanding `A` could alternate indefinitely between `A` and `B`.

4. The technique follows the standard conceptual model used for conforming macro replacement: tokens produced by a macro expansion carry information suppressing immediate re-expansion of applicable macro names.

#### 13-4. `origin`

1. `origin` points to the original token from which a macro-expanded token was produced.

2. It preserves diagnostic traceability after replacement tokens have been copied or synthesized.

```c
#define VALUE invalid_name
int x = VALUE;
```

3. If an error involves `invalid_name`, `origin` helps associate the generated token with the macro invocation in the source.

### 14. Diagnostic functions

```c
noreturn void error(char *fmt, ...)
  __attribute__((format(printf, 1, 2)));

noreturn void error_at(char *loc, char *fmt, ...)
  __attribute__((format(printf, 2, 3)));

noreturn void error_tok(Token *tok, char *fmt, ...)
  __attribute__((format(printf, 2, 3)));

void warn_tok(Token *tok, char *fmt, ...)
  __attribute__((format(printf, 2, 3)));
```

#### 14-1. `error()`

1. `error()` reports a general fatal error and terminates compilation.

```c
error("cannot open %s: %s", path, strerror(errno));
```

2. `noreturn` tells the compiler that execution cannot continue past the call.

#### 14-2. `error_at()`

1. `error_at()` reports a fatal error at a character location in the source buffer.

```c
error_at(input, "invalid token");
```

2. The format string is argument 2, so the attribute uses `format(printf, 2, 3)`.

#### 14-3. `error_tok()`

1. `error_tok()` reports a fatal error associated with a complete token.

```c
error_tok(tok, "expected an expression");
```

2. It can use `tok->file`, `tok->line_no`, `tok->loc`, and `tok->len` to generate a source-oriented diagnostic.

#### 14-4. `warn_tok()`

1. `warn_tok()` reports a token-specific warning but does not terminate compilation.

```c
warn_tok(tok, "unused variable");
```

2. It has no `noreturn` qualifier because execution continues after the warning.

### 15. Token matching helpers

#### 15-1. `equal()`

```c
bool equal(Token *tok, char *op);
```

1. `equal()` checks whether a token’s spelling matches a specified string.

```c
if (equal(tok, "return")) {
  /* Parse a return statement. */
}
```

2. It typically compares both the token length and bytes.

```c
tok->len == strlen(op) &&
memcmp(tok->loc, op, tok->len) == 0
```

#### 15-2. `skip()`

```c
Token *skip(Token *tok, char *op);
```

1. `skip()` verifies that the current token matches `op` and returns the next token.

```c
tok = skip(tok, ";");
```

2. If the token does not match, it reports an error instead of returning a status value.

#### 15-3. `consume()`

```c
bool consume(Token **rest, Token *tok, char *str);
```

1. `consume()` conditionally matches a token without producing an error.

```c
Token *next;

if (consume(&next, tok, ",")) {
  tok = next;
  /* A comma was present. */
}
```

2. On success, `*rest` receives `tok->next`; on failure, the function returns `false`.

3. The distinction is that `skip()` represents mandatory syntax, while `consume()` represents optional syntax.

### 16. Preprocessing-number conversion

```c
void convert_pp_tokens(Token *tok);
```

1. The preprocessor initially recognizes numeric-looking text as `TK_PP_NUM`.

2. After macro expansion, `convert_pp_tokens()` converts preprocessing numbers into ordinary `TK_NUM` tokens and determines their values and types.

```c
#define EXPONENT e+3
1 EXPONENT
```

3. Delaying conversion is necessary because preprocessing tokenization and C numeric-constant parsing follow different grammar stages.

4. During conversion, fields such as `val`, `fval`, and `ty` are populated.

### 17. Input-file access

```c
File **get_input_files(void);
```

1. `get_input_files()` returns the collection of files read during compilation.

2. The collection can include the main source file and headers introduced by `#include`.

3. A null-terminated `File **` representation allows iteration similar to:

```c
File **files = get_input_files();

for (int i = 0; files[i]; i++) {
  printf("%s\n", files[i]->name);
}
```

### 18. File-object creation

```c
File *new_file(char *name, int file_no, char *contents);
```

1. `new_file()` constructs and initializes a `File` object from a pathname, numeric identifier, and source buffer.

2. A conceptual use is:

```c
File *file = new_file("example.c", 1, source_text);
```

3. The function can initialize defaults such as `display_name = name` and `line_delta = 0`.

### 19. String-literal retokenization

```c
Token *tokenize_string_literal(Token *tok, Type *basety);
```

1. `tokenize_string_literal()` converts or retokenizes a string literal using the supplied base character type.

2. The base type can distinguish ordinary, wide, UTF-16, and UTF-32 literals.

```text
"abc"    -> char elements
L"abc"   -> wide-character elements
u"abc"   -> char16_t-like elements
U"abc"   -> char32_t-like elements
```

3. This processing determines both the decoded byte representation and the array type stored in `tok->ty`.

### 20. Tokenization entry points

#### 20-1. `tokenize()`

```c
Token *tokenize(File *file);
```

1. `tokenize()` converts the in-memory contents of a `File` into a linked list of tokens.

```text
File contents
     |
     v
Tokenizer
     |
     v
TK_KEYWORD -> TK_IDENT -> TK_PUNCT -> TK_NUM -> TK_PUNCT -> TK_EOF
```

2. For this input:

```c
int x = 42;
```

3. The approximate token sequence is:

```text
TK_KEYWORD "int"
TK_IDENT   "x"
TK_PUNCT   "="
TK_PP_NUM  "42"
TK_PUNCT   ";"
TK_EOF
```

4. The preprocessing number can later become `TK_NUM` through `convert_pp_tokens()`.

#### 20-2. `tokenize_file()`

```c
Token *tokenize_file(char *filename);
```

1. `tokenize_file()` is a higher-level entry point that loads a named file and tokenizes its contents.

```c
Token *tok = tokenize_file("example.c");
```

2. Its responsibilities typically include opening the file, reading the contents, creating a `File`, recording it among the input files, and invoking `tokenize()`.

3. A null result may indicate a file-loading failure, depending on the implementation’s documented error convention.

### 21. Overall data flow

1. `tokenize_file()` reads source text and creates a `File`.

2. `tokenize()` scans `File.contents` and produces a linked list of `Token` objects.

3. The preprocessor uses `at_bol`, `has_space`, `hideset`, and `origin` while processing directives and expanding macros.

4. `convert_pp_tokens()` converts remaining `TK_PP_NUM` tokens into typed numeric constants.

5. The parser consumes tokens using helpers such as `equal()`, `consume()`, and `skip()`.

6. On invalid input, `error_tok()` reports the filename, line number, source position, and explanatory message.

```text
Source file
    |
    v
tokenize_file()
    |
    v
File + preprocessing tokens
    |
    v
Macro expansion and directives
    |
    v
convert_pp_tokens()
    |
    v
Typed token stream
    |
    v
Parser and type checker
```

User:
#define unreachable() \
  error("internal error at %s:%d", __FILE__, __LINE__)

//
// preprocess.c
//

char *search_include_paths(char *filename);
void init_macros(void);
void define_macro(char *name, char *buf);
void undef_macro(char *name);
Token *preprocess(Token *tok);

//
// parse.c
//

// Variable or function
typedef struct Obj Obj;
struct Obj {
  Obj *next;
  char *name;    // Variable name
  Type *ty;      // Type
  Token *tok;    // representative token
  bool is_local; // local or global/function
  int align;     // alignment

  // Local variable
  int offset;

  // Global variable or function
  bool is_function;
  bool is_definition;
  bool is_static;

  // Global variable
  bool is_tentative;
  bool is_tls;
  char *init_data;
  Relocation *rel;

  // Function
  bool is_inline;
  Obj *params;
  Node *body;
  Obj *locals;
  Obj *va_area;
  Obj *alloca_bottom;
  int stack_size;

  // Static inline function
  bool is_live;
  bool is_root;
  StringArray refs;
};

// Global variable can be initialized either by a constant expression
// or a pointer to another global variable. This struct represents the
// latter.
typedef struct Relocation Relocation;
struct Relocation {
  Relocation *next;
  int offset;
  char **label;
  long addend;
};

// AST node
typedef enum {
  ND_NULL_EXPR, // Do nothing
  ND_ADD,       // +
  ND_SUB,       // -
  ND_MUL,       // *
  ND_DIV,       // /
  ND_NEG,       // unary -
  ND_MOD,       // %
  ND_BITAND,    // &
  ND_BITOR,     // |
  ND_BITXOR,    // ^
  ND_SHL,       // <<
  ND_SHR,       // >>
  ND_EQ,        // ==
  ND_NE,        // !=
  ND_LT,        // <
  ND_LE,        // <=
  ND_ASSIGN,    // =
  ND_COND,      // ?:
  ND_COMMA,     // ,
  ND_MEMBER,    // . (struct member access)
  ND_ADDR,      // unary &
  ND_DEREF,     // unary *
  ND_NOT,       // !
  ND_BITNOT,    // ~
  ND_LOGAND,    // &&
  ND_LOGOR,     // ||
  ND_RETURN,    // "return"
  ND_IF,        // "if"
  ND_FOR,       // "for" or "while"
  ND_DO,        // "do"
  ND_SWITCH,    // "switch"
  ND_CASE,      // "case"
  ND_BLOCK,     // { ... }
  ND_GOTO,      // "goto"
  ND_GOTO_EXPR, // "goto" labels-as-values
  ND_LABEL,     // Labeled statement
  ND_LABEL_VAL, // [GNU] Labels-as-values
  ND_FUNCALL,   // Function call
  ND_EXPR_STMT, // Expression statement
  ND_STMT_EXPR, // Statement expression
  ND_VAR,       // Variable
  ND_VLA_PTR,   // VLA designator
  ND_NUM,       // Integer
  ND_CAST,      // Type cast
  ND_MEMZERO,   // Zero-clear a stack variable
  ND_ASM,       // "asm"
  ND_CAS,       // Atomic compare-and-swap
  ND_EXCH,      // Atomic exchange
} NodeKind;


the code continues from the above.

Assistant:
### 1. `unreachable()` Macro

```c
// File: chibicc.h
#define unreachable() \
  error("internal error at %s:%d", __FILE__, __LINE__)
```

1. `unreachable()` reports an internal compiler error at the location where the macro is expanded. It is used for states that should be impossible if the compiler itself is correct.

2. `__FILE__` and `__LINE__` are predefined C macros containing the current source filename and line number.

```c
// File: parse.c
switch (tok->kind) {
case TK_IDENT:
  break;
case TK_NUM:
  break;
default:
  unreachable();
}
```

3. A resulting diagnostic may look like:

```text
internal error at parse.c:125
```

4. This differs from `error_tok()`: `error_tok()` diagnoses invalid user input, whereas `unreachable()` diagnoses a compiler implementation defect or violated invariant.

5. Although named `unreachable`, it does not provide optimization semantics like GCC’s `__builtin_unreachable()`. It calls the `noreturn` function `error()`, which terminates compilation.

### 2. Preprocessor Interface

```c
// File: chibicc.h
char *search_include_paths(char *filename);
void init_macros(void);
void define_macro(char *name, char *buf);
void undef_macro(char *name);
Token *preprocess(Token *tok);
```

1. These functions implement the interface between tokenization and C preprocessing. They handle include files, predefined macros, user-defined macros, conditional directives, and macro expansion.

### 3. `search_include_paths()`

```c
// File: chibicc.h
char *search_include_paths(char *filename);
```

1. `search_include_paths()` searches configured include directories for a header filename.

2. For this directive:

```c
// File: example.c
#include <stdio.h>
```

3. The compiler may test paths conceptually similar to:

```text
/usr/local/include/stdio.h
/usr/include/stdio.h
```

4. If the file is found, the function returns its resolved pathname. The exact failure behavior must be checked in `preprocess.c`, but a null pointer is a common result for “not found.”

5. Quoted includes and angle-bracket includes have different search rules:

```c
// File: example.c
#include "project.h"
#include <system.h>
```

6. A quoted include normally searches relative to the including file before searching standard include directories, while an angle-bracket include searches configured system paths.

### 4. `init_macros()`

```c
// File: chibicc.h
void init_macros(void);
```

1. `init_macros()` initializes predefined macros before preprocessing begins.

2. Typical compiler-provided macros include architecture, operating-system, language-version, and implementation markers.

```c
// File: example.c
#ifdef __linux__
int platform_is_linux;
#endif
```

3. It may also install predefined macros whose values are generated dynamically, such as `__FILE__`, `__LINE__`, `__DATE__`, and `__TIME__`.

4. This function should normally be called once during compiler initialization, before `preprocess()` processes the main translation unit.

### 5. `define_macro()`

```c
// File: chibicc.h
void define_macro(char *name, char *buf);
```

1. `define_macro()` programmatically defines a macro using `name` as its identifier and `buf` as its replacement text.

2. It supports command-line definitions conceptually equivalent to:

```text
-DVERSION=3
```

3. The corresponding call could be:

```c
// File: main.c
define_macro("VERSION", "3");
```

4. The replacement text generally must be tokenized because macro expansion operates on preprocessing tokens rather than raw character strings.

5. A definition without an explicit value is often represented as a replacement value such as `1`, depending on the command-line driver’s policy.

### 6. `undef_macro()`

```c
// File: chibicc.h
void undef_macro(char *name);
```

1. `undef_macro()` removes a macro definition from the preprocessor’s macro table.

2. It supports command-line behavior conceptually equivalent to:

```text
-UDEBUG
```

3. The corresponding call could be:

```c
// File: main.c
undef_macro("DEBUG");
```

4. After removal, `#ifdef DEBUG` evaluates as false unless the source defines `DEBUG` again.

### 7. `preprocess()`

```c
// File: chibicc.h
Token *preprocess(Token *tok);
```

1. `preprocess()` accepts a linked list of preprocessing tokens and returns the processed token stream consumed by the parser.

2. Its major responsibilities generally include:
   1. Expanding object-like and function-like macros.
   2. Processing `#include`.
   3. Evaluating conditional directives.
   4. Handling `#define` and `#undef`.
   5. Processing `#line`.
   6. Concatenating adjacent string literals where appropriate.
   7. Converting preprocessing numbers to typed numeric tokens.

3. For example:

```c
// File: example.c
#define SIZE 4
int values[SIZE];
```

4. The parser receives a token stream approximately equivalent to:

```c
// File: preprocessed-example.c
int values[4];
```

5. The overall front-end pipeline is:

```text
source characters
      |
      v
tokenize()
      |
      v
preprocessing tokens
      |
      v
preprocess()
      |
      v
parser-ready tokens
      |
      v
parse()
```

### 8. `Obj`: Variables and Functions

```c
// File: chibicc.h
typedef struct Obj Obj;

struct Obj {
  Obj *next;
  char *name;
  Type *ty;
  Token *tok;
  bool is_local;
  int align;

  int offset;

  bool is_function;
  bool is_definition;
  bool is_static;

  bool is_tentative;
  bool is_tls;
  char *init_data;
  Relocation *rel;

  bool is_inline;
  Obj *params;
  Node *body;
  Obj *locals;
  Obj *va_area;
  Obj *alloca_bottom;
  int stack_size;

  bool is_live;
  bool is_root;
  StringArray refs;
};
```

1. `Obj` represents a declared program object in chibicc’s internal model. Despite the name, it is used for variables and functions.

2. Some fields are common to every object, while other fields are meaningful only for local variables, global variables, or functions.

3. This “tagged structure” design avoids a hierarchy of separate C structures. Boolean fields such as `is_local` and `is_function` determine how the remaining fields should be interpreted.

### 9. Common `Obj` Fields

#### 9-1. `next`

```c
// File: chibicc.h
Obj *next;
```

1. `next` links objects into a singly linked list.

2. Separate lists can represent global objects, function parameters, and function-local variables.

```text
globals:
main -> counter -> message -> NULL

parameters:
argc -> argv -> NULL
```

#### 9-2. `name`

```c
// File: chibicc.h
char *name;
```

1. `name` stores the source-level or generated identifier associated with the object.

2. Examples include `counter`, `main`, and compiler-generated names for anonymous static objects.

#### 9-3. `ty`

```c
// File: chibicc.h
Type *ty;
```

1. `ty` points to the complete C type of the object.

2. For a variable, it can represent `int`, a pointer, an array, a structure, or another object type.

3. For a function, it represents a function type containing information such as return type, parameter types, and variadic status.

```c
// File: example.c
long sum(int left, int right);
```

4. Conceptually, `sum->ty` describes a function taking two `int` arguments and returning `long`.

#### 9-4. `tok`

```c
// File: chibicc.h
Token *tok;
```

1. `tok` is a representative source token associated with the declaration.

2. It is primarily useful for diagnostics and source-location tracking.

```text
example.c:12: conflicting types for 'value'
```

#### 9-5. `is_local`

```c
// File: chibicc.h
bool is_local;
```

1. `is_local` distinguishes stack-based local variables from global variables and functions.

2. The classification is approximately:

```text
is_local = true   -> local variable or parameter
is_local = false  -> global variable or function
```

#### 9-6. `align`

```c
// File: chibicc.h
int align;
```

1. `align` records the required memory alignment in bytes.

2. A typical object might have size 8 and alignment 8, although exact values depend on the target ABI and type.

3. Stack and global-data layout use this value to place objects at valid addresses.

[
\operatorname{aligned}(n,a)
=
\left\lceil \frac{n}{a} \right\rceil a
]

### 10. Local-Variable Field

```c
// File: chibicc.h
int offset;
```

1. `offset` identifies a local variable’s position within a function’s stack frame.

2. On an architecture using a frame pointer, code generation may access the object relative to that register.

```text
rbp - 8   -> first local variable
rbp - 16  -> second local variable
rbp - 24  -> third local variable
```

3. For the following function:

```c
// File: example.c
int function(void) {
  int x;
  long y;
  return x;
}
```

4. `x` and `y` receive stack offsets after their size and alignment requirements have been calculated.

5. Whether the generated address is `frame_pointer - offset` or `frame_pointer + offset` depends on the code generator’s convention.

### 11. Global Object and Function Flags

#### 11-1. `is_function`

```c
// File: chibicc.h
bool is_function;
```

1. `is_function` distinguishes function objects from variable objects.

```text
false -> variable
true  -> function
```

2. This flag determines whether the later function-specific fields are valid.

#### 11-2. `is_definition`

```c
// File: chibicc.h
bool is_definition;
```

1. `is_definition` distinguishes an object definition from a declaration that merely refers to an externally defined entity.

```c
// File: example.c
extern int external_value;  // Declaration
int actual_value = 10;      // Definition

int calculate(int);         // Declaration
int calculate(int x) {      // Definition
  return x * 2;
}
```

2. Code generation normally emits storage or a function body only for definitions.

#### 11-3. `is_static`

```c
// File: chibicc.h
bool is_static;
```

1. `is_static` records internal linkage or another relevant `static` property.

```c
// File: example.c
static int counter;
static void helper(void) {
}
```

2. A file-scope static object is not exported as a public symbol from the translation unit.

3. For a local static variable, storage is static rather than automatic even though the name has block scope.

### 12. Global-Variable Fields

#### 12-1. `is_tentative`

```c
// File: chibicc.h
bool is_tentative;
```

1. `is_tentative` marks a tentative definition.

```c
// File: example.c
int counter;
```

2. At file scope, this declaration is tentative when no initializer or prior complete definition changes its status.

3. If no full definition appears later in the translation unit, it behaves as a zero-initialized definition.

```c
// File: example.c
int value;
int value;
```

4. The two tentative declarations do not normally require two separate objects.

#### 12-2. `is_tls`

```c
// File: chibicc.h
bool is_tls;
```

1. `is_tls` identifies thread-local storage objects.

```c
// File: example.c
_Thread_local int thread_counter;
```

2. Each thread receives a separate instance of the variable.

3. Code generation and linking require a TLS-specific storage section and addressing model.

#### 12-3. `init_data`

```c
// File: chibicc.h
char *init_data;
```

1. `init_data` holds the byte representation of a global variable’s initial value.

2. For a scalar integer, it contains the target representation of that integer; for an aggregate, it contains the initialized bytes including zero-filled areas.

```c
// File: example.c
int numbers[4] = {1, 2};
```

3. The conceptual initial bytes represent:

```text
numbers[0] = 1
numbers[1] = 2
numbers[2] = 0
numbers[3] = 0
```

4. Pointer values cannot always be fully represented in `init_data`, because their final addresses are assigned by the assembler and linker. Those values require relocations.

#### 12-4. `rel`

```c
// File: chibicc.h
Relocation *rel;
```

1. `rel` points to a linked list of relocation records associated with the global initializer.

2. Each record tells the backend that a portion of `init_data` must refer to another symbol.

```c
// File: example.c
int target;
int *pointer = &target;
```

3. The final address of `target` is unknown to the compiler, so the generated object file needs a relocation referring to the `target` symbol.

### 13. Function-Specific Fields

#### 13-1. `is_inline`

```c
// File: chibicc.h
bool is_inline;
```

1. `is_inline` records whether the function was declared with the `inline` specifier.

```c
// File: example.c
static inline int square(int x) {
  return x * x;
}
```

2. In C, `inline` affects linkage and emission rules; it is not an unconditional command to perform optimization-time inlining.

3. chibicc may still generate an ordinary callable function rather than substituting its body at every call site.

#### 13-2. `params`

```c
// File: chibicc.h
Obj *params;
```

1. `params` points to the linked list of function parameter objects in declaration order.

```c
// File: example.c
long add(int left, int right);
```

2. The conceptual list is:

```text
left -> right -> NULL
```

3. Each parameter is also an `Obj`, usually with `is_local == true`, because parameters are handled similarly to local variables after function entry.

#### 13-3. `body`

```c
// File: chibicc.h
Node *body;
```

1. `body` points to the root AST node of the parsed function body.

```c
// File: example.c
int twice(int x) {
  return x * 2;
}
```

2. Its simplified AST is:

```text
ND_BLOCK
  |
  +-- ND_RETURN
        |
        +-- ND_MUL
             +-- ND_VAR(x)
             +-- ND_NUM(2)
```

#### 13-4. `locals`

```c
// File: chibicc.h
Obj *locals;
```

1. `locals` points to all local variables allocated for the function.

2. This can include parameters, named local variables, compiler-generated temporaries, and hidden implementation objects.

3. The backend traverses this list to assign stack offsets and calculate the stack-frame size.

#### 13-5. `va_area`

```c
// File: chibicc.h
Obj *va_area;
```

1. `va_area` identifies compiler-managed stack storage used by a variadic function.

```c
// File: example.c
int sum(int count, ...);
```

2. A target ABI may pass arguments partly in registers and partly on the stack. The variadic machinery needs saved register values and metadata so `va_start` and `va_arg` can retrieve unnamed arguments.

3. This field represents an implementation detail required by the target calling convention.

#### 13-6. `alloca_bottom`

```c
// File: chibicc.h
Obj *alloca_bottom;
```

1. `alloca_bottom` tracks a reference point used when implementing dynamic stack allocation.

2. Calls to `alloca()` can move the stack pointer during function execution, so the compiler needs bookkeeping to preserve or relocate temporary stack arguments correctly.

3. This is particularly relevant when a function both performs dynamic allocation and makes calls with stack-passed arguments.

#### 13-7. `stack_size`

```c
// File: chibicc.h
int stack_size;
```

1. `stack_size` is the total number of bytes reserved for the function’s stack frame.

2. It is calculated after local offsets and alignment requirements are known.

3. If the ABI requires 16-byte stack alignment, the final value is rounded accordingly.

[
\text{stack size}
=
\operatorname{align\_to}(\text{required bytes},16)
]

### 14. Static Inline Function Liveness

```c
// File: chibicc.h
bool is_live;
bool is_root;
StringArray refs;
```

1. These fields support reachability analysis for static inline functions. The compiler can avoid emitting functions that are not reachable from code requiring externally observable output.

#### 14-1. `is_live`

1. `is_live` indicates that the function has been determined to be reachable and therefore should be emitted.

2. A function becomes live if it is a root or is referenced by another live function.

#### 14-2. `is_root`

1. `is_root` marks a function that begins the reachability traversal.

2. Non-inline functions and otherwise externally required definitions can act as roots, while an unreferenced static inline function generally does not need emission.

#### 14-3. `refs`

1. `refs` records names of functions referenced by this function.

```c
// File: example.c
static inline int helper(int x) {
  return x + 1;
}

int public_function(int x) {
  return helper(x);
}
```

2. Conceptually, the dependency graph is:

```text
public_function [root]
        |
        v
helper [live]
```

3. A graph traversal marks `public_function` and then follows its `refs` entry to mark `helper`.

### 15. `Relocation`

```c
// File: chibicc.h
typedef struct Relocation Relocation;

struct Relocation {
  Relocation *next;
  int offset;
  char **label;
  long addend;
};
```

1. `Relocation` represents a symbolic address embedded in a global initializer.

2. It is necessary because the compiler does not know final addresses before assembly and linking.

### 16. Relocation Fields

#### 16-1. `next`

```c
// File: chibicc.h
Relocation *next;
```

1. `next` links multiple relocations belonging to the same initialized object.

```c
// File: example.c
int first;
int second;
int *table[] = {&first, &second};
```

2. The initializer for `table` requires at least one relocation for `first` and one for `second`.

#### 16-2. `offset`

```c
// File: chibicc.h
int offset;
```

1. `offset` specifies the byte position inside `init_data` where the symbolic address must be written.

2. For an array of two 8-byte pointers, the offsets might be:

```text
table + 0  -> relocation for first
table + 8  -> relocation for second
```

#### 16-3. `label`

```c
// File: chibicc.h
char **label;
```

1. `label` identifies the symbol referenced by the relocation.

2. It is a `char **` rather than `char *`, allowing the symbol name to be resolved or replaced indirectly after the relocation record is created.

3. This is useful when labels are compiler-generated or finalized after parsing.

#### 16-4. `addend`

```c
// File: chibicc.h
long addend;
```

1. `addend` is a constant displacement added to the symbol address.

2. A relocation conceptually computes:

[
\text{stored value}
=
\operatorname{address}(\text{label})+\text{addend}
]

3. For example:

```c
// File: example.c
int values[10];
int *middle = values + 5;
```

4. If `sizeof(int) == 4`, the relocation refers to `values` with an addend of 20 bytes.

### 17. AST `NodeKind`

```c
// File: chibicc.h
typedef enum {
  ND_NULL_EXPR,
  ND_ADD,
  ND_SUB,
  ND_MUL,
  ND_DIV,
  ND_NEG,
  ND_MOD,
  ND_BITAND,
  ND_BITOR,
  ND_BITXOR,
  ND_SHL,
  ND_SHR,
  ND_EQ,
  ND_NE,
  ND_LT,
  ND_LE,
  ND_ASSIGN,
  ND_COND,
  ND_COMMA,
  ND_MEMBER,
  ND_ADDR,
  ND_DEREF,
  ND_NOT,
  ND_BITNOT,
  ND_LOGAND,
  ND_LOGOR,
  ND_RETURN,
  ND_IF,
  ND_FOR,
  ND_DO,
  ND_SWITCH,
  ND_CASE,
  ND_BLOCK,
  ND_GOTO,
  ND_GOTO_EXPR,
  ND_LABEL,
  ND_LABEL_VAL,
  ND_FUNCALL,
  ND_EXPR_STMT,
  ND_STMT_EXPR,
  ND_VAR,
  ND_VLA_PTR,
  ND_NUM,
  ND_CAST,
  ND_MEMZERO,
  ND_ASM,
  ND_CAS,
  ND_EXCH,
} NodeKind;
```

1. `NodeKind` identifies the semantic category of an abstract syntax tree node. The corresponding `Node` structure, defined later, stores the operands and metadata required by each category.

2. A single enum supports expressions, statements, declarations lowered into operations, and implementation-specific nodes.

### 18. Arithmetic Nodes

| Kind | Meaning | Example |
|---|---|---|
| `ND_ADD` | Addition | `a + b` |
| `ND_SUB` | Subtraction | `a - b` |
| `ND_MUL` | Multiplication | `a * b` |
| `ND_DIV` | Division | `a / b` |
| `ND_NEG` | Unary negation | `-a` |
| `ND_MOD` | Remainder | `a % b` |

1. Binary arithmetic nodes usually store a left-hand operand and a right-hand operand.

```text
a + b * c

       ND_ADD
       /    \
 ND_VAR(a)  ND_MUL
            /    \
      ND_VAR(b) ND_VAR(c)
```

2. Operator precedence is represented by the tree structure: multiplication is nested below addition and therefore evaluated as one operand of the addition.

3. `ND_ADD` and `ND_SUB` also represent pointer arithmetic after type checking.

```c
// File: example.c
int *next = pointer + 3;
```

4. Pointer addition must scale the integer by the pointed-to element size:

[
\text{result address}
=
\text{pointer address}
+
3 \times \operatorname{sizeof}(*\text{pointer})
]

### 19. Bitwise and Shift Nodes

| Kind | Meaning | Example |
|---|---|---|
| `ND_BITAND` | Bitwise AND | `a & b` |
| `ND_BITOR` | Bitwise OR | `a \| b` |
| `ND_BITXOR` | Bitwise XOR | `a ^ b` |
| `ND_SHL` | Left shift | `a << b` |
| `ND_SHR` | Right shift | `a >> b` |
| `ND_BITNOT` | Bitwise complement | `~a` |

1. These nodes operate on integer representations rather than Boolean truth values.

2. They are distinct from `ND_LOGAND`, `ND_LOGOR`, and `ND_NOT`, which implement logical semantics.

### 20. Comparison Nodes

| Kind | Meaning | Example |
|---|---|---|
| `ND_EQ` | Equal | `a == b` |
| `ND_NE` | Not equal | `a != b` |
| `ND_LT` | Less than | `a < b` |
| `ND_LE` | Less than or equal | `a <= b` |

1. `>` and `>=` do not require separate node kinds because the parser can reverse their operands.

```text
a > b   -> b < a
a >= b  -> b <= a
```

2. This normalization reduces the number of cases the type checker and code generator must handle.

### 21. Assignment and Sequencing Nodes

#### 21-1. `ND_ASSIGN`

1. `ND_ASSIGN` represents assignment.

```c
// File: example.c
value = expression;
```

2. Its left operand must designate a modifiable object, while its right operand supplies the new value.

#### 21-2. `ND_COND`

1. `ND_COND` represents the conditional operator.

```c
// File: example.c
result = condition ? left : right;
```

2. It normally needs three child nodes: condition, then-expression, and else-expression.

#### 21-3. `ND_COMMA`

1. `ND_COMMA` represents the comma operator.

```c
// File: example.c
result = (initialize(), calculate());
```

2. The left operand is evaluated first, and the expression’s result is the right operand’s value.

### 22. Structure and Pointer Nodes

#### 22-1. `ND_MEMBER`

1. `ND_MEMBER` represents structure or union member access.

```c
// File: example.c
record.value
pointer->value
```

2. `pointer->value` can be lowered conceptually to `(*pointer).value`, so both forms can use the same node kind after parsing.

3. The node refers to a `Member` descriptor containing the member’s type, offset, and related metadata.

#### 22-2. `ND_ADDR`

1. `ND_ADDR` represents the unary address-of operator.

```c
// File: example.c
int *pointer = &value;
```

#### 22-3. `ND_DEREF`

1. `ND_DEREF` represents pointer dereference.

```c
// File: example.c
int value = *pointer;
```

2. The generated load or store behavior depends on whether the node is used as an rvalue or as an assignment destination.

### 23. Logical Nodes

| Kind | Meaning | Example |
|---|---|---|
| `ND_NOT` | Logical negation | `!a` |
| `ND_LOGAND` | Logical AND | `a && b` |
| `ND_LOGOR` | Logical OR | `a \|\| b` |

1. `ND_LOGAND` and `ND_LOGOR` require short-circuit evaluation.

```c
// File: example.c
pointer != NULL && pointer->value > 0
```

2. The right operand must not be evaluated when the left operand already determines the result.

3. Consequently, these nodes are generally emitted using branches rather than a simple arithmetic instruction.

### 24. Control-Flow Statement Nodes

| Kind | Meaning |
|---|---|
| `ND_RETURN` | Return statement |
| `ND_IF` | `if` statement |
| `ND_FOR` | `for` or `while` loop |
| `ND_DO` | `do-while` loop |
| `ND_SWITCH` | `switch` statement |
| `ND_CASE` | `case` or `default` label |
| `ND_BLOCK` | Compound statement |

1. `ND_FOR` represents both `for` and `while` because a `while` loop can use the same internal shape with missing initialization and increment components.

```c
// File: example.c
while (condition)
  body;
```

2. It can be normalized approximately as:

```c
// File: normalized-example.c
for (; condition;)
  body;
```

3. `ND_BLOCK` contains a sequence of statement nodes corresponding to `{ ... }`.

4. `ND_SWITCH` typically points to linked `ND_CASE` nodes so code generation can compare the controlling value and dispatch to the correct labels.

### 25. Labels and `goto`

| Kind | Meaning |
|---|---|
| `ND_GOTO` | Ordinary `goto` |
| `ND_GOTO_EXPR` | Computed `goto` |
| `ND_LABEL` | Labeled statement |
| `ND_LABEL_VAL` | Address of a label |

1. `ND_GOTO` and `ND_LABEL` implement standard C label-based control flow.

```c
// File: example.c
goto cleanup;

cleanup:
  release_resources();
```

2. `ND_LABEL_VAL` and `ND_GOTO_EXPR` implement GNU labels-as-values extensions.

```c
// File: example.c
void *target = &&label;
goto *target;

label:
  return;
```

3. The parser must resolve label names after parsing because a `goto` may refer to a label declared later in the function.

### 26. Calls and Statement Expressions

#### 26-1. `ND_FUNCALL`

1. `ND_FUNCALL` represents a function call.

```c
// File: example.c
result = calculate(left, right);
```

2. The node stores the function expression, argument nodes, and usually the resolved return type.

#### 26-2. `ND_EXPR_STMT`

1. `ND_EXPR_STMT` wraps an expression used as a statement.

```c
// File: example.c
counter++;
function();
```

2. The expression’s result is discarded.

#### 26-3. `ND_STMT_EXPR`

1. `ND_STMT_EXPR` represents the GNU statement-expression extension.

```c
// File: example.c
int result = ({
  int temporary = 3;
  temporary * 2;
});
```

2. Its value is generally the value of the final expression statement inside the block.

### 27. Values, Variables, and Casts

#### 27-1. `ND_VAR`

1. `ND_VAR` refers to a variable represented by an `Obj`.

```c
// File: example.c
value
```

2. The node normally stores a pointer to the corresponding `Obj`, allowing code generation to determine whether it is local, global, TLS, or another object category.

#### 27-2. `ND_NUM`

1. `ND_NUM` represents an integer or floating-point constant after token conversion.

```c
// File: example.c
42
3.14
```

2. Its value and type are stored in the `Node` fields defined later.

#### 27-3. `ND_CAST`

1. `ND_CAST` represents an explicit or compiler-inserted type conversion.

```c
// File: example.c
double result = (double)integer_value;
```

2. The compiler can also insert casts for ordinary arithmetic conversions, assignments, argument passing, and return values.

### 28. Variable-Length Array Node

```c
// File: chibicc.h
ND_VLA_PTR
```

1. `ND_VLA_PTR` represents an internal pointer associated with a variable-length array.

```c
// File: example.c
void function(int length) {
  int values[length];
  values[0] = 1;
}
```

2. A VLA’s size is computed at runtime, so it cannot be handled entirely like a fixed-offset local array.

3. The compiler typically allocates dynamic stack space and stores the resulting address in a hidden local variable; `ND_VLA_PTR` refers to that runtime address.

### 29. Internal Memory-Zeroing Node

```c
// File: chibicc.h
ND_MEMZERO
```

1. `ND_MEMZERO` is an internal node that clears a stack object before applying an initializer.

```c
// File: example.c
struct Pair {
  int left;
  int right;
};

void function(void) {
  struct Pair pair = {.left = 10};
}
```

2. The compiler can first zero the entire `pair` object and then store `10` into `pair.left`.

3. This simplifies handling of omitted aggregate members, which C requires to be zero-initialized.

### 30. Assembly and Atomic Nodes

#### 30-1. `ND_ASM`

1. `ND_ASM` represents an inline assembly statement.

```c
// File: example.c
asm("nop");
```

2. This is an implementation extension rather than a core ISO C expression.

#### 30-2. `ND_CAS`

1. `ND_CAS` represents an atomic compare-and-swap operation.

2. Its conceptual behavior is:

```c
// File: conceptual-cas.c
if (*address == expected) {
  *address = desired;
  success = true;
} else {
  expected = *address;
  success = false;
}
```

3. The actual operation must be indivisible with respect to concurrent threads and therefore requires target-specific atomic instructions or runtime support.

#### 30-3. `ND_EXCH`

1. `ND_EXCH` represents an atomic exchange operation.

2. Conceptually, it performs:

```c
// File: conceptual-exchange.c
old_value = *address;
*address = new_value;
return old_value;
```

3. Unlike the conceptual non-atomic code, the generated operation must atomically replace the value.

### 31. Relationship Among `Obj`, `Node`, and `Relocation`

1. `Obj` describes declared variables and functions.

2. `Node` describes executable syntax and expressions.

3. An `ND_VAR` node refers to an `Obj`, while a function `Obj` refers back to its AST through `body`.

4. A global-variable `Obj` contains raw initialization bytes in `init_data` and symbolic address fixups in `rel`.

```text
Function Obj
  |
  +-- params  -> Obj -> Obj
  +-- locals  -> Obj -> Obj -> Obj
  +-- body    -> Node AST

Global-variable Obj
  |
  +-- init_data -> byte sequence
  +-- rel       -> Relocation -> Relocation
```

### 32. Example Internal Representation

```c
// File: example.c
int global_value = 10;

int add(int left, int right) {
  int result = left + right;
  return result;
}
```

1. `global_value` becomes a non-local, non-function `Obj` with `init_data` containing the representation of `10`.

2. `add` becomes a non-local function `Obj` whose `params` list contains `left` and `right`.

3. The function’s `locals` list contains `left`, `right`, and `result`, each with an assigned stack `offset`.

4. Its body is represented approximately as:

```text
ND_BLOCK
  |
  +-- initialization/assignment
  |     |
  |     +-- ND_ASSIGN
  |           +-- ND_VAR(result)
  |           +-- ND_ADD
  |                 +-- ND_VAR(left)
  |                 +-- ND_VAR(right)
  |
  +-- ND_RETURN
        |
        +-- ND_VAR(result)
```

5. Code generation traverses this representation to emit stack allocation, arithmetic instructions, variable stores and loads, and the function return sequence.

User:
// AST node type
struct Node {
  NodeKind kind; // Node kind
  Node *next;    // Next node
  Type *ty;      // Type, e.g. int or pointer to int
  Token *tok;    // Representative token

  Node *lhs;     // Left-hand side
  Node *rhs;     // Right-hand side

  // "if" or "for" statement
  Node *cond;
  Node *then;
  Node *els;
  Node *init;
  Node *inc;

  // "break" and "continue" labels
  char *brk_label;
  char *cont_label;

  // Block or statement expression
  Node *body;

  // Struct member access
  Member *member;

  // Function call
  Type *func_ty;
  Node *args;
  bool pass_by_stack;
  Obj *ret_buffer;

  // Goto or labeled statement, or labels-as-values
  char *label;
  char *unique_label;
  Node *goto_next;

  // Switch
  Node *case_next;
  Node *default_case;

  // Case
  long begin;
  long end;

  // "asm" string literal
  char *asm_str;

  // Atomic compare-and-swap
  Node *cas_addr;
  Node *cas_old;
  Node *cas_new;

  // Atomic op= operators
  Obj *atomic_addr;
  Node *atomic_expr;

  // Variable
  Obj *var;

  // Numeric literal
  int64_t val;
  long double fval;
};

Node *new_cast(Node *expr, Type *ty);
int64_t const_expr(Token **rest, Token *tok);
Obj *parse(Token *tok);

//
// type.c
//

typedef enum {
  TY_VOID,
  TY_BOOL,
  TY_CHAR,
  TY_SHORT,
  TY_INT,
  TY_LONG,
  TY_FLOAT,
  TY_DOUBLE,
  TY_LDOUBLE,
  TY_ENUM,
  TY_PTR,
  TY_FUNC,
  TY_ARRAY,
  TY_VLA, // variable-length array
  TY_STRUCT,
  TY_UNION,
} TypeKind;

struct Type {
  TypeKind kind;
  int size;           // sizeof() value
  int align;          // alignment
  bool is_unsigned;   // unsigned or signed
  bool is_atomic;     // true if _Atomic
  Type *origin;       // for type compatibility check

  // Pointer-to or array-of type. We intentionally use the same member
  // to represent pointer/array duality in C.
  //
  // In many contexts in which a pointer is expected, we examine this
  // member instead of "kind" member to determine whether a type is a
  // pointer or not. That means in many contexts "array of T" is
  // naturally handled as if it were "pointer to T", as required by
  // the C spec.
  Type *base;

  // Declaration
  Token *name;
  Token *name_pos;

  // Array
  int array_len;

  // Variable-length array
  Node *vla_len; // # of elements
  Obj *vla_size; // sizeof() value

  // Struct
  Member *members;
  bool is_flexible;
  bool is_packed;

  // Function type
  Type *return_ty;
  Type *params;
  bool is_variadic;
  Type *next;
};


the code continues from the avobe.

Assistant:
### 1. Overall Role of `Node`

1. `Node` is the central representation of expressions and statements after parsing. Its `kind` field determines which of the remaining fields are meaningful.

2. Rather than defining a different structure for every AST node category, chibicc uses one large structure with conditionally used fields. This simplifies allocation and traversal at the cost of storing some unused fields in each node.

```c
// File: chibicc.h
struct Node {
  NodeKind kind;
  Node *next;
  Type *ty;
  Token *tok;

  /* Fields selected according to kind. */
};
```

3. For example, an `ND_ADD` node uses `lhs` and `rhs`, an `ND_IF` node uses `cond`, `then`, and `els`, and an `ND_NUM` node uses `val` or `fval`.

### 2. Common `Node` Fields

#### 2-1. `kind`

```c
// File: chibicc.h
NodeKind kind;
```

1. `kind` identifies the node’s semantic operation. It controls how the parser, type checker, and code generator interpret the node.

```c
// File: codegen.c
switch (node->kind) {
case ND_ADD:
  /* Generate addition. */
  break;
case ND_RETURN:
  /* Generate function return. */
  break;
default:
  unreachable();
}
```

2. Accessing fields that are not applicable to the current `kind` is logically invalid even though C permits the access.

#### 2-2. `next`

```c
// File: chibicc.h
Node *next;
```

1. `next` links sibling nodes into a singly linked list. It is commonly used for function arguments, statements in a block, and other ordered node sequences.

```text
ND_EXPR_STMT -> ND_IF -> ND_RETURN -> NULL
```

2. A block’s first statement may be stored in `body`, with subsequent statements reached through `next`.

#### 2-3. `ty`

```c
// File: chibicc.h
Type *ty;
```

1. `ty` contains the C type assigned to an expression node by semantic analysis. Examples include `int`, `double`, `int *`, and `struct Pair`.

2. Statement nodes usually do not require a value type, but expression nodes depend on it for conversion rules and code generation.

```c
// File: example.c
long value;
int number;

value + number;
```

3. The addition node may have type `long` after the usual arithmetic conversions promote `number`.

#### 2-4. `tok`

```c
// File: chibicc.h
Token *tok;
```

1. `tok` points to a representative source token for diagnostics. It allows later compiler stages to report the location associated with the AST node.

```c
// File: type.c
if (invalid_operand)
  error_tok(node->tok, "invalid operands");
```

2. For a binary operator, this is commonly the token containing the operator itself.

### 3. Binary and Unary Operands

```c
// File: chibicc.h
Node *lhs;
Node *rhs;
```

1. `lhs` and `rhs` represent the left and right operands of binary operations.

```c
// File: example.c
left + right
```

2. Its simplified representation is:

```text
ND_ADD
├── lhs: ND_VAR(left)
└── rhs: ND_VAR(right)
```

3. Unary nodes generally use `lhs` and leave `rhs` unused.

```c
// File: example.c
-value
*pointer
&object
!condition
```

4. Their representation is approximately:

```text
ND_NEG
└── lhs: ND_VAR(value)
```

5. Assignment also uses both fields, with `lhs` designating the destination and `rhs` supplying the value.

```text
ND_ASSIGN
├── lhs: ND_VAR(destination)
└── rhs: ND_ADD
    ├── lhs: ND_VAR(a)
    └── rhs: ND_VAR(b)
```

### 4. Conditional and Loop Fields

```c
// File: chibicc.h
Node *cond;
Node *then;
Node *els;
Node *init;
Node *inc;
```

1. These fields support `if`, `for`, and normalized `while` statements. Unused components are represented by null pointers.

#### 4-1. `if` Statement

```c
// File: example.c
if (value > 0)
  positive();
else
  nonpositive();
```

1. An `ND_IF` node uses:

```text
kind = ND_IF
cond = value > 0
then = positive()
els  = nonpositive()
```

2. `init` and `inc` are unused for an `ND_IF` node.

#### 4-2. `for` Statement

```c
// File: example.c
for (int i = 0; i < 10; i++)
  process(i);
```

1. An `ND_FOR` node uses:

```text
kind = ND_FOR
init = int i = 0
cond = i < 10
inc  = i++
then = process(i)
```

2. In this representation, `then` stores the loop body rather than an “if-then” branch.

#### 4-3. `while` Statement

```c
// File: example.c
while (ready())
  execute();
```

1. A `while` statement can be represented by `ND_FOR` with no initialization or increment.

```text
kind = ND_FOR
init = NULL
cond = ready()
inc  = NULL
then = execute()
```

2. Reusing `ND_FOR` avoids a separate `ND_WHILE` code-generation path.

#### 4-4. Conditional Operator

1. The same `cond`, `then`, and `els` fields can represent `ND_COND`.

```c
// File: example.c
maximum = left > right ? left : right;
```

2. Its conceptual AST is:

```text
ND_COND
├── cond: ND_LT(right, left)
├── then: ND_VAR(left)
└── els:  ND_VAR(right)
```

### 5. Break and Continue Labels

```c
// File: chibicc.h
char *brk_label;
char *cont_label;
```

1. `brk_label` stores the generated assembly label targeted by a `break` statement. It is relevant to loops and `switch` statements.

2. `cont_label` stores the generated label targeted by `continue`. It is relevant to loops but not ordinarily to `switch`.

```c
// File: example.c
for (;;) {
  if (stop)
    break;

  if (skip)
    continue;

  process();
}
```

3. The generated control flow is conceptually:

```text
loop_start:
    if (stop)
        goto break_label

    if (skip)
        goto continue_label

    process()

continue_label:
    goto loop_start

break_label:
```

4. Unique generated labels prevent collisions between nested loops.

### 6. Block and Statement-Expression Body

```c
// File: chibicc.h
Node *body;
```

1. For `ND_BLOCK`, `body` points to the first statement in the block. Remaining statements are reached through each statement’s `next` field.

```c
// File: example.c
{
  initialize();
  process();
  cleanup();
}
```

2. The representation is:

```text
ND_BLOCK.body
    |
    v
ND_EXPR_STMT(initialize)
    |
   next
    v
ND_EXPR_STMT(process)
    |
   next
    v
ND_EXPR_STMT(cleanup)
```

3. `body` is also used by `ND_STMT_EXPR`, the GNU statement-expression extension.

```c
// File: example.c
int value = ({
  int temporary = calculate();
  temporary * 2;
});
```

4. The value of the statement expression is derived from its final expression statement.

### 7. Structure and Union Member Access

```c
// File: chibicc.h
Member *member;
```

1. `member` identifies the selected field for an `ND_MEMBER` node. The `Member` object normally provides its name, type, byte offset, and bit-field properties.

```c
// File: example.c
struct Point {
  int x;
  int y;
};

int value = point.y;
```

2. The AST is conceptually:

```text
ND_MEMBER
├── lhs: ND_VAR(point)
└── member: Point.y
```

3. If `y` has offset 4, the generated address is conceptually:

[
\operatorname{address}(\text{point.y})
=
\operatorname{address}(\text{point})+4
]

4. Pointer member access is normalized similarly:

```c
// File: example.c
pointer->y
```

5. It can be represented as:

```text
ND_MEMBER
├── lhs: ND_DEREF
│   └── lhs: ND_VAR(pointer)
└── member: Point.y
```

### 8. Function-Call Fields

```c
// File: chibicc.h
Type *func_ty;
Node *args;
bool pass_by_stack;
Obj *ret_buffer;
```

1. These fields are primarily meaningful for `ND_FUNCALL`.

#### 8-1. `func_ty`

1. `func_ty` is the resolved function type of the called expression. It describes the return type, parameter types, and variadic status.

```c
// File: example.c
long calculate(int value, double ratio);
```

2. The associated function type conceptually contains:

```text
return_ty  = long
params     = int -> double
is_variadic = false
```

3. This information is needed even when the call is made through a function pointer.

```c
// File: example.c
long (*callback)(int, double);
long result = callback(10, 2.5);
```

#### 8-2. `args`

1. `args` points to the first argument expression. Additional arguments are linked by `next`.

```c
// File: example.c
function(first, second, third);
```

2. The representation is:

```text
args
 |
 v
ND_VAR(first) -> ND_VAR(second) -> ND_VAR(third) -> NULL
```

3. Before code generation, argument expressions are converted to the parameter types required by `func_ty`.

#### 8-3. `pass_by_stack`

1. `pass_by_stack` records that an argument or call-related value must be passed through the stack under the target ABI.

2. A target ABI may pass initial integer arguments in registers and remaining arguments on the stack.

```text
arg1 -> register
arg2 -> register
...
argN -> stack
```

3. Aggregate size and classification can also force stack passing even when argument registers remain available.

#### 8-4. `ret_buffer`

1. `ret_buffer` points to a hidden object used when a function returns a structure or union through memory.

```c
// File: example.c
struct Pair {
  long first;
  long second;
};

struct Pair make_pair(void);
```

2. Some return values cannot be returned entirely in ordinary result registers. The caller allocates storage and passes its address as an implicit argument.

```text
Caller:
    allocate return buffer
    pass &return_buffer to make_pair()
    use return_buffer after the call
```

3. `ret_buffer` identifies that compiler-generated storage.

### 9. Goto and Label Fields

```c
// File: chibicc.h
char *label;
char *unique_label;
Node *goto_next;
```

1. These fields support ordinary labels, `goto`, computed goto, and labels-as-values.

#### 9-1. `label`

1. `label` contains the source-level label name.

```c
// File: example.c
goto cleanup;

cleanup:
  release();
```

2. Both the `goto` and the labeled statement initially refer to `"cleanup"`.

#### 9-2. `unique_label`

1. `unique_label` contains an internal assembler-level label generated by the compiler.

2. Source labels only need to be unique inside one function, but assembly labels often share a wider namespace. chibicc therefore generates names such as:

```text
.L.label.12
```

3. A resolved goto uses this unique label during code generation.

#### 9-3. `goto_next`

1. `goto_next` links unresolved goto and label-reference nodes.

2. C allows a goto to refer to a label appearing later, so immediate resolution is not always possible.

```c
// File: example.c
goto finish;

/* More statements. */

finish:
  return;
```

3. The parser can collect goto nodes into a list and resolve them after parsing the entire function body.

### 10. Switch Fields

```c
// File: chibicc.h
Node *case_next;
Node *default_case;
```

1. An `ND_SWITCH` node needs access to all associated `case` labels. `case_next` links those labels into a list.

```c
// File: example.c
switch (value) {
case 1:
  first();
  break;
case 2:
  second();
  break;
default:
  other();
}
```

2. The switch metadata is conceptually:

```text
ND_SWITCH
├── case_next ──> case 1 ──> case 2
└── default_case ──> default
```

3. `default_case` directly identifies the default label, if present. It is null when no default label exists.

4. The body of the switch remains an ordinary statement tree; the case list provides an efficient index for code generation.

### 11. Case Value Range

```c
// File: chibicc.h
long begin;
long end;
```

1. For an ordinary `case`, `begin` and `end` contain the same value.

```c
// File: example.c
case 10:
```

2. Its representation is:

```text
begin = 10
end   = 10
```

3. Separate endpoints also support GNU case ranges.

```c
// File: example.c
case 10 ... 20:
```

4. Its representation is:

```text
begin = 10
end   = 20
```

5. Code generation checks whether the switch expression satisfies:

[
\text{begin} \leq \text{value} \leq \text{end}
]

### 12. Inline Assembly String

```c
// File: chibicc.h
char *asm_str;
```

1. `asm_str` stores the decoded assembly text for an `ND_ASM` node.

```c
// File: example.c
asm("nop");
```

2. Its simplified representation is:

```text
kind    = ND_ASM
asm_str = "nop"
```

3. The code generator emits the string directly into the generated assembly output.

4. Because inline assembly bypasses much of the compiler’s semantic understanding, malformed or target-incompatible text may fail later in the assembler.

### 13. Atomic Compare-and-Swap Fields

```c
// File: chibicc.h
Node *cas_addr;
Node *cas_old;
Node *cas_new;
```

1. These fields represent the three central operands of an `ND_CAS` operation.

2. `cas_addr` identifies the atomic object, `cas_old` identifies the expected-value storage, and `cas_new` contains the desired replacement value.

3. The operation is conceptually similar to:

```c
// File: conceptual-cas.c
bool compare_exchange(
  int *address,
  int *expected,
  int desired
) {
  if (*address == *expected) {
    *address = desired;
    return true;
  }

  *expected = *address;
  return false;
}
```

4. The real operation must be atomic, so it cannot be implemented by the ordinary non-atomic C sequence shown above.

5. The node layout is:

```text
ND_CAS
├── cas_addr: address of atomic object
├── cas_old:  address of expected value
└── cas_new:  desired value
```

### 14. Atomic Compound-Assignment Fields

```c
// File: chibicc.h
Obj *atomic_addr;
Node *atomic_expr;
```

1. These fields support atomic compound-assignment operators such as `+=`, `-=`, `|=`, and `^=`.

```c
// File: example.c
_Atomic int counter;
counter += amount;
```

2. A non-atomic compound assignment can conceptually perform a load, calculation, and store. That sequence would be unsafe for an atomic object because another thread could modify the value between operations.

3. chibicc can lower the operation into a compare-and-swap loop:

```c
// File: conceptual-atomic-add.c
do {
  old_value = atomic_load(address);
  new_value = old_value + amount;
} while (!atomic_compare_exchange(address, &old_value, new_value));
```

4. `atomic_addr` stores a compiler-generated local object containing the address being updated, which ensures that an address expression with side effects is evaluated only once.

5. `atomic_expr` stores the expression used to calculate the desired value from the previously loaded value.

### 15. Variable Reference

```c
// File: chibicc.h
Obj *var;
```

1. `var` is meaningful for an `ND_VAR` node and points to the corresponding variable object.

```c
// File: example.c
int count;
int result = count;
```

2. The reference to `count` is represented approximately as:

```text
kind = ND_VAR
var  = Obj(count)
ty   = int
```

3. The code generator inspects `var->is_local`, `var->offset`, and other fields to determine how to access it.

```text
Local variable:
    address = frame pointer - var->offset

Global variable:
    address = symbol represented by var->name

TLS variable:
    address = target-specific TLS expression
```

### 16. Numeric Literal Fields

```c
// File: chibicc.h
int64_t val;
long double fval;
```

1. `val` stores integer constants, while `fval` stores floating-point constants.

```c
// File: example.c
42
0xff
3.14159
1.0L
```

2. Integer nodes use `val` together with `ty`.

```text
val = 42, ty = int
val = 42, ty = unsigned long
```

3. Floating-point nodes use `fval` together with `ty`.

```text
fval = 3.14159, ty = double
fval = 1.0,     ty = long double
```

4. The type is essential because the stored mathematical value alone does not determine signedness, width, or generated instruction form.

### 17. Field Usage by `NodeKind`

1. The principal field combinations are approximately:

| `NodeKind` category | Relevant fields |
|---|---|
| Binary expression | `lhs`, `rhs`, `ty`, `tok` |
| Unary expression | `lhs`, `ty`, `tok` |
| Numeric literal | `val` or `fval`, `ty`, `tok` |
| Variable | `var`, `ty`, `tok` |
| Member access | `lhs`, `member`, `ty` |
| Conditional operator | `cond`, `then`, `els`, `ty` |
| `if` | `cond`, `then`, `els` |
| `for` or `while` | `init`, `cond`, `inc`, `then` |
| Block | `body` |
| Function call | `lhs`, `func_ty`, `args`, `ret_buffer` |
| Goto or label | `label`, `unique_label`, `goto_next` |
| Switch | `cond`, `then`, `case_next`, `default_case` |
| Case | `begin`, `end`, `unique_label` |
| Inline assembly | `asm_str` |
| Atomic CAS | `cas_addr`, `cas_old`, `cas_new` |

2. This conditional interpretation is a central invariant: the node’s `kind` must always agree with its initialized fields.

### 18. `new_cast()`

```c
// File: chibicc.h
Node *new_cast(Node *expr, Type *ty);
```

1. `new_cast()` creates an `ND_CAST` node that converts `expr` to `ty`.

```c
// File: example.c
int integer = 10;
double floating = (double)integer;
```

2. The generated node is conceptually:

```text
ND_CAST
├── lhs: ND_VAR(integer)
└── ty: double
```

3. The parser also inserts implicit casts even when the source contains no explicit cast.

```c
// File: example.c
double result = 10;
```

4. This may become:

```text
ND_ASSIGN
├── lhs: ND_VAR(result)
└── rhs: ND_CAST
    ├── lhs: ND_NUM(10)
    └── ty: double
```

5. Explicit cast nodes simplify code generation because type conversion is represented directly rather than inferred repeatedly.

### 19. `const_expr()`

```c
// File: chibicc.h
int64_t const_expr(Token **rest, Token *tok);
```

1. `const_expr()` parses and evaluates an integer constant expression beginning at `tok`.

2. `rest` receives the first token not consumed by the expression.

```c
// File: conceptual-use.c
Token *rest;
int64_t value = const_expr(&rest, tok);
tok = rest;
```

3. Integer constant expressions are required in contexts such as array lengths, enumeration values, case labels, and alignment specifications.

```c
// File: example.c
enum {
  BUFFER_SIZE = 4 * 1024
};

int buffer[BUFFER_SIZE];

switch (value) {
case 1 + 2:
  break;
}
```

4. The compiler evaluates `4 * 1024` and `1 + 2` during compilation rather than generating runtime calculations.

5. Invalid nonconstant operations produce a diagnostic.

```c
// File: example.c
int runtime_value;

enum {
  INVALID = runtime_value
};
```

### 20. `parse()`

```c
// File: chibicc.h
Obj *parse(Token *tok);
```

1. `parse()` consumes the preprocessed token stream and constructs the translation unit’s object list.

2. The returned `Obj *` generally points to a linked list containing global variables and function objects.

```text
Obj(main)
  |
 next
  v
Obj(global_counter)
  |
 next
  v
Obj(helper)
```

3. Function objects contain their AST in `body`, while global-variable objects contain initialization data and relocations.

4. The front-end flow is:

```text
Token stream
     |
     v
parse()
     |
     +----> global variable Obj
     |
     +----> function Obj
                 |
                 +----> parameter Obj list
                 +----> local Obj list
                 +----> Node AST
```

### 21. `TypeKind`

```c
// File: chibicc.h
typedef enum {
  TY_VOID,
  TY_BOOL,
  TY_CHAR,
  TY_SHORT,
  TY_INT,
  TY_LONG,
  TY_FLOAT,
  TY_DOUBLE,
  TY_LDOUBLE,
  TY_ENUM,
  TY_PTR,
  TY_FUNC,
  TY_ARRAY,
  TY_VLA,
  TY_STRUCT,
  TY_UNION,
} TypeKind;
```

1. `TypeKind` identifies the primary category of a C type. Additional properties such as signedness, array length, and base type are stored in `Type`.

### 22. Scalar Type Kinds

#### 22-1. Integer Types

1. The principal integer categories are:

| Kind | Typical C spelling |
|---|---|
| `TY_BOOL` | `_Bool` |
| `TY_CHAR` | `char`, `signed char`, `unsigned char` |
| `TY_SHORT` | `short`, `unsigned short` |
| `TY_INT` | `int`, `unsigned int` |
| `TY_LONG` | `long`, `unsigned long`, possibly implementation-related wider forms |
| `TY_ENUM` | Enumeration type |

2. Signedness is not encoded in separate enum constants. It is stored in `Type.is_unsigned`.

```text
unsigned int:
    kind = TY_INT
    is_unsigned = true

signed int:
    kind = TY_INT
    is_unsigned = false
```

3. chibicc may represent multiple source spellings using the same internal kind when their target size and code-generation behavior are shared.

#### 22-2. Floating-Point Types

1. The floating categories are:

| Kind | C spelling |
|---|---|
| `TY_FLOAT` | `float` |
| `TY_DOUBLE` | `double` |
| `TY_LDOUBLE` | `long double` |

2. Their sizes, alignments, calling conventions, and instruction forms depend on the target ABI.

#### 22-3. `TY_VOID`

1. `TY_VOID` represents the incomplete `void` type.

```c
// File: example.c
void function(void);
void *generic_pointer;
```

2. `void` itself has no ordinary object value, but `void *` is a valid object pointer type.

### 23. Derived and Aggregate Type Kinds

1. The derived and aggregate categories are:

| Kind | Meaning |
|---|---|
| `TY_PTR` | Pointer |
| `TY_FUNC` | Function type |
| `TY_ARRAY` | Fixed-length array |
| `TY_VLA` | Variable-length array |
| `TY_STRUCT` | Structure |
| `TY_UNION` | Union |

2. These types require additional fields beyond `kind`, such as `base`, `array_len`, `members`, or `return_ty`.

### 24. Common `Type` Fields

```c
// File: chibicc.h
struct Type {
  TypeKind kind;
  int size;
  int align;
  bool is_unsigned;
  bool is_atomic;
  Type *origin;

  /* Kind-specific fields follow. */
};
```

#### 24-1. `kind`

1. `kind` selects the principal type category and determines which kind-specific fields are valid.

#### 24-2. `size`

1. `size` stores the result used for `sizeof` in bytes for complete, non-VLA object types.

```c
// File: example.c
sizeof(int)
sizeof(struct Point)
```

2. A structure’s size includes internal and trailing padding required by its members and alignment.

3. For VLAs, the runtime size cannot be represented by this fixed integer alone; `vla_size` is used instead.

#### 24-3. `align`

1. `align` stores the required byte alignment.

2. If a type has alignment 8, its object address must generally satisfy:

[
\operatorname{address} \bmod 8 = 0
]

3. Structure layout, local-stack allocation, and global-data emission all rely on this value.

#### 24-4. `is_unsigned`

1. `is_unsigned` distinguishes signed and unsigned variants of integer categories.

```text
signed char:
    kind = TY_CHAR
    is_unsigned = false

unsigned char:
    kind = TY_CHAR
    is_unsigned = true
```

2. It affects integer promotion, comparisons, division, remainder, right shifts, and conversion behavior.

#### 24-5. `is_atomic`

1. `is_atomic` indicates that the type has an `_Atomic` qualification.

```c
// File: example.c
_Atomic int counter;
```

2. Accesses to such objects require atomic semantics and may be lowered to specialized AST operations such as `ND_CAS` or `ND_EXCH`.

#### 24-6. `origin`

1. `origin` points to an earlier type object from which the current type was copied or derived.

2. It assists type-compatibility checking where separately allocated `Type` objects still represent the same declared structure, union, or canonical type lineage.

3. This is useful because pointer equality alone is not always sufficient after type objects have been copied and decorated.

### 25. `base`: Pointer and Array Element Type

```c
// File: chibicc.h
Type *base;
```

1. `base` contains the pointed-to type for a pointer and the element type for an array or VLA.

```text
int *:
    kind = TY_PTR
    base = int

int[10]:
    kind = TY_ARRAY
    base = int
    array_len = 10
```

2. For nested types, `base` forms a chain.

```c
// File: example.c
int *table[4];
```

3. This declaration is an array of four pointers to `int`:

```text
TY_ARRAY
├── array_len = 4
└── base = TY_PTR
    └── base = TY_INT
```

4. A pointer to an array has the reverse nesting:

```c
// File: example.c
int (*pointer)[4];
```

5. Its structure is:

```text
TY_PTR
└── base = TY_ARRAY
    ├── array_len = 4
    └── base = TY_INT
```

6. The shared `base` field also helps implement array-to-pointer decay. In many expression contexts, an array of `T` behaves like a pointer to its first `T` element.

7. The compiler must still inspect `kind` where arrays and pointers behave differently, such as `sizeof`, `_Alignof`, string initialization, and array declaration compatibility.

### 26. Declaration Name Fields

```c
// File: chibicc.h
Token *name;
Token *name_pos;
```

#### 26-1. `name`

1. `name` points to the identifier token introduced by a declarator.

```c
// File: example.c
int *values[10];
```

2. The token for `values` is retained while the parser constructs its nested array and pointer type.

3. Abstract declarators may have no identifier.

```c
// File: example.c
sizeof(int *)
```

4. In that case, `name` can be null.

#### 26-2. `name_pos`

1. `name_pos` stores the token position where a declarator name is expected or located.

2. It supports better diagnostics for complicated declarations, including cases where the actual name is omitted or nested inside parentheses.

```c
// File: example.c
int (*handler)(double);
```

3. Declarator parsing is recursive, so retaining a representative name position helps associate errors with the correct part of the declaration.

### 27. Fixed-Length Array Fields

```c
// File: chibicc.h
int array_len;
```

1. `array_len` stores the compile-time number of elements for `TY_ARRAY`.

```c
// File: example.c
int values[20];
```

2. Its type is represented as:

```text
kind      = TY_ARRAY
base      = int
array_len = 20
size      = 20 * sizeof(int)
```

3. The total size is generally:

[
\operatorname{size}(\text{array})
=
\text{array\_len}
\times
\operatorname{size}(\text{base})
]

4. An incomplete array may use a special length value until an initializer or later declaration supplies its size.

```c
// File: example.c
extern int values[];
```

### 28. Variable-Length Array Fields

```c
// File: chibicc.h
Node *vla_len;
Obj *vla_size;
```

#### 28-1. `vla_len`

1. `vla_len` stores the runtime expression that calculates the number of elements.

```c
// File: example.c
void process(int count) {
  int values[count + 1];
}
```

2. The VLA type is conceptually:

```text
kind    = TY_VLA
base    = int
vla_len = ND_ADD(ND_VAR(count), ND_NUM(1))
```

#### 28-2. `vla_size`

1. `vla_size` points to a compiler-generated local variable holding the computed byte size of the VLA type.

2. The runtime size is conceptually:

[
\text{VLA byte size}
=
\text{element count}
\times
\operatorname{size}(\text{base})
]

3. Saving the result avoids repeatedly evaluating the original length expression, which may have side effects.

```c
// File: example.c
int next_length(void);

void function(void) {
  int values[next_length()];
}
```

4. `next_length()` must not be invoked repeatedly whenever the compiler needs the VLA size.

### 29. Structure and Union Fields

```c
// File: chibicc.h
Member *members;
bool is_flexible;
bool is_packed;
```

#### 29-1. `members`

1. `members` points to the linked list describing a structure or union’s fields.

```c
// File: example.c
struct Record {
  char tag;
  int value;
  double ratio;
};
```

2. The conceptual list is:

```text
tag -> value -> ratio -> NULL
```

3. For a structure, members receive distinct offsets with padding as necessary. For a union, members generally begin at offset zero.

#### 29-2. `is_flexible`

1. `is_flexible` indicates that the structure ends with a flexible array member.

```c
// File: example.c
struct Packet {
  size_t length;
  unsigned char data[];
};
```

2. The flexible array contributes no fixed elements to the nominal structure size, but additional storage may follow an allocated object.

```c
// File: example.c
struct Packet *packet =
  malloc(sizeof(struct Packet) + payload_size);
```

3. Only the final member of an eligible structure can be a flexible array member.

#### 29-3. `is_packed`

1. `is_packed` indicates that ordinary padding and alignment rules have been modified by a packed-layout extension.

```c
// File: example.c
struct __attribute__((packed)) Header {
  char tag;
  int value;
};
```

2. Without packing, padding may be inserted before `value`; with packing, `value` may begin immediately after `tag`.

3. Packed fields can become unaligned, so generated loads and stores must remain valid for the target architecture.

### 30. Function-Type Fields

```c
// File: chibicc.h
Type *return_ty;
Type *params;
bool is_variadic;
Type *next;
```

#### 30-1. `return_ty`

1. `return_ty` contains the function’s result type.

```c
// File: example.c
long calculate(int input);
```

2. Its function type contains:

```text
kind      = TY_FUNC
return_ty = TY_LONG
```

#### 30-2. `params`

1. `params` points to the first parameter type. Additional parameter types are connected through `Type.next`.

```c
// File: example.c
long calculate(int count, double ratio, char *name);
```

2. The list is conceptually:

```text
TY_INT
  |
 next
  v
TY_DOUBLE
  |
 next
  v
TY_PTR(base = TY_CHAR)
```

3. Parameter type adjustment occurs when required by C. An array parameter, for example, is adjusted to a pointer type.

```c
// File: example.c
void process(int values[10]);
```

4. As a function parameter, `values` is adjusted to `int *`; the bound does not make the parameter an array object.

#### 30-3. `is_variadic`

1. `is_variadic` indicates that the function accepts unnamed arguments after its declared parameters.

```c
// File: example.c
int printf(const char *format, ...);
```

2. The function type is approximately:

```text
return_ty   = int
params      = pointer to const char
is_variadic = true
```

3. Variadic arguments undergo default argument promotions because no parameter type is available for each unnamed argument.

#### 30-4. `next`

1. `next` links types in a function’s parameter list.

2. It is not part of the semantic identity of an isolated type in every context; it primarily gives the type object a position in a list.

### 31. Example Type Representations

#### 31-1. Pointer to Integer

```c
// File: example.c
int *pointer;
```

1. Its type graph is:

```text
TY_PTR
└── base: TY_INT
```

#### 31-2. Array of Pointers

```c
// File: example.c
char *names[8];
```

1. Its type graph is:

```text
TY_ARRAY
├── array_len: 8
└── base: TY_PTR
    └── base: TY_CHAR
```

#### 31-3. Pointer to Function

```c
// File: example.c
double (*operation)(int, long);
```

1. Its type graph is:

```text
TY_PTR
└── base: TY_FUNC
    ├── return_ty: TY_DOUBLE
    └── params:
        TY_INT -> TY_LONG
```

#### 31-4. Function Returning a Pointer

```c
// File: example.c
double *operation(int, long);
```

1. This differs from the previous declaration:

```text
TY_FUNC
├── return_ty: TY_PTR
│   └── base: TY_DOUBLE
└── params:
    TY_INT -> TY_LONG
```

2. The declarator syntax differs only in parentheses, but the resulting type graphs are structurally different.

### 32. Relationship Between `Node` and `Type`

1. Parsing first determines the syntactic AST shape, while type analysis fills or validates each expression node’s `ty`.

2. For an addition expression:

```c
// File: example.c
long left;
int right;

left + right;
```

3. The typed AST is approximately:

```text
ND_ADD: TY_LONG
├── lhs: ND_VAR(left), TY_LONG
└── rhs: ND_CAST, TY_LONG
    └── ND_VAR(right), TY_INT
```

4. The inserted `ND_CAST` makes the integer conversion explicit.

5. Type information then determines code generation details such as operation width, signedness, floating-point instructions, pointer scaling, and calling-convention behavior.

### 33. Front-End Data Flow

1. Tokenization converts source characters into preprocessing tokens.

2. Preprocessing expands macros, processes directives, and produces parser-ready tokens.

3. `parse()` constructs `Obj`, `Node`, and `Type` objects.

4. Type analysis annotates expression nodes and inserts conversions through nodes such as `ND_CAST`.

5. Constant-expression evaluation processes compile-time-required expressions.

6. Code generation traverses function `Obj` objects and their typed `Node` trees.

```text
Source text
    |
    v
Token list
    |
    v
Preprocessed token list
    |
    v
parse()
    |
    +-- Obj list
    |     |
    |     +-- global initialization
    |     +-- function metadata
    |
    +-- Node AST
    |
    +-- Type graph
          |
          v
Typed code generation
```

User:
the following code continues from the above.
---

// Struct member
struct Member {
  Member *next;
  Type *ty;
  Token *tok; // for error message
  Token *name;
  int idx;
  int align;
  int offset;

  // Bitfield
  bool is_bitfield;
  int bit_offset;
  int bit_width;
};

extern Type *ty_void;
extern Type *ty_bool;

extern Type *ty_char;
extern Type *ty_short;
extern Type *ty_int;
extern Type *ty_long;

extern Type *ty_uchar;
extern Type *ty_ushort;
extern Type *ty_uint;
extern Type *ty_ulong;

extern Type *ty_float;
extern Type *ty_double;
extern Type *ty_ldouble;

bool is_integer(Type *ty);
bool is_flonum(Type *ty);
bool is_numeric(Type *ty);
bool is_compatible(Type *t1, Type *t2);
Type *copy_type(Type *ty);
Type *pointer_to(Type *base);
Type *func_type(Type *return_ty);
Type *array_of(Type *base, int size);
Type *vla_of(Type *base, Node *expr);
Type *enum_type(void);
Type *struct_type(void);
void add_type(Node *node);

//
// codegen.c
//

void codegen(Obj *prog, FILE *out);
int align_to(int n, int align);

//
// unicode.c
//

int encode_utf8(char *buf, uint32_t c);
uint32_t decode_utf8(char **new_pos, char *p);
bool is_ident1(uint32_t c);
bool is_ident2(uint32_t c);
int display_width(char *p, int len);

//
// hashmap.c
//

typedef struct {
  char *key;
  int keylen;
  void *val;
} HashEntry;

typedef struct {
  HashEntry *buckets;
  int capacity;
  int used;
} HashMap;

void *hashmap_get(HashMap *map, char *key);
void *hashmap_get2(HashMap *map, char *key, int keylen);
void hashmap_put(HashMap *map, char *key, void *val);
void hashmap_put2(HashMap *map, char *key, int keylen, void *val);
void hashmap_delete(HashMap *map, char *key);
void hashmap_delete2(HashMap *map, char *key, int keylen);
void hashmap_test(void);

//
// main.c
//

bool file_exists(char *path);

extern StringArray include_paths;
extern bool opt_fpic;
extern bool opt_fcommon;
extern char *base_file;

Assistant:
### 1. `Member`: Structure and Union Member Metadata

```c
// File: chibicc.h
struct Member {
  Member *next;
  Type *ty;
  Token *tok;
  Token *name;
  int idx;
  int align;
  int offset;

  bool is_bitfield;
  int bit_offset;
  int bit_width;
};
```

1. `Member` describes one field of a `struct` or `union`. A `Type` with `kind == TY_STRUCT` or `TY_UNION` points to the first member through `Type.members`.

```c
// File: example.c
struct Record {
  char tag;
  int value;
  double ratio;
};
```

2. The corresponding metadata forms a linked list:

```text
Type(TY_STRUCT).members
    |
    v
Member(tag) -> Member(value) -> Member(ratio) -> NULL
```

### 2. `Member.next`

```c
// File: chibicc.h
Member *next;
```

1. `next` points to the next member in declaration order. The final member has `next == NULL`.

2. Declaration order must be preserved because structure layout, initialization, and member iteration depend on it.

```c
// File: conceptual-member-iteration.c
for (Member *mem = ty->members; mem; mem = mem->next) {
  /* Process members in declaration order. */
}
```

### 3. `Member.ty`

```c
// File: chibicc.h
Type *ty;
```

1. `ty` describes the member’s complete type.

```c
// File: example.c
struct Object {
  int count;
  char *name;
  int values[8];
};
```

2. The member types are approximately:

```text
count  -> TY_INT
name   -> TY_PTR(base = TY_CHAR)
values -> TY_ARRAY(base = TY_INT, array_len = 8)
```

3. The compiler uses this field to calculate member size, alignment, access instructions, initialization rules, and nested member expressions.

### 4. `Member.tok` and `Member.name`

```c
// File: chibicc.h
Token *tok;
Token *name;
```

1. `tok` is a representative source token retained for diagnostics. It allows errors discovered during structure layout or semantic analysis to point to the member declaration.

```c
// File: example.c
struct Invalid {
  void value;
};
```

2. A diagnostic can use `tok` to identify the declaration of `value`.

3. `name` specifically points to the member’s identifier token.

```c
// File: example.c
struct Point {
  int x;
  int y;
};
```

4. Member lookup conceptually compares an access token against each `Member.name`.

```c
// File: conceptual-member-lookup.c
for (Member *mem = ty->members; mem; mem = mem->next) {
  if (mem->name && equal(member_name_token, mem->name->loc))
    return mem;
}
```

5. The real implementation must compare token lengths as well as bytes because token spellings are not necessarily null-terminated.

6. Anonymous structure or union members may not have an ordinary identifier, so `name` can be null in applicable extension-supported cases.

### 5. `Member.idx`

```c
// File: chibicc.h
int idx;
```

1. `idx` records the member’s zero-based declaration index.

```c
// File: example.c
struct Triple {
  int first;
  int second;
  int third;
};
```

2. Its members are indexed approximately as:

```text
first  -> idx 0
second -> idx 1
third  -> idx 2
```

3. The index is useful when matching positional initializers to members.

```c
// File: example.c
struct Triple value = {10, 20, 30};
```

4. The initializer values correspond to members by index:

```text
initializer[0] -> first
initializer[1] -> second
initializer[2] -> third
```

### 6. `Member.align`

```c
// File: chibicc.h
int align;
```

1. `align` stores the effective alignment requirement for the member. It normally derives from `mem->ty->align` but can be changed by alignment attributes or packed layout.

2. A structure member’s offset is rounded up to satisfy its alignment:

[
\operatorname{offset}
=
\operatorname{align\_to}(\operatorname{current\ size},\operatorname{member\ alignment})
]

3. For example, assuming `char` has alignment 1 and `int` has alignment 4:

```c
// File: example.c
struct Example {
  char tag;
  int value;
};
```

4. A typical layout is:

```text
Offset 0: tag
Offset 1: padding
Offset 2: padding
Offset 3: padding
Offset 4: value
```

5. `value.align` is 4, so its offset is rounded from 1 to 4.

### 7. `Member.offset`

```c
// File: chibicc.h
int offset;
```

1. `offset` is the byte displacement of the member from the beginning of its containing structure.

```c
// File: example.c
struct Record {
  char tag;
  int value;
  double ratio;
};
```

2. A possible LP64 layout is:

```text
tag.offset   = 0
value.offset = 4
ratio.offset = 8
```

3. An `ND_MEMBER` expression uses this value to calculate the field address:

[
\operatorname{address}(\text{object.member})
=
\operatorname{address}(\text{object})
+
\text{member.offset}
]

4. For a union, ordinary members generally have offset zero because they overlap.

```c
// File: example.c
union Value {
  int integer;
  double floating;
};
```

```text
integer.offset  = 0
floating.offset = 0
```

### 8. Bit-Field Metadata

```c
// File: chibicc.h
bool is_bitfield;
int bit_offset;
int bit_width;
```

1. These fields describe members declared as bit-fields.

```c
// File: example.c
struct Flags {
  unsigned int readable : 1;
  unsigned int writable : 1;
  unsigned int mode     : 3;
};
```

#### 8-1. `is_bitfield`

1. `is_bitfield` distinguishes a bit-field from an ordinary member.

```text
readable.is_bitfield = true
writable.is_bitfield = true
mode.is_bitfield     = true
```

2. Ordinary members have `is_bitfield == false`; their location is determined only by byte offset and type size.

#### 8-2. `bit_offset`

1. `bit_offset` identifies the first bit occupied by the member within its storage unit or layout region, according to chibicc’s target-layout convention.

2. A possible conceptual layout is:

```text
Bit 0: readable
Bit 1: writable
Bit 2-4: mode
```

```text
readable.bit_offset = 0
writable.bit_offset = 1
mode.bit_offset     = 2
```

#### 8-3. `bit_width`

1. `bit_width` stores the number of bits assigned to the member.

```text
readable.bit_width = 1
writable.bit_width = 1
mode.bit_width     = 3
```

2. Reading a bit-field conceptually requires loading its storage unit, shifting, and masking:

[
\text{value}
=
(\text{storage} \gg \text{bit offset})
\mathbin{\&}
(2^{\text{bit width}}-1)
]

3. Writing a bit-field conceptually preserves the surrounding bits:

[
\text{new storage}
=
(\text{old storage} \mathbin{\&} \sim\text{field mask})
\mathbin{|}
((\text{new value} \ll \text{bit offset}) \mathbin{\&} \text{field mask})
]

4. Signed bit-fields additionally require sign extension when read.

### 9. Canonical Built-In Type Objects

```c
// File: chibicc.h
extern Type *ty_void;
extern Type *ty_bool;

extern Type *ty_char;
extern Type *ty_short;
extern Type *ty_int;
extern Type *ty_long;

extern Type *ty_uchar;
extern Type *ty_ushort;
extern Type *ty_uint;
extern Type *ty_ulong;

extern Type *ty_float;
extern Type *ty_double;
extern Type *ty_ldouble;
```

1. These declarations expose shared `Type` objects representing fundamental C types. `extern` declares the variables here while their definitions exist in one implementation file, normally `type.c`.

2. A declaration such as:

```c
// File: chibicc.h
extern Type *ty_int;
```

3. Is paired with one definition conceptually similar to:

```c
// File: type.c
Type *ty_int = /* address of canonical integer type */;
```

4. The header must not provide another ordinary definition, or including it in multiple translation units could cause multiple-definition linker errors.

### 10. Signed Built-In Types

1. The principal signed type objects are:

| Object | C type represented |
|---|---|
| `ty_char` | Plain `char` |
| `ty_short` | `short` |
| `ty_int` | `int` |
| `ty_long` | `long` |
| `ty_bool` | `_Bool` |
| `ty_void` | `void` |

2. Their `Type` records contain target-specific size and alignment information.

3. On the target ABI used by chibicc, the values may conceptually resemble:

```text
ty_char:  size 1, align 1
ty_short: size 2, align 2
ty_int:   size 4, align 4
ty_long:  size 8, align 8
```

4. These values are target properties rather than universal C requirements.

5. Plain `char` is distinct from both `signed char` and `unsigned char` at the language level, even when it shares representation with one of them.

### 11. Unsigned Built-In Types

1. The unsigned type objects are:

| Object | C type represented |
|---|---|
| `ty_uchar` | `unsigned char` |
| `ty_ushort` | `unsigned short` |
| `ty_uint` | `unsigned int` |
| `ty_ulong` | `unsigned long` |

2. Their `kind` values can be the same as their signed counterparts while `is_unsigned` distinguishes them.

```text
ty_int:
    kind = TY_INT
    is_unsigned = false

ty_uint:
    kind = TY_INT
    is_unsigned = true
```

3. Signedness affects comparisons and arithmetic instructions.

```c
// File: example.c
unsigned int left;
unsigned int right;
bool result = left < right;
```

4. The backend must generate an unsigned comparison rather than a signed comparison.

### 12. Floating-Point Built-In Types

| Object | C type represented |
|---|---|
| `ty_float` | `float` |
| `ty_double` | `double` |
| `ty_ldouble` | `long double` |

1. These objects allow the parser and type checker to refer to canonical floating-point types without repeatedly allocating equivalent `Type` structures.

2. They determine storage size, alignment, conversion behavior, register classification, and generated instruction selection.

```c
// File: example.c
float first;
double second;
long double third;
```

### 13. Benefits and Restrictions of Canonical Types

1. Shared type objects reduce allocation and make common comparisons inexpensive.

```c
// File: conceptual-type-check.c
if (node->ty == ty_int) {
  /* Exact canonical int type. */
}
```

2. Pointer equality is not sufficient for every type-compatibility test because copied, qualified, derived, structure, and function types may be represented by separate objects.

3. The canonical objects should not be modified directly when a declaration needs properties such as `_Atomic`. The compiler can copy the type first.

```c
// File: conceptual-atomic-type.c
Type *ty = copy_type(ty_int);
ty->is_atomic = true;
```

4. Modifying `ty_int->is_atomic` directly would incorrectly make every subsequent `int` atomic.

### 14. `is_integer()`

```c
// File: chibicc.h
bool is_integer(Type *ty);
```

1. `is_integer()` reports whether a type belongs to an integer category.

2. It normally recognizes categories such as:

```text
TY_BOOL
TY_CHAR
TY_SHORT
TY_INT
TY_LONG
TY_ENUM
```

3. Example use:

```c
// File: conceptual-type-check.c
if (!is_integer(node->ty))
  error_tok(node->tok, "an integer operand is required");
```

4. This predicate is useful for `%`, bitwise operators, shifts, array indexes, and many constant-expression checks.

### 15. `is_flonum()`

```c
// File: chibicc.h
bool is_flonum(Type *ty);
```

1. `is_flonum()` reports whether a type is floating-point.

2. It recognizes:

```text
TY_FLOAT
TY_DOUBLE
TY_LDOUBLE
```

3. Example:

```c
// File: conceptual-codegen.c
if (is_flonum(node->ty)) {
  /* Generate floating-point operation. */
} else {
  /* Generate integer or pointer operation. */
}
```

### 16. `is_numeric()`

```c
// File: chibicc.h
bool is_numeric(Type *ty);
```

1. `is_numeric()` combines the integer and floating-point categories.

2. Its conceptual definition is:

```c
// File: conceptual-type.c
bool is_numeric(Type *ty) {
  return is_integer(ty) || is_flonum(ty);
}
```

3. Arithmetic operators such as `+`, `-`, `*`, and `/` use this predicate when validating ordinary arithmetic operands, although pointer arithmetic needs separate handling.

### 17. `is_compatible()`

```c
// File: chibicc.h
bool is_compatible(Type *t1, Type *t2);
```

1. `is_compatible()` implements C type-compatibility rules. Compatibility is more involved than comparing `kind`, size, or pointer identity.

2. Two pointer types are compatible when their pointed-to types satisfy the required compatibility rules.

```c
// File: example.c
int *first;
int *second;
```

3. Their type graphs may be different allocations but still describe compatible types.

4. Array compatibility can depend on element type and known bounds.

```c
// File: example.c
extern int values[];
extern int values[10];
```

5. Function compatibility depends on return type, parameter types, variadic status, and declaration form.

```c
// File: example.c
int function(int);
int function(int value);
```

6. The `origin` field in `Type` helps recognize copied types that originated from the same declaration identity.

### 18. `copy_type()`

```c
// File: chibicc.h
Type *copy_type(Type *ty);
```

1. `copy_type()` creates a copy of a type descriptor so declaration-specific properties can be changed without mutating a shared type.

```c
// File: conceptual-type-copy.c
Type *qualified = copy_type(ty_int);
qualified->is_atomic = true;
```

2. This is particularly important for canonical objects such as `ty_int`.

3. The function is generally a structural copy of the top-level `Type`; nested objects such as `base` may remain shared unless deeper copying is required.

4. A copied top-level type can therefore change its own flags without duplicating the entire type graph.

### 19. `pointer_to()`

```c
// File: chibicc.h
Type *pointer_to(Type *base);
```

1. `pointer_to()` constructs a pointer type whose `base` is the supplied type.

```c
// File: conceptual-type-construction.c
Type *int_pointer = pointer_to(ty_int);
```

2. The resulting graph is:

```text
Type
├── kind = TY_PTR
├── base = ty_int
├── size = target pointer size
└── align = target pointer alignment
```

3. Nested pointers are built by repeated calls.

```c
// File: conceptual-type-construction.c
Type *int_pointer = pointer_to(ty_int);
Type *int_pointer_pointer = pointer_to(int_pointer);
```

4. This represents `int **`.

### 20. `func_type()`

```c
// File: chibicc.h
Type *func_type(Type *return_ty);
```

1. `func_type()` creates a function type with the supplied return type.

```c
// File: conceptual-type-construction.c
Type *function_ty = func_type(ty_long);
```

2. Parameter types and variadic status are filled later while parsing the function declarator.

```text
kind        = TY_FUNC
return_ty   = ty_long
params      = parameter type list
is_variadic = true or false
```

3. For this declaration:

```c
// File: example.c
long calculate(int, double);
```

4. The resulting type is approximately:

```text
TY_FUNC
├── return_ty: TY_LONG
└── params:
    TY_INT -> TY_DOUBLE
```

### 21. `array_of()`

```c
// File: chibicc.h
Type *array_of(Type *base, int size);
```

1. `array_of()` creates a fixed-length array type.

```c
// File: conceptual-type-construction.c
Type *array_ty = array_of(ty_int, 10);
```

2. The result represents `int[10]`:

```text
kind      = TY_ARRAY
base      = ty_int
array_len = 10
size      = 10 * ty_int->size
align     = ty_int->align
```

3. Nested arrays are created inside out.

```c
// File: conceptual-type-construction.c
Type *row = array_of(ty_int, 4);
Type *matrix = array_of(row, 3);
```

4. `matrix` represents `int[3][4]`.

### 22. `vla_of()`

```c
// File: chibicc.h
Type *vla_of(Type *base, Node *expr);
```

1. `vla_of()` creates a variable-length array type whose element count is determined at runtime by `expr`.

```c
// File: example.c
void function(int count) {
  int values[count];
}
```

2. Its type is approximately:

```text
kind    = TY_VLA
base    = ty_int
vla_len = ND_VAR(count)
```

3. Unlike `array_of()`, the total byte size cannot always be stored as a compile-time integer.

4. The parser later generates code to calculate:

[
\text{VLA size}
=
\text{runtime element count}
\times
\operatorname{sizeof}(\text{base})
]

### 23. `enum_type()`

```c
// File: chibicc.h
Type *enum_type(void);
```

1. `enum_type()` creates a fresh enumeration type.

```c
// File: example.c
enum Color {
  COLOR_RED,
  COLOR_GREEN,
  COLOR_BLUE
};
```

2. Enumeration constants have integer values, but the enumeration itself retains a distinct type identity needed for compatibility checking.

3. A fresh type object is appropriate because unrelated enumeration declarations are not the same declared type merely because they use the same representation.

### 24. `struct_type()`

```c
// File: chibicc.h
Type *struct_type(void);
```

1. `struct_type()` creates an initially incomplete structure type.

```c
// File: example.c
struct Node;
```

2. At this point, the compiler knows the type exists but does not yet know its members or size.

3. When the definition is parsed, the same type is completed:

```c
// File: example.c
struct Node {
  int value;
  struct Node *next;
};
```

4. Completion conceptually fills fields such as:

```text
members
size
align
is_flexible
is_packed
```

5. Although the function is named `struct_type()`, the implementation may also be adapted or copied for unions by changing `kind` to `TY_UNION`.

### 25. `add_type()`

```c
// File: chibicc.h
void add_type(Node *node);
```

1. `add_type()` performs semantic type annotation over an AST node and its children.

2. It fills `node->ty`, validates operand combinations, applies language rules, and may insert implicit conversions.

```c
// File: example.c
long left;
int right;
long result = left + right;
```

3. Before complete typing, the AST may be:

```text
ND_ADD
├── ND_VAR(left):  TY_LONG
└── ND_VAR(right): TY_INT
```

4. After type analysis, the result is conceptually:

```text
ND_ADD: TY_LONG
├── ND_VAR(left): TY_LONG
└── ND_CAST: TY_LONG
    └── ND_VAR(right): TY_INT
```

5. Type analysis also implements pointer arithmetic.

```c
// File: example.c
int *pointer;
pointer + 3;
```

6. The result has type `int *`, and code generation scales `3` by `sizeof(int)`.

7. `add_type()` can safely return early when a node has already been annotated, depending on the implementation’s traversal design.

### 26. Type Helper Summary

| Function | Primary result |
|---|---|
| `is_integer()` | Tests integer categories |
| `is_flonum()` | Tests floating-point categories |
| `is_numeric()` | Tests arithmetic categories |
| `is_compatible()` | Applies C type-compatibility rules |
| `copy_type()` | Copies a top-level type descriptor |
| `pointer_to()` | Constructs `TY_PTR` |
| `func_type()` | Constructs `TY_FUNC` |
| `array_of()` | Constructs fixed `TY_ARRAY` |
| `vla_of()` | Constructs runtime-sized `TY_VLA` |
| `enum_type()` | Constructs a fresh `TY_ENUM` |
| `struct_type()` | Constructs an incomplete structure type |
| `add_type()` | Annotates and validates AST nodes |

### 27. `codegen()`

```c
// File: chibicc.h
void codegen(Obj *prog, FILE *out);
```

1. `codegen()` converts the parsed and typed program into target assembly text written to `out`.

2. `prog` points to the linked list of global variables and functions returned by `parse()`.

```c
// File: conceptual-driver.c
Obj *prog = parse(tokens);
codegen(prog, output_file);
```

3. Its high-level responsibilities include:

   1. Emitting global variables and initializers.
   2. Emitting relocatable symbol references.
   3. Assigning local-variable stack offsets.
   4. Generating function prologues and epilogues.
   5. Evaluating expression and statement AST nodes.
   6. Implementing the target calling convention.
   7. Emitting source-location metadata where supported.

4. The pipeline is:

```text
Obj list and typed AST
          |
          v
      codegen()
          |
          v
    Assembly text
          |
          v
       Assembler
          |
          v
      Object file
```

5. `FILE *out` allows output to a regular file, temporary file, or another C stream.

```c
// File: conceptual-output.c
FILE *out = fopen("output.s", "w");

if (!out)
  error("cannot open output.s: %s", strerror(errno));

codegen(prog, out);

if (fclose(out) != 0)
  error("cannot close output.s: %s", strerror(errno));
```

### 28. `align_to()`

```c
// File: chibicc.h
int align_to(int n, int align);
```

1. `align_to()` rounds `n` upward to the next multiple of `align`.

2. For a positive power-of-two alignment, the common formula is:

[
\operatorname{align\_to}(n,a)
=
\frac{n+a-1}{a}\times a
]

3. Examples are:

```text
align_to(0, 8)  = 0
align_to(1, 8)  = 8
align_to(8, 8)  = 8
align_to(9, 8)  = 16
align_to(17, 8) = 24
```

4. Structure layout uses it when assigning member offsets.

```c
// File: conceptual-struct-layout.c
offset = align_to(offset, member->align);
member->offset = offset;
offset += member->ty->size;
```

5. Stack layout also uses it to satisfy the ABI’s required stack alignment.

```c
// File: conceptual-stack-layout.c
function->stack_size = align_to(required_size, 16);
```

6. Callers must provide a valid positive alignment; zero alignment would make the arithmetic invalid.

### 29. Unicode Support Overview

```c
// File: chibicc.h
int encode_utf8(char *buf, uint32_t c);
uint32_t decode_utf8(char **new_pos, char *p);
bool is_ident1(uint32_t c);
bool is_ident2(uint32_t c);
int display_width(char *p, int len);
```

1. These functions allow chibicc to tokenize Unicode identifiers, process universal character values, create UTF-8 text, and align diagnostics correctly.

2. They work with Unicode code points represented by `uint32_t`.

### 30. `encode_utf8()`

```c
// File: chibicc.h
int encode_utf8(char *buf, uint32_t c);
```

1. `encode_utf8()` writes the UTF-8 encoding of code point `c` into `buf` and returns the number of bytes written.

2. UTF-8 uses between one and four bytes for valid Unicode scalar values.

| Code-point range | Encoded length |
|---|---:|
| `U+0000`–`U+007F` | 1 byte |
| `U+0080`–`U+07FF` | 2 bytes |
| `U+0800`–`U+FFFF` | 3 bytes |
| `U+10000`–`U+10FFFF` | 4 bytes |

3. Example usage is:

```c
// File: conceptual-unicode.c
char buf[4];
int len = encode_utf8(buf, 0x03BB);
```

4. `U+03BB` is Greek small letter lambda, encoded as two UTF-8 bytes.

5. The function writes bytes, not necessarily a terminating null character, so the caller tracks the returned length.

### 31. `decode_utf8()`

```c
// File: chibicc.h
uint32_t decode_utf8(char **new_pos, char *p);
```

1. `decode_utf8()` reads one UTF-8 sequence beginning at `p` and returns its Unicode code point.

2. `*new_pos` is updated to point immediately after the decoded sequence.

```c
// File: conceptual-unicode.c
char *p = input;
char *next;
uint32_t codepoint = decode_utf8(&next, p);
p = next;
```

3. The caller can iterate over code points without assuming one byte per character.

```c
// File: conceptual-unicode-iteration.c
for (char *p = input; *p;) {
  uint32_t c = decode_utf8(&p, p);
  /* Process c. */
}
```

4. Invalid UTF-8 input must be diagnosed or handled according to the implementation’s policy.

### 32. `is_ident1()`

```c
// File: chibicc.h
bool is_ident1(uint32_t c);
```

1. `is_ident1()` checks whether code point `c` is permitted as the first character of a C identifier.

2. ASCII examples normally accepted in the first position include:

```text
A-Z
a-z
_
```

3. Digits are not valid in the first position.

```text
value1   -> valid
1value   -> not an identifier beginning
```

4. The function also recognizes applicable non-ASCII Unicode ranges supported by chibicc.

### 33. `is_ident2()`

```c
// File: chibicc.h
bool is_ident2(uint32_t c);
```

1. `is_ident2()` checks whether a code point is permitted after the first identifier character.

2. It normally accepts everything accepted by `is_ident1()` plus decimal digits and other allowed continuation characters.

```c
// File: example.c
int value2;
```

3. Tokenization conceptually follows:

```c
// File: conceptual-identifier-scan.c
if (is_ident1(first_codepoint)) {
  while (is_ident2(next_codepoint))
    /* Continue scanning. */;
}
```

### 34. `display_width()`

```c
// File: chibicc.h
int display_width(char *p, int len);
```

1. `display_width()` calculates the terminal display width of a UTF-8 byte sequence.

2. Byte length, Unicode code-point count, and terminal column width are different measurements.

```text
ASCII "A":
    UTF-8 bytes = 1
    code points = 1
    display width = 1

A typical wide CJK character:
    UTF-8 bytes = 3
    code points = 1
    display width = 2
```

3. Diagnostics need display width to place a caret under the correct source column.

```text
source.c:1: error: invalid token
変数 = @;
       ^
```

4. Counting bytes alone would misalign the caret after multibyte or wide characters.

5. Exact terminal rendering can vary, but this function applies the width model selected by chibicc.

### 35. `HashEntry`

```c
// File: chibicc.h
typedef struct {
  char *key;
  int keylen;
  void *val;
} HashEntry;
```

1. `HashEntry` stores one key-value association in chibicc’s generic hash map.

2. Keys are represented by a pointer and explicit length rather than requiring null termination.

3. Values use `void *`, allowing the map to store different compiler objects.

```text
Macro name     -> Macro *
Variable name  -> VarScope *
Struct tag     -> Type *
Identifier     -> Obj *
```

### 36. `HashEntry.key` and `keylen`

```c
// File: chibicc.h
char *key;
int keylen;
```

1. `key` points to the key’s first byte, while `keylen` gives its byte length.

2. This matches chibicc’s token representation, where `Token.loc` and `Token.len` identify text without requiring a null terminator.

```c
// File: conceptual-token-lookup.c
void *value = hashmap_get2(
  &map,
  token->loc,
  token->len
);
```

3. Explicit lengths also permit lookup of substrings within a larger source buffer.

4. Keys must remain valid for as long as the map relies on their stored pointers unless the implementation copies them internally.

### 37. `HashEntry.val`

```c
// File: chibicc.h
void *val;
```

1. `val` stores an untyped object pointer.

```c
// File: conceptual-map-use.c
hashmap_put(&map, "counter", variable_object);

Obj *obj = hashmap_get(&map, "counter");
```

2. C implicitly converts between `void *` and object-pointer types, but the caller remains responsible for using the correct expected type.

3. A mismatched cast or interpretation is not detected by the C type system.

### 38. `HashMap`

```c
// File: chibicc.h
typedef struct {
  HashEntry *buckets;
  int capacity;
  int used;
} HashMap;
```

1. `HashMap` stores the table’s bucket array and occupancy metadata.

#### 38-1. `buckets`

1. `buckets` points to a dynamically allocated array of `HashEntry`.

2. The hash value determines an initial bucket, while collision resolution is handled by the implementation.

```text
buckets[0]
buckets[1] -> {"name", value}
buckets[2]
buckets[3] -> {"count", value}
...
```

#### 38-2. `capacity`

1. `capacity` is the number of allocated bucket slots.

2. When occupancy becomes too high, the map increases its capacity and reinserts entries according to the new bucket count.

3. Resizing keeps average lookup and insertion time close to constant under ordinary conditions.

#### 38-3. `used`

1. `used` tracks occupied or consumed bucket slots according to the implementation’s deletion policy.

2. It helps determine the load factor:

[
\text{load factor}
=
\frac{\text{used}}{\text{capacity}}
]

3. A high load factor increases collisions and probe lengths, so the table is resized before it becomes full.

### 39. Null-Terminated-Key Hash-Map Functions

```c
// File: chibicc.h
void *hashmap_get(HashMap *map, char *key);
void hashmap_put(HashMap *map, char *key, void *val);
void hashmap_delete(HashMap *map, char *key);
```

1. These convenience functions accept ordinary null-terminated C strings.

2. `hashmap_get()` returns the associated value or null when the key is absent.

```c
// File: conceptual-map-use.c
Obj *obj = hashmap_get(&variables, "counter");

if (obj) {
  /* Found counter. */
}
```

3. `hashmap_put()` inserts a new key-value pair or replaces the value associated with an existing key.

```c
// File: conceptual-map-use.c
hashmap_put(&variables, "counter", counter_obj);
```

4. `hashmap_delete()` removes the entry if present.

```c
// File: conceptual-map-use.c
hashmap_delete(&variables, "counter");
```

5. If null is also a meaningful stored value, `hashmap_get()` alone cannot distinguish “missing key” from “present key with null value.” Compiler tables typically avoid storing ambiguous null values where that distinction matters.

### 40. Explicit-Length Hash-Map Functions

```c
// File: chibicc.h
void *hashmap_get2(HashMap *map, char *key, int keylen);
void hashmap_put2(
  HashMap *map,
  char *key,
  int keylen,
  void *val
);
void hashmap_delete2(HashMap *map, char *key, int keylen);
```

1. The `2` variants accept pointer-and-length keys.

2. They avoid allocating temporary null-terminated strings when the key comes directly from a token.

```c
// File: conceptual-scope-lookup.c
Obj *obj = hashmap_get2(
  &scope,
  tok->loc,
  tok->len
);
```

3. The ordinary variants can be understood as wrappers:

```c
// File: conceptual-hashmap-wrapper.c
void *hashmap_get(HashMap *map, char *key) {
  return hashmap_get2(map, key, strlen(key));
}
```

4. The real implementation may differ, but the public behavior follows this relationship.

### 41. Hash-Map Use in a Compiler

1. A hash map is suitable for macro lookup because preprocessing repeatedly asks whether an identifier names a macro.

```c
// File: conceptual-preprocessor.c
Macro *macro = hashmap_get2(
  &macros,
  tok->loc,
  tok->len
);
```

2. It is also suitable for scope lookup.

```c
// File: conceptual-parser.c
Obj *var = hashmap_get2(
  &current_scope,
  tok->loc,
  tok->len
);
```

3. Compared with scanning a linked list, a hash map generally reduces average lookup cost from linear time to approximately constant time.

[
\text{linked-list lookup: } O(n)
]

[
\text{average hash lookup: } O(1)
]

4. Worst-case hash-map behavior can still be linear when many keys collide.

### 42. `hashmap_test()`

```c
// File: chibicc.h
void hashmap_test(void);
```

1. `hashmap_test()` runs the hash-map implementation’s internal tests.

2. Such tests commonly verify:

   1. Insertion and retrieval.
   2. Replacement of existing values.
   3. Deletion.
   4. Collision handling.
   5. Resizing.
   6. Explicit-length keys.
   7. Large numbers of entries.

3. It can be invoked from a test mode in the compiler driver rather than forming part of ordinary compilation.

### 43. `file_exists()`

```c
// File: chibicc.h
bool file_exists(char *path);
```

1. `file_exists()` tests whether a filesystem path refers to an accessible existing file according to the implementation’s check.

2. It is useful when searching include paths and locating compiler support files.

```c
// File: conceptual-include-search.c
char *path = format("%s/%s", include_dir, filename);

if (file_exists(path))
  return path;
```

3. The name does not by itself specify whether directories count as existing files or whether permission failures are treated as nonexistence; those details are determined by `main.c`.

4. There is an inherent time-of-check/time-of-use race if the file is checked and opened later. For correctness-sensitive access, opening the file directly and handling failure is safer.

### 44. `include_paths`

```c
// File: chibicc.h
extern StringArray include_paths;
```

1. `include_paths` stores directories searched for header files.

2. Command-line options such as `-I` append paths to this array.

```text
-Iinclude
-Ithird_party/include
```

3. The resulting array may contain:

```text
include
third_party/include
/usr/local/include
/usr/include
```

4. `search_include_paths()` traverses these entries when resolving an angle-bracket include or after local quoted-include lookup fails.

```c
// File: example.c
#include <project/config.h>
```

5. Candidate paths are conceptually created as:

```text
include/project/config.h
third_party/include/project/config.h
/usr/local/include/project/config.h
/usr/include/project/config.h
```

### 45. `opt_fpic`

```c
// File: chibicc.h
extern bool opt_fpic;
```

1. `opt_fpic` records whether position-independent code generation is enabled, typically through a command-line option such as `-fPIC` or `-fpic`.

2. Position-independent code avoids embedding absolute addresses that prevent code from being loaded at different virtual addresses.

3. Global symbol access may require target-specific PC-relative addressing or a global offset table.

```text
Non-PIC conceptual access:
    load from absolute address of global

PIC conceptual access:
    calculate address relative to instruction pointer
    or load through relocation/GOT machinery
```

4. This option mainly affects `codegen.c` and emitted relocation forms.

### 46. `opt_fcommon`

```c
// File: chibicc.h
extern bool opt_fcommon;
```

1. `opt_fcommon` controls how tentative global definitions are emitted.

```c
// File: example.c
int counter;
```

2. With common-symbol behavior enabled, tentative definitions can be emitted as common symbols, allowing compatible tentative definitions from multiple translation units to be merged by the linker.

3. With common-symbol behavior disabled, the compiler emits an ordinary zero-initialized definition, commonly in a BSS-like section.

4. Conceptually:

```text
-fcommon:
    counter -> common symbol

-fno-common:
    counter -> regular zero-initialized definition
```

5. This option is closely related to `Obj.is_tentative`.

### 47. `base_file`

```c
// File: chibicc.h
extern char *base_file;
```

1. `base_file` stores the primary source file for the current translation unit.

2. It is distinct from included files.

```text
base_file:
    src/main.c

input files:
    src/main.c
    include/project.h
    /usr/include/stddef.h
```

3. It can be used for built-in macros, dependency generation, diagnostics, and compiler-driver behavior.

4. In particular, a built-in macro such as `__BASE_FILE__`, where supported, refers to the main translation-unit file rather than the current included file.

```c
// File: header.h
const char *current = __FILE__;
const char *base = __BASE_FILE__;
```

5. While processing `header.h`, `__FILE__` can identify the header, whereas `__BASE_FILE__` identifies the original source file.

### 48. Ownership and Lifetime Considerations

1. Many structures in this header store raw pointers rather than owning, reference-counting, or copying their targets.

2. Token-backed keys can point into a `File.contents` buffer, so that source buffer must remain alive while tokens or maps reference it.

```text
File.contents
   |
   +-- Token.loc
   |
   +-- HashEntry.key
```

3. Canonical type objects must remain alive throughout compilation because many AST and object nodes point to them.

4. `Member`, `Type`, `Node`, `Obj`, and hash-map values similarly need translation-unit or compiler-process lifetimes unless the implementation explicitly manages shorter ownership.

5. chibicc’s architecture generally benefits from arena-like “allocate during compilation, release at process termination” behavior, although exact allocation policy must be confirmed from each implementation file.

### 49. Data Relationships in This Section

1. A structure `Type` owns or references a linked list of `Member` descriptors.

2. Each `Member` points to another `Type` describing that field.

3. `add_type()` annotates AST nodes using canonical or derived `Type` objects.

4. `codegen()` consumes typed nodes and structure-layout information.

5. Unicode helpers support tokenization and diagnostics.

6. Hash maps provide efficient macro, scope, and symbol lookup.

7. Driver globals communicate include paths and command-line code-generation options.

```text
Type(TY_STRUCT)
    |
    +-- members
          |
          v
       Member
          |
          +-- ty ------> Type
          +-- next ----> Member

Token stream
    |
    v
Hash-map-assisted parsing
    |
    v
Node AST + Obj list + Type graph
    |
    v
add_type()
    |
    v
codegen()
    |
    v
Assembly output
```


