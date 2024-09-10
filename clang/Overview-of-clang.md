# Overview of clang

- Supporting C, C++, Objective C/C++, OpenCL, CUDA, RenderScript.

- Clang is a compiler driver.
    - Driving all phases of a compiler invocaƟon, e.g. preprocessing, compiling, linking.
    - Seƫng flags for current build/installaƟon (e.g. paths to include files).

- Clang is a C language family frontend.
    - Compiling C-like code to LLVM IR.
    - Also known as CFE (C frontend), cc1, or clang_cc1.

- Compiler driver phases

```
C File -> ||| Preprocessor -> Frontend -> Middlend -> Backend(codegen) -> Assembler -> Linker ||| -> binary
```

```
> clang -ccc-print-phases factorial.c
0: input, "factorial.c", c
1: preprocessor , {0}, cpp-output
2: compiler , {1}, ir
3: backend , {2}, assembler
4: assembler , {3}, object
5: linker, {4}, image
```

- Phases combined into tool executions.
- Driver invokes the frontend(cc1), linker, ... with the appropriate flags.

```
> clang -### factorial.c
clang version 10.0.0
Target: x86_64-unknown -linux-gnu
Thread model: posix
InstalledDir: /data/llvm/build/bin
"/data/llvm/build/bin/clang -10" "-cc1" "-triple" "x86_64-unknown -linux-gnu" "-emit-obj"
"-mrelax-all" "-disable -free" "-main-file-name" "factorial.c"
"-mrelocation -model" "static" "-mthread -model" "posix"
"-mframe-pointer=all" "-fmath-errno"
"-internal -isystem" "/data/llvm/build/lib/clang/10.0.0/include"
...
"-x" "c" "factorial.c"
"/usr/bin/ld" "-z" "relro" "--hash-style=gnu" "--eh-frame-hdr" "-m" "elf_x86_64"
"-dynamic -linker" "/lib64/ld-linux-x86-64.so.2" "-o" "a.out"

```

## Clang as language Frontend

- Compiling C-like code to LLVM IR.
    - …and emit helpful diagnostics.
    - …and support various standards and dialects.
    - …and record source locations for debug informaƟon.
    - …and provide foundation for many other tools (syntax highlighƟng, code compleƟon,
    - code refactoring, static analysis, …).

### Core components of clang

```
Preprocessor -> Frontend
```

```
  ||Preprocessor & Lexer|| -> 'Tokens' -> ||Parser|| -> ||Sema|| -> 'AST' -> ||CodeGen|| -> 'LLVM IR'
```
