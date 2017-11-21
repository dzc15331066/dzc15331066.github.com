# 编译原理实验——基于表达式的计算器`EXPREVAL`

[TOC]

## 1.3.1 	讨论语法定义的二义性

表达式的BNF存在二义性。

文法改写成：

```
A -> decimal | (A)
	|A + A |A - A
	|A * A |A / A
	|A ^ A
	|-A
	|B?A:A
	|U
	|V
U -> sin(A) | cos(A)
V -> max(A,AL)
	|min(A,AL)
AL -> A | A,AL
B -> true|false
	|(B)
	|A > A
	|A >= A
	|A <= A
	|A = A
	|A <> A
	|B & B
	|B | B
	|!B
```

一个有二义性的句子是decimal * decimal + decimal

它有两棵不同的语法树：

![2017-11-18 10-05-48屏幕截图](/home/dengzhc/图片/2017-11-18 10-05-48屏幕截图.png)

ExprEval是通过算符优先级来解决二义性，比如上述的句子，规定`*`优先级高于`+`,那它只能采用第一棵语法树。

## 1.3.2	设计并实现词法分析程序		

有限状态自动机：

integral:

![2017-11-11 14-15-53屏幕截图](/home/dengzhc/图片/2017-11-11 14-15-53屏幕截图.png)

fration:

![2017-11-11 14-32-43屏幕截图](/home/dengzhc/图片/2017-11-11 14-32-43屏幕截图.png)

exponent:

![2017-11-11 14-33-00屏幕截图](/home/dengzhc/图片/2017-11-11 14-33-00屏幕截图.png)

decimal:

![2017-11-11 14-38-01屏幕截图](/home/dengzhc/图片/2017-11-11 14-38-01屏幕截图.png)

### 单词分类

#### 运算符分类：

实验中我把每个运算符看做一类单词。用一个数组存放预定义函数名和布尔常量。decimal 和 bool 分别用一个类来表示。记录字符串的当前识别位置，和字符串的长度，以此判断字符串的边界。

## 1.3.3 构造算符优先关系表

要根据算符之间的优先关系来决定是移入或规约，首先必须构造一个优先级关系表和一个与位置相关的比较关系。

大体上的算符优先关系如下

![2017-11-18 10-59-22屏幕截图](/home/dengzhc/图片/2017-11-18 10-59-22屏幕截图.png)

用java实现为

```java
public OpTable() {
		hash.put("(", new Op("(",Op.NONE,1));
		hash.put(")", new Op(")",Op.NONE,1));
		hash.put("sin", new Op("sin",Op.NONE,2));
		hash.put("cos", new Op("cos",Op.NONE,2));
		hash.put("max", new Op("max",Op.NONE,2));
		hash.put("min", new Op("min",Op.NONE,2));
		hash.put("sin", new Op("sin",Op.NONE,2));
		hash.put("^", new Op("^",Op.RIGHT,4));
		hash.put("*", new Op("*",Op.LEFT,5));
		hash.put("/", new Op("/",Op.LEFT,5));
		hash.put("+", new Op("+",Op.LEFT,6));
		hash.put("-", new Op("-",Op.LEFT,6));
		hash.put("=", new Op("=",Op.LEFT,7));
		hash.put("<>", new Op("<>",Op.LEFT,7));
		hash.put("<", new Op("<",Op.LEFT,7));
		hash.put("<=", new Op("<=",Op.LEFT,7));
		hash.put(">", new Op(">",Op.LEFT,7));
		hash.put(">=", new Op(">=",Op.LEFT,7));
		hash.put("!", new Op("!",Op.RIGHT,8));
		hash.put("&", new Op("&",Op.LEFT,9));
		hash.put("|", new Op("|",Op.LEFT,10));
		hash.put("?", new Op("?",Op.RIGHT,11));
		hash.put(":", new Op(":",Op.RIGHT,11));
		hash.put(",", new Op(",",Op.RIGHT,12));
	}
```

我把结合性质分为三类:左结合(`Op.LEFT`)、右结合(`Op.RIGHT`)、不结合(`Op.NONE`)

增加了`,`的结合性和优先级。这里我设为右结合，优先级为12

