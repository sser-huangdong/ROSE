分步骤完成ex4
=============
大致步骤：  

1. 完成语法翻译(ENBF转java)
2. 完成画图(画图api)
3. 完成语义异常检查

``Appendix: 最后会提供实验装置。``

##1. 语法翻译：
###1.1 准备阶段:

1. 新建一个`OberonParser.java`。
2. 沿用ex3的`Symbol.java`。
	最好改名成`MySymbol.java`, 免去与java_cup的Symbol类`java_cup.runtime.Symbol`冲突。
3. 沿用ex3的OberonScanner.java
	如果在`2.` 的时候进行了改名的话,记得修改一下`.flex`文件重新生成`OberonScanner.java`。改的话就是将里面的`Symbol.`改成`MySymbol.`, 但是`java_cup.runtime.Symbol`就不能改, 所以最好不要直接使用全局替换。我相信要是你自己完成ex3的话这部分完全没难度。
4. 我提供的实验装置基本上可以沿用ex3的, 
	主要是bin下的java_cup的class要保留。

5. 如果你不想沿用实验三里面的java_cup, 请自写一个Symbol类。并且修改一下`.jlex`文件

6. 最好开IDE(Eclipse)写, debug是比较重要的

7. 最好复习一下下神马叫 LL(1)

接下来可以开写OberonParser.java了

###1.2 完成大框架:
####1.2.1 OberonParse应该有的成员变量:  
```java
OberonScanner scanner;  
Symbol lookahead;   
```
这个Symbol是java_cup的类, 主要有两个两个成员变量  
`.value`表示返回对象, 没有时为null  
`.sym`  表示类型, 类型是int 对应`MySymbol`里面的值  


####1.2.2 应该有的公有函数:  
```java
public OberonParser(OberonScanner scanner) {  
	this.scanner = scanner;  
}  
public void parse() throws Exception {...}  
```

####1.2.3 私有函数分为两种:  
1. 一种是基本操作的:  
至少有获得lookahead(调用scanner.next_token())和match  
还可以添加获得Symbol的内容函数(返回String), 这个比较常用  
说说我的吧:  
我还有一个 bool isMatch(MySymbol) 的函数  
这个是判断lookahead类型是否跟预测的类型相等  
而String match(MySymbol) throws Exception 定义为  
	直接把lookahead匹配我认为的类型, 如果不匹配应该抛出语义异常  
	最后返回我match到内容  
	同时刷新我的lookahead  


2. 一种是进行语法翻译的:  
每个非终结符对应一个函数  
一开始返回值先设置为void, 比如  
void module() throws Exception {...}  
这部分根据EBNF可以轻易完成。  
后面可以稍微增加一些非终结符, 以保证一个方法不会太冗长。  

###1.3 如何将EBNF转成java代码:  

先忽略可能遇到的文法修改, 先开始写  
`公式:`  

```java
// 大写是非终结符, 小写是终结符
A = aB ==>
A() {
	match(a);
	B();
}

A = BC ==>
A() {
	B();
	C();
}

A = B|C ==> 
A() { // make sure first(B) (intersect) first(C) == emptySet
	if (lookahead in first(B)) {
		B();
	} else if (lookahead in first(C)){
		C();
	} else throw new SemanticException();
}

A = [B] ==>
A() {
	if (lookahead in first(B)) {
		B();
	}
	// empty
}

A = {C} ==>
A() {
	while (lookahead in first(C)) {
		C();
	} // empty
}

// 这个公式不是必要的, 不过可以处理相当一部分所以就加上了
A = C{bC} ==>
A() { // b is terminal symbol
	C();
	while (lookahead == b) {
		shift_token(); //把lookahead指向下一位, 其实不应该叫shift的
		C();
	} // empty
}
```

`解释:`
里面的first(A) 可以先算, 也可以一边写, 一边观察
我是看别人的代码然后直接观察出来

1. 例子1
```java
module  = 
		"MODULE" identifier ";"  
            declarations  
        ["BEGIN"  
            statement_sequence]  
        "END" identifier "." ;  
```
```java
void module() throws Exception {
	// 'module' 已经在parse里面match了!
	String id = match(MySymbol.IDENTIFIER);

	match(MySymbol.SEMI);
	
	declarations();
	
	if (isMatch(MySymbol.BEGIN)) {
		shift_token();
		statement_sequence();
	} // empty
	
	match(MySymbol.END);
	
	String id2 = match(MySymbol.IDENTIFIER);
	if (!id.equals(id2))  throw new ModuleNameMismatchedException();

	match(MySymbol.PERIOD);
	
	if (!isMatch(MySymbol.EOF)) {
		throw new SemanticException();
	} 
}
```

