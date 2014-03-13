The Compiler
============
[原文地址][1]
[Issue #6 Build Tools][2], November 2013
By [Chris Eidhof][3]

What Does a Compiler Do?
========================

In this article we’ll have a look at what a compiler does, and how we can use that to our advantage.

Roughly speaking, the compiler has two tasks: converting our Objective-C code into low-level code, and analyzing our code to make sure we didn’t make any obvious mistakes.

These days, Xcode ships with clang as the compiler. Wherever we write compiler, you can read it as clang. clang is the tool that takes Objective-C code, analyzes it, and transforms it into a more low-level representation that resembles assembly code: LLVM Intermediate Representation. LLVM IR is low level, and operating system independent. LLVM takes instructions and compiles them into native bytecode for the target platform. This can either be done just-in-time or at the same time as compilation.

The nice thing about having those LLVM instructions is that you can generate and run them on any platform that is supported by LLVM. For example, if you write your iOS app, it automatically runs on two very different architectures (Intel and ARM), and it’s LLVM that takes care of translating the IR code into native bytecode for those platforms.

LLVM benefits from having a three-tier architecture, which means it supports a lot of input languages (e.g. C, Objective-C, and C++, but also Haskell) in the first tier, then a shared optimizer in the second tier (which optimizes the LLVM IR), and different targets in the third tier (e.g. Intel, ARM, and PowerPC). If you want to add a language, you can focus on the first tier, and if you want to add another compilation target, you don’t have to worry too much about the input languages. In the book The Architecture of Open Source Applications there’s a great chapter by LLVM’s creator, Chris Lattner, about the [LLVM architecture][4].

When compiling a source file, the compiler proceeds through several phases. To see the different phases, we can ask clang what it will do to compile the file hello.m:

    % clang -ccc-print-phases hello.m
    
    0: input, "hello.m", objective-c
    1: preprocessor, {0}, objective-c-cpp-output
    2: compiler, {1}, assembler
    3: assembler, {2}, object
    4: linker, {3}, image
    5: bind-arch, "x86_64", {4}, image

In this article, we’ll focus on phases one and two. In [Mach-O Executables][5], Daniel explains phases three and four.

Preprocessing
-------------

The first thing that happens when you’re compiling a source file is preprocessing. The preprocessor handles a macro processing language, which means it will replace macros in your text by their definitions. For example, if you write the following:

    #import <Foundation/Foundation.h>

The preprocessor will take that line, and substitute it with the contents of that file. If that header file contains any other macro definitions, they will get substituted too.

This is the reason why people tell you to keep your header files free of imports as much as you can, because anytime you import something, the compiler has to do more work. For example, in your header file, instead of writing:

    #import "MyClass.h"

You could write:

    @class MyClass;

And by doing that you promise the compiler that there will be a class, MyClass. In the implementation file (the .m file), you can then import the MyClass.h and use it.

Now suppose we have a very simple pure C program, named hello.c:

    #include <stdio.h>
    
    int main() {
      printf("hello world\n");
      return 0;
    }

We can run the preprocessor on this to see what the effect is:

    clang -E hello.c | less

Now, have a look at that code. It’s 401 lines. If we also add the following line to the top:

    #import <Foundation/Foundation.h>

We can run the command again, and see that our file has expanded to a whopping 89,839 lines. There are entire operating systems written in less lines of code.

Luckily, the situation has recently improved a bit. There’s now a feature called [modules][6] that makes this process a bit more high level.

Custom Macros
-------------

Another example is when you define or use custom macros, like this:

    #define MY_CONSTANT 4

Now, anytime you write MY_CONSTANT after this line, it’ll get replaced by 4 before the rest of compilation starts. You can also define more interesting macros that take arguments:

    #define MY_MACRO(x) x

This article is too short to discuss the full scope of what is possible with the preprocessor, but it’s a very powerful tool. Often, the preprocessor is used to inline code. We strongly discourage this. For example, suppose you have the following innocent-looking program:

    #define MAX(a,b) a > b ? a : b
    
    int main() {
      printf("largest: %d\n", MAX(10,100));
      return 0;
    }

This will work fine. However, what about the following program:

    #define MAX(a,b) a > b ? a : b
    
    int main() {
      int i = 200;
      printf("largest: %d\n", MAX(i++,100));
      printf("i: %d\n", i);
      return 0;
    }

If we compile this with clang max.c, we get the following result:

    largest: 201
    i: 202

This is quite obvious when we run the preprocessor and expand all the macros by issuing clang -E max.c:

    int main() {
      int i = 200;
      printf("largest: %d\n", i++ > 100 ? i++ : 100);
      printf("i: %d\n", i);
      return 0;
    }

In this case, it’s an obvious example of what could go wrong with macros, but things can go also wrong in more unexpected and hard-to-debug ways. Instead of using a macro, you should use static inline functions:

    #include <stdio.h>
    
    static const int MyConstant = 200;
    
    static inline int max(int l, int r) {
       return l > r ? l : r;
    }
    
    int main() {
      int i = MyConstant;
      printf("largest: %d\n", max(i++,100));
      printf("i: %d\n", i);
      return 0;
    }

This will print the correct result (i: 201). Because the code is inlined, it will have the same performance as the macro variant, but it’s a lot less error-prone. Also, you can set breakpoints, have type checking, and avoid unexpected behavior.

The only time when macros are a reasonable solution is for logging, since you can use  `__FILE__` and `__LINE__`  and assert macros.

Tokenization (Lexing)
---------------------

After preprocessing is done, every source .m file now has a bunch of definitions. This text is converted from a string to a stream of tokens. For example, in the case of a simple Objective-C hello world program:

    int main() {
      NSLog(@"hello, %@", @"world");
      return 0;
    }

We can ask clang to dump the tokens for this program by issuing the following command: clang -Xclang -dump-tokens hello.m:

    int 'int'        [StartOfLine]  Loc=<hello.m:4:1>
    identifier 'main'        [LeadingSpace] Loc=<hello.m:4:5>
    l_paren '('             Loc=<hello.m:4:9>
    r_paren ')'             Loc=<hello.m:4:10>
    l_brace '{'      [LeadingSpace] Loc=<hello.m:4:12>
    identifier 'NSLog'       [StartOfLine] [LeadingSpace]   Loc=<hello.m:5:3>
    l_paren '('             Loc=<hello.m:5:8>
    at '@'          Loc=<hello.m:5:9>
    string_literal '"hello, %@"'            Loc=<hello.m:5:10>
    comma ','               Loc=<hello.m:5:21>
    at '@'   [LeadingSpace] Loc=<hello.m:5:23>
    string_literal '"world"'                Loc=<hello.m:5:24>
    r_paren ')'             Loc=<hello.m:5:31>
    semi ';'                Loc=<hello.m:5:32>
    return 'return'  [StartOfLine] [LeadingSpace]   Loc=<hello.m:6:3>
    numeric_constant '0'     [LeadingSpace] Loc=<hello.m:6:10>
    semi ';'                Loc=<hello.m:6:11>
    r_brace '}'      [StartOfLine]  Loc=<hello.m:7:1>
    eof ''          Loc=<hello.m:7:2>

We can see that each token consists of a piece of text and a source location. The source location is from before macro expansion, so that if something goes wrong, clang can point you to the right spot.

Parsing
-------

Now the interesting part starts: our stream of tokens is parsed into an abstract syntax tree. Because Objective-C is a rather complicated language, parsing is not always easy. After parsing, a program is now available as an abstract syntax tree: a tree that represents the original program. Suppose we have a program hello.m:

    #import <Foundation/Foundation.h>
    
    @interface World
    - (void)hello;
    @end
    
    @implementation World
    - (void)hello {
      NSLog(@"hello, world");
    }
    @end
    
    int main() {
       World* world = [World new];
       [world hello];
    }

When we issue clang -Xclang -ast-dump -fsyntax-only hello.m, we get the following result printed to our command line:

    @interface World- (void) hello;
    @end
    @implementation World
    - (void) hello (CompoundStmt 0x10372ded0 <hello.m:8:15, line:10:1>
      (CallExpr 0x10372dea0 <line:9:3, col:24> 'void'
        (ImplicitCastExpr 0x10372de88 <col:3> 'void (*)(NSString *, ...)' <FunctionToPointerDecay>
          (DeclRefExpr 0x10372ddd8 <col:3> 'void (NSString *, ...)' Function 0x1023510d0 'NSLog' 'void (NSString *, ...)'))
        (ObjCStringLiteral 0x10372de38 <col:9, col:10> 'NSString *'
          (StringLiteral 0x10372de00 <col:10> 'char [13]' lvalue "hello, world"))))
    
    
    @end
    int main() (CompoundStmt 0x10372e118 <hello.m:13:12, line:16:1>
      (DeclStmt 0x10372e090 <line:14:4, col:30>
        0x10372dfe0 "World *world =
          (ImplicitCastExpr 0x10372e078 <col:19, col:29> 'World *' <BitCast>
            (ObjCMessageExpr 0x10372e048 <col:19, col:29> 'id':'id' selector=new class='World'))")
      (ObjCMessageExpr 0x10372e0e8 <line:15:4, col:16> 'void' selector=hello
        (ImplicitCastExpr 0x10372e0d0 <col:5> 'World *' <LValueToRValue>
          (DeclRefExpr 0x10372e0a8 <col:5> 'World *' lvalue Var 0x10372dfe0 'world' 'World *'))))

Every node in the abstract syntax tree is annotated with the original source position, so that if there’s any problem later on, clang can warn about your program and give you the correct location.

**See Also**

 - [Introduction to the clang AST][7]

Static Analysis
===============

Once the compiler has an abstract syntax tree, it can perform analyses on that tree to help catch you errors, such as in type checking, where it checks whether your program is type correct. For example, when you send a message to an object, it checks if the object actually implements that message. Also, clang does more advanced analyses where it walks through your program to make sure you’re not doing anything weird.

Type Checking
-------------

Any time you write code, clang is there to help you check that you didn’t make any mistakes. One of the obvious things to check is if your program sends the correct messages to the correct objects and calls the correct functions on the correct values. If you have a plain NSObject*, you can’t just send it the hello message, as clang will report an error. Also, if you create a class Test that subclasses NSObject, like this:

    @interface Test : NSObject
    @end

And you then try to assign an object with a different type to that object, the compiler will help you and warn you that what you’re doing is probably not correct.

There are two types of typing: dynamic and static typing. Dynamic typing means that a type is checked at runtime, and static typing means that the types are checked at compile time. In the past, you could always send any message to any object, and at runtime, it would be determined if the object responds to that message. When this is checked only at runtime, it’s called dynamic typing.

With static typing, this is checked at compile time. When you use ARC, the compiler checks a lot more types at compile time, because it needs to know which objects it works with. For example, you can not write the following code anymore:

    [myObject hello]

If there’s no hello method defined anywhere in your program.

Other Analyses
--------------

There are a lot of other analyses that clang does for you. If you clone the clang repository and go to lib/StaticAnalyzer/Checkers, you’ll see all the static checkers. For example, there’s ObjCUnusedIVarsChecker.cpp, which checks if ivars are unused. Or there is ObjCSelfInitChecker.cpp, which checks if you called [self initWith...] or [super init] before you start using self inside your initializer. Some of the other checks happen in other parts of the compiler. For example, in line 2,534 of lib/Sema/SemaExprObjC.cpp, you can see the line that does the following:

     Diag(SelLoc, diag::warn_arc_perform_selector_leaks);

Which produces the dreaded “performSelector may cause a leak because its selector is unknown” warning.

Code Generation
===============

Now, once your code is fully tokenized, parsed, and analyzed by clang, it can generate the LLVM code for you. To see what happens, we can have a look again at the program hello.c:

    #include <stdio.h>
    
    int main() {
      printf("hello world\n");
      return 0;
    }

To compile this into LLVM bitcode (which is, most of the time, represented in a binary format) we can issue the following command:

    clang -O3 -emit-LLVM hello.c -c -o hello.bc

This generates a binary file that we can then inspect using another command:

    llvm-dis < hello.bc | less

Which gives us the following output:

    ; ModuleID = '<stdin>'
    target datalayout = "e-p:64:64:64-i1:8:8-i8:8:8-i16:16:16-i32:32:32-i64:64:64-f32:32:32-f64:64:64-v64:64:64-v128:128:128-a0:0:64-s0:64:64-f80:128:128-n8:16:32:64-S128"
    target triple = "x86_64-apple-macosx10.8.0"
    
    @str = private unnamed_addr constant [12 x i8] c"hello world\00"
    
    ; Function Attrs: nounwind ssp uwtable
    define i32 @main() #0 {
      %puts = tail call i32 @puts(i8* getelementptr inbounds ([12 x i8]* @str, i64 0, i64 0))
      ret i32 0
    }
    
    ; Function Attrs: nounwind
    declare i32 @puts(i8* nocapture) #1
    
    attributes #0 = { nounwind ssp uwtable }
    attributes #1 = { nounwind }

You can see that the main function is only two lines: one to print the string and one to return 0.

It’s also interesting to do the same thing for a very simple Objective-C program five.m that we compile and then view using LLVM-dis < five.bc | less:

    #include <stdio.h>
    #import <Foundation/Foundation.h>
    
    int main() {
      NSLog(@"%@", [@5 description]);
      return 0;
    }

There are a lot more things going on around, but here’s the main function:

    define i32 @main() #0 {
      %1 = load %struct._class_t** @"\01L_OBJC_CLASSLIST_REFERENCES_$_", align 8
      %2 = load i8** @"\01L_OBJC_SELECTOR_REFERENCES_", align 8, !invariant.load !4
      %3 = bitcast %struct._class_t* %1 to i8*
      %4 = tail call %0* bitcast (i8* (i8*, i8*, ...)* @objc_msgSend to %0* (i8*, i8*, i32)*)(i8* %3, i8* %2, i32 5)
      %5 = load i8** @"\01L_OBJC_SELECTOR_REFERENCES_2", align 8, !invariant.load !4
      %6 = bitcast %0* %4 to i8*
      %7 = tail call %1* bitcast (i8* (i8*, i8*, ...)* @objc_msgSend to %1* (i8*, i8*)*)(i8* %6, i8* %5)
      tail call void (%1*, ...)* @NSLog(%1* bitcast (%struct.NSConstantString* @_unnamed_cfstring_ to %1*), %1* %7)
      ret i32 0
    }

