在平时的开发中经常需要阅读学习其他人的代码，当开始阅读一份自己完全不熟悉的代码时，通常会遇到一些麻烦，因为我必须要先找到代码逻辑的入口点并沿着逻辑链路将其梳理一遍，一份代码文件通常会伴随着许多的方法调用，这一个阶段往往是比较痛苦的，因为我必须花上许多时间来将这些方法之间的关系理清楚，这样才能在我的大脑中生成一份逻辑关系图。如果我们能自动生成源码中的方法调用图(Call Graph)，那样一定会对源码阅读有很大的帮助。  

我们需要一个能够自动生成源码方法调用图的工具，那么这个工具必须能够理解并分析我们的代码，而最能理解代码的当然就是编译器了。我们编译Objective-C的代码所用的前端是Clang，Clang提供了一系列的工具来帮助我们分析源码，我们可以基于Clang来构建自己的工具。在这之前简单介绍一些相关概念：

### 抽象语法树
抽象语法树（Abstract Syntax Code, AST）是源代码语法结构的树状表示，其中的每一个节点都表示一个源码中的结构，AST在编译中扮演了一个十分重要的角色，Clang分析输入的源码并生成AST，之后根据AST生成LLVM IR(中间码)。  

我们可以使用Clang提供的工具`clang-check`来查看AST，创建一个代码文件test.c
```c
int square(int num) {
	return num * num;
}

int main() {
	int result = square(2);
}
```
在终端执行命令`clang-check -ast-dump test.m`，可以看到转换后的AST结构：
```
|-FunctionDecl 0x7fa933840e00 </Users/lzephyr/Desktop/test.c:1:1, line:3:1> line:1:5 used square 'int (int)'
| |-ParmVarDecl 0x7fa93302f720 <col:12, col:16> col:16 used num 'int'
| `-CompoundStmt 0x7fa933840fa0 <col:21, line:3:1>
|   `-ReturnStmt 0x7fa933840f88 <line:2:2, col:15>
|     `-BinaryOperator 0x7fa933840f60 <col:9, col:15> 'int' '*'
|       |-ImplicitCastExpr 0x7fa933840f30 <col:9> 'int' <LValueToRValue>
|       | `-DeclRefExpr 0x7fa933840ee0 <col:9> 'int' lvalue ParmVar 0x7fa93302f720 'num' 'int'
|       `-ImplicitCastExpr 0x7fa933840f48 <col:15> 'int' <LValueToRValue>
|         `-DeclRefExpr 0x7fa933840f08 <col:15> 'int' lvalue ParmVar 0x7fa93302f720 'num' 'int'
`-FunctionDecl 0x7fa933841010 <line:5:1, line:7:1> line:5:5 main 'int ()'
  `-CompoundStmt 0x7fa9338411f8 <col:12, line:7:1>
    `-DeclStmt 0x7fa9338411e0 <line:6:2, col:24>
      `-VarDecl 0x7fa9338410c0 <col:2, col:23> col:6 result 'int' cinit
        `-CallExpr 0x7fa9338411b0 <col:15, col:23> 'int'
          |-ImplicitCastExpr 0x7fa933841198 <col:15> 'int (*)(int)' <FunctionToPointerDecay>
          | `-DeclRefExpr 0x7fa933841120 <col:15> 'int (int)' Function 0x7fa933840e00 'square' 'int (int)'
          `-IntegerLiteral 0x7fa933841148 <col:22> 'int' 2
```

### LibTooling和Clang Plugin

`LibTooling`是一个库，提供了对AST的访问和修改的能力，`LibTooling`可以用来编写可独立运行的程序，如我们上面所使用的`clang-check`，`LibTooling`提供了一系列便捷的方法来访问语法树。  

`Clang Plugin`与`LibTooling`类似，对AST有完全的控制权，但是不同的是`Clang Plugin`是作为插件注入到编译流程中的，并且可以嵌入xCode中。实际上使用`LibTooling`编写的独立工具只需要经过少许的改动就可以变成`Clang Plugin`来使用。

##访问抽象语法树
要获得函数之间的调用关系，我们必须分析AST，Clang提供了两种方法：`ASTMatchers`和`RecursiveASTVisitor`。

### ASTMatchers

`ASTMatchers`提供了一系列的函数，以DSL的方式编写匹配表达式来查找我们感兴趣的节点，并使用`bind`方法绑定到指定的名称上：

```c
StatementMatcher matcher = callExpr(hasAncestor(functionDecl().bind("caller")), 
                                    callee(functionDecl().bind("callee")));
```
上面的表达式匹配了源码中普通C函数的调用，并将调用者绑定到字符串"caller"，被调用者绑定到字符串"callee"，随后在回调方法中可以通过名称caller和callee来获取`FunctionDecl`类型的对象：
```c++
class FindFuncCall : public MatchFinder::MatchCallback {
public :
    virtual void run(const MatchFinder::MatchResult &Result) {
        // 获取调用者的函数定义
        if (const FunctionDecl *caller = Result.Nodes.getNodeAs<clang::FunctionDecl>("caller")) {
            caller->dump();
        }
        // 获取被调用者的函数定义
        if (const FunctionDecl *callee = Result.Nodes.getNodeAs<clang::FunctionDecl>("callee")) {
            callee->dump();
        }
    }
};