3. 例子2
```java
declarations =   
		[const_declarations]
        [type_declarations]
        [var_declarations]
        {procedure_declarations} ;
```
```java
void declarations() throws Exception {
	if (isMatch(MySymbol.CONST)) {
		shift_token();
		const_declarations();
	} // empty
		
	if (isMatch(MySymbol.TYPE)) {
		shift_token();
		type_declarations();
	} // empty
		
	if (isMatch(MySymbol.VAR)) {
		shift_token();
		var_declarations();
	} // empty
		
	while (isMatch(MySymbol.PROCEDURE)) {
		shift_token();
		procedure_declarations();
	} // empty
}
```

做到这里, 已经算完成了绝大部分的代码了。
可以开始测试了, 语法错误一般都能报错, 虽然不能完全指定类型。语义错误没法识别的。
正确的测试样例应该是不会报错才对, 假如报错就开始debug吧。
debug的时候把尽量把样例的代码减少, 然后使用单步调试,相对而言还是能比较快完成的。

这部分做的时候最好完成一些简单的语法错误的检查。(报出正确的异常)

##2. 完成画图
首先，画图有两种方法：一种是把画图相关的代码写进`OberonParser`里面，另一种把画图相关的代码都写出到另外一个java文件里面，然后再调用批处理来运行。  
虽然有一位比较早做完的同学一直推荐我使用写到外部去的做法，但是我思考了一下其实画图的代码真心不多，就使用了第一种方法。  
下面简单说一下画图需要用到的API还有画图改到需要对`OberonParser`做哪些修改。  
###2.1 画图API
```java
Module mainModule = new Module(moduleName); // Tpye of moduleName is `String`
Procedure mainProcedure = mainModule.add(procedureName); // Tpye of procedureName is `String`
mainProcedure.add(statements); // Type of statements is `StatementSequence` 
statements.add(statement)// Type of statement is `AbstractStatement`

WhileStatement whileStatement = new WhileStatement(expr); // Type of expr is `String`. 
whileStatement.getLoopBody().add(statements);
IfStatement ifStatement = new IfStatement(expr);WhileStatement(expr); // Type of expr is `String`. 
ifStatement.getTrueBody().add(statements);
ifStatement.getFalseBody().add(statements);
PrimitiveStatement(expr); // for a single statement.
// PrimitiveStatement, IfStatement, WhileStatement and StatementSequence both extend `AbstractStatement`

```

###2.2 修改OberonParser
你需要开始定义非终结符对应的函数的返回值了。有些需要使用`String`有些是`AbstractStatement`, 这些是自己思考的。去到后面的语义异常处理，最好定义一些自己写的类的。  


##3. 完成语义异常检查
你觉得我有心情做这些东西?!  
这部分内容跟ex3差不多，至少处理id和type的关系可以使用map这些技巧差不多。感觉总体会比ex3容易一些，因为这个类是自己写的，而且LL语法相对而言跟人类思维比较相似。  
我自己的情况是下午5点到晚上1点就写完了语法翻译，第二天早上大概花了2个多小时完成了画图。语义异常检查我还真没做，就不忽悠了。

##Appendix 我提供的实验装置
```
.\ex4
│  build.bat
│  debugparser.bat
│  doc.bat
│  EBNF
│  gen.bat
│  readme.txt
│  run.bat
│  test.bat
│
├─bin
│  ├─exceptions
│  │      ......
│  └─java_cup
│      │  ......
│      ├─anttask   
│      │    ......
│      └─runtime
│           ......
├─doc
├─lib
│   ......
└─src
    │  Main.java
    │  MySymbol.java
    │  oberon.flex
    │  OberonParser.java
    │
    ├─exceptions
    │      ......
    └─testcases
        ├─OberonSrc
        └─Samples
            │  .......
            ├─LexicalErrors
            │       ......
            ├─SemanticErrors
            │       ......
            └─SyntacticErrors
                    ......
```

1. bin里面已经放了java_cup运行时的类还有异常，异常跟我ex2的文档一样。
2. 不过加入Eclipse的时候异常的类好像必须要源代码，所以源代码也提供到了src下面。
3. 自己的代码放到OberonSrc里面即可。
4. ex4下还提供了原来的EBNF，好像从`.pdf`拷贝出来有点问题，于是就会从别处搬运过来了。
5. 批处理文件基本是基于ex3修改了，修改了之前我们提供的ex3的批处理文件里面的逻辑错误。同时还添加了`debugparser.bat`是只编译`parse.java`的批处理。同时保留了`gen.bat`用来生成`OberonSanner.java`的

---
祝顺利