The most important lines are line 4, which creates the NSNumber object, line 7, which sends the description message to the number object, and line 8, which logs the string returned from the description message.

Optimizations
-------------

To see which kind of optimizations LLVM and clang can do, it’s interesting to look at a slightly more complicated C example, the recursively defined factorial function:

    #include <stdio.h>

    int factorial(int x) {
       if (x > 1) return x * factorial(x-1);
       else return 1;
    }
    
    int main() {
      printf("factorial 10: %d\n", factorial(10));
    }

To compile this without optimizations, run the following command:

    clang -O0 -emit-llvm factorial.c  -c -o factorial.bc && llvm-dis < factorial.bc

The interesting part is to look at the code generated for the factorial function:

    define i32 @factorial(i32 %x) #0 {
      %1 = alloca i32, align 4
      %2 = alloca i32, align 4
      store i32 %x, i32* %2, align 4
      %3 = load i32* %2, align 4
      %4 = icmp sgt i32 %3, 1
      br i1 %4, label %5, label %11
    
    ; <label>:5                                       ; preds = %0
      %6 = load i32* %2, align 4
      %7 = load i32* %2, align 4
      %8 = sub nsw i32 %7, 1
      %9 = call i32 @factorial(i32 %8)
      %10 = mul nsw i32 %6, %9
      store i32 %10, i32* %1
      br label %12
    
    ; <label>:11                                      ; preds = %0
      store i32 1, i32* %1
      br label %12
    
    ; <label>:12                                      ; preds = %11, %5
      %13 = load i32* %1
      ret i32 %13
    }