int main(int argv, const char **argv) {
	StatementMatcher matcher = callExpr(hasAncestor(functionDecl().bind("caller")),
                                        callee(functionDecl().bind("callee")));
    MatchFinder finder;
    FindFuncCall callback;
    finder.addMatcher(matcher, &callback);
	
    // 执行Matcher
    CommonOptionsParser OptionsParser(argc, argv, MyToolCategory);
    ClangTool Tool(OptionsParser.getCompilations(), OptionsParser.getSourcePathList());
    Tool.run(newFrontendActionFactory(&finder).get());
    return 0;
}
```

上述匹配表达式中的每一个函数(如callExpr)被称为一个`Matcher`，所有的`Matcher`可以分为三类：
- **Node Matchers**：匹配表达式的核心，用来匹配特定类型的所有节点，所有的匹配表达式都是由一个`Node Matcher`来开始的，并且只有在`Node Matcher`上可以调用`bind`方法。`Node Mathcher`可以包含任意数量的参数，在参数中传入其他的Matcher来操纵匹配的节点，但是需要注意的是所有作为参数传入的Matcher都会作用在同一个被匹配的节点上，如：
  ```
  DeclarationMatcher matcher = recordDecl(cxxRecordDecl().bind("class"),
  										hasName("MyClass"));
  ```
  该matcher的含义是查找名字为“MyClass”的c++类，`recordDecl`是一个`Node Matcher`，匹配所有的class、struct和union的定义；`hasName`匹配名字为"MyClass"的节点；`cxxRecordDecl`匹配C++类定义的节点，并将其绑定到字符串"class"上。
- **Narrowing Matchers**：顾名思义，这种Matcher提供了条件判断能力用来缩小匹配范围，如第二个例子中的`hasName`就是一个`Narrowing Matcher`，只匹配名称为"MyClass"的节点。
- **Traversal Matchers**：以当前匹配的节点作为起点，用来限定匹配表达式查找的范围。如第一个例子中的`hasAncestor`，在当前节点的祖先节点中进行下一步的匹配。

### RecursiveASTVisitor

`RecursiveASTVisitor`是Clang提供的另一种访问AST的方式，使用起来很简单，你需要定义三个类，分别继承自`ASTFrontendAction`、`ASTConsumer`和`RecursiveASTVisitor`。  
在自定义的MyFrontendAction中返回一个自定义的MyConsumer实例

```c++
class MyFrontendAction : public clang::ASTFrontendAction {
public:
    virtual std::unique_ptr<clang::ASTConsumer> CreateASTConsumer(
      clang::CompilerInstance &Compiler, llvm::StringRef InFile) {
      return std::unique_ptr<clang::ASTConsumer>(new MyConsumer);
    }
};
```
在AST解析完毕后会调用MyConsumer的`HandleTranslationUnit`方法，`TranslationUnitDecl`是一个AST的根节点，`ASTContext`中保存了AST相关的所有信息，获取`TranslationUnitDecl`并将其交给MyVisitor，我们主要的操作都在Visitor中完成
```c++
class MyConsumer : public clang::ASTConsumer {
public:
    virtual void HandleTranslationUnit(clang::ASTContext &Context) {
      Visitor.TraverseDecl(Context.getTranslationUnitDecl());
    }
private:
  	MyVisitor Visitor;
};
```
在Visitor中访问感兴趣的节点只需要重写该类型节点的Visit方法就行了，比如我想访问代码中所有的C++类定义，只需要重写`VisitCXXRecordDecl`方法，就可以访问所有的的所有的C++类定义了
```c++
class MyVisitor : public RecursiveASTVisitor<FindNamedClassVisitor> {
public:
  	bool VisitCXXRecordDecl(CXXRecordDecl *decl) {
    	decl->dump();
    	return true; // 返回true继续遍历，false则直接停止
  	}
};
```
之后在main函数中使用`newFrontendActionFactory`创建`ToolAction`就可以了:
```c++
Tool.run(newFrontendActionFactory<CallGraphAction>().get());
```

## 构建CallGraph工具

在Clang源码中的`Analysis`文件夹中提供了一个名为`CallGraph`的类，参考这份源码的实现编写了自己的CallGraph工具。其中核心部分主要为三个类：`CallGraph`、`CallGraphNode`和`CGBuilder`：

- **CallGraph**：继承自`RecursiveASTVisitor`，实现`VisitFunctionDecl`和`VisitObjCMethodDecl`方法，遍历所有的C函数和Objective-C方法：
  ```c++
  bool VisitObjCMethodDecl(ObjCMethodDecl *MD) {
      if (isInSystem(MD)) { // 忽略系统库中的定义
          return true;
      }

      if (canBeCallerInGraph(MD)) {
          addRootNode(MD); // 添加一个Node到Roots
      }
      return true;
  }
  ```
  在`addRootNode`中将其封装成`CallGraphNode`对象并保存在一个map类型的成员对象`Roots`中。随后获取函数体(`CompoundStmt`类型)，将其传递给`CGBuilder`查找在函数体中被调用的方法。
  ```c++
  void CallGraph::addRootNode(Decl *decl) {
    CallGraphNode *Node = getOrInsertNode(decl); // 将decl封装成Node，并添加到Roots中
    
    // 初始化CGBuilder遍历函数里中所有的方法调用
    CGBuilder builder(this, Node, Context);
    if (Stmt *Body = decl->getBody())
        builder.Visit(Body);
  }
  ```
- **CallGraphNode**：封装了一个`Decl`类型的的实例（C函数或OC方法的定义），用来表示一个AST节点，所有被该函数所调用的其他函数会被添加到vector类型的成员变量`CalledFunctions`中。
  ```c++
  class CallGraphNode {
  private:
      // C函数或OC方法的定义
      Decl *decl;
      // 保存所有被decl调用的Node
      SmallVector<CallGraphNode*, 5> CalledFunctions;
  ...
  ```
- **CGBuilder**：继承自`StmtVisitor`，初始化时获取一个CallerNode，遍历该CallerNode对应函数的函数体，查找函数体中的方法调用：`CallExpr`和`ObjCMessageExpr`。`CallExpr`表示普通的C函数调用，`ObjCMessageExpr`表示Objective-C方法调用。获取被调用函数的定义并封装成`CallGraphNode`类型，然后将其添加到CallerNode的`CalledFunctions`中。
  ```c++
  class CGBuilder : public StmtVisitor<CGBuilder> {
    CallGraph *G;
    CallGraphNode *CallerNode;
    ASTContext &Context;
  public:
    void VisitObjCMessageExpr(ObjCMessageExpr *ME) {
        // 从ObjCMessageExpr中获取被调用方法的Decl
        Decl *decl = ...
        
        // 将decl封装在CallGraphNode中并添加到CallerNode的CalledFunctions中
        addCalledDecl(decl); 
    }
  ...
  ```

目前只实现了一个基础版本，支持C和Objecive-C，实现了最基本的功能，代码也比较简单，之后会继续优化并增加新的功能，所有代码已经托管到github上：https://github.com/L-Zephyr/clang-mapper 

## 使用

可以下载并自行编译源码，或者直接使用release文件夹中预先编译好的二进制文件`clang-mapper`(使用Clang5.0.0编译)，由于采用了`Graphviz`来生成调用图，请确保在运行前已正确安装了`Graphviz`  

### 编译源码

关于如何编译用LibTooling编写的工具，[Clang官方文档](https://clang.llvm.org/docs/LibASTMatchersTutorial.html)中有详细的说明

1. 首先下载LLVM和Clang的源码。

2. 将`clang-mapper`文件夹拷贝到`llvm/tools/clang/tools/`中。

3. 编辑文件`llvm/tools/clang/tools/CMakeLists.txt`，在最后加上一句`add_clang_subdirectory(clang-mapper)`

4. 建议采用外部编译，在包含llvm文件夹的目录下创建build文件夹，在build目录中编译源码
   ```
   $ mkdir build
   $ cd build
   $ cmake -G 'Unix Makefiles' ../llvm
   $ make
   ```
   也可以按照文档中介绍的使用Ninja来编译，编译过程过会生成20多个G的中间文件，编译结束后在`build/bin/`中就能找到`clang-mapper`文件了，将其拷贝到`/usr/local/bin`目录下

### 基本使用

传入任意数量的文件或是文件夹，`clang-mapper`会自动处理所有文件并在当前执行命令的路径下生成函数的调用图，以代码文件的命名做区分。如下，我们用clang-mapper分析大名鼎鼎的AFNetworking的核心代码。我不希望将分析的生成的结果和源码文件混在一起，所以我创建了一个文件夹CallGraph并在该目录下调用

```
$ cd ./AFNetworking-master
$ mkdir CallGraph
$ cd ./CallGraph
$ clang-mapper ../AFNetworking --
```
之后程序会自动分析`../AFNetworking`下的所有代码文件，并在CallGraph目录下生成对应的png文件：  
<img src="./res/4.png"/>
![](https://raw.githubusercontent.com/L-Zephyr/static_resource/master/Resources/clang_mapper/ef862a4410793adc464e372e5494535a.png)

### 命令行参数

clang-mapper提供了一些可选的命令行参数

- **-graph-only**：只生成调用图png文件，不保留dot文件，这个是默认选项
- **-dot-only**：只生成dot文件，不生成png文件
- **-dot-graph**：同时生成dot文件和png文件
- **-ignore-header**：在iOS开发中头文件通常只用来声明，加上该选项可以忽略文件夹中的.h文件

## 参考资料

- https://clang.llvm.org/docs/LibASTMatchersTutorial.html
- https://clang.llvm.org/docs/RAVFrontendAction.html
- https://clang.llvm.org/docs/LibASTMatchersReference.html
- https://clang.llvm.org/docs/IntroductionToTheClangAST.html