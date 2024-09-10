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
--------
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


```c
KEYWORD(if , KEYALL)
KEYWORD(inline , KEYC99|KEYCXX|KEYGNU)
KEYWORD(int , KEYALL)
```

- Token is consumed by include/clang/Parse/Parser.h

```cpp
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



- Parser
---------

- HandwriƩen recursive-descent parser.
- TentaƟve parsing by looking at the tokens ahead.
- Tries to recover from errors to parse as much as possible (and suggest fix-it hints).

`Parser Call Stack`
```
1 Call stack:
2 clang::Parser::ParseRHSOfBinaryExpression
3 clang::Parser::ParseAssignmentExpression
4 clang::Parser::ParseExpression
5 clang::Parser::ParseParenExprOrCondition
6 clang::Parser::ParseIfStatement
7 ...
8 clang::Parser::ParseStatementOrDeclaration
9 clang::Parser::ParseCompoundStatementBody
10 ...
11 clang::Parser::ParseFunctionDefinition
12 ...
13 clang::Parser::ParseTopLevelDecl
14 clang::Parser::ParseFirstTopLevelDecl
15 clang::ParseAST
16 ...
17 clang::FrontendAction::Execute
18 clang::CompilerInstance::ExecuteAction
19 clang::ExecuteCompilerInvocation
20 cc1_main
```

- First few frames 17-20 belogs to compiler driver.
- First entry point to parsing is ParseAST, where it starts to parse tokens.


-------
- Sema
-------

- Tight coupling with parser.
- Biggest client of the DiagnosƟcs subsystem.
- Checks program validity.
    - If correct generate AST.
    - If incorrect report errors and warnings using clang Diagnostics.
-------
- DiagnosƟcs subsystem
-----------------------

- Purpose: communicate with human through diagnosƟcs:
    - Severity, e.g. note, warning, or error.
    - A source locaƟon, e.g. factorial.c:2:1.
    - A message, e.g. “unknown type name ’inƩ’; did you mean ’int’?”
- Defined in Diagnostic*Kinds.td TableGen files.
- EmiƩed through helper funcƟon Diag().

- DiagnosƟcs example
```c
    factorial.c:2:1: error: unknown type name 'i'
    i factorial(int n) {
    ^
```

- Defined in include/clang/Basic/DiagnosticSemaKinds.td:

```TableGen
def err_unknown_typename : Error<
"unknown type name %0">;
```

- Triggered in lib/Sema/SemaDecl.cpp:


```cpp
void Sema::DiagnoseUnknownTypeName(IdentifierInfo *&II,
SourceLocation IILoc,
...
if (!SS || (!SS->isSet() && !SS->isInvalid()))
Diag(IILoc, IsTemplateName ? diag::err_no_template
: diag::err_unknown_typename)
<< II;
```

- Once we parse our program which is symmantically correct,
  what happend next is AST. (Very close to source representation of program.)
- AST is immutable, once created can't be modified. unless,
    - apart from special case like templates, where clang provides functionality to perform some modification.


-------------
- AST Example
-------------

```cpp
1 > clang -c -Xclang -ast-dump factorial.c
2 FunctionDecl <factorial.c:2:1, line:6:1> line:2:5 referenced factorial 'int (int)'
3 |-ParmVarDecl <col:15, col:19> col:19 used n 'int'
4 `-CompoundStmt <col:22, line:6:1>
5 |-IfStmt <line:3:3, line:4:12>
6 | |-BinaryOperator <line:3:7, col:12> 'int' '<='
7 | | |-ImplicitCastExpr <col:7> 'int' <LValueToRValue>
8 | | | `-DeclRefExpr <col:7> 'int' lvalue ParmVar 'n' 'int'
9 | | `-IntegerLiteral <col:12> 'int' 1
10 | `-ReturnStmt <line:4:5, col:12>
11 | `-IntegerLiteral <col:12> 'int' 1
12 `-ReturnStmt <line:5:3, col:29>
13 `-...

```
-------------

--------------
- AST Visitors
--------------

- RecursiveASTVisitor for visiƟng the full AST.
- StmtVisitor for visiƟng Stmt and Expr.
- TypeVisitor for visiƟng Type hierarchy.
--------------


---------
- CodeGen
---------

- Not to be confused with LLVM CodeGen! (which generates machine code)
- Uses AST visitors, IRBuilder, and TargetInfo.
- CodeGenModule class keeps global state, e.g. LLVM type cache.
  Emits global and some shared enƟƟes.
- CodeGenFunction class keeps per funcƟon state.
  Emits LLVM IR for funcƟon body statements.
---------