You can see that at the line marked with %9, it calls itself recursively. This is quite inefficient, because the stack increases with each recursive call. To turn on optimizations, we can pass the flag -O3 to clang:

    clang -O3 -emit-llvm factorial.c  -c -o factorial.bc && llvm-dis < factorial.bc

Now the code for the factorial function looks like this:

    define i32 @factorial(i32 %x) #0 {
      %1 = icmp sgt i32 %x, 1
      br i1 %1, label %tailrecurse, label %tailrecurse._crit_edge
    
    tailrecurse:                                      ; preds = %tailrecurse, %0
      %x.tr2 = phi i32 [ %2, %tailrecurse ], [ %x, %0 ]
      %accumulator.tr1 = phi i32 [ %3, %tailrecurse ], [ 1, %0 ]
      %2 = add nsw i32 %x.tr2, -1
      %3 = mul nsw i32 %x.tr2, %accumulator.tr1
      %4 = icmp sgt i32 %2, 1
      br i1 %4, label %tailrecurse, label %tailrecurse._crit_edge
    
    tailrecurse._crit_edge:                           ; preds = %tailrecurse, %0
      %accumulator.tr.lcssa = phi i32 [ 1, %0 ], [ %3, %tailrecurse ]
      ret i32 %accumulator.tr.lcssa
    }

Even though our function was not written in a [tail-recursive][8] way, clang can still optimize it and now it’s just one function with a loop. There are a lot more optimizations that clang will do on your code. A good example of what gcc can do is at [ridiculousfish.com][9].

**More Reading**

 - [LLVM blog: posts tagged ‘optimization’][10]
 - [LLVM blog: vectorization improvements][11]
 - [LLVM blog: greedy register allocation][12]
 - [The Polly project][13]

How to Use this to Your Advantage
=================================

Now that we’ve seen a full compile, from tokenization to parsing, from abstract syntax trees to analysis and compilation, we can wonder: why should we care?

Using libclang or clang Plugins
-------------------------------

The cool thing about clang is that it’s open-source and a really well-built project: almost everything is a library. That means that it’s possible to create your own version of clang and only change those parts that you need. For example, you can change the way clang generates code, add better type checking, or perform your analyses. There are a lot of ways in which you can do this; the easiest way is by using a C library called [libclang][14]. libclang provides you with a simple C API to clang, and you can use it to analyze all your sources. However, in my experience, as soon as you want to do something that’s a bit more advanced, libclang is too limited. Also, there’s [ClangKit][15], which is an Objective-C wrapper around some of the functionality provided by clang.

Another way is to use the C++ library provided by clang by directly using LibTooling. This is a lot more work, and involves C++, but gives you the full power of clang. You can do any kind of analyses, or even rewrite programs. If you want to add custom analyses to clang, want to write your own refactorer, need to rewrite a massive code base, or want to generate graphs and documentation from your project, LibTooling is your friend.

Writing an Analyzer
-------------------

Follow the instructions on [Tutorial for building tools using LibTooling][16] to build LLVM, clang, and clang-tools-extra. Be sure to leave some time aside for compilation; even though I have a very fast machine, I was still able to do the dishes during the time LLVM compiled.