decimal 和 bool 常量的优先级为0

当然还有一些敏感关系要处理，我在Op类中定义了compare函数，处理依赖于位置关系的算符优先关系。

java 代码如下

```java
public static int compare(Op op_1, Op op_2) {
		Op op1 = new Op(op_1.lexeme,op_1.assoc,op_1.pr);
		Op op2 = new Op(op_2.lexeme,op_2.assoc,op_2.pr);
		if(op1.lexeme.equals("?")&&op2.lexeme.equals(":")) {
			return 0;
		}
		if(op1.lexeme.equals(")")) {
			return 1;
		}
		if(op2.lexeme.equals("(")) {
			op2.pr = 2;
		}
		if(op1.lexeme.equals("(")) {
			op1.pr = 99;
		}
		if(op2.lexeme.equals(")")) {
			op2.pr = 99;
		}
		if(op1.pr < op2.pr) {
			return 1;
		}
		if(op1.pr > op2.pr) {
			return -1;
		}
		if(op1.assoc == LEFT) {
			return 1;
		}
		if(op1.assoc == NONE) {
			return 0;
		}
		if(op1.assoc == RIGHT) {
			return -1;
		}
		
		return 0;
	}
```

实现过程中主要处理括号的优先级变化，三目运算符的优先关系。

`)`在左边 ，优先级最高(比常量还高)，所以直接return 1;

`)`在右边，优先级最低(除了`$`是最低的)，我将它的优先级设为99。

`(`在右边，优先级和函数相同，设为２

`(`在左边，优先级为99。

三目运算符处理:

如果`?`在`:`右边，则优先级是相等的。

其它情况按照优先级和结合性来判断最终的结合性。

`-`的二义性问题：

我在词法分析里面处理：

用一个变量存前一个识别的`token`,如果为`null`或者不为左括号和decimal,那么就当负号处理，优先级设为3,否则当减号处理，优先级设为6

所以优先级的问题就解决了。接下来就是语法分析和语义动作的处理了。

## 1.3.4 设计并实现语法分析和语义处理程序

Parser 的核心算法

![2017-11-18 10-18-15屏幕截图](/home/dengzhc/图片/2017-11-18 10-18-15屏幕截图.png)

当然在实验过程中reduce会处理语义动作，代码较复杂，所以单独作为一个方法

处理过程中我将BNF表示改了一下

```
A -> decimal | (A)
	|A + A |A - A
	|A * A |A / A
	|A ^ A
	|-A
	|B?A:A
 	|sin(A) | cos(A)
	|max(A,AL)
	|min(A,AL)
AL -> A | A,AL
B -> true|false
	|(B)
	|A > A
	|A >= A
	|A <= A
	|A = A
	|A <> A
	|B & B
	|B | B
	|!B
```



所以函数直接规约成了A

所以用java代码实现为:

```java
while(true) {
			printStack();
			if(lookahead.lexeme.equals("$")&&terminal.peek().lexeme.equals("$"){
				if(stack.peek().lexeme.equals("B")) {
					throw new TypeMismatchedException();
				}
				return accept();
			}
			topOp = terminal.peek();
			
			if(Op.compare(topOp, (Op)lookahead) <= 0) {
				shift();
			}else {
				reduce();
			}
		}
```

其中terminal存的是终结符号，stack存的是终结符号和非终结符号

shift的代码如下

```java
private void shift() throws ExpressionException {
		stack.push(lookahead);
		terminal.push((Op)lookahead);
		lookahead = Scanner.getNextToken();
}
```

reduce 代码如下:

