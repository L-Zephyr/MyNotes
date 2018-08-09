在很久之前的一篇[博客](http://www.jianshu.com/p/e19aafbaddca)中，曾经用clang提供的库`LibTooling`编写了一个简单的导出iOS代码中函数调用关系图的工具，然而这种实现方式存在一些很明显的缺点：

1. 在分析一个工程中的单个代码文件时，无法得知定义在其他文件中的类或方法，导致生成的语法树节点缺失，对最终的结果造成不小的影响。
2. 在解析时clang会进行预处理，导致最终生成的结果可能包括一些外部系统库的函数，这对于我们来说是无用的信息（当然这个应该是我的使用姿势问题）。
3. 无法支持swift。swift编译器的前端并不是clang，而这个工具是基于clang的库来开发的，所以也就没有支持swift的可能。

由于这几个缺点（主要是第三点，因为在日常工作中还是以swift为主），后来也没有再继续使用和完善。直到最近因为工作上的安排，需要维护一份较为陈旧的代码，面对动辄数千行的代码文件，觉得还是需要一个比较趁手的工具来辅助阅读。前段时间正好恰逢国庆长假，抽空用swift重新写了一个工具：drafter，如名字所示，它的目的在于生成描述代码的草图。

## Drafter是什么

- Drafter是一个命令行工具，用于分析iOS工程的代码，支持Objective-C和Swift。
- 自动解析代码并生成方法调用关系图。
- 自动解析代码并生成类继承关系图。

## 安装和使用

完整的代码在这里：[https://github.com/L-Zephyr/Drafter](https://github.com/L-Zephyr/Drafter)

这里提供了一个快速安装的脚本，在shell中执行指令：

```shell
curl "https://raw.githubusercontent.com/L-Zephyr/Drafter/master/install.sh" | /bin/sh
```

`drafter`程序会自动安装到  /usr/local/bin 目录中，之后直接在终端使用即可。

具体使用方法请查看[使用介绍](https://github.com/L-Zephyr/Drafter#基本使用)

## 实现原理

*注：解析器部分后来已用parser combinator重构，文章所讲述的代码对应于0.1.0的tag*

在之前的做法中对源码的解析全交给clang，只对生成的AST做处理，这其实是一种比较偷懒的做法，对最后生成的结果不可控，而且也断了支持swift的可能。为了获得更优化的输出并同时支持Swift和OC，源码解析这一步还是得自己来做。幸运的是我们只需要解析类、方法定义、方法调用这几块，实际工作并不是很复杂。

### 词法解析

词法解析是程序编译的第一步，所谓词法解析就是将代码分割成一系列的词法单元。词法单元是一个有特殊意义的标记，也是语法分析程序在处理源代码时的最小单元。比如说一个简单的赋值表达式`int i = 3`，在经过词法分析之后被处理成了一系列的词法单元：`int`、`i`、`=`、`3`。

```swift
struct Token {
    var type: TokenType
    var text: String
}

enum TokenType {
    case endOfFile   // 文件结束
    case name        // 变量名
    case colon       // 冒号 	
    case comma       // 逗号     
  	...
}
```

先定义一个名为`Token`的结构体，用来表示词法单元，其中枚举值`type`用来表示词法单元的类型，`text`保存该词法单元的原始数据，如：对于一个变量n，它在解析成Token之后type为`.name`，text为`n`。由于我们的目的只是解析类和方法，所以这里只定义了在类和方法的定义中会用到的词法单元类型，对于那些我们不关心的词法则一概忽略。

词法解析器会将任何输入的源代码解析成词法单元流，对于上层使用者来说就像是迭代器一样遍历词法单元直到文件结束，所以这里可以定义一个基本的词法解析器类型，只有一个计算属性`nextToken`，用来获取下一个词法单元：

```swift
protocol Lexer {
    var nextToken: Token { get }
}
```

### 语法解析

在经过第一步的词法分析将源代码分割成带有类型的词法单元之后，就可以进入语法解析的阶段了。要分析一段程序，如表达式`1 + 2`，我们是无法直接从字面上来处理的，必须将其转换成某种可以处理的中间形式，这就是语法解析要做的事情。语法解析器根据语言的文法规则扫描词法单元流，同时生成中间表示形式（IR），通常来说会生成一棵抽象语法树（AST），之后的语义分析阶段会基于这一步生成的AST进行分析。Drafter只处理到语法解析这一步，仅对代码中的类、方法定义和方法调用进行解析，解析后生成的数据结构也比较简单。

#### 语言的文法描述

程序是由多个有效的表达式组成的，我们要做的就是将这些符合特定规则的式子识别出来，语言特定的语法规则称为这门语言的文法，这种规则可以用一种DSL来描述（BNF范式）。

举个例子（来源于《编程语言实现模式》一书），对于一个可以包含任意字母的列表声明如`[a, b, c]`，它的文法规则描述如下：

```c
list = '[' elements ']'; // 单引号之间的内容直接匹配
elements = elemenet (',' element)*; // *表示0个或多个
element = NAME | list; // |表示或，元素可能是另一个列表
NAME = ('a'..'z' | 'A'..'Z')+; // +表示一个或多个
```

上面每一条式子都描述了一条文法规则，这里将词法规则和文法规则做了区分，文法规则的名称小写，词法规则的名称大写。像`list`这样的规则称为产生式，它可以继续向下推导，如`list`会产生`elements`。另外有一些被单引号包围的符号，这样的符号是实际要匹配的内容，称为终结符，因为它无法再继续往下推导了。

这个文法描述了一个列表声明的语法，每个规则都包含一个或多个解析选项，多个解析选项通过`|`符号分隔。上面声明了三个文法规则和一个词法规则：词法规则`NAME`匹配包含至少一个字母的词法单元；`list`规则表示列表必须由中括号包围，并至少包含一个元素，多个元素之间用逗号分隔，元素可以是一个变量也可以是另一个列表声明。

有了明确的文法规则定义我们才能够去编写语法解析器，对Objective-C的文法我参考了[这里](https://kornel.ski/objective-c-grammar)。

#### 递归下降分析法

定义了语法的结构和相关的词法单元之后，在解析时只需要识别出相应的式子即可，简单来说解析器的工作就是：遇到某种结构，就执行某种操作。具体到实现上，我们为每一种文法规则提供一个专用匹配函数，对于词法规则则统一用`match`函数来匹配：

```swift
@discardableResult
func match(_ t: TokenType) throws -> Token // 匹配指定类型词法单元，匹配成功返回该词法单元
```

对于上面那个列表的例子，可以编写如下用于识别的函数：

```swift
func list() throws
func elements() throws
func element() throws
```

每个函数都识别一个特定的子结构，并且可能会调用其他的识别函数或递归调用自身。在识别时从起始的词法单元开始，自上而下进行推导。所以这种分析的方法也被称为递归下降分析法，以这种方法编写的解析器称为LL解析器。第一个L表示解析内容的输入顺序是从左到右，第二个L表示解析时也是从左向右进行推导（最左推导）。

对于上面的`element`规则，它可能匹配一个变量名或是另一个列表，在进入`element`函数时需要先进行判断，所幸`list`规则始终以`[`符号开始，变量的规则始终以字母开始，只需要检查当前的词法单元类型就可以做出判断：

```swift
func element() throws {
    if currentToken.type == .leftSquare {
      	try list()
    } else {
        try match(.name)
    }
}
```

在这个列表的文法规则中，从当前的位置开始只需要检查一个词法单元的类型就可以做出决断，像这样的文法称为`LL(1)`文法，相应的解析器称为LL(1)解析器，1表示该解析器只能从解析位置向前查看一个词法单元，通常这个词法单元被称为前瞻符号（lookahead）。

#### LL(k)解析器

LL(1)解析器十分简单，但是解析能力不足。比如在上面列表语法的例子中，为列表的元素添加一个赋值的操作：`[a, b = c, d]`，这样一来，`element`规则就变成了：

```c
element = NAME
		| NAME '=' NAME
		| list
```

`element`文法中有两个解析选项都是以词法单元`NAME`开头的，仅查看一个词法单元无法确定，在解析时需要向前检查更多的词法单元，也就是说这个语法不再是LL(1)的了。

在实际解析时情况比这里要复杂很多，可能需要向前检查看多个词法单元才能确定解析策略，所以需要构建一个能够根据需要查看任意多符号的解析器，也就是LL(k)解析器。目前在应用上有一些能够根据特定DSL自动生成解析器的工具，如Antlr等，但是考虑通过DSL生成的代码并不是特别便于调试，而且Drafter只是做了一些非常简单的解析工作，所以还是自己编写了一个简单的LL(k)解析器。在Drafter中提供一个这样一个基础的解析器：

```swift
class BacktrackParser: Parser {
    init(lexer: Lexer) {
        self.input = lexer
    }
  
  	func token(at index: Int = 0) -> Token {
        ...
    }
  	...
}
```

以一个词法解析器(Lexer)作为初始化参数，`token()`方法提供从当前位置开始向前查看任意位置词法单元的能力，而具体的文法规则解析则通过各个子类化的解析器来完成。Objective-C和Swift的代码通过不同的解析器来进行，解析完成后输出相同的数据结构，如表示类型的节点：

```swift
class ClassNode: Node {
    var superCls: ClassNode? = nil // 父类
    var className: String = ""     // 类名
    var protocols: [String] = []   // 实现的协议
}
```

在将所有关心的语法节点信息解析出来之后，剩下的就是对这些信息进行处理和展示了。Drafter中提供了一些对语法节点进行过滤和搜索的选项，通过提供的参数过滤出感兴趣的信息，最后将这些数据传递给`DotGenerator`类，这个类的作用是根据节点信息生成`Dot`语言（一种描述图形的语言）的代码，传递给[Graphviz](http://www.graphviz.org/Download_macos.php)生成图片。

#### 方法调用解析

单独讨论一下对于方法调用的解析，首先为方法调用定义一个语法节点类型：

```swift
enum MethodInvoker {
    case name(String)    // 普通变量
    case method(MethodInvokeNode) // 另一个方法调用
}

class MethodInvokeNode: Node {
    var isSwift: Bool = false
    var invoker: MethodInvoker = .name("") // 调用者
    var params: [String] = [] // 参数名
    var methodName: String = "" 
}
```

一个方法的调用者可能是一个变量，也可能是另一个方法调用的返回值（链式调用），所以`invoker`被定义为一个枚举值。

OC方法调用的Parser由类`ObjcMessageSendParser`实现，swift方法调用的Parse由类`SwiftInvokeParser`实现。以OC为例，对于这样的简单调用：

```objective-c
[self.view insertSubview:subview atIndex:0];
```

匹配的结果为:`[self.view insertSubview: atIndex:]`，忽略参数的具体内容。对于链式的方法调用：

```objective-c
[[self objectAtIndex: 1] doSomethingWith: param];
```

解析的结果只保留一个链式调用的表示：`[[self objectAtIndex:] doSomethingWith:]`，而不是`objectAtIndex:`和`doSomethingWith:`。

而对于一些更加复杂的形式，如参数为一个Block的定义，Block中还调用了其他方法，如：

```objective-c
[Post globalTimelinePostsWithBlock:^(NSArray *posts, NSError *error) {
    if (!error) {
        self.posts = posts;
        [self.tableView reloadData];
    }
}];
```

先看看对于OC方法调用文法的一个简单定义：

```c
message_send = '[' receiver param_list ']'
receiver = message_send | NAME
param_list = NAME | (NAME ':' param)+
param = ...
```

方法调用中具体的参数是通过规则`param`来解析的，`param`要知道自己当前是否位于另一个闭包或是其他子结构中，这样才能在正确的时机结束匹配，这一步可以通过计算左右括号的数量来判断，`param`在碰到另一个方法调用语句时进入`message_send`规则并将结果添加到最后的匹配结果中，伪代码如下：

```swift
func param() throws {
        while 文件未结束 {
            if 不在子结构中 && 参数匹配结束 {
				return
            }
          
            if isMessageSend() {
                try messageSend() // 匹配方法调用
				保存到最终的匹配结果中
                continue
            }
            consume()
        }
    }
```

## 后记

以上就是Drafter实现的基本思路，开头提到的三个问题基本上得到了解决。在这段时间的工作中Drafter给了我不少帮助，至少当我在面对一个这样的文件：

![1](https://raw.githubusercontent.com/L-Zephyr/static_resource/master/Resources/drafter/51b3df7d634d8560e1b97f8734dcf736.png)

以及动辄数百行的方法时不再那么头疼，导出指定方法的调用流可以更迅速的理清代码逻辑上的关系：

![2](https://raw.githubusercontent.com/L-Zephyr/static_resource/master/Resources/drafter/3c7ea652b5939487a852988c23e4d41b.png)

之后如果有需要的话会为Drafter添加更多的功能、增强解析能力等，希望这个小工具能稍微减轻你在阅读代码时的负担😁。