Next, go to your LLVM directory and do a cd ~/llvm/tools/clang/tools/. In this directory, you can create your own standalone clang tools. As an example, we created a small tool to help us detect correct usage of a library. Clone the [example repository][17] into this directory, and type make. This will provide you with a binary called example.

Our use case is as follows: suppose we have an Observer class, which looks like this:

    @interface Observer
    + (instancetype)observerWithTarget:(id)target action:(SEL)selector;
    @end

Now, we want to check whenever this class is used that the action is a method that exists on the target object. We can write a quick C++ function that does this (be warned, this is the first C++ I ever wrote, so it’s definitely not idiomatic):

    virtual bool VisitObjCMessageExpr(ObjCMessageExpr *E) {
      if (E->getReceiverKind() == ObjCMessageExpr::Class) {
        QualType ReceiverType = E->getClassReceiver();
        Selector Sel = E->getSelector();
        string TypeName = ReceiverType.getAsString();
        string SelName = Sel.getAsString();
        if (TypeName == "Observer" && SelName == "observerWithTarget:action:") {
          Expr *Receiver = E->getArg(0)->IgnoreParenCasts();
          ObjCSelectorExpr* SelExpr = cast<ObjCSelectorExpr>(E->getArg(1)->IgnoreParenCasts());
          Selector Sel = SelExpr->getSelector();
          if (const ObjCObjectPointerType *OT = Receiver->getType()->getAs<ObjCObjectPointerType>()) {
            ObjCInterfaceDecl *decl = OT->getInterfaceDecl();
            if (! decl->lookupInstanceMethod(Sel)) {
              errs() << "Warning: class " << TypeName << " does not implement selector " << Sel.getAsString() << "\n";
              SourceLocation Loc = E->getExprLoc();
              PresumedLoc PLoc = astContext->getSourceManager().getPresumedLoc(Loc);
              errs() << "in " << PLoc.getFilename() << " <" << PLoc.getLine() << ":" << PLoc.getColumn() << ">\n";
            }
          }
        }
      }
      return true;
    }

This method starts by looking for message expressions that have Observer as the receiver, and observerWithTarget:action: as the selector, and then looks at the target and checks if the method exists. Of course, this is a bit of a contrived example, but if you want to mechanically verify things in your code base using the AST, this is the way to go.

More clang Possibilities
------------------------

There are many more ways in which we can leverage clang. For example, it’s possible to write compiler plugins (e.g. with the same checker as above) and load them dynamically into your compiler. I haven’t tested yet, but it should work with Xcode. For example, if you want to get a warning about code style, you could write a clang plugin for that. (For simple checks, see the [Build process][18] article.)

Also, if you need to do a massive refactoring of your codebase where the normal refactoring tools built into Xcode or AppCode are not sufficient, you could write a simple refactoring tool using clang. It may sound daunting, but in the tutorials linked below you’ll see that it’s not that hard.

Finally, if you really need to, you can compile your own clang and instruct Xcode to use that. Again, this is not as hard as it sounds, and definitely fun to play around with.

**More Reading**

 - [Clang Tutorial][19]
 - [X86_64 Assembly Language Tutorial][20]
 - [Custom clang Build with Xcode (I)][21] and [(II)][22]
 - [Clang Tutorial (I)][23], [(II)][24] and [(III)][25]
 - [Clang Plugin Tutorial][26]
 - [LLVM blog: What every C programmer should know (I)][27] , [(II)][28]
   and [(III)][29]

