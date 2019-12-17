---
layout: post
title:  "LLVM教程（三）LLVM IR代码生成"
date:   2019-12-17"
excerpt: "本章将介绍如何把第二章中构造的抽象语法树转换成LLVM IR。"
tag:
- 编译器
comments: false
---

本文为LLVM教程的笔记（三）。

## 1.准备工作

先在每个AST类添加一个虚函数Codegen（code generation），实现代码生成。

Codegen()方法负责生成该类型AST节点的IR代码及其他必要信息，生成的内容会以LLVM对象的形式返回。

LLVM用value类表示静态一次性赋值寄存器或者说SSA值（static single assignment），SSA值由对应的指令运算得出之后就会固定下来，是不变的，而且知道该指令再次执行之前，SSA值都不可修改。

“这个概念并不难，习惯了就好。”

    /// ExprAST - Base class for all expression nodes.
    class ExprAST {
    public:
      virtual ~ExprAST() {}
      virtual Value *Codegen() = 0;
    };

    /// NumberExprAST - Expression class for numeric literals like "1.0".
    class NumberExprAST : public ExprAST {
      double Val;
    public:
      NumberExprAST(double val) : Val(val) {}
      virtual Value *Codegen();
    };
    ...
    
除了在ExprAST类中添加虚方法，还可以利用visitor模式等其他方法实现代码生成，“但就当前需求，添加虚函数是最好的方法”。

此外，还需要Error方法，就像语法解析器里的报错函数，它用来报告代码生成过程中发生的错误，比如引用未经声明的参数：

    Value *ErrorV(const char *Str) { Error(Str); return 0; }

    // TheModule是LLVM中用于存放代码段中所有函数和全局变量的结构。从某种意义上讲，可以把它当作LLVM IR代码的顶层容器。
    static Module *TheModule; 
    // Builder是用于简化LLVM指令生成的辅助对象，IRBuilder类模板的实例可用于跟踪当前插入指令的位置，同时还带有用于生成新指令的方法。
    static IRBuilder<> Builder(getGlobalContext());
    // NamedValues映射表用于记录定义于当前作用域内的变量及与之相对应的LLVM表示（换言之，也就是代码的符号表）。
    static std::map<std::string, Value*> NamedValues;

在kaleidoscope中，可引用的变量只有函数的参数，会存在 NamedValues表中。

生成代码之前比如设置Builder对象，指明写入代码的位置。

## 2.表达式代码生成

1. 数值常量

LLVM IR中的数值常量由ConstantFP表示，在其内部又具体由APFloat表示。

        Value *NumberExprAST::Codegen() {
        return ConstantFP::get(getGlobalContext(), APFloat(Val));
        }

2. 引用变量

假设引用的变量已经在某处定义赋值，NamedValues映射表中的变量只可能是函数的调用参数。

这段代码首先确认给定的变量名是否存在于映射表中（如果不存在，就说明引用了未定义的变量）然后返回该变量的值。

        Value *VariableExprAST::Codegen() {
        // Look this variable up in the function.
        Value *V = NamedValues[Name];
        return V ? V : ErrorV("Unknown variable name");
        }

3. 二元运算符

递归地生成代码，先处理表达式的左侧，再处理表达式的右侧，最后计算整个二元表达式的值。

        Value *BinaryExprAST::Codegen() {
          Value *L = LHS->Codegen();
          Value *R = RHS->Codegen();
          if (L == 0 || R == 0) return 0;

          // opcode的取值用了一个简单的switch语句，从而为各种二元运算符创建出相应的LLVM指令。
          switch (Op) {
          // 只需想清楚该用哪些操作数（即此处的L和R）生成哪条指令（通过调用CreateFAdd等方法）即可，至于新指令该插入到什么位置，交给IRBuilder就可以了
          // 此处的指令名只是一个提示。假设代码生成了多条“addtmp”指令，LLVM会自动给每条指令的名字追加一个自增的唯一数字后缀。
          case '+': return Builder.CreateFAdd(L, R, "addtmp");
          case '-': return Builder.CreateFSub(L, R, "subtmp");
          case '*': return Builder.CreateFMul(L, R, "multmp");
          case '<':
          // add指令的Left、Right操作数必须同属一个类型，结果的类型则必须与操作数的类型相容。
          // fcmp指令的返回值类型必须是‘i1’（单比特整数），而Kaleidoscope只能接受0.0或1.0。为了弥合语义上的差异，给fcmp指令配上一条uitofp指令。
          // 这条指令会将输入的整数视作无符号数，并将之转换成浮点数。
          // 如果用的是sitofp指令，Kaleidoscope的‘<’运算符将视输入的不同而返回0.0或-1.0。
            L = Builder.CreateFCmpULT(L, R, "cmptmp");
            // Convert bool 0/1 to double 0.0 or 1.0
            return Builder.CreateUIToFP(L, Type::getDoubleTy(getGlobalContext()),
                                        "booltmp");
          default: return ErrorV("invalid binary operator");
          }
        }
        
4. 函数调用

代码开头的几行将在LLVM Module的符号表中查找函数名，LLVM Module是个容器，待处理的函数全都在里面。

只要函数的名字与用户指定的函数名一致，LLVM的符号表就可以替我们完成函数名的解析。

等取到了该调用的函数，就能递归地生成传入的各参数代码，并创建一条LLVM call指令。

        Value *CallExprAST::Codegen() {
        // Look up the name in the global module table.
        Function *CalleeF = TheModule->getFunction(Callee);
        if (CalleeF == 0)
        return ErrorV("Unknown function referenced");

        // If argument mismatch error.
        if (CalleeF->arg_size() != Args.size())
        return ErrorV("Incorrect # arguments passed");

        std::vector<Value*> ArgsV;
        for (unsigned i = 0, e = Args.size(); i != e; ++i) {
        ArgsV.push_back(Args[i]->Codegen());
        if (ArgsV.back() == 0) return 0;
        }

        return Builder.CreateCall(CalleeF, ArgsV, "calltmp");
        }

## 3.函数的代码生成

1. 函数原型的代码生成过程

函数的返回值类型将是Function \*而不是Value \*，因为函数原型描述的是函数的对外接口而不是表达式计算出的值。

        Function *PrototypeAST::Codegen() {
        // Make the function type:  double(double,double) etc.
        // 创建了double的vector
        std::vector<Type*> Doubles(Args.size(),
        Type::getDoubleTy(getGlobalContext()));
        // FunctionType::get调用用于为给定的函数原型创建对应的FunctionType对象，创建出一个参数不可变的函数类型（false）。
        FunctionType *FT = FunctionType::get(Type::getDoubleTy(getGlobalContext()),
        Doubles, false);

        // 创建与该函数原型相对应的函数，包含了类型、链接方式和函数名等，并指定该函数待插入的模块。
        // “ExternalLinkage”表示该函数可能定义于当前模块之外，且/或可以被当前模块之外的函数调用。
        // Name是用户指定的函数名.
        // 函数定义在“TheModule”内，函数名自然也注册在“TheModule”的符号表内。
        Function *F = Function::Create(FT, Function::ExternalLinkage, Name, TheModule);
        
2. 检查函数是否被定义过

处理名称冲突时，Module的符号表与Function的符号表类似：在模块中添加新函数时，如果发现函数名与符号表中现有的名称重复，新函数会被默默地重命名。

“对于Kaleidoscope，在两种情况下允许重定义函数：第一，允许对同一个函数进行多次extern声明，前提是所有声明中的函数原型保持一致（由于只有一种参数类型，我们只需要检查参数的个数是否匹配即可）。第二，允许先对函数进行extern声明，再定义函数体。这样一来，才能定义出相互递归调用的函数。”

        // If F conflicted, there was already something named 'Name'.  If it has a
        // body, don't allow redefinition or reextern.
        // 如果有函数名冲突，（调用eraseFunctionParent）将刚刚创建的函数对象删除
        if (F->getName() != Name) {
        // Delete the one we just made and get the existing one.
        F->eraseFromParent();
        // 然后调用getFunction获取与函数名相对应的函数对象。
        F = TheModule->getFunction(Name);


3. 检查函数基本块与参数个数

"看看之前定义的函数对象是否为“空”。换言之，也就是看看该函数有没有定义基本块。没有基本块就意味着该函数尚未定义函数体，只是一个前导声明。如果已经定义了函数体，就不能继续下去了，抛出错误予以拒绝。"

"如果之前的函数对象只是个“extern”声明，则检查该函数的参数个数是否与当前的参数个数相符。如果不符，抛出错误。"

    // If F already has a body, reject this.
    
    if (!F->empty()) {
    ErrorF("redefinition of function");
    return 0;
    }

    // If F took a different number of args, reject.
    if (F->arg_size() != Args.size()) {
    ErrorF("redefinition of function with different # args");
    return 0;
    }
    
4. 设置参数名
    
接着遍历函数原型所有参数，为这些LLVM Argument对象设置参数名，并把这些参数注册到NameValues映射表，最后返回Function对象。

    // Set names for all arguments.
    unsigned Idx = 0;
    for (Function::arg_iterator AI = F->arg_begin(); Idx != Args.size();
    ++AI, ++Idx) {
    AI->setName(Args[Idx]);

    // Add arguments to variable symbol table.
    NamedValues[Args[Idx]] = AI;
    }
    
5. 函数定义的代码生成过程

生成函数原型（proto）的代码并且校验，清空NamedValues映射表，确保之前生成代码过程产生的内容不会残留。

        Function *FunctionAST::Codegen() {
        NamedValues.clear();

        Function *TheFunction = Proto->Codegen();
        if (TheFunction == 0)
        return 0;
        
6. 设置Builder对象

        // Create a new basic block to start insertion into.
        // 新建entry基本块，将该对象插入TheFunction
        BasicBlock *BB = BasicBlock::Create(getGlobalContext(), "entry", TheFunction);
        // 告诉Builder后续指令插至新建的基本块末尾
        Builder.SetInsertPoint(BB);
        
    LLVM基本块是用于定义控制流图（Control Flow Graph）的重要部件。当前我们还不涉及到控制流，所以所有的函数都只有一个基本块。
    
7. 调用函数主表达式的CodegGen方法

        if (Value *RetVal = Body->Codegen()) {
        // Finish off the function.
        Builder.CreateRet(RetVal);

        // Validate the generated code, checking for consistency.
        verifyFunction(*TheFunction);

        return TheFunction;
        }
        
## 4.完整代码

    // To build this:
    // See example below.

    #include "llvm/DerivedTypes.h"
    #include "llvm/IRBuilder.h"
    #include "llvm/LLVMContext.h"
    #include "llvm/Module.h"
    #include "llvm/Analysis/Verifier.h"
    #include <cstdio>
    #include <string>
    #include <map>
    #include <vector>
    using namespace llvm;

    //===----------------------------------------------------------------------===//
    // Lexer
    //===----------------------------------------------------------------------===//

    // The lexer returns tokens [0-255] if it is an unknown character, otherwise one
    // of these for known things.
    enum Token {
    tok_eof = -1,

    // commands
    tok_def = -2, tok_extern = -3,

    // primary
    tok_identifier = -4, tok_number = -5
    };

    static std::string IdentifierStr;  // Filled in if tok_identifier
    static double NumVal;              // Filled in if tok_number

    /// gettok - Return the next token from standard input.
    static int gettok() {
    static int LastChar = ' ';

    // Skip any whitespace.
    while (isspace(LastChar))
    LastChar = getchar();

    if (isalpha(LastChar)) { // identifier: [a-zA-Z][a-zA-Z0-9]*
    IdentifierStr = LastChar;
    while (isalnum((LastChar = getchar())))
    IdentifierStr += LastChar;

    if (IdentifierStr == "def") return tok_def;
    if (IdentifierStr == "extern") return tok_extern;
    return tok_identifier;
    }

    if (isdigit(LastChar) || LastChar == '.') {   // Number: [0-9.]+
    std::string NumStr;
    do {
    NumStr += LastChar;
    LastChar = getchar();
    } while (isdigit(LastChar) || LastChar == '.');

    NumVal = strtod(NumStr.c_str(), 0);
    return tok_number;
    }

    if (LastChar == '#') {
    // Comment until end of line.
    do LastChar = getchar();
    while (LastChar != EOF && LastChar != '\n' && LastChar != '\r');

    if (LastChar != EOF)
    return gettok();
    }

    // Check for end of file.  Don't eat the EOF.
    if (LastChar == EOF)
    return tok_eof;

    // Otherwise, just return the character as its ascii value.
    int ThisChar = LastChar;
    LastChar = getchar();
    return ThisChar;
    }

    //===----------------------------------------------------------------------===//
    // Abstract Syntax Tree (aka Parse Tree)
    //===----------------------------------------------------------------------===//

    /// ExprAST - Base class for all expression nodes.
    class ExprAST {
    public:
    virtual ~ExprAST() {}
    virtual Value *Codegen() = 0;
    };

    /// NumberExprAST - Expression class for numeric literals like "1.0".
    class NumberExprAST : public ExprAST {
    double Val;
    public:
    NumberExprAST(double val) : Val(val) {}
    virtual Value *Codegen();
    };

    /// VariableExprAST - Expression class for referencing a variable, like "a".
    class VariableExprAST : public ExprAST {
    std::string Name;
    public:
    VariableExprAST(const std::string &name) : Name(name) {}
    virtual Value *Codegen();
    };

    /// BinaryExprAST - Expression class for a binary operator.
    class BinaryExprAST : public ExprAST {
    char Op;
    ExprAST *LHS, *RHS;
    public:
    BinaryExprAST(char op, ExprAST *lhs, ExprAST *rhs)
    : Op(op), LHS(lhs), RHS(rhs) {}
    virtual Value *Codegen();
    };

    /// CallExprAST - Expression class for function calls.
    class CallExprAST : public ExprAST {
    std::string Callee;
    std::vector<ExprAST*> Args;
    public:
    CallExprAST(const std::string &callee, std::vector<ExprAST*> &args)
    : Callee(callee), Args(args) {}
    virtual Value *Codegen();
    };

    /// PrototypeAST - This class represents the "prototype" for a function,
    /// which captures its name, and its argument names (thus implicitly the number
    /// of arguments the function takes).
    class PrototypeAST {
    std::string Name;
    std::vector<std::string> Args;
    public:
    PrototypeAST(const std::string &name, const std::vector<std::string> &args)
    : Name(name), Args(args) {}

    Function *Codegen();
    };

    /// FunctionAST - This class represents a function definition itself.
    class FunctionAST {
    PrototypeAST *Proto;
    ExprAST *Body;
    public:
    FunctionAST(PrototypeAST *proto, ExprAST *body)
    : Proto(proto), Body(body) {}

    Function *Codegen();
    };

    //===----------------------------------------------------------------------===//
    // Parser
    //===----------------------------------------------------------------------===//

    /// CurTok/getNextToken - Provide a simple token buffer.  CurTok is the current
    /// token the parser is looking at.  getNextToken reads another token from the
    /// lexer and updates CurTok with its results.
    static int CurTok;
    static int getNextToken() {
    return CurTok = gettok();
    }

    /// BinopPrecedence - This holds the precedence for each binary operator that is
    /// defined.
    static std::map<char, int> BinopPrecedence;

    /// GetTokPrecedence - Get the precedence of the pending binary operator token.
    static int GetTokPrecedence() {
    if (!isascii(CurTok))
    return -1;

    // Make sure it's a declared binop.
    int TokPrec = BinopPrecedence[CurTok];
    if (TokPrec <= 0) return -1;
    return TokPrec;
    }

    /// Error* - These are little helper functions for error handling.
    ExprAST *Error(const char *Str) { fprintf(stderr, "Error: %s\n", Str);return 0;}
    PrototypeAST *ErrorP(const char *Str) { Error(Str); return 0; }
    FunctionAST *ErrorF(const char *Str) { Error(Str); return 0; }

    static ExprAST *ParseExpression();

    /// identifierexpr
    ///   ::= identifier
    ///   ::= identifier '(' expression* ')'
    static ExprAST *ParseIdentifierExpr() {
    std::string IdName = IdentifierStr;

    getNextToken();  // eat identifier.

    if (CurTok != '(') // Simple variable ref.
    return new VariableExprAST(IdName);

    // Call.
    getNextToken();  // eat (
    std::vector<ExprAST*> Args;
    if (CurTok != ')') {
    while (1) {
    ExprAST *Arg = ParseExpression();
    if (!Arg) return 0;
    Args.push_back(Arg);

    if (CurTok == ')') break;

    if (CurTok != ',')
    return Error("Expected ')' or ',' in argument list");
    getNextToken();
    }
    }

    // Eat the ')'.
    getNextToken();

    return new CallExprAST(IdName, Args);
    }

    /// numberexpr ::= number
    static ExprAST *ParseNumberExpr() {
    ExprAST *Result = new NumberExprAST(NumVal);
    getNextToken(); // consume the number
    return Result;
    }

    /// parenexpr ::= '(' expression ')'
    static ExprAST *ParseParenExpr() {
    getNextToken();  // eat (.
    ExprAST *V = ParseExpression();
    if (!V) return 0;

    if (CurTok != ')')
    return Error("expected ')'");
    getNextToken();  // eat ).
    return V;
    }

    /// primary
    ///   ::= identifierexpr
    ///   ::= numberexpr
    ///   ::= parenexpr
    static ExprAST *ParsePrimary() {
    switch (CurTok) {
    default: return Error("unknown token when expecting an expression");
    case tok_identifier: return ParseIdentifierExpr();
    case tok_number:     return ParseNumberExpr();
    case '(':            return ParseParenExpr();
    }
    }

    /// binoprhs
    ///   ::= ('+' primary)*
    static ExprAST *ParseBinOpRHS(int ExprPrec, ExprAST *LHS) {
    // If this is a binop, find its precedence.
    while (1) {
    int TokPrec = GetTokPrecedence();

    // If this is a binop that binds at least as tightly as the current binop,
    // consume it, otherwise we are done.
    if (TokPrec < ExprPrec)
    return LHS;

    // Okay, we know this is a binop.
    int BinOp = CurTok;
    getNextToken();  // eat binop

    // Parse the primary expression after the binary operator.
    ExprAST *RHS = ParsePrimary();
    if (!RHS) return 0;

    // If BinOp binds less tightly with RHS than the operator after RHS, let
    // the pending operator take RHS as its LHS.
    int NextPrec = GetTokPrecedence();
    if (TokPrec < NextPrec) {
    RHS = ParseBinOpRHS(TokPrec+1, RHS);
    if (RHS == 0) return 0;
    }

    // Merge LHS/RHS.
    LHS = new BinaryExprAST(BinOp, LHS, RHS);
    }
    }

    /// expression
    ///   ::= primary binoprhs
    ///
    static ExprAST *ParseExpression() {
    ExprAST *LHS = ParsePrimary();
    if (!LHS) return 0;

    return ParseBinOpRHS(0, LHS);
    }

    /// prototype
    ///   ::= id '(' id* ')'
    static PrototypeAST *ParsePrototype() {
    if (CurTok != tok_identifier)
    return ErrorP("Expected function name in prototype");

    std::string FnName = IdentifierStr;
    getNextToken();

    if (CurTok != '(')
    return ErrorP("Expected '(' in prototype");

    std::vector<std::string> ArgNames;
    while (getNextToken() == tok_identifier)
    ArgNames.push_back(IdentifierStr);
    if (CurTok != ')')
    return ErrorP("Expected ')' in prototype");

    // success.
    getNextToken();  // eat ')'.

    return new PrototypeAST(FnName, ArgNames);
    }

    /// definition ::= 'def' prototype expression
    static FunctionAST *ParseDefinition() {
    getNextToken();  // eat def.
    PrototypeAST *Proto = ParsePrototype();
    if (Proto == 0) return 0;

    if (ExprAST *E = ParseExpression())
    return new FunctionAST(Proto, E);
    return 0;
    }

    /// toplevelexpr ::= expression
    static FunctionAST *ParseTopLevelExpr() {
    if (ExprAST *E = ParseExpression()) {
    // Make an anonymous proto.
    PrototypeAST *Proto = new PrototypeAST("", std::vector<std::string>());
    return new FunctionAST(Proto, E);
    }
    return 0;
    }

    /// external ::= 'extern' prototype
    static PrototypeAST *ParseExtern() {
    getNextToken();  // eat extern.
    return ParsePrototype();
    }

    //===----------------------------------------------------------------------===//
    // Code Generation
    //===----------------------------------------------------------------------===//

    static Module *TheModule;
    static IRBuilder<> Builder(getGlobalContext());
    static std::map<std::string, Value*> NamedValues;

    Value *ErrorV(const char *Str) { Error(Str); return 0; }

    Value *NumberExprAST::Codegen() {
    return ConstantFP::get(getGlobalContext(), APFloat(Val));
    }

    Value *VariableExprAST::Codegen() {
    // Look this variable up in the function.
    Value *V = NamedValues[Name];
    return V ? V : ErrorV("Unknown variable name");
    }

    Value *BinaryExprAST::Codegen() {
    Value *L = LHS->Codegen();
    Value *R = RHS->Codegen();
    if (L == 0 || R == 0) return 0;

    switch (Op) {
    case '+': return Builder.CreateFAdd(L, R, "addtmp");
    case '-': return Builder.CreateFSub(L, R, "subtmp");
    case '*': return Builder.CreateFMul(L, R, "multmp");
    case '<':
    L = Builder.CreateFCmpULT(L, R, "cmptmp");
    // Convert bool 0/1 to double 0.0 or 1.0
    return Builder.CreateUIToFP(L, Type::getDoubleTy(getGlobalContext()),
    "booltmp");
    default: return ErrorV("invalid binary operator");
    }
    }

    Value *CallExprAST::Codegen() {
    // Look up the name in the global module table.
    Function *CalleeF = TheModule->getFunction(Callee);
    if (CalleeF == 0)
    return ErrorV("Unknown function referenced");

    // If argument mismatch error.
    if (CalleeF->arg_size() != Args.size())
    return ErrorV("Incorrect # arguments passed");

    std::vector<Value*> ArgsV;
    for (unsigned i = 0, e = Args.size(); i != e; ++i) {
    ArgsV.push_back(Args[i]->Codegen());
    if (ArgsV.back() == 0) return 0;
    }

    return Builder.CreateCall(CalleeF, ArgsV, "calltmp");
    }

    Function *PrototypeAST::Codegen() {
    // Make the function type:  double(double,double) etc.
    std::vector<Type*> Doubles(Args.size(),
    Type::getDoubleTy(getGlobalContext()));
    FunctionType *FT = FunctionType::get(Type::getDoubleTy(getGlobalContext()),
    Doubles, false);

    Function *F = Function::Create(FT, Function::ExternalLinkage, Name, TheModule);

    // If F conflicted, there was already something named 'Name'.  If it has a
    // body, don't allow redefinition or reextern.
    if (F->getName() != Name) {
    // Delete the one we just made and get the existing one.
    F->eraseFromParent();
    F = TheModule->getFunction(Name);

    // If F already has a body, reject this.
    if (!F->empty()) {
    ErrorF("redefinition of function");
    return 0;
    }

    // If F took a different number of args, reject.
    if (F->arg_size() != Args.size()) {
    ErrorF("redefinition of function with different # args");
    return 0;
    }
    }

    // Set names for all arguments.
    unsigned Idx = 0;
    for (Function::arg_iterator AI = F->arg_begin(); Idx != Args.size();
    ++AI, ++Idx) {
    AI->setName(Args[Idx]);

    // Add arguments to variable symbol table.
    NamedValues[Args[Idx]] = AI;
    }

    return F;
    }

    Function *FunctionAST::Codegen() {
    NamedValues.clear();

    Function *TheFunction = Proto->Codegen();
    if (TheFunction == 0)
    return 0;

    // Create a new basic block to start insertion into.
    BasicBlock *BB = BasicBlock::Create(getGlobalContext(), "entry", TheFunction);
    Builder.SetInsertPoint(BB);

    if (Value *RetVal = Body->Codegen()) {
    // Finish off the function.
    Builder.CreateRet(RetVal);

    // Validate the generated code, checking for consistency.
    verifyFunction(*TheFunction);

    return TheFunction;
    }

    // Error reading body, remove function.
    TheFunction->eraseFromParent();
    return 0;
    }

    //===----------------------------------------------------------------------===//
    // Top-Level parsing and JIT Driver
    //===----------------------------------------------------------------------===//

    static void HandleDefinition() {
    if (FunctionAST *F = ParseDefinition()) {
    if (Function *LF = F->Codegen()) {
    fprintf(stderr, "Read function definition:");
    LF->dump();
    }
    } else {
    // Skip token for error recovery.
    getNextToken();
    }
    }

    static void HandleExtern() {
    if (PrototypeAST *P = ParseExtern()) {
    if (Function *F = P->Codegen()) {
    fprintf(stderr, "Read extern: ");
    F->dump();
    }
    } else {
    // Skip token for error recovery.
    getNextToken();
    }
    }

    static void HandleTopLevelExpression() {
    // Evaluate a top-level expression into an anonymous function.
    if (FunctionAST *F = ParseTopLevelExpr()) {
    if (Function *LF = F->Codegen()) {
    fprintf(stderr, "Read top-level expression:");
    LF->dump();
    }
    } else {
    // Skip token for error recovery.
    getNextToken();
    }
    }

    /// top ::= definition | external | expression | ';'
    static void MainLoop() {
    while (1) {
    fprintf(stderr, "ready> ");
    switch (CurTok) {
    case tok_eof:    return;
    case ';':        getNextToken(); break;  // ignore top-level semicolons.
    case tok_def:    HandleDefinition(); break;
    case tok_extern: HandleExtern(); break;
    default:         HandleTopLevelExpression(); break;
    }
    }
    }

    //===----------------------------------------------------------------------===//
    // "Library" functions that can be "extern'd" from user code.
    //===----------------------------------------------------------------------===//

    /// putchard - putchar that takes a double and returns 0.
    extern "C"
    double putchard(double X) {
    putchar((char)X);
    return 0;
    }

    //===----------------------------------------------------------------------===//
    // Main driver code.
    //===----------------------------------------------------------------------===//

    int main() {
    LLVMContext &Context = getGlobalContext();

    // Install standard binary operators.
    // 1 is lowest precedence.
    BinopPrecedence['<'] = 10;
    BinopPrecedence['+'] = 20;
    BinopPrecedence['-'] = 20;
    BinopPrecedence['*'] = 40;  // highest.

    // Prime the first token.
    fprintf(stderr, "ready> ");
    getNextToken();

    // Make the module, which holds all the code.
    TheModule = new Module("my cool jit", Context);

    // Run the main "interpreter loop" now.
    MainLoop();

    // Print out all of the generated code.
    TheModule->dump();

    return 0;
    }

Reference:

1. [LLVM Tutorial](http://llvm.org/docs/tutorial/index.html)
