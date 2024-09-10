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

- Lexer
--------

- Converts input program into sequence of tokens.
    - Performance-criƟcal.
        -Also handles preprocessing.
        - Various “fast paths” for e.g. skipping through #if 0 blocks, MultipleIncludeOpt, …
- Supports tentaƟve parsing.

- Lexer Example


```lex
1        int factorial(int n) {
2        if (n <= 1)
3        return 1;
4        return n * factorial(n - 1);
5        }

1        > clang -c -Xclang -dump-tokens factorial.c

2        int 'int' [StartOfLine] Loc=<factorial.c:1:1>
3        identifier 'factorial' [LeadingSpace] Loc=<factorial.c:1:5>
4        l_paren '(' Loc=<factorial.c:1:14>
5        int 'int' Loc=<factorial.c:1:15>
6        identifier 'n' [LeadingSpace] Loc=<factorial.c:1:19>
7        r_paren ')' Loc=<factorial.c:1:20>
8        l_brace '{' [LeadingSpace] Loc=<factorial.c:1:22>
9        if 'if' [StartOfLine] [LeadingSpace] Loc=<factorial.c:2:3>
10       l_paren '(' [LeadingSpace] Loc=<factorial.c:2:6>
11       identifier 'n' Loc=<factorial.c:2:7>
12       lessequal '<=' [LeadingSpace] Loc=<factorial.c:2:9>
13       numeric_constant '1' [LeadingSpace] Loc=<factorial.c:2:12>
14       r_paren ')' Loc=<factorial.c:2:13>
15       ...
```

- Lexer Internals

    - Tokens declared in include/clang/Basic/TokenKinds.def
```
...
KEYWORD(if , KEYALL)
KEYWORD(inline , KEYC99|KEYCXX|KEYGNU)
KEYWORD(int , KEYALL)
...
```

    - Token is consumed by include/clang/Parse/Parser.h

```
SourceLocation ConsumeToken() {
...
PP.Lex(Tok);
...
}
bool TryConsumeToken(tok::TokenKind Expected) {
if (Tok.isNot(Expected))
return false;
PP.Lex(Tok);
...
```



