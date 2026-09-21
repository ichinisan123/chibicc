This is an excellent piece of `chibicc` to study because **`main()` is essentially the compiler driver's control center**.

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

# 1. Function signature

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

# 2. Register cleanup function

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

# 3. Initialize macros

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

# 4. Parse command-line arguments

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

# 5. Special `cc1` mode

```c
if (opt_cc1) {
  add_default_include_paths(argv[0]);
  cc1();
  return 0;
}
```

This is an important compiler architecture concept.

## What is `cc1`?

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

## `opt_cc1`

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

# 6. Check invalid `-o` usage

```c
if (input_paths.len > 1 && opt_o && (opt_c || opt_S | opt_E))
  error("cannot specify '-o' with '-c,' '-S' or '-E' with multiple files");
```

This checks for an invalid command-line combination.

There are three conditions.

### Condition 1

```c
input_paths.len > 1
```

There are multiple input files.

For example:

```bash
chibicc a.c b.c
```

### Condition 2

```c
opt_o
```

The user specified an output filename:

```bash
-o something
```

### Condition 3

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

# 7. Create linker argument array

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

# 8. Process every input file

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