[More articles in issue #6][30]


  [1]: http://www.objc.io/issue-6/compiler.html
  [2]: http://www.objc.io/issue-6/index.html
  [3]: http://twitter.com/chriseidhof
  [4]: http://www.aosabook.org/en/llvm.html
  [5]: http://www.objc.io/issue-6/mach-o-executables.html
  [6]: http://clang.llvm.org/docs/Modules.html
  [7]: http://clang.llvm.org/docs/IntroductionToTheClangAST.html
  [8]: http://en.wikipedia.org/wiki/Tail_call
  [9]: http://ridiculousfish.com/blog/posts/will-it-optimize.html
  [10]: http://blog.llvm.org/search/label/optimization
  [11]: http://blog.llvm.org/2013/05/llvm-33-vectorization-improvements.html
  [12]: http://blog.llvm.org/2011/09/greedy-register-allocation-in-llvm-30.html
  [13]: http://polly.llvm.org/index.html
  [14]: http://clang.llvm.org/doxygen/group__CINDEX.html
  [15]: https://github.com/macmade/ClangKit
  [16]: http://clang.llvm.org/docs/LibASTMatchersTutorial.html
  [17]: https://github.com/objcio/issue6-compiler-tool
  [18]: http://www.objc.io/issue-6/build-process.html
  [19]: https://github.com/loarabia/Clang-tutorial
  [20]: http://cocoafactory.com/blog/2012/11/23/x86-64-assembly-language-tutorial-part-1/
  [21]: http://clang-analyzer.llvm.org/xcode.html
  [22]: http://stackoverflow.com/questions/3297986/using-an-external-xcode-clang-static-analyzer-binary-with-additional-checks
  [23]: http://kevinaboos.wordpress.com/2013/07/23/clang-tutorial-part-i-introduction/
  [24]: http://kevinaboos.wordpress.com/2013/07/23/clang-tutorial-part-ii-libtooling-example/
  [25]: http://kevinaboos.wordpress.com/2013/07/29/clang-tutorial-part-iii-plugin-example/
  [26]: http://getoffmylawnentertainment.com/blog/2011/10/01/clang-plugin-development-tutorial/
  [27]: http://blog.llvm.org/2011/05/what-every-c-programmer-should-know.html
  [28]: http://blog.llvm.org/2011/05/what-every-c-programmer-should-know_14.html
  [29]: http://blog.llvm.org/2011/05/what-every-c-programmer-should-know_21.html
  [30]: http://www.objc.io/issue-6




The Compiler 编译器
================

[Issue #6 Build Tools][1], November 2013
By [Chris Eidhof][2]

编译器是如何工作的？
==========

本文主要探讨一下编译器是如何工作的，以及如何有效的利用编译器。

简单的说，编译器有两个职责：把Objective-C代码转化成低阶代码，以及对代码做分析，确保代码中没有什么显而易见的错误。

现在，Xcode的默认编译器是clang。以下我们提到的编译器都会以clang代替。clang会先对Objective-C代码做分析检查，然后再转化成类似汇编语言但与操作系统无关的低阶中间表达形式码：LLVM Intermediate Representation（LLVM中间表达码）。接着LLVM会执行自己的指令将LLVM IR编译成目标运行平台上可执行的机器码，这个过程可能是实时的也可以是随着汇编过程一起进行的。

LLVM的指令非常棒，只要一个平台支持LLVM，那么LLVM就可以在这个平台上执行它的指令。比如说一个iOS的app，它就可以同时在完全不同架构的Intel和ARM平台上运行，所有针对不同平台的兼容性问题都是靠LLVM利用自己的中间表达码生成不同的原生机器码来解决的。

LLVM的优秀跨平台特性得益于其特定的“三层式”架构，即在第一层支持多种输入语言（比如：C,ObjectiveC,C++以及Haskell），第二层利用共享优化器来对LLVM中间表达码进行优化，第三层挂接了不同的平台（比如:Intel,ARM和PowerPC）。这样的话第一层和第三层之间做到了比较好的弱相关，如果想要增加对不同的输入语言的支持，只需要解决第一层的支持即可，要是想要增加目标编译平台，也不太需要操心输入语言的问题。如果对LLVM的架构感兴趣，可以参阅书籍*The Architecture of Open Source Application*，里面收录了由LLVM创造者Chris Lattner编写的介绍[LLVM架构][3]的章节。

编译器在编译文件的过程中通常会分几个阶段。如果要详细研究每个阶段的情况，拿编译hello.m文件来说，可以让clang输出每一阶段的信息：

    % clang -ccc-print-phases hello.m
    
    0: input, "hello.m", objective-c
    1: preprocessor, {0}, objective-c-cpp-output
    2: compiler, {1}, assembler
    3: assembler, {2}, object
    4: linker, {3}, image
    5: bind-arch, "x86_64", {4}, image

本文重点关注阶段1和阶段2。在文章[Mach-O Executables][4]中，Daniel会对阶段3和阶段4进行阐述。



Preprocessing 预处理
----------------

每当编译文件的时候，编译器最先做的是一些预处理工作。比如预处理器会处理宏定义，将文本中的宏用其对应定义的具体内容进行替换。

例如，如果在代码文件中的出现的下述格式的引入：

    #import <Foundation/Foundation.h>

预处理器对这行代码的处理是用真实Foundation.h头文件中的内容去替换这行宏引入，如果Foundation.h中也使用了类似的宏引入，则会按照同样的处理方式用各个宏对应的真正代码进行逐级替代。

这也就是为什么人们主张头文件最好尽量少的去引入其他的类或库，因为引入的东西越多，编译器需要做的处理就越多。

例如，在头文件中用：

    @class MyClass;

代替：

    #import "MyClass.h"

这么写是告诉编译器MyClass.h文件本身是存在的，并且在.m文件中会对MyClass.h文件做引入使用。

例如，写一个简单的C程序hello.c：

    #include <stdio.h>

    int main() {
      printf("hello world\n");
      return 0;
    }

然后执行一下命令，看看预处理器是怎么处理这段代码的：

    clang -E hello.c | less

接下来看看处理后的代码，一共是401行。如果在代码中再增加一行引入：

    #import <Foundation/Foundation.h>

再执行一下上面的命令，处理后的文件代码行数暴增到89,839行。这个数字比某些操作系统的总代码行数还要多。

幸好现在有了[模块][5]引入特性，使得引用关系处理变得更加智能。

Custom Macros 自定义宏
-----------------

另一种情形是自定义宏，比如定义：

    #define MY_CONSTANT 4

凡是在此行宏定义作用域允许的代码内键入MY_CONSTANT，在预处理过程中MY_CONSTANT都会被替换成4。宏定义也是可以携带参数的， 比如：

    #define MY_MACRO(x) x

鉴于本文的内容所限，就不对强大的预处理做更多、更全面的展开讨论了。但是还是要强调一点，建议大家不要在需要预处理的代码中加入内联代码逻辑。

例如，下面这段代码，这样用没什么问题：

    #define MAX(a,b) a > b ? a : b

int main() {
  printf("largest: %d\n", MAX(10,100));
  return 0;
}

但是如果换成这么写：

    #define MAX(a,b) a > b ? a : b

    int main() {
      int i = 200;
      printf("largest: %d\n", MAX(i++,100));
      printf("i: %d\n", i);
      return 0;
    }

用clang的max.c编译一下，结果是：

    largest: 201
    i: 202

用`clang -E max.c`进行宏展开的预处理结果是：

    int main() {
      int i = 200;
      printf("largest: %d\n", i++ > 100 ? i++ : 100);
      printf("i: %d\n", i);
      return 0;
    }

本例是典型的宏使用不当，而且通常这类问题会更加的隐蔽且难以debug。针对本例这类情况，最好使用静态函数而不是宏。

    #include <stdio.h>
    static const int MyConstant = 200;
    
    static inline int max(int l, int r) {
       return l > r ? l : r;
    }
    
    int main() {
      int i = MyConstant;
      printf("largest: %d\n", max(i++,100));
      printf("i: %d\n", i);
      return 0;
    }

这样改过之后，就可以输出正常的结果(i:201)。因为之前定义的静态函数是直接插入在主体代码中的，所以它的效率和宏变量差不多，但是可靠性比宏定义要好许多。再者，还可以通过设断点debug、类型检查等手段来避免一些异常的产生。

基本上，宏的最佳使用场景是日志输出，可以使用`__FILE__` 和 `__LINE__` 这种宏来做断言。

Tokenization (Lexing) 词语法解析标记
----------------------------

预处理完成以后，每一个.m源文件里都有一堆的声明和定义。这些代码文本都会从string转化成特殊的标记流。

例如，一段简单的Objective-C的hello word：

    int main() {
      NSLog(@"hello, %@", @"world");
      return 0;
    }

利用clang 命令 -Xclang -dump-tokens 来将hello world.m的标记流导出：

    int 'int'        [StartOfLine]  Loc=<hello.m:4:1>
    identifier 'main'        [LeadingSpace] Loc=<hello.m:4:5>
    l_paren '('             Loc=<hello.m:4:9>
    r_paren ')'             Loc=<hello.m:4:10>
    l_brace '{'      [LeadingSpace] Loc=<hello.m:4:12>
    identifier 'NSLog'       [StartOfLine] [LeadingSpace]   Loc=<hello.m:5:3>
    l_paren '('             Loc=<hello.m:5:8>
    at '@'          Loc=<hello.m:5:9>
    string_literal '"hello, %@"'            Loc=<hello.m:5:10>
    comma ','               Loc=<hello.m:5:21>
    at '@'   [LeadingSpace] Loc=<hello.m:5:23>
    string_literal '"world"'                Loc=<hello.m:5:24>
    r_paren ')'             Loc=<hello.m:5:31>
    semi ';'                Loc=<hello.m:5:32>
    return 'return'  [StartOfLine] [LeadingSpace]   Loc=<hello.m:6:3>
    numeric_constant '0'     [LeadingSpace] Loc=<hello.m:6:10>
    semi ';'                Loc=<hello.m:6:11>
    r_brace '}'      [StartOfLine]  Loc=<hello.m:7:1>
    eof ''          Loc=<hello.m:7:2>

仔细观察可以发现，每一个标记都包含了对应的源码内容和其在源码中的位置。注意这个位置是宏展开之前的位置，这样clang可以保证在出现什么问题的时候，能够在源码中精确定位到出错位置。

Parsing 解析
---------

接下来要说的东西比较有意思：之前生成的标记流将会被解析成抽象语法树。由于Objective-C语言本身比较复杂，这导致解析过程也比较繁复。解析过后，源程序变成了抽象语法树。

还是以前面的hello world为例：

    #import <Foundation/Foundation.h>

    @interface World
    - (void)hello;
    @end
    
    @implementation World
    - (void)hello {
      NSLog(@"hello, world");
    }
    @end
    
    int main() {
       World* world = [World new];
       [world hello];
    }

执行clang 命令-Xclang -ast-dump -fsyntax-only后，命令行输出了hello world的抽象语法树结果：

    @interface World- (void) hello;
    @end
    @implementation World
    - (void) hello (CompoundStmt 0x10372ded0 <hello.m:8:15, line:10:1>
      (CallExpr 0x10372dea0 <line:9:3, col:24> 'void'
        (ImplicitCastExpr 0x10372de88 <col:3> 'void (*)(NSString *, ...)' <FunctionToPointerDecay>
          (DeclRefExpr 0x10372ddd8 <col:3> 'void (NSString *, ...)' Function 0x1023510d0 'NSLog' 'void (NSString *, ...)'))
        (ObjCStringLiteral 0x10372de38 <col:9, col:10> 'NSString *'
          (StringLiteral 0x10372de00 <col:10> 'char [13]' lvalue "hello, world"))))
    
    
    @end
    int main() (CompoundStmt 0x10372e118 <hello.m:13:12, line:16:1>
      (DeclStmt 0x10372e090 <line:14:4, col:30>
        0x10372dfe0 "World *world =
          (ImplicitCastExpr 0x10372e078 <col:19, col:29> 'World *' <BitCast>
            (ObjCMessageExpr 0x10372e048 <col:19, col:29> 'id':'id' selector=new class='World'))")
      (ObjCMessageExpr 0x10372e0e8 <line:15:4, col:16> 'void' selector=hello
        (ImplicitCastExpr 0x10372e0d0 <col:5> 'World *' <LValueToRValue>
          (DeclRefExpr 0x10372e0a8 <col:5> 'World *' lvalue Var 0x10372dfe0 'world' 'World *'))))

生成的抽象语法树中的每个节点都标注了其对应源码中的位置，同样的，如果产生了什么问题，clang可以定位到问题所在处的源码位置。

**延伸阅读：**

 - [Introduction to the clang AST][6]

Static Analysis静态分析
===================

一旦源码生成了抽象语法树，编译器就可以围绕其做检查分析，比如可以做类型检查，即检查程序中是否有类型错误。举例来说：如果对某个对象发送了一个消息，编译器会检查这个对象是否实现了这个消息（函数、方法）。此外，clang做了许多其他的检查，来确保目标程序中没有什么杂七杂八的错误。

Type Checking 类型检查
-----------------

每当开发人员编写代码的时候，clang都会帮忙检查错误。其中最常见的就是检查是否对程序对象发送正确的消息，是否在数值上使用了正确的函数。比如若对一个单纯的NSObject*对象发送了一个hello消息，clang就会报错。

下面对NSObject创建一个Test子类：

    @interface Test : NSObject
    @end

然后给这个子类中某个属性设置一个与其自身声明类型不相符的值，clang同样会给出一些类型不匹配的警告。

一般会把类型们分类两类：动态的和静态的。动态的在运行时做检查，静态的在编译时做检查。以往，编写代码时可以向对象发送任何消息，因为在真正运行时，才会检查对象是否能够响应这些消息。鉴于只是在运行时做此类检查，所以叫做动态类检查型。至于静态类型检查，例如如果使用ARC，编译器会在编译过程中做许多检查，因为编译器需要确保对其编译的对象有明确的理解和认识。比如下面的代码中，如果myObject没有hello方法，这行代码就编不过。

    [myObject hello]

Other Analyses其他分析

其实clang还有许多其他的分析能力。看一下clang源码的lib/StaticAnalyzer/Checkers目录，可以查看所有的静态检查。比如ObjCUnusedIVarsChecker.cpp用来检查代码中是否有多余的实例变量ivars的定义。ObjCSelfInitChecker.cpp检查代码中自定义初始化方法中是否调用了 [self initWith…]或[super init]。编译器还进行了一些其他的检查，例如在lib/Sema/SemaExprObjC.cpp的2,534行，有这样一句：

    Diag(SelLoc, diag::warn_arc_perform_selector_leaks);

这个会生成严重错误的提醒 “performSelector may cause a leak because its selector is unknown” 。

代码生成
===================

clang完成了代码的标记，解析和分析后，接着就会生成LLVM代码了。下面看看hello.c的变化：

    #include <stdio.h>
    
    int main() {
      printf("hello world\n");
      return 0;
    }


要把这段代码编译成LLVM位码（绝大多数情况下是二进制码），执行下面的命令：

    clang -O3 -emit-LLVM hello.c -c -o hello.bc

接着用另一个命令来查看刚刚生成的二进制文件：

    llvm-dis < hello.bc | less

输出如下：

    ; ModuleID = '<stdin>'
    target datalayout = "e-p:64:64:64-i1:8:8-i8:8:8-i16:16:16-i32:32:32-i64:64:64-f32:32:32-f64:64:64-v64:64:64-v128:128:128-a0:0:64-s0:64:64-f80:128:128-n8:16:32:64-S128"
    target triple = "x86_64-apple-macosx10.8.0"
    
    @str = private unnamed_addr constant [12 x i8] c"hello world\00"
    
    ; Function Attrs: nounwind ssp uwtable
    define i32 @main() #0 {
      %puts = tail call i32 @puts(i8* getelementptr inbounds ([12 x i8]* @str, i64 0, i64 0))
      ret i32 0
    }
    
    ; Function Attrs: nounwind
    declare i32 @puts(i8* nocapture) #1
    
    attributes #0 = { nounwind ssp uwtable }
    attributes #1 = { nounwind }

观察发现main函数只有两行：一行输出string一行返回0。

再换一个程序，拿five.m为例，执行`LLVM-dis < five.bc | less`:

    #include <stdio.h>
    #import <Foundation/Foundation.h>
    
    int main() {
      NSLog(@"%@", [@5 description]);
      return 0;
    }

抛开其他的不说，单看main函数：

    define i32 @main() #0 {
      %1 = load %struct._class_t** @"\01L_OBJC_CLASSLIST_REFERENCES_$_", align 8
      %2 = load i8** @"\01L_OBJC_SELECTOR_REFERENCES_", align 8, !invariant.load !4
      %3 = bitcast %struct._class_t* %1 to i8*
      %4 = tail call %0* bitcast (i8* (i8*, i8*, ...)* @objc_msgSend to %0* (i8*, i8*, i32)*)(i8* %3, i8* %2, i32 5)
      %5 = load i8** @"\01L_OBJC_SELECTOR_REFERENCES_2", align 8, !invariant.load !4
      %6 = bitcast %0* %4 to i8*
      %7 = tail call %1* bitcast (i8* (i8*, i8*, ...)* @objc_msgSend to %1* (i8*, i8*)*)(i8* %6, i8* %5)
      tail call void (%1*, ...)* @NSLog(%1* bitcast (%struct.NSConstantString* @_unnamed_cfstring_ to %1*), %1* %7)
      ret i32 0
    }

看看最重要的第4行，创建了一个NSNumber对象。第7行，对number对象发送了一个执行description的消息。第8行，对description的返回结果做log。

Optimizations 优化
---------------

要想了解LLVM和clang能做哪些优化，先从一个略微复杂的C程序看起，这个函数主要是在递归计算阶乘：

    #include <stdio.h>
    
    int factorial(int x) {
       if (x > 1) return x * factorial(x-1);
       else return 1;
    }
    
    int main() {
      printf("factorial 10: %d\n", factorial(10));
    }

先看看直接编译不做优化，执行下面命令：

    clang -O0 -emit-llvm factorial.c  -c -o factorial.bc && llvm-dis < factorial.bc

重点看一下阶乘部分的代码：

    define i32 @factorial(i32 %x) #0 {
      %1 = alloca i32, align 4
      %2 = alloca i32, align 4
      store i32 %x, i32* %2, align 4
      %3 = load i32* %2, align 4
      %4 = icmp sgt i32 %3, 1
      br i1 %4, label %5, label %11
    
    ; <label>:5                                       ; preds = %0
      %6 = load i32* %2, align 4
      %7 = load i32* %2, align 4
      %8 = sub nsw i32 %7, 1
      %9 = call i32 @factorial(i32 %8)
      %10 = mul nsw i32 %6, %9
      store i32 %10, i32* %1
      br label %12
    
    ; <label>:11                                      ; preds = %0
      store i32 1, i32* %1
      br label %12
    
    ; <label>:12                                      ; preds = %11, %5
      %13 = load i32* %1
      ret i32 %13
    }

