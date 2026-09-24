[About main method](#about-main-method)

[What is AST?](#what-is-ast)

[what is atexit()](#what-is-atexit)

---

## About main method

**`main()` is essentially the compiler driver's control center**.

One important point first:

> This `main.c` is **not where C source code is parsed into an AST**.
> It decides **what should happen to each input file** and orchestrates the tokenizer/parser/code generator, assembler, and linker.

A useful mental model is:

```text
           chibicc driver
                │
                ▼
            parse_args()
                │
       ┌────────┴────────┐
       │                 │
    cc1 mode          normal mode
       │                 │
       ▼                 ▼
     cc1()        inspect input files
                         │
    ┌────────────────────┼────────────────────┐
    ▼                    ▼                    ▼
preprocess             compile             link
    │                    │                    │
    ▼                    ▼                    ▼
 output               .s / .o             executable
```

Let's go through the code carefully.

---

### 1. Function signature

```c
int main(int argc, char** argv) {
```

This is the normal C program entry point.

`argc` means:

```text
argument count
```

`argv` means:

```text
argument vector
```

For example, if you run:

```bash
./chibicc hello.c -o hello
```

conceptually:

```text
argc = 4

argv[0] = "./chibicc"
argv[1] = "hello.c"
argv[2] = "-o"
argv[3] = "hello"
```

So `main()` receives the command-line arguments.

---

### 2. Register cleanup function

```c
atexit(cleanup);
```

`atexit()` is a standard C library function.

It registers a function that should be called when the program terminates normally.

So:

```c
atexit(cleanup);
```

means approximately:

```text
"When chibicc exits, call cleanup()."
```

This is useful because chibicc creates temporary files and allocates resources during compilation.

For example:

```text
source.c
   │
   ▼
temporary assembly
   │
   ▼
temporary object
   │
   ▼
executable
```

The temporary files should eventually be removed.

Instead of manually calling:

```c
cleanup();
```

at every possible exit point, chibicc registers it once:

```c
atexit(cleanup);
```

Then normal termination automatically invokes it.

---

### 3. Initialize macros

```c
init_macros();
```

This initializes the C preprocessor's predefined macros.

For example, a C compiler normally provides macros such as:

```c
__FILE__
__LINE__
__DATE__
__TIME__
__STDC__
```

and architecture/platform-related macros.

So conceptually:

```text
init_macros()
   │
   ▼
Initialize preprocessor state
   │
   ├── predefined macros
   ├── compiler-specific macros
   └── target-specific macros
```

This is part of the preprocessing infrastructure.

---

### 4. Parse command-line arguments

```c
parse_args(argc, argv);
```

This is one of the most important calls in `main()`.

Suppose you execute:

```bash
chibicc -S test.c
```

or:

```bash
chibicc -c test.c
```

or:

```bash
chibicc test.c -o test
```

`parse_args()` examines these arguments and sets global compiler options.

For example:

```text
-S
```

means:

```text
compile to assembly
```

while:

```text
-c
```

means:

```text
compile and assemble, but don't link
```

and:

```text
-E
```

means:

```text
preprocess only
```

After `parse_args()` runs, variables such as:

```c
opt_c
opt_S
opt_E
opt_M
opt_o
opt_cc1
```

contain the selected options.

So think of:

```c
parse_args(argc, argv);
```

as:

> "Figure out what the user wants chibicc to do."

---

### 5. Special `cc1` mode

```c
if (opt_cc1) {
  add_default_include_paths(argv[0]);
  cc1();
  return 0;
}
```

This is an important compiler architecture concept.

#### What is `cc1`?

You can think of chibicc as having two levels:

```text
             chibicc
                │
     ┌──────────┴──────────┐
     │                     │
  driver                  cc1
     │                     │
     │             actual compilation
     │                     │
     ▼                     ▼
command-line          lexer/parser/
  handling             codegen
```

The **driver** decides:

```text
Should I preprocess?
Should I compile?
Should I assemble?
Should I link?
```

The `cc1` part performs the actual compilation.

---

#### `opt_cc1`

```c
if (opt_cc1)
```

checks whether chibicc was explicitly requested to operate in its internal compiler mode.

If so:

```c
add_default_include_paths(argv[0]);
```

sets up the standard include paths.

Then:

```c
cc1();
```

performs the actual compilation.

Finally:

```c
return 0;
```

terminates the driver.

So:

```text
opt_cc1 == true
     │
     ▼
add include paths
     │
     ▼
cc1()
     │
     ▼
return
```

This is similar conceptually to how GCC historically separates its driver functionality from its internal compilation stages.

---

### 6. Check invalid `-o` usage

```c
if (input_paths.len > 1 && opt_o && (opt_c || opt_S | opt_E))
  error("cannot specify '-o' with '-c,' '-S' or '-E' with multiple files");
```

This checks for an invalid command-line combination.

There are three conditions.

#### Condition 1

```c
input_paths.len > 1
```

There are multiple input files.

For example:

```bash
chibicc a.c b.c
```

#### Condition 2

```c
opt_o
```

The user specified an output filename:

```bash
-o something
```

#### Condition 3

```c
opt_c || opt_S | opt_E
```

The user is requesting a non-linking operation.

The intended logic is essentially:

```text
-c OR -S OR -E
```

There appears to be a `|` here:

```c
opt_S | opt_E
```

rather than:

```c
opt_S || opt_E
```

Because these options are normally integer/boolean flags, bitwise OR can produce the intended truthy result here, although `||` would be the clearer boolean expression.

The problem is that:

```bash
chibicc -c a.c b.c -o output.o
```

would have an ambiguous output.

Should:

```text
a.c → output.o
b.c → ??? 
```

both produce the same file?

Therefore chibicc rejects it.

---

### 7. Create linker argument array

```c
StringArray ld_args = {};
```

This creates an empty string array.

`ld_args` means approximately:

```text
linker arguments
```

Eventually it might contain:

```text
foo.o
bar.o
-lm
-lpthread
```

Then:

```c
run_linker(&ld_args, ...);
```

uses these arguments to invoke the linker.

So:

```text
ld_args
   │
   ├── foo.o
   ├── bar.o
   ├── -lm
   └── -lpthread
```

---

### 8. Process every input file

```c
for (int i = 0; i < input_paths.len; i++) {
```

This is the central loop.

Suppose you run:

```bash
chibicc main.c foo.c bar.o
```

then:

```text
input_paths

0 → main.c
1 → foo.c
2 → bar.o
```

The loop processes them one by one.

---

# 9. Get current input filename

```c
char* input = input_paths.data[i];
```

This retrieves the current input path.

For example:

```text
i = 0
input = "main.c"
```

Then on the next iteration:

```text
i = 1
input = "foo.c"
```

---

# 10. Handle `-l` arguments

```c
if (!strncmp(input, "-l", 2)) {
  strarray_push(&ld_args, input);
  continue;
}
```

This checks whether the input begins with:

```text
-l
```

For example:

```bash
-lm
```

means:

```text
link against libm
```

So:

```text
-lm
```

isn't actually a source file.

It is a linker option.

Therefore:

```c
strarray_push(&ld_args, input);
```

adds it to the linker argument list.

Then:

```c
continue;
```

means:

> Stop processing this input and immediately go to the next iteration of the `for` loop.

So:

```text
-lm
 │
 ▼
ld_args
 │
 ▼
continue
 │
 ▼
next input
```

---

# 11. Handle `-Wl,`

```c
if (!strncmp(input, "-Wl,", 4)) {
```

`-Wl,` is used to pass options to the linker.

For example:

```bash
-Wl,--gc-sections
```

The compiler driver needs to extract:

```text
--gc-sections
```

and pass it to the linker.

---

## Copy the string

```c
char* s = strdup(input + 4);
```

Suppose:

```text
input = "-Wl,--gc-sections"
```

Then:

```c
input + 4
```

points after:

```text
-Wl,
```

so:

```text
input + 4
      ↓
"--gc-sections"
```

`strdup()` makes a writable copy.

---

# 12. Split linker arguments

```c
char* arg = strtok(s, ",");
```

`strtok()` splits a string using a delimiter.

For example:

```text
-Wl,--foo,--bar
```

after removing `-Wl,`:

```text
--foo,--bar
```

then:

```c
strtok(s, ",")
```

produces:

```text
--foo
```

and subsequent calls:

```c
strtok(NULL, ",")
```

produce:

```text
--bar
```

So:

```c
while (arg) {
  strarray_push(&ld_args, arg);
  arg = strtok(NULL, ",");
}
```

effectively transforms:

```text
-Wl,--foo,--bar
```

into:

```text
ld_args
├── --foo
└── --bar
```

Then:

```c
continue;
```

moves to the next input.

---

# 13. Determine output filename

```c
char* output;
```

Now chibicc needs to determine where the output should go.

There are three possibilities.

---

## Explicit `-o`

```c
if (opt_o)
  output = opt_o;
```

For:

```bash
chibicc hello.c -o hello.o
```

we get:

```text
output = "hello.o"
```

---

## Assembly output

```c
else if (opt_S)
  output = replace_extn(input, ".s");
```

For:

```bash
chibicc hello.c -S
```

the default output becomes:

```text
hello.s
```

Conceptually:

```text
hello.c
   ↓
replace_extn()
   ↓
hello.s
```

---

## Otherwise object output

```c
else
  output = replace_extn(input, ".o");
```

For:

```bash
chibicc hello.c -c
```

the output becomes:

```text
hello.o
```

So:

```text
-o specified?
       │
    yes│ no
       ▼
 explicit output
       │
       └─────────┐
                 ▼
             -S specified?
                 │
             yes │ no
                 ▼
                .s
                 │
                 └─────── .o
```

---

# 14. Determine input file type

```c
FileType type = get_file_type(input);
```

This examines the filename extension.

For example:

```text
foo.c   → FILE_C
foo.s   → FILE_ASM
foo.o   → FILE_OBJ
libfoo.a → FILE_AR
libfoo.so → FILE_DSO
```

This allows the driver to decide what to do next.

---

# 15. Object files and libraries

```c
if (type == FILE_OBJ || type == FILE_AR || type == FILE_DSO) {
  strarray_push(&ld_args, input);
  continue;
}
```

This handles:

```text
.o
.a
.so
```

For example:

```bash
chibicc main.c foo.o libbar.a
```

`foo.o` and `libbar.a` don't need to be compiled.

They simply need to be given to the linker.

So:

```text
foo.o
  │
  ▼
ld_args

libbar.a
  │
  ▼
ld_args
```

---

# 16. Assembly files

```c
if (type == FILE_ASM) {
  if (!opt_S) assemble(input, output);
  continue;
}
```

This handles `.s` files.

For example:

```bash
chibicc foo.s
```

The assembly needs to be assembled:

```text
foo.s
  │
  ▼
assembler
  │
  ▼
foo.o
```

So:

```c
assemble(input, output);
```

does the assembly step.

But notice:

```c
if (!opt_S)
```

If `-S` was specified, chibicc is being asked to stop at assembly output, so it doesn't assemble.

---

# 17. Assert that remaining input is C

```c
assert(type == FILE_C);
```

At this point, chibicc has already handled:

```text
-l...
-Wl,...
.o
.a
.so
.s
```

Therefore the remaining expected type is:

```text
.c
```

So this:

```c
assert(type == FILE_C);
```

is essentially saying:

> "At this point, this must be a C source file."

---

# 18. Preprocess only

```c
if (opt_E || opt_M) {
  run_cc1(argc, argv, input, NULL);
  continue;
}
```

This corresponds to options such as:

```bash
chibicc -E test.c
```

`-E` means:

> Run the preprocessor, but don't compile.

So:

```text
test.c
  │
  ▼
preprocessor
  │
  ▼
preprocessed C
```

No assembly.

No object file.

No linking.

Notice:

```c
run_cc1(..., NULL);
```

The output parameter is `NULL`.

That tells `cc1` that there isn't a normal assembly output file to generate.

---

# 19. Compile to assembly

```c
if (opt_S) {
  run_cc1(argc, argv, input, output);
  continue;
}
```

For:

```bash
chibicc -S test.c
```

the pipeline is:

```text
test.c
  │
  ▼
cc1
  │
  ├── tokenize
  ├── preprocess
  ├── parse
  ├── type analysis
  └── code generation
  │
  ▼
test.s
```

This is the point where you should connect this `main.c` code to the other chibicc files you were asking about earlier.

`run_cc1()` eventually gets you into the actual compiler implementation.

---

# 20. Compile and assemble

```c
if (opt_c) {
  char* tmp = create_tmpfile();
  run_cc1(argc, argv, input, tmp);
  assemble(tmp, output);
  continue;
}
```

This corresponds to:

```bash
chibicc -c test.c
```

The user wants an object file:

```text
test.o
```

but not a final executable.

So chibicc needs two stages:

```text
test.c
  │
  ▼
cc1
  │
  ▼
temporary assembly
  │
  ▼
assembler
  │
  ▼
test.o
```

That's why:

```c
char* tmp = create_tmpfile();
```

creates a temporary file.

Then:

```c
run_cc1(argc, argv, input, tmp);
```

generates assembly into the temporary file.

Then:

```c
assemble(tmp, output);
```

converts that assembly into:

```text
test.o
```

---

# 21. Compile, assemble, and link

If none of these options were specified:

```text
-E
-S
-c
```

then the user wants the normal behavior:

```bash
chibicc test.c
```

which means:

```text
test.c
   ↓
compile
   ↓
assembly
   ↓
object
   ↓
link
   ↓
a.out
```

The code:

```c
char* tmp1 = create_tmpfile();
char* tmp2 = create_tmpfile();
```

creates two temporary files.

Then:

```c
run_cc1(argc, argv, input, tmp1);
```

does:

```text
test.c
 ↓
tmp1
```

where `tmp1` contains assembly.

Then:

```c
assemble(tmp1, tmp2);
```

does:

```text
tmp1 (assembly)
       ↓
   assembler
       ↓
tmp2 (object)
```

Then:

```c
strarray_push(&ld_args, tmp2);
```

adds the object file to the linker arguments.

So after processing the C source:

```text
ld_args
   │
   └── tmp2
```

---

# 22. Run the linker

After the `for` loop:

```c
if (ld_args.len > 0)
  run_linker(&ld_args, opt_o ? opt_o : "a.out");
```

Now all input files have been processed.

Suppose:

```bash
chibicc main.c foo.c foo.o -lm
```

After processing everything, `ld_args` might conceptually contain:

```text
ld_args
├── temporary object for main.c
├── temporary object for foo.c
├── foo.o
└── -lm
```

Then:

```c
run_linker(&ld_args, "a.out");
```

links them together:

```text
object files
     │
     ├──────┐
     ├──────┤
     └──────┤
            ▼
         linker
            │
            ▼
          a.out
```

If the user specified:

```bash
-o myprogram
```

then:

```c
opt_o ? opt_o : "a.out"
```

evaluates to:

```text
"myprogram"
```

Otherwise:

```text
"a.out"
```

is used.

---

# 23. The entire function in one picture

This is the most important thing to take away from this `main.c`:

```text
                  chibicc
                     │
                     ▼
              parse_args()
                     │
                     ▼
             What did user request?
                     │
       ┌─────────────┼────────────────┐
       │             │                │
       ▼             ▼                ▼
      -E            -S               -c
       │             │                │
       ▼             ▼                ▼
 preprocess      compile          compile
 only            → .s             → .o
                                      │
                                      ▼
                                  no linker
       
                     OR

                  no -E/-S/-c
                       │
                       ▼
                    compile
                       │
                       ▼
                     .s
                       │
                       ▼
                    assemble
                       │
                       ▼
                     .o
                       │
                       ▼
                    linker
                       │
                       ▼
                    a.out
```

---

# 24. Connect this to the files you should study

Now the earlier recommended study order makes more sense.

This `main.c` is the **driver**.

When you see:

```c
run_cc1(...)
```

that's your signal to start investigating the actual compiler.

The relationship is roughly:

```text
main.c
 │
 ├── parse_args()
 │
 ├── run_cc1()
 │      │
 │      ▼
 │     cc1()
 │      │
 │      ├── tokenize.c
 │      │
 │      ├── preprocess.c
 │      │
 │      ├── parse.c
 │      │
 │      ├── type.c
 │      │
 │      └── codegen.c
 │      │
 │      ▼
 │     .s
 │
 ├── assemble()
 │      │
 │      ▼
 │     .o
 │
 └── run_linker()
        │
        ▼
     executable
```

**This is the key distinction when studying chibicc:**

* `main.c` = **compiler driver**
* `tokenize.c` = **lexical analysis**
* `preprocess.c` = **C preprocessing**
* `parse.c` = **parsing + AST construction**
* `type.c` = **type information / semantic processing**
* `codegen.c` = **AST → x86-64 assembly**

So after understanding this `main.c`, I would **not continue reading `main.c`**. Your next target should be **`run_cc1()` and then `cc1()`**, because those functions provide the bridge from the compiler driver into the actual compiler pipeline. From there, go into `tokenize.c`.

---

## What is AST?

---

**AST** stands for **Abstract Syntax Tree**. It is one of the most important concepts in compiler construction.

In the sentence:

> "`main.c` is not where C source code is parsed into an AST"

it means that `main.c` mainly **controls the compilation process**, while another part of chibicc—primarily `parse.c`—takes the C tokens and builds the AST.

---

# 1. What is an AST?

An AST is a **tree-shaped representation of the structure of a program**.

Consider this C code:

```c
int x = 10 + 20;
```

A compiler first sees characters:

```text
i n t   x   =   1 0   +   2 0   ;
```

The tokenizer converts them into tokens:

```text
int
x
=
10
+
20
;
```

Then the parser analyzes the relationship between those tokens and constructs an AST.

Conceptually:

```text
       declaration
            │
       ┌────┴────┐
       │         │
      x:int    initializer
                   │
                   +
                 /   \
               10     20
```

The important thing is that the AST represents **meaningful syntactic structure**, rather than individual characters.

---

# 2. Why is it called a "tree"?

Because each piece of the program can have children.

For:

```c
10 + 20
```

the structure is:

```text
       +
      / \
    10   20
```

The `+` operation is the parent node.

`10` and `20` are its child nodes.

This is a tree:

```text
        +
       / \
      10  20
```

For:

```c
10 + 20 * 3
```

the AST is:

```text
          +
         / \
       10   *
           / \
         20   3
```

Notice something important.

It is **not**:

```text
          *
         / \
        +   3
       / \
     10  20
```

because multiplication has higher precedence than addition.

Therefore:

```c
10 + 20 * 3
```

means:

```text
10 + (20 * 3)
```

The AST captures that structure.

---

# 3. AST vs source code

Consider:

```c
x = 10 + 20;
```

The source code is just text:

```text
x = 10 + 20;
```

The tokenizer produces something like:

```text
[x] [=] [10] [+] [20] [;]
```

The parser produces:

```text
          =
         / \
        x   +
           / \
         10  20
```

So the transformation is:

```text
Source code
     │
     ▼
" x = 10 + 20; "
     │
     ▼
Tokenizer
     │
     ▼
Tokens
[x] [=] [10] [+] [20] [;]
     │
     ▼
Parser
     │
     ▼
AST
    =
   / \
  x   +
     / \
   10  20
```

---

# 4. What does "abstract" mean?

The word **abstract** is important.

The AST doesn't need to preserve every character from the original source.

For example:

```c
x = 10 + 20;
```

contains:

* spaces
* semicolon
* exact formatting
* comments
* etc.

The AST doesn't need most of that.

It cares about the program's structure:

```text
assignment
    │
    ├── variable x
    │
    └── addition
          ├── 10
          └── 20
```

That's why it's an **Abstract** Syntax Tree.

---

# 5. AST in chibicc

This is where `parse.c` becomes very important.

chibicc defines different kinds of AST nodes.

You'll encounter concepts such as:

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

These represent different kinds of syntax.

For example:

```c
x + y
```

might be represented as:

```text
ND_ADD
├── lhs → x
└── rhs → y
```

And:

```c
x = 10;
```

as:

```text
ND_ASSIGN
├── lhs → x
└── rhs → 10
```

And:

```c
return x + 1;
```

as:

```text
ND_RETURN
    │
    ▼
   ND_ADD
   /    \
  x      1
```

---

# 6. Why does the compiler need an AST?

Because the compiler needs to **understand the program's structure** before generating machine code.

For example:

```c
int result = (10 + 20) * 3;
```

The compiler needs to know:

```text
          *
         / \
        +   3
       / \
     10   20
```

Then code generation can produce something conceptually like:

```asm
mov $10, ...
add $20, ...
imul $3, ...
```

So the AST acts as an intermediate representation between parsing and code generation:

```text
C source
   │
   ▼
Tokenizer
   │
   ▼
Tokens
   │
   ▼
Parser
   │
   ▼
AST
   │
   ▼
Type analysis
   │
   ▼
Code generation
   │
   ▼
Assembly
```

---

# 7. AST is not machine code

This distinction is important.

An AST:

```text
       +
      / \
    10   20
```

does **not** mean:

```text
add instruction
```

It means:

> "This program contains an addition operation whose left operand is 10 and whose right operand is 20."

Then `codegen.c` decides how to translate that operation into machine instructions.

For example:

```text
AST

      +
     / \
   10   20

      │
      ▼

Code generator

      │
      ▼

Assembly
```

So:

```text
AST = representation of program structure
Assembly = instructions for the CPU
```

---

# 8. A more realistic example

Consider:

```c
int main() {
    int x = 10;

    if (x > 5)
        return 1;

    return 0;
}
```

The AST is conceptually something like:

```text
Function: main
│
└── Body
    │
    ├── Variable declaration
    │     │
    │     └── x = 10
    │
    ├── if
    │    │
    │    ├── condition
    │    │      │
    │    │      └── x > 5
    │    │
    │    └── then
    │           │
    │           └── return 1
    │
    └── return 0
```

This is much easier for the compiler to work with than:

```text
i n t   m a i n ( ) { i n t ...
```

---

# 9. How this relates to the chibicc files

Now you can see why I recommended studying the files in this order:

```text
tokenize.c
     │
     │ creates
     ▼
  Tokens
     │
     │ consumed by
     ▼
 parse.c
     │
     │ creates
     ▼
   AST
     │
     │ analyzed by
     ▼
  type.c
     │
     ▼
Typed AST
     │
     │ consumed by
     ▼
codegen.c
     │
     ▼
Assembly
```

In particular:

### `tokenize.c`

Answers:

> "What are the individual pieces of the source code?"

For:

```c
x + 10
```

it produces roughly:

```text
identifier(x)
+
number(10)
```

### `parse.c`

Answers:

> "How are these pieces related?"

It produces:

```text
    +
   / \
  x  10
```

### `type.c`

Answers:

> "What types do these things have?"

For example:

```text
x → int
10 → int
x + 10 → int
```

### `codegen.c`

Answers:

> "How do I turn this structure into CPU instructions?"

---

## The key idea

When studying chibicc, keep this mental model in your head:

```text
                 C source
                    │
                    ▼
             ┌─────────────┐
             │  tokenize.c │
             └──────┬──────┘
                    │
                    ▼
                  Tokens
                    │
                    ▼
             ┌─────────────┐
             │   parse.c   │
             └──────┬──────┘
                    │
                    ▼
                   AST
                    │
              ┌─────┴─────┐
              │           │
              ▼           ▼
           type.c      codegen.c
              │           │
              ▼           ▼
          type info     Assembly
```

So when you see **AST** throughout chibicc, think:

> **"The compiler's structured representation of what the source code means syntactically."**

And when you next study `parse.c`, the most useful thing is to understand **how a sequence of tokens such as `10 + 20 * 3` gets turned into the tree `10 + (20 * 3)`**. That is the point where compiler construction starts becoming much more concrete.

---

## what is atexit()

---

`atexit()` registers a function to run automatically when the program exits normally — either via `return` from `main()` or a call to `exit()`.

It does **not** run on `abort()`, `_exit()`, or a crash (e.g., segfault).

### Signature

```c
#include <stdlib.h>

int atexit(void (*func)(void));
```

- Takes a function pointer to a function with no arguments and no return value.
- Returns `0` on success, nonzero on failure.
- You can register multiple functions (at least 32 guaranteed by the C standard).

### Multiple handlers — LIFO order

Registered functions run in **reverse order of registration** (last-in, first-out):

```c
#include <stdio.h>
#include <stdlib.h>

void cleanup(void) { printf("Cleaning up...\n"); }
void cleanup2(void) { printf("Cleaning2 up...\n"); }
void cleanup3(void) { printf("Cleaning3 up...\n"); }

int main(void) {
  atexit(cleanup);
  atexit(cleanup2);
  atexit(cleanup3);
  printf("Main running...\n");
  return 0;
}
```

```bash
Main running...
Cleaning3 up...
Cleaning2 up...
Cleaning up...
```

### Works with `exit()` too

```c
#include <stdio.h>
#include <stdlib.h>

void notify(void) { printf("exit handler called\n"); }

void do_work(int fail) {
  if (fail) {
    printf("failure detected\n");
    exit(EXIT_FAILURE);  // triggers atexit handlers
  }
}

int main(void) {
  atexit(notify);
  do_work(1);
  printf("this line never runs\n");
  return 0;
}
```

```
failure detected
exit handler called
```

### Common gotchas

- **No arguments allowed.** If you need to pass state, use a global/static variable or a closure-like pattern (C has no closures, so use file-scope statics).

- **Not called on `abort()`/crash.** Don't rely on it for critical cleanup like flushing to disk on a signal — use signal handlers for that.

- **Order matters** if handlers depend on each other's side effects (LIFO, as shown above).

- **`exit()` inside an atexit handler** is undefined/implementation-defined behavior in some standards — avoid calling `exit()` from within a registered function.

---

## parse_args()

---

### 1. Purpose of `parse_args()`

- 1. `parse_args()`
- interprets the compiler command line and
- stores the results in global option variables and dynamic string arrays.

- 2. In chibicc, the executable acts as both a compiler driver and an internal compiler process, so this function handles preprocessing, compilation, assembly, linking, and internal `-cc1` options.

- 3. For a command such as:

```bash
chibicc -Iinclude -DDEBUG -c main.c -o main.o
```

the function approximately produces:

```text
include_paths = ["include"]
input_paths   = ["main.c"]
opt_c         = true
opt_o         = "main.o"
macro DEBUG   = defined
```

### 2. Function parameters

```c
static void parse_args(int argc, char **argv)
```

1. `argc` is the number of command-line arguments.

2. `argv` is an array of argument strings, and the C runtime guarantees that `argv[argc]` is a null pointer.

3. `argv[0]` is the executable name, so both loops begin at index `1`.

### 3. First pass: validating options with operands

```c
for (int i = 1; i < argc; i++)
  if (take_arg(argv[i]))
    if (!argv[++i])
      usage(1);
```

1. The first pass verifies that every option requiring a following argument actually has one.

2. `take_arg()` identifies separate-argument options such as `-o`, `-D`, `-U`, `-include`, `-x`, `-MF`, `-MT`, `-MQ`, `-Xlinker`, and `-L`.

3. When such an option is found, `++i` moves to its operand; that operand is skipped as an option during this validation pass.

4. If the option is the last command-line argument, `argv[++i]` evaluates to `argv[argc]`, which is guaranteed to be `NULL`, and `usage(1)` terminates with an error.

For example:

```bash
chibicc -o
```

is rejected because `-o` has no output filename.

The double `if` is equivalent to:

```c
for (int i = 1; i < argc; i++) {
  if (take_arg(argv[i])) {
    i++;

    if (argv[i] == NULL)
      usage(1);
  }
}
```

### 4. Delayed `-idirafter` paths

```c
StringArray idirafter = {};
```

1. `idirafter` temporarily stores include directories supplied by `-idirafter`.

2. These directories must be searched after ordinary `-I` directories, so they are not immediately inserted into `include_paths`.

3. `{}` zero-initializes the structure in chibicc’s build environment; `{0}` is the conventional form for older C standards.

Conceptually, `StringArray` contains fields similar to:

```c
typedef struct {
  char **data;
  int capacity;
  int len;
} StringArray;
```

### 5. Main parsing loop

```c
for (int i = 1; i < argc; i++) {
  // Interpret argv[i].
}
```

1. The second pass performs the actual parsing.

2. Each recognized option updates a global variable or appends a value to an array.

3. Nearly every recognized branch ends with `continue`, ensuring that the argument is not later mistaken for an input filename.

4. Options with separate operands increment `i` so the outer loop does not process the operand again.

### 6. Driver and internal compiler options

#### 6-1. `-###`

```c
if (!strcmp(argv[i], "-###")) {
  opt_hash_hash_hash = true;
  continue;
}
```

1. `-###` asks the compiler driver to print the external commands it would execute.
2. The associated global flag is later checked when invoking the assembler, linker, or internal compiler stage.

#### 6-2. `-cc1`

```c
if (!strcmp(argv[i], "-cc1")) {
  opt_cc1 = true;
  continue;
}
```

1. `-cc1` selects chibicc’s internal compiler mode.
2. The driver can launch another chibicc process with `-cc1` to perform the actual preprocessing and C-to-assembly compilation.

#### 6-3. Internal input and output names

```c
if (!strcmp(argv[i], "-cc1-input")) {
  base_file = argv[++i];
  continue;
}

if (!strcmp(argv[i], "-cc1-output")) {
  output_file = argv[++i];
  continue;
}
```

1. `-cc1-input` specifies the source file used by the internal compiler stage.
2. `-cc1-output` specifies where that stage writes its result.
3. These are internal communication options rather than normal user-facing compiler options.

### 7. General output and compilation-stage options

#### 7-1. Help

```c
if (!strcmp(argv[i], "--help"))
  usage(0);
```

1. `usage(0)` prints usage information and exits successfully.
2. A nonzero argument, such as `usage(1)`, conventionally indicates an error.

#### 7-2. Output filename

```c
if (!strcmp(argv[i], "-o")) {
  opt_o = argv[++i];
  continue;
}

if (!strncmp(argv[i], "-o", 2)) {
  opt_o = argv[i] + 2;
  continue;
}
```

1. The first branch accepts a separate argument:

```bash
chibicc -o output main.c
```

2. The second branch accepts the attached form:

```bash
chibicc -ooutput main.c
```

3. `argv[i] + 2` points immediately after the two characters `-o`.
4. The exact `-o` test must appear before the prefix test; otherwise bare `-o` would produce an empty filename.

#### 7-3. Stop-after-stage options

```c
if (!strcmp(argv[i], "-S")) {
  opt_S = true;
  continue;
}

if (!strcmp(argv[i], "-c")) {
  opt_c = true;
  continue;
}

if (!strcmp(argv[i], "-E")) {
  opt_E = true;
  continue;
}
```

1. `-E` stops after preprocessing.
2. `-S` stops after generating assembly.
3. `-c` stops after generating an object file.
4. Without these options, the driver normally continues through linking.

The pipeline is approximately:

```text
source.c
   |
   | preprocessing
   v
preprocessed C
   |
   | compilation
   v
assembly
   |
   | assembly
   v
object file
   |
   | linking
   v
executable or shared object
```

### 8. Common-symbol behavior

```c
if (!strcmp(argv[i], "-fcommon")) {
  opt_fcommon = true;
  continue;
}

if (!strcmp(argv[i], "-fno-common")) {
  opt_fcommon = false;
  continue;
}
```

1. These options control the treatment of tentative global definitions such as:

```c
int value;
```

2. With `-fcommon`, multiple tentative definitions can be emitted as common symbols and combined by the linker.
3. With `-fno-common`, the compiler emits ordinary definitions, allowing the linker to diagnose duplicate definitions more strictly.
4. Since arguments are processed from left to right, the last occurrence wins:

```bash
chibicc -fcommon -fno-common main.c
```

results in `opt_fcommon == false`.

### 9. Include-path handling

#### 9-1. `-I`

```c
if (!strncmp(argv[i], "-I", 2)) {
  strarray_push(&include_paths, argv[i] + 2);
  continue;
}
```

1. This branch recognizes attached include paths such as:

```bash
chibicc -Iinclude main.c
```

2. `argv[i] + 2` points to `"include"`.
3. The path is appended to `include_paths`, which the preprocessor later searches for headers.

For example:

```text
argv[i]     -> "-Iinclude"
argv[i] + 2 ->   "include"
```

This particular branch does not independently handle the separate form `-I include`; support for that form depends on the surrounding chibicc version and its option-validation logic.

#### 9-2. `-include`

```c
if (!strcmp(argv[i], "-include")) {
  strarray_push(&opt_include, argv[++i]);
  continue;
}
```

1. `-include file.h` makes the preprocessor process `file.h` before the main source file.
2. Multiple occurrences are preserved in command-line order.

Example:

```bash
chibicc -include config.h -include platform.h main.c
```

produces approximately:

```text
opt_include = ["config.h", "platform.h"]
```

#### 9-3. `-idirafter`

The pasted code contains:

```c
if (!strcmp(argv[i], "-idirafter")) {
  strarray_push(&idirafter, argv[i++]);
  continue;
}
```

1. As written, this appends the literal string `"-idirafter"` rather than the following directory.
2. The post-increment moves `i` to the directory, and then the `for` loop increments it again, so the directory is skipped.
3. The intended implementation should normally be:

```c
if (!strcmp(argv[i], "-idirafter")) {
  strarray_push(&idirafter, argv[++i]);
  continue;
}
```

For:

```bash
chibicc -idirafter /opt/sdk/include main.c
```

the corrected code stores:

```text
idirafter = ["/opt/sdk/include"]
```

The original chibicc source/version should be checked because `argv[i++]` at this location is behaviorally inconsistent with the purpose of the option.

### 10. Macro definitions and undefinitions

#### 10-1. `-D`

```c
if (!strcmp(argv[i], "-D")) {
  define(argv[++i]);
  continue;
}

if (!strncmp(argv[i], "-D", 2)) {
  define(argv[i] + 2);
  continue;
}
```

1. Both separate and attached forms are supported:

```bash
chibicc -D DEBUG main.c
chibicc -DDEBUG main.c
chibicc -DVERSION=3 main.c
```

2. `define()` interprets the text as a macro definition.
3. A definition without `=` is typically treated as if its replacement value were `1`.

Conceptually:

```text
-DDEBUG       -> #define DEBUG 1
-DVERSION=3   -> #define VERSION 3
```

#### 10-2. `-U`

```c
if (!strcmp(argv[i], "-U")) {
  undef_macro(argv[++i]);
  continue;
}

if (!strncmp(argv[i], "-U", 2)) {
  undef_macro(argv[i] + 2);
  continue;
}
```

1. `-U` removes a macro definition.
2. It also accepts separate and attached forms:

```bash
chibicc -U DEBUG main.c
chibicc -UDEBUG main.c
```

3. Option ordering matters:

```bash
chibicc -DDEBUG -UDEBUG main.c
```

leaves `DEBUG` undefined.

### 11. Input-language selection with `-x`

```c
if (!strcmp(argv[i], "-x")) {
  opt_x = parse_opt_x(argv[++i]);
  continue;
}

if (!strncmp(argv[i], "-x", 2)) {
  opt_x = parse_opt_x(argv[i] + 2);
  continue;
}
```

1. `-x` explicitly specifies the input language instead of relying on the filename extension.
2. Both forms are accepted:

```bash
chibicc -x c source
chibicc -xc source
```

3. `parse_opt_x()` converts the textual language name into an internal file-type value such as `FILE_C`.
4. This is useful for extensionless files or standard input.

### 12. Linker inputs and options

#### 12-1. Libraries and comma-separated linker options

```c
if (!strncmp(argv[i], "-l", 2) ||
    !strncmp(argv[i], "-Wl,", 4)) {
  strarray_push(&input_paths, argv[i]);
  continue;
}
```

1. `-lfoo` asks the linker to search for library `foo`.
2. `-Wl,...` forwards comma-separated arguments to the linker.
3. They are placed in `input_paths` because linker argument ordering can be significant.

Examples:

```bash
chibicc main.c -lm
chibicc main.c -Wl,--as-needed
```

Keeping `-lm` among the ordered inputs matters because static-library resolution is generally left-to-right.

#### 12-2. `-Xlinker`

```c
if (!strcmp(argv[i], "-Xlinker")) {
  strarray_push(&ld_extra_args, argv[++i]);
  continue;
}
```

1. `-Xlinker ARG` forwards exactly one following argument to the linker.
2. Unlike `-Wl,a,b`, it does not split a comma-separated list.

#### 12-3. Strip symbols

```c
if (!strcmp(argv[i], "-s")) {
  strarray_push(&ld_extra_args, "-s");
  continue;
}
```

1. `-s` is forwarded to the linker.
2. It requests removal of symbol information from the resulting binary.

#### 12-4. Static linking

```c
if (!strcmp(argv[i], "-static")) {
  opt_static = true;
  strarray_push(&ld_extra_args, "-static");
  continue;
}
```

1. `opt_static` records the mode for chibicc’s own driver logic.
2. The same option is also forwarded to the linker.
3. Static linking attempts to use static libraries instead of runtime shared libraries.

#### 12-5. Shared-library output

```c
if (!strcmp(argv[i], "-shared")) {
  opt_shared = true;
  strarray_push(&ld_extra_args, "-shared");
  continue;
}
```

1. `-shared` requests a shared object rather than a normal executable.
2. The flag is both stored internally and forwarded to the linker.

#### 12-6. Library search directories

```c
if (!strcmp(argv[i], "-L")) {
  strarray_push(&ld_extra_args, "-L");
  strarray_push(&ld_extra_args, argv[++i]);
  continue;
}

if (!strncmp(argv[i], "-L", 2)) {
  strarray_push(&ld_extra_args, "-L");
  strarray_push(&ld_extra_args, argv[i] + 2);
  continue;
}
```

1. Both `-L path` and `-Lpath` are accepted.
2. chibicc normalizes both forms into two linker arguments:

```text
"-L"
"path"
```

For example:

```bash
chibicc main.c -L/usr/local/lib -lfoo
```

adds approximately:

```text
ld_extra_args = ["-L", "/usr/local/lib"]
input_paths    = ["main.c", "-lfoo"]
```

### 13. Dependency-generation options

#### 13-1. `-M`

```c
if (!strcmp(argv[i], "-M")) {
  opt_M = true;
  continue;
}
```

1. `-M` requests Makefile-style dependency output.
2. The generated rule lists headers on which the source file depends.

Example output:

```makefile
main.o: main.c config.h common.h
```

#### 13-2. `-MF`

```c
if (!strcmp(argv[i], "-MF")) {
  opt_MF = argv[++i];
  continue;
}
```

1. `-MF file` chooses the destination for dependency output.
2. Without it, dependencies may be written to standard output or a derived `.d` filename, depending on the active dependency mode.

#### 13-3. `-MP`

```c
if (!strcmp(argv[i], "-MP")) {
  opt_MP = true;
  continue;
}
```

1. `-MP` adds dummy targets for headers.
2. This prevents `make` from failing immediately if a previously included header is later deleted.

Example:

```makefile
main.o: main.c config.h

config.h:
```

#### 13-4. `-MT`

```c
if (!strcmp(argv[i], "-MT")) {
  if (opt_MT == NULL)
    opt_MT = argv[++i];
  else
    opt_MT = format("%s %s", opt_MT, argv[++i]);
  continue;
}
```

1. `-MT target` explicitly sets the target appearing on the left side of the dependency rule.
2. Repeated `-MT` options are concatenated with spaces.
3. Only one `argv[++i]` expression is executed because only one branch of the `if` statement runs.

For:

```bash
chibicc -M -MT main.o -MT backup.o main.c
```

the result is approximately:

```text
opt_MT = "main.o backup.o"
```

#### 13-5. `-MQ`

```c
if (!strcmp(argv[i], "-MQ")) {
  if (opt_MT == NULL)
    opt_MT = quote_makefile(argv[++i]);
  else
    opt_MT = format("%s %s", opt_MT,
                    quote_makefile(argv[++i]));
  continue;
}
```

1. `-MQ` serves the same basic purpose as `-MT`.
2. `quote_makefile()` escapes characters that have special meaning in Makefile syntax, particularly `$`.
3. Repeated `-MT` and `-MQ` options contribute to the same target string.

#### 13-6. `-MD` and `-MMD`

```c
if (!strcmp(argv[i], "-MD")) {
  opt_MD = true;
  continue;
}

if (!strcmp(argv[i], "-MMD")) {
  opt_MD = opt_MMD = true;
  continue;
}
```

1. `-MD` generates dependency information while continuing normal compilation.
2. `-MMD` does the same but typically excludes system headers.
3. The chained assignment sets both flags to `true`:

```c
opt_MMD = true;
opt_MD = opt_MMD;
```

### 14. Position-independent code

```c
if (!strcmp(argv[i], "-fpic") ||
    !strcmp(argv[i], "-fPIC")) {
  opt_fpic = true;
  continue;
}
```

1. Both common spellings enable position-independent code generation.
2. Position-independent code avoids embedding fixed absolute addresses and is normally required for shared libraries.
3. chibicc treats `-fpic` and `-fPIC` identically here, even though some architectures distinguish their address-range assumptions.

### 15. Internal hash-map test

```c
if (!strcmp(argv[i], "-hashmap-test")) {
  hashmap_test();
  exit(0);
}
```

1. This is a project-specific self-test option.
2. It runs tests for chibicc’s hash-map implementation and exits without compiling any source file.
3. Because `exit(0)` terminates immediately, the later “no input files” check is never reached.

### 16. Accepted but ignored compatibility options

```c
if (!strncmp(argv[i], "-O", 2) ||
    !strncmp(argv[i], "-W", 2) ||
    !strncmp(argv[i], "-g", 2) ||
    !strncmp(argv[i], "-std=", 5) ||
    !strcmp(argv[i], "-ffreestanding") ||
    !strcmp(argv[i], "-fno-builtin") ||
    !strcmp(argv[i], "-fno-omit-frame-pointer") ||
    !strcmp(argv[i], "-fno-stack-protector") ||
    !strcmp(argv[i], "-fno-strict-aliasing") ||
    !strcmp(argv[i], "-m64") ||
    !strcmp(argv[i], "-mno-red-zone") ||
    !strcmp(argv[i], "-w"))
  continue;
```

1. These options are recognized so that build systems written for GCC or Clang can invoke chibicc without immediately failing.
2. They currently have no effect in this parser.
3. Prefix checks accept whole option families:

```text
-O0, -O1, -O2, -Os
-Wall, -Wextra, -Werror
-g, -g3, -gdwarf
```

4. Broad prefix matching also accepts unknown variants beginning with those prefixes, so this is intentionally permissive rather than strict validation.

### 17. Unknown-option detection

```c
if (argv[i][0] == '-' && argv[i][1] != '\0')
  error("unknown argument: %s", argv[i]);
```

1. Any remaining argument beginning with `-` is considered an unsupported option.
2. A single `"-"` is excluded because `argv[i][1] == '\0'`; it may represent standard input.
3. This check occurs only after all supported and intentionally ignored options have been tested.

Example:

```bash
chibicc --unknown main.c
```

produces an error similar to:

```text
unknown argument: --unknown
```

### 18. Collecting input files

```c
strarray_push(&input_paths, argv[i]);
```

1. Any remaining non-option argument is treated as an input.
2. Inputs can include C source files, assembly files, object files, archives, and possibly `"-"` for standard input.
3. Their order is preserved because link order can affect the final result.

Example:

```bash
chibicc main.c helper.o libutil.a -lm
```

produces an ordered input list similar to:

```text
["main.c", "helper.o", "libutil.a", "-lm"]
```

### 19. Appending `-idirafter` directories

```c
for (int i = 0; i < idirafter.len; i++)
  strarray_push(&include_paths, idirafter.data[i]);
```

1. After all arguments have been parsed, delayed directories are appended to `include_paths`.
2. This ensures that normal `-I` directories precede `-idirafter` directories in the search order.

For a corrected parser:

```bash
chibicc -idirafter late -Iearly main.c
```

the resulting order is:

```text
include_paths = ["early", "late"]
```

This remains true even though `-idirafter late` appeared first on the command line.

### 20. Requiring at least one input

```c
if (input_paths.len == 0)
  error("no input files");
```

1. Normal compilation requires at least one source, object, library, or other linker input.
2. Options such as `--help` and `-hashmap-test` exit earlier, so they do not trigger this check.

### 21. Special treatment of `-E`

```c
if (opt_E)
  opt_x = FILE_C;
```

1. `-E` forces the input to be treated as C preprocessing input.
2. This is especially relevant for extensionless files or standard input.
3. It prevents the driver from rejecting an input merely because its filename does not have a recognized C extension.

Example:

```bash
chibicc -E source_without_extension
```

is treated as C preprocessor input.

### 22. Why the parser uses two passes

1. The validation pass detects missing operands before the parser performs side effects such as defining macros or changing global flags.
2. The parsing pass can then safely use expressions such as `argv[++i]`, assuming `take_arg()` correctly identifies every separate-argument option.
3. This design introduces an important invariant: every option consumed with `argv[++i]` in the second pass must also be recognized by `take_arg()`.
4. If that invariant is violated, a trailing option could read `argv[argc]` as though it were a valid string.

Conceptually:

```text
Pass 1:
    Validate option structure.

Pass 2:
    Apply options and collect inputs.

Finalization:
    Append delayed include paths.
    Check for inputs.
    Apply implied option behavior.
```

### 23. Important implementation details

1. `strcmp(a, b) == 0` means the two complete strings are equal.
2. `strncmp(a, prefix, n) == 0` checks only the first `n` characters and is used for attached options.
3. Expressions such as `argv[i] + 2` do not allocate or copy a string; they create a pointer into the existing argument string.
4. Therefore, values such as `opt_o`, `base_file`, and attached option operands remain valid because the `argv` strings remain available for the lifetime of the process.
5. `strarray_push()` stores strings in a dynamically growing array.
6. `format()` returns a newly formatted string and is used when multiple dependency targets must be combined.
7. The parser is order-sensitive: later scalar options overwrite earlier values, while array-valued options accumulate entries.

### 24. Simplified behavior summary

| Option          | Stored result                | Purpose                                  |
|-----------------|------------------------------|------------------------------------------|
| `-E`            | `opt_E`                      | Preprocess only                          |
| `-S`            | `opt_S`                      | Generate assembly only                   |
| `-c`            | `opt_c`                      | Generate object file only                |
| `-o file`       | `opt_o`                      | Set output filename                      |
| `-Ipath`        | `include_paths`              | Add header search directory              |
| `-Dname`        | Macro table                  | Define macro                             |
| `-Uname`        | Macro table                  | Undefine macro                           |
| `-include file` | `opt_include`                | Force-include header                     |
| `-x c`          | `opt_x`                      | Select input language                    |
| `-Lpath`        | `ld_extra_args`              | Add library search directory             |
| `-lfoo`         | `input_paths`                | Link library `foo`                       |
| `-static`       | `opt_static` and linker args | Request static linking                   |
| `-shared`       | `opt_shared` and linker args | Build shared object                      |
| `-fPIC`         | `opt_fpic`                   | Generate position-independent code       |
| `-M`            | `opt_M`                      | Emit dependencies                        |
| `-MD`           | `opt_MD`                     | Compile and emit dependencies            |
| `-MMD`          | `opt_MD`, `opt_MMD`          | Exclude system headers from dependencies |
| `-MF file`      | `opt_MF`                     | Set dependency output file               |
| `-MT target`    | `opt_MT`                     | Set dependency target                    |
| `-MQ target`    | `opt_MT`                     | Set and quote dependency target          |
| `-MP`           | `opt_MP`                     | Add dummy header targets                 |
| `-cc1`          | `opt_cc1`                    | Enter internal compiler mode             |
| `-###`          | `opt_hash_hash_hash`         | Print driver commands                    |

### 25. Central interpretation

1. This function does not perform compilation itself; it converts textual command-line arguments into structured compiler state.
2. The rest of chibicc reads that state to decide which pipeline stages to run and which arguments to pass to the preprocessor, compiler, assembler, and linker.
3. Its most important design properties are left-to-right processing, preservation of linker-input order, delayed handling of `-idirafter`, and compatibility with a useful subset of GCC-style options.
4. The `argv[i++]` expression in the pasted `-idirafter` branch is the notable inconsistency and should be `argv[++i]` if the intention is to record the directory following the option.