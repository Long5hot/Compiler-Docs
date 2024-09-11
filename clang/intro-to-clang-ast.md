# Introduction to clang AST

- The Structure of the Clang AST

    ● rich AST representation
    ● fully type resolved

### ASTContext

● Keeps information around the AST
    ○ Identifier Table
    ○ Source Manager
● Entry point into the AST
    ○ TranslationUnitDecl* getTranslationUnitDecl()

### Core Classes

- Decl
    ○ CXXRecordDecl
    ○ VarDecl
    ○ UnresolvedUsingTypenameDec

- Stmt
    ○ CompoundStmt
    ○ CXXTryStmt
    ○ BinaryOperator - BinaryOperator is an expression. in clang AST expressions are statements.

- Type
    ○ PointerType
    ○ ParenType
    ○ SubstTemplateTypeParmType 

### Glue Classes