看一下%9标注的那一行，这行代码正递归调用阶乘函数本身，这样是非常低效的，因为每次递归调用都要重新压栈。接下来可以看一下优化后的效果，在clang命令中增加-03标志：

    clang -O3 -emit-llvm factorial.c  -c -o factorial.bc && llvm-dis < factorial.bc

优化后编译的阶乘计算代码如下：

    define i32 @factorial(i32 %x) #0 {
      %1 = icmp sgt i32 %x, 1
      br i1 %1, label %tailrecurse, label %tailrecurse._crit_edge
    
    tailrecurse:                                      ; preds = %tailrecurse, %0
      %x.tr2 = phi i32 [ %2, %tailrecurse ], [ %x, %0 ]
      %accumulator.tr1 = phi i32 [ %3, %tailrecurse ], [ 1, %0 ]
      %2 = add nsw i32 %x.tr2, -1
      %3 = mul nsw i32 %x.tr2, %accumulator.tr1
      %4 = icmp sgt i32 %2, 1
      br i1 %4, label %tailrecurse, label %tailrecurse._crit_edge
    
    tailrecurse._crit_edge:                           ; preds = %tailrecurse, %0
      %accumulator.tr.lcssa = phi i32 [ 1, %0 ], [ %3, %tailrecurse ]
      ret i32 %accumulator.tr.lcssa
    }


