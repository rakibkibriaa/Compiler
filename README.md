# C Compiler with Flex and Bison

A compiler for a subset of the **C** language, built in four stages: a symbol table, a lexical analyzer, a syntax and semantic analyzer, and an intermediate code generator that outputs **8086 assembly**. The scanner is written with **Flex**, the parser with **Bison (Yacc)**, and everything else in **C++**.

Built as part of the **CSE 310: Compiler Sessional** coursework.

---

## Overview

The project follows the classic compiler pipeline. Each stage builds on the one before it, and each lives in its own folder:

```
source.c ──▶ Lexical Analysis ──▶ Syntax & Semantic Analysis ──▶ Code Generation ──▶ code.asm
                 (Flex)                  (Bison)                    (8086 asm)
                    │                       │                           │
                    └───────────── Symbol Table (shared) ───────────────┘
```

| Stage | Folder | Output |
|---|---|---|
| 1. Symbol Table | `Symbol Table` | Scoped symbol table, tested with a command file |
| 2. Lexical Analysis | `Lexical Analysis` | Token stream and lexical error log |
| 3. Syntax & Semantic Analysis | `Syntax and Symantic` | Parse tree, semantic error report and log |
| 4. Intermediate Code Generation | `ICG` | 8086 assembly, plus an optimized version |

---

## Supported Language

The compiler accepts a C subset with:

- **Types:** `int`, `float`, `void`
- **Declarations:** global and local variables, one-dimensional arrays, function declarations and definitions
- **Statements:** `if`, `if-else`, `for`, `while`, `return`, compound blocks, and `println(x)` for output
- **Expressions:** arithmetic (`+ - * / %`), relational, logical (`&& ||`), unary (`- !`), increment and decrement, assignment, array indexing and function calls

---

## Stage 1: Symbol Table

A scoped symbol table that the later stages use to track identifiers.

- Each scope is a **hash table** with separate chaining, using the **SDBM** hash function.
- Scopes are stacked, so entering a block creates a new scope and exiting it restores the parent. Lookups search from the innermost scope outward.

It is tested with a command file of operations:

| Command | Action |
|---|---|
| `I <name> <type>` | Insert a symbol into the current scope |
| `L <name>` | Look up a symbol, searching from the innermost scope |
| `D <name>` | Delete a symbol from the current scope |
| `P A` / `P C` | Print all scopes / the current scope |
| `S` | Enter a new scope |
| `E` | Exit the current scope |
| `Q` | Quit |

---

## Stage 2: Lexical Analysis

A Flex scanner that turns source code into tokens and inserts identifiers into the symbol table.

- **Recognizes** keywords, identifiers, integer, float and character literals, operators, punctuation, single-line and multi-line strings, and both comment styles.
- **Handles escapes** in characters and strings, and line continuations with `\`.
- **Reports lexical errors** with line numbers, including:
  - Too many decimal points, and ill-formed numbers
  - Empty, unfinished or multi-character character constants
  - Unfinished strings and comments
  - Identifiers that start with a digit
  - Unrecognized characters

---

## Stage 3: Syntax and Semantic Analysis

A Bison parser that checks the program against the grammar, builds a **parse tree**, and performs **semantic checks** with the symbol table. It resolves the dangling-`else` ambiguity with precedence rules.

Semantic errors and warnings caught include:

- **Declarations:** undeclared variables and functions, multiple declarations, redeclaration as a different kind of symbol, variables declared `void`, and functions declared but never defined
- **Functions:** conflicting return types, too few or too many arguments, argument type mismatches, and parameter redefinition
- **Types:** non-integer array subscripts, non-integer operands for `%`, `void` used in expressions, and indexing something that isn't an array
- **Warnings:** division by zero, and possible data loss when assigning or returning a `float` as an `int`

---

## Stage 4: Intermediate Code Generation

Extends the parser to generate **8086 assembly** while it parses, so the whole compiler runs in a single pass.

- **Memory layout:** globals go in the data segment. Locals and parameters live on the stack and are addressed through `BP`, with negative offsets for locals and positive offsets for parameters. Arrays are indexed with `SI`.
- **Control flow:** `if-else`, `for` and `while` are translated with generated labels and conditional jumps. Relational and logical operators produce `0` or `1` values.
- **Functions:** follow a standard `PUSH BP` / `MOV BP,SP` prologue and epilogue, with the return value passed in `AX`.
- **Output:** `println` calls a built-in procedure that prints signed integers.

A **peephole optimizer** then makes a second pass over the assembly and removes redundant instruction pairs:

- A `MOV A,B` directly followed by `MOV B,A`
- A `PUSH X` directly followed by `POP X`

---

## Repository Structure

```
Compiler/
├── Symbol Table/
│   └── 1905098.cpp                 # Symbol table and command-file driver
├── Lexical Analysis/
│   ├── 1905098.l                   # Flex scanner
│   └── 1905098_Symbol_Table.h
├── Syntax and Symantic/
│   ├── 1905098.l                   # Scanner feeding the parser
│   ├── 1905098.y                   # Bison grammar with semantic checks
│   ├── 1905098_Helper.h            # Parse tree node
│   └── 1905098_Symbol_Table.h
└── ICG/
    ├── 1905098.l
    ├── 1905098.y                   # Grammar with code generation and optimizer
    ├── 1905098_Helper.h
    ├── 1905098_Symbol_Table.h
    └── 1905098.sh                  # Build and run script
```

---

## Getting Started

### Prerequisites
- `g++`
- `flex`
- `bison` (run as `yacc`)
- For running the generated assembly: an 8086 emulator such as **EMU8086**, or **MASM/TASM** in DOSBox

On Ubuntu:

```bash
sudo apt install g++ flex bison
```

### Stage 1: Symbol Table

Place the commands in `sample_input.txt`. The first line is the number of buckets per scope.

```bash
cd "Symbol Table"
g++ 1905098.cpp -o symbol_table
./symbol_table
```

Output: `1905098_out.txt`

### Stage 2: Lexical Analysis

```bash
cd "Lexical Analysis"
flex 1905098.l
g++ lex.yy.c -o scanner
./scanner input.c
```

Output: `1905098_token.txt` and `1905098_log.txt`

### Stage 3: Syntax and Semantic Analysis

```bash
cd "Syntax and Symantic"
yacc -d -y 1905098.y
g++ -w -c -o y.o y.tab.c
flex 1905098.l
g++ -fpermissive -w -c -o l.o lex.yy.c
g++ y.o l.o -o parser
./parser input.c
```

Output: `1905098_parsetree.txt`, `1905098_error.txt` and `1905098_log.txt`

### Stage 4: Code Generation

Place the source program in `input.c`, then run the build script:

```bash
cd ICG
bash 1905098.sh
```

Output: `code.asm` and `optimized_code.asm`, along with the parse tree, error and log files. Load either `.asm` file in EMU8086 or assemble it with MASM to run it.

---

## Example

```c
int main(){
    int a, b;
    a = 3;
    b = a * 2 + 1;
    println(b);
    return 0;
}
```

Compiling this program generates `code.asm`. Running it in an 8086 emulator prints the value of `b`, which is `7`.