```java
private void reduce() throws ExpressionException {
		form = new Vector<Token>();
		formString = "";
		nString = "";
		res = null;
		boolean end = false;
		topOp = new Op("$",Op.RIGHT,100);
		
		do {
			top = stack.pop();
			if(top.tag == Tag.OPERATOR) {
				terminal.pop();
				topOp = (Op)top;
			}
			if(stack.peek().tag == Tag.OPERATOR) {
				if(Op.compare((Op)stack.peek(),topOp) < 0) {
					end = true;
				}
			}
			form.add(0,top);
			formString = top.lexeme + formString;
			if(top.tag == Tag.NONTERMINAL) {
				nString = "N"+nString;
			}else {
				String tmp = top.lexeme;
				if(tmp.equals("(")||tmp.equals(")")) {
					nString = tmp + nString;
				}else if(isFunc(top)){
					nString = "F" + nString;
				}else if(isConstant(top)) {
					nString = "C" + nString;
				}else {
					nString = "o" + nString;
				}
				
			}
			if(end) {
				break;
			}
		}while(true);
		System.out.println(formString);
		checkAndSet();
		if(res != null) {
			stack.push(res);
		}else {
			
			throw new SyntacticException();
		}
	}
```

我用两个string类型的变量来处理法分析和语义动作。`nString`保存的是抽象的符号,其中`o`表示运算符，`C`表示常量(decimal 和 bool)，`N`表示非终结符号,`F`表示函数。`formString`是具体的句型，是将出栈的token的lexeme拼接在一起的。`form` 保存的是出栈顶的`token` ,  `res`保存的是规约之后的新的`token`,`res`是`static`的变量，每次规约都会更新`res`的值，如果`res`为`null`,就说明出现错误

checkAndSet的处理是先判断nString,然后判断formString,最后根据对应formString执行语义动作。

checkAndSet代码如下

```java
private void checkAndSet() throws ExpressionException {
		switch(nString) {
			case "oN":
				if(formString.charAt(0) == '-') {
					checkUnaryOp();
				}else {
					throw new MissingOperandException();
				}
				break;
			case "NoN":
				if(formString.equals("A:A")) {
					throw new TrinaryOperationException();
				}
				checkBinaryOp();
				break;
			case "NoNoN":
				checkTrinaryOp();
				break;
			case "(N":
			case "F(N":
				throw new MissingRightParenthesisException();
			case "N)":
				throw new MissingLeftParenthesisException();
			case "No":
				throw new MissingOperandException();
			case "NC":
			case "NN":
			case "N(N)":
				throw new MissingOperatorException();
			case "FN":
			case "FN)":
				throw new FunctionCallException();
			case "F(N)":
				checkFunc();
				break;
			case "F()":
			case "()":
				throw new MissingOperandException();
			default:
				checkOther();
				break;
		}
		
```

checkAndSet主要分成checkUnaryOp(),checkBinaryOp(),checkTrinaryOp(),checkFunc(),checkOther()五个部分去实现。现在以checkUnaryOp()为例解释一下语义处理过程

checkUnaryOp

```java
private void checkUnaryOp() throws TypeMismatchedException {
		switch(formString) {
		case "-A":
			res = new ArithExpr(-((ArithExpr)form.elementAt(1)).value);
			break;
		case "!B":
			res = new BoolExpr(!((BoolExpr)form.elementAt(1)).value);
			break;
		default:
			throw new TypeMismatchedException();
		}
	}
```

程序写到这里基本完成了

接下来就是测试了

## 1.3.5 实验结果和测试

程序运行的截图

![2017-11-18 12-15-57屏幕截图](/home/dengzhc/图片/2017-11-18 12-15-57屏幕截图.png)

![2017-11-18 12-16-19屏幕截图](/home/dengzhc/图片/2017-11-18 12-16-19屏幕截图.png)

![2017-11-18 12-18-25屏幕截图](/home/dengzhc/图片/2017-11-18 12-18-25屏幕截图.png)

![2017-11-18 12-20-14屏幕截图](/home/dengzhc/图片/2017-11-18 12-20-14屏幕截图.png)



测试结果

![2017-11-18 11-48-00屏幕截图](/home/dengzhc/图片/2017-11-18 11-48-00屏幕截图.png)

![2017-11-18 11-48-52屏幕截图](/home/dengzhc/图片/2017-11-18 11-48-52屏幕截图.png)

![2017-11-18 11-49-15屏幕截图](/home/dengzhc/图片/2017-11-18 11-49-15屏幕截图.png)

自定义的测试样例我保存在/testcases/mytest.xml，并且测试通过了

![2017-11-18 11-53-07屏幕截图](/home/dengzhc/图片/2017-11-18 11-53-07屏幕截图.png)