即便我们的源码书写并不是[尾递归][7]的方式，clang仍然能很好的优化编译，编译的结果中只包含一个循环。当然clang能对代码进行的优化还有很多方面。可以看以下这个比较不错的gcc的优化例子[ridiculousfish.com][8]。

**延伸阅读：**

 - [LLVM blog: posts tagged ‘optimization’][9]
 - [LLVM blog: vectorization improvements][10]
 - [LLVM blog: greedy register allocation][11]
 - [The Polly project][12]

如何在实际中应用这些特性
=============================================

刚刚我们探讨了编译的全过程，从标记到解析，从抽象语法树到分析检查再到汇编。读者不禁要问，为什么要关注这些？

使用libclang或clang插件
-------------------

clang最优秀的特点：它是本身构建得非常好且又开源的项目，几乎可以说到处是宝。使用者可以创建自己的clang分支，针对自己的需求进行改造。比如说，可以改变clang生成代码的方式，增加更强的类型检查，或者按照自己的定义进行代码的检查分析等等。要想达成以上的目标，有很多种方法，其中最简单的就是使用C类库[libclang][13]。libclang提供了API，可以对C和clang做桥接，可以用它对源码做分析。但是，按照我的经验，如果使用者的要求更加复杂高端，libclang就不太够用了。接下来，推荐一下[Clangkit][14]，它是Objective-C基于clang的一些功能做的封装。最后，clang还提供了一个直接使用LibTooling的C++类库。这里要做的事儿比较多，而且涉及到C++，但是它能够发挥clang的全部武功。如果有以下诉求：对代码做各种分析，重写程序，对clang增加分析方法，创建自己的重构器，或者对现有代码做大量的重写，甚至是想基于工程生成图例者说明文档，LibTooling都是很好的选择。


自定义分析
-----

使用者可以按照[Tutorial for building tools using LibTooling][15]的说明去构造LLVM，clang以及其他clang附加工具。需要注意的是，一定要为编译预留时间，就算你的机器已经挺快了，你还是有机会在LLVM编译的过程中吃顿饭什么的。

接下来去使用者机器的LLVM目录下执行命令cd ~/llvm/tools/clang/tools/，可以在这个目录下创建独立的clang工具。比如说，我们要创建一个小工具来帮助检查类库是否使用正确。把[样例工程][16]拷贝到这个目录下，然后执行make。接下来会生成一个叫example的二进制文件。

使用场景：假如有一个Observer观察者类：

    @interface Observer
    + (instancetype)observerWithTarget:(id)target action:(SEL)selector;
    @end

接下来，我们想要检查一下每当这个类被调用的时候，在目标对象中是否都有对应的响应action方法存在。可以写个C++函数来做这件事（注意，这是我第一次写C++程序，可能不那么严谨）：

    virtual bool VisitObjCMessageExpr(ObjCMessageExpr *E) {
      if (E->getReceiverKind() == ObjCMessageExpr::Class) {
        QualType ReceiverType = E->getClassReceiver();
        Selector Sel = E->getSelector();
        string TypeName = ReceiverType.getAsString();
        string SelName = Sel.getAsString();
        if (TypeName == "Observer" && SelName == "observerWithTarget:action:") {
          Expr *Receiver = E->getArg(0)->IgnoreParenCasts();
          ObjCSelectorExpr* SelExpr = cast<ObjCSelectorExpr>(E->getArg(1)->IgnoreParenCasts());
          Selector Sel = SelExpr->getSelector();
          if (const ObjCObjectPointerType *OT = Receiver->getType()->getAs<ObjCObjectPointerType>()) {
            ObjCInterfaceDecl *decl = OT->getInterfaceDecl();
            if (! decl->lookupInstanceMethod(Sel)) {
              errs() << "Warning: class " << TypeName << " does not implement selector " << Sel.getAsString() << "\n";
              SourceLocation Loc = E->getExprLoc();
              PresumedLoc PLoc = astContext->getSourceManager().getPresumedLoc(Loc);
              errs() << "in " << PLoc.getFilename() << " <" << PLoc.getLine() << ":" << PLoc.getColumn() << ">\n";
            }
          }
        }
      }
      return true;
    }

这段程序先是扫描消息方法的特定存在形式：以观察者作为消息方法的接收者，以observerWithTarget:action:作为selector，接着检查target中是否存在相应的selector。虽然这个例子有点儿刻意，但如果你想要利用AST对自己的代码库做某些机械检查，按照上面的例子来就可以了。


clang的其他特性
==========

clang还有许多其他的用途。比如，可以写编译器插件（类似上面的检查器例子）并且动态的加载到编译器中。虽然我没有亲自实验过，但是我觉得在Xcode中应该是可行的。再比如，也可以通过编写clang插件来自定义代码样式（具体可以参见[Build process][17]）。

另外，如果想对现有的代码做大规模的重构，如果Xcode本身集成的重构工具或者AppCode（一款第三方IDE）无法达你的要求，你完全可以用clang自己写个重构工具。听起来有点儿可怕，读读下面的文档和教程，你会发现其实没那么难。

最后，如果是真的有这种需求，你完全可以引导Xcdoe使用你自己编译的clang。再一次，如果你去尝试，其实这些事儿真的没想象中那么复杂，反而会发现许多个中乐趣。

**延伸阅读：**

 - [Clang Tutorial][18]
 - [X86_64 Assembly Language Tutorial][19]
 - [Custom clang Build with Xcode (I)][20] and [(II)][21]
 - [Clang Tutorial (I)][22], [(II)][23] and [(III)][24]
 - [Clang Plugin Tutorial][25]
 - [LLVM blog: What every C programmer should know (I)][26] , [(II)][27] and [(III)][28]

[更多issue #6文章][29]


  [1]: http://www.objc.io/issue-6/index.html
  [2]: http://twitter.com/chriseidhof
  [3]: http://www.aosabook.org/en/llvm.html
  [4]: http://www.objc.io/issue-6/mach-o-executables.html
  [5]: http://clang.llvm.org/docs/Modules.html
  [6]: http://clang.llvm.org/docs/IntroductionToTheClangAST.html
  [7]: http://en.wikipedia.org/wiki/Tail_call
  [8]: http://ridiculousfish.com/blog/posts/will-it-optimize.html
  [9]: http://blog.llvm.org/search/label/optimization
  [10]: http://blog.llvm.org/2013/05/llvm-33-vectorization-improvements.html
  [11]: http://blog.llvm.org/2011/09/greedy-register-allocation-in-llvm-30.html
  [12]: http://polly.llvm.org/index.html
  [13]: http://clang.llvm.org/doxygen/group__CINDEX.html
  [14]: https://github.com/macmade/ClangKit
  [15]: http://clang.llvm.org/docs/LibASTMatchersTutorial.html
  [16]: https://github.com/objcio/issue6-compiler-tool
  [17]: http://www.objc.io/issue-6/build-process.html
  [18]: https://github.com/loarabia/Clang-tutorial
  [19]: http://cocoafactory.com/blog/2012/11/23/x86-64-assembly-language-tutorial-part-1/
  [20]: http://clang-analyzer.llvm.org/xcode.html
  [21]: http://stackoverflow.com/questions/3297986/using-an-external-xcode-clang-static-analyzer-binary-with-additional-checks
  [22]: http://kevinaboos.wordpress.com/2013/07/23/clang-tutorial-part-i-introduction/
  [23]: http://kevinaboos.wordpress.com/2013/07/23/clang-tutorial-part-ii-libtooling-example/
  [24]: http://kevinaboos.wordpress.com/2013/07/29/clang-tutorial-part-iii-plugin-example/
  [25]: http://getoffmylawnentertainment.com/blog/2011/10/01/clang-plugin-development-tutorial/
  [26]: http://blog.llvm.org/2011/05/what-every-c-programmer-should-know.html
  [27]: http://blog.llvm.org/2011/05/what-every-c-programmer-should-know_14.html
  [28]: http://blog.llvm.org/2011/05/what-every-c-programmer-should-know_21.html
  [29]: http://www.objc.io/issue-6
