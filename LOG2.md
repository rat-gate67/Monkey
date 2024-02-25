# 評価
## シンボルに意味を与える

例えば以下のようなプログラムがあったとする。

```
let one = fn() {
	println("one");
	return 1;
};

let two = fn() {
	println("two");
	return 2;
};

add(one(), two());
```

この場合、最初にoneが出るのかtwoが出るのかそれとも逆だろうか。それはインタープリタの設計にかかっている。

## 評価の戦略

抽象木を辿っていきながら実行していくインタプリタは「tree-walking型インタプリタ」と呼ばれる。

事前にASTを辿ってそれをバイトコードに変換するインタプリタもある。
一般的にいえば、オペコードは大半のアセンブリ言語のニーモニックに似ている。
ネイティブの機械語でないのでCPU上では実行できない。
そこでインタプリタの一部である仮想マシンによって解釈される。



この戦略のバリエーションとしてASTを構築せずバイトコードを出力するものもある。

## Tree-Walkingインタプリタ

以下の二つを用意する。

* tree-walking評価器
* ホスト言語であるGoでMonkeyの値を表現する方法

評価器はexval関数一つである。

## オブジェクトを表現する

### オブジェクトシステムの基礎

Monkeyソースコードに出てくる値はすべてObjectで表現する。
すべての値はObjectObjectインターフェイスを満たす必要がある。

```go
// object/object.go
package object

type ObjectType string

type Object interface {
	Type() ObjectType
	Inspect() string
}

```

現時点ではMonkeyインタプリタには3のデータ型しかない。null,真偽値,整数だ。

### 整数

object.Integer型は短くてすむ。

```go
// object/object.go

import "fmt"

type Integer struct {
	Value int64
}

func (i *Integer) Inspect() string { return fmt.Sprintf("%d", i.Value) }
```

ソースコードの中で整数リテラルに出会うたびにそれをast.IntegerLiteralに変換する。そして、そのASTノードを評価する際にobject.Integerへと変換する。

object.Integerがobject.Objectのインターフェイスを満たすには、Type()メソッドも必要。


```go
// object/object.go

const (
	INTEGER_OBJ = "INTEGER"
)

func (i *Integer) Type() ObjectType { return INTEGER_OBJ }
```

### 真偽値

こいつも小さい。

```go
func (i *Integer) Type() ObjectType { return INTEGER_OBJ }
...
type Boolean struct {
	Value bool
}

func (b *Boolean) Type() ObjectType { return BOOLREAN_OBJ }
func (b *Boolean) Inspect() string  { return fmt.Sprintf("%t", b.Value) }


```

### null

「10億ドルの失敗」

```go

	NULL_OBJ    = "NULL"

...

type Null struct{}

func (n *Null) Type() ObjectType { return NULL_OBJ }
func (n *Null) Inspect() string  { return "null" }

```

なんの値もラップしていない。これは値の不存在を表現している。

## 式の評価

eval関数を書いていく。

まずは「5を入力したら5を返す」とか「trueを入力したらtrueを返す」といった簡単なものたち。

### 整数リテラル

「5を入力したら5を返す」は一体何を意味しているのだろうか。
それは、\*ast.IntegerLiteralが与えられた時、Eval関数は*object.Integerを返す必要があるということである。
そのValueフィールドはast\*ast.IntegerLiteral.Valueと同じ整数値を含んでいる。

このテストは簡単に書ける。

```go
// evaluator/evaluator_tes.go

package evaluator

import (
	"Monkey/lexer"
	"Monkey/object"
	"Monkey/parser"
	"testing"
)

func TestEvalIntegerExpression(t *testing.T) {
	tests := []struct {
		input   string
		expected int64
	}{
		{"5", 5},
		{"10", 10},
	}

	for _, tt := range tests {
		evaluated := testEval(tt.input)
		testIntegerObject(t, evaluated, tt.expected)
	}
}

func testEval(input string) {
	l := lexer.New(input)
	p := parser.New(l)
	program := p.ParseProgram()
	return Eval(program)
}

func testIntegerObject(t *testing.T, obj object.Object, expected int64) {
	result, ok := obj.(*object.Integer)
	if !ok {
		t.Errorf("object is not Integer. got=%T (%+v)", obj, obj)
		return false
	}

	if result.Value != expected {
		t.Errorf("object is not Integer. got=%d, want=%d", result.Value, expected)
		return false
	}

	return true
}
```


Eval関数が定義されていないのでエラーになる。

````go
// evaluator/evaluator.go

package evaluator

import (
	"Monkey/ast"
	"Monkey/object"
)

func Eval(node ast.Node) object.Object {
	switch node := node.(type) {
	case *ast.Program:
		return evalStatements(node.Statements)
	case *ast.ExpressionStatement:
		return Eval(node.Expression)
	case *ast.IntegerLiteral:
		return &object.Integer{Value: node.Value}
	}

	return nil
}

func evalStatements(stmt []ast.Statement) object.Object {
	var result object.Object

	for _, statement := range stmt {
		result = Eval(statement)
	}

	return result
}
````

```
=== RUN   TestEvalIntegerExpression
--- PASS: TestEvalIntegerExpression (0.00s)
PASS
ok      Monkey/evaluator        0.352s
```

### REPLを完成させる

REPL(Read, Evaluate, Print, Loop)を完成させる。

```go
// repl/repl.go

package repl

import (
	"Monkey/evaluator"
	...
)

const PROMPT = ">> "

func Start(in io.Reader, out io.Writer) {
	scanner := bufio.NewScanner(in)

	for {
		fmt.Printf(PROMPT)
		scanned := scanner.Scan()
		if !scanned {
			return
		}

		line := scanner.Text()
		l := lexer.New(line)
		p := parser.New(l)

		program := p.ParseProgram()
		if len(p.Errors()) != 0 {
			printParserErrors(out, p.Errors())
			continue
		}

		evaluated := evaluator.Eval(program)
		if evaluated != nil {
			io.WriteString(out, evaluated.Inspect())
			io.WriteString(out, "\n")
		}

		// io.WriteString(out, program.String())
		// io.WriteString(out, "\n")
	}
}
...
```

```
Monkey％go run main/main.go                                                
Hello kubotadaichi! This is the Monkey programming language!
Feel free to type in commands
>> 5
5
>> 10
10
>> 999
999
>> 0
0
>>
 ```

### 真偽値リテラル

真偽値リテラルは整数リテラルと同じように評価するとそれ自身になる。
テストを書く。

```go
// evaluator_test.go
func TestEvalBooleanExpression(t *testing.T) {
	tests := []struct {
		input   string
		expected bool
	}{
		{"true", true},
		{"false", false},
	}

	for _, tt := range tests {
		evaluated := testEval(tt.input)
		testBooleanObject(t, evaluated, tt.expected)
	}
}

func testBooleanObject(t *testing.T, obj object.Object, expected bool) bool {
	result, ok := obj.(*object.Boolean)
	if !ok {
		t.Errorf("object is not Boolean. got=%T (%+v)", obj, obj)
		return false
	}

	if result.Value != expected {
		t.Errorf("object has wrong value. got=%t, want=%t",
			result.Value, expected)
		return false
	}

	return true
}

```

```go
func Eval...

	case *ast.Boolean:
		return &object.Boolean{Value: node.Value}
```

```
=== RUN   TestEvalBooleanExpression
--- PASS: TestEvalBooleanExpression (0.00s)
PASS
ok      Monkey/evaluator        0.285s
```

新しい値を生成するのではなくその参照を使うパターンも考える。


```go
// evaluator/evaluator.go

var (
	TRUE = &object.Boolean{Value: true}
	FALSE = &object.Boolean{Value: false}
)

func Eval(node ast.Node) object.Object {
	switch node := node.(type) {
	case *ast.Program:
		return evalStatements(node.Statements)
	case *ast.ExpressionStatement:
		return Eval(node.Expression)
	case *ast.Boolean:
		// return &object.Boolean{Value: node.Value}
		return nativeBoolToBooleanObject(node.Value)
	case *ast.IntegerLiteral:
		return &object.Integer{Value: node.Value}
	}

	return nil
}

func nativeBoolToBooleanObject(input bool) *object.Boolean {
	if input {
		return TRUE
	}
	return FALSE
}

```

### null

いちいち新しいobject.Nullを生成することはしない。

```go
// evaluator/evaluator
var (
	Null = &object.Null{}
	TRUE = &object.Boolean{Value: true}
	FALSE = &object.Boolean{Value: false}
)

```
### 前置式

Monkeyがサポートする前置式は「!」「-」がある。

気をつけて実装する。

それでは「!」演算子の対応を実装する。
テストはこの演算子がオペランドを真偽値に変換してその否定を返すことを期待している。

```go
// evaluator_test.go

func TestBangOperator(t *testing.T) {
	tests := []struct {
		input    string
		expected bool
	}{
		{"!true", false},
		{"!false", true},
		{"!5", false},
		{"!!true", true},
		{"!!false", false},
		{"!!5", true},
	}

	for _, tt := range tests {
		evaluated := testEval(tt.input)
		testBooleanObject(t, evaluated, tt.expected)
	}
}

```

Evalはnilを返すためテストは失敗する。
```go
// evaluator.go

func Eval(node ast.Node) object.Object {
...
	case *ast.PrefixExpression:
		right := Eval(node.Right)
		return evalPrefixExpression(node.Operator, right)
...


func evalPrefixExpression(operator string, right object.Object) object.Object {
	switch operator {
	case "!":
		return evalBangOperatorExpression(right)
	default:
		return NULL
	}
}

func evalBangOperatorExpression(right object.Object) object.Object {
	switch right {
	case TRUE:
		return FALSE
	case FALSE:
		return TRUE
	case NULL:
		return NULL
	default:
		return FALSE
	}
}
```

```
=== RUN   TestBangOperator
--- PASS: TestBangOperator (0.00s)
PASS
ok      Monkey/evaluator        0.290s

```

続いて「-」を対応する。
これは「!」の時を拡張して作ることができる。

```go
// evaluator_test.go
func TestMinusOperator(t *testing.T) {
	tests := []struct {
		input    string
		expected int64
	}{
		{"5", 5},
		{"10", 10},
		{"-5", -5},
		{"-10", -10},
	}
...

```

```go
// evaluator.go

func evalPrefixExpression(operator string, right object.Object) object.Object {
	switch operator {
	case "!":
		return evalBangOperatorExpression(right)
	case "-":
		return evalMinusPrefixOperatorExpression(right)
	default:
		return NULL
	}
}

func evalMinusPrefixOperatorExpression(right object.Object) object.Object {
	if right.Type() != object.INTEGER_OBJ {
		return NULL
	}
	value := right.(*object.Integer).Value
	return &object.Integer{Value: -value}
}
```

```
=== RUN   TestMinusOperator
--- PASS: TestMinusOperator (0.00s)
PASS
ok      Monkey/evaluator        0.273s
```

REPLで前置式を試すことができるようになった。

```
Monkey％go run main/main.go
Hello kubotadaichi! This is the Monkey programming language!
Feel free to type in commands
>> !true
false
>> -10
-10
>> !44 
false
>> !0
false
>> !!!false
true
>> 
```

### 中置式

Monkeyがサポートするのは以下の8つの式である。

```
5 + 5;
5 - 5;
5 * 5;
5 / 5;

5 > 5;
5 < 5;
5 >= 5;
5 <= 5;
```
まずは真偽値を生成しない上の４つの式から対応する。

テストはTestEvalIntegerExpressionテスト関数を拡張したものを使用する。
```go
// evaluatir_test.go

func TestEvalIntegerExpression(t *testing.T) {
	tests := []struct {
		input    string
		expected int64
	}{
		{"5", 5},
		{"10", 10},
		{"5 + 5 + 5 + 5 - 10", 10},
		{"3 + 3 - 2 -2 - 1", 1},
		{"5 * 2", 10},
		{"12 / 2", 6},
		{"50 / 2 * 2 + 10 - 5", 55},
		{"5 * 2 + 10", 20},
		{"5 + 2 * 10", 25},
		{"5 * (2 + 10)", 60},
		{"10-5", 5},
	}
...

```

まずEval関数のswtichを拡張するところから始める。
```go
// evaluator.go

...
	case *ast.InfixExpression:
		left := Eval(node.Left)
		right := Eval(node.Right)
		return evalInfixExpression(node.Operator, left, right)
...


func evalInfixExpression(
	operator string,
	left, right object.Object,
) object.Object {
	switch {
	case left.Type() == object.INTEGER_OBJ && right.Type() == object.INTEGER_OBJ:
		return evalIntegerInfixExpression(operator, left, right)
	default:
		return NULL
	}
}

func evalIntegerInfixExpression(
	operator string,
	left, right object.Object,
) object.Object {
	leftVal := left.(*object.Integer).Value
	rightVal := right.(*object.Integer).Value

	switch operator {
	case "+":
		return &object.Integer{Value: leftVal + rightVal}
	case "-":
		return &object.Integer{Value: leftVal - rightVal}
	case "*":
		return &object.Integer{Value: leftVal * rightVal}
	case "/":
		return &object.Integer{Value: leftVal / rightVal}
	default:
		return NULL
	}
}
```

```

=== RUN   TestEvalIntegerExpression
--- PASS: TestEvalIntegerExpression (0.00s)
PASS
ok      Monkey/evaluator        0.645s

```
これだけでテストが通る。

続けて真偽値をかえす演算子を追加する。

```go
// evaluator_test.go
func TestEvalBooleanExpression(t *testing.T) {
	tests := []struct {
		input    string
		expected bool
	}{
		{"true", true},
		{"false", false},
		{"1 < 2", true},
		{"1 > 2", false},
		{"1 == 1", true},
		{"1 != 1", true},
		{"1 == 2", false},
		{"1 != 2", true},
	// 	{"true == true", true},
	// 	{"true != true", false},
	// 	{"true == false", false},
	}
...
```

数行を追加するだけでテストは通る。
```go
// evaluator.go

func evalIntegerInfixExpression(
	operator string,
	left, right object.Object,
) object.Object {
	leftVal := left.(*object.Integer).Value
	rightVal := right.(*object.Integer).Value

	switch operator {
	case "+":
		return &object.Integer{Value: leftVal + rightVal}
	case "-":
		return &object.Integer{Value: leftVal - rightVal}
	case "*":
		return &object.Integer{Value: leftVal * rightVal}
	case "/":
		return &object.Integer{Value: leftVal / rightVal}
	case "<":
		return nativeBoolToBooleanObject(leftVal < rightVal)
	case ">":
		return nativeBoolToBooleanObject(leftVal > rightVal)
	case "==":
		return nativeBoolToBooleanObject(leftVal == rightVal)
	case "!=":
		return nativeBoolToBooleanObject(leftVal != rightVal)
	default:
		return NULL
	}
}
```

```
=== RUN   TestEvalBooleanExpression
--- PASS: TestEvalBooleanExpression (0.00s)
PASS
ok      Monkey/evaluator        0.288s
````

より真偽値についても演算ができるようにする.

```go
...
	{
		{"true", true},
		{"false", false},
		{"1 < 2", true},
		{"1 > 2", false},
		{"1 == 1", true},
		{"1 != 1", false},
		{"1 == 2", false},
		{"1 != 2", true},
		{"true == true", true},
		{"true != true", false},
		{"true == false", false},
		{"(1 > 0) != true", false},
		{"(1 > 0) == true", true},
		{"(1 > 0) == false", false},
		{"(1 > 0) != false", true},
	}
...
```

これはNULLが出て失敗する。

```
=== RUN   TestEvalBooleanExpression
    /Users/kubotadaichi/Desktop/PLP/Monkey/evaluator/evaluator_test.go:77: object is not Boolean. got=*object.Null (&{})
    /Users/kubotadaichi/Desktop/PLP/Monkey/evaluator/evaluator_test.go:77: object is not Boolean. got=*object.Null (&{})
    /Users/kubotadaichi/Desktop/PLP/Monkey/evaluator/evaluator_test.go:77: object is not Boolean. got=*object.Null (&{})
    /Users/kubotadaichi/Desktop/PLP/Monkey/evaluator/evaluator_test.go:77: object is not Boolean. got=*object.Null (&{})
    /Users/kubotadaichi/Desktop/PLP/Monkey/evaluator/evaluator_test.go:77: object is not Boolean. got=*object.Null (&{})
    /Users/kubotadaichi/Desktop/PLP/Monkey/evaluator/evaluator_test.go:77: object is not Boolean. got=*object.Null (&{})
    /Users/kubotadaichi/Desktop/PLP/Monkey/evaluator/evaluator_test.go:77: object is not Boolean. got=*object.Null (&{})
--- FAIL: TestEvalBooleanExpression (0.00s)
FAIL
FAIL    Monkey/evaluator        0.287s

```
これは以下のようにすれば直ちにテストが通ってしまう。

```go
// evaluator.go
func evalInfixExpression(
	operator string,
	left, right object.Object,
) object.Object {
	switch {
	case left.Type() == object.INTEGER_OBJ && right.Type() == object.INTEGER_OBJ:
		return evalIntegerInfixExpression(operator, left, right)
	case operator == "==":
		return nativeBoolToBooleanObject(left == right)
	case operator == "!=":
		return nativeBoolToBooleanObject(left != right)
	default:
		return NULL
	}
}
```

```
=== RUN   TestEvalBooleanExpression
--- PASS: TestEvalBooleanExpression (0.00s)
PASS
ok      Monkey/evaluator        0.117s
```
一応、電卓が完成した。
```
Monkey％go run main/main.go
Hello kubotadaichi! This is the Monkey programming language!
Feel free to type in commands
>> 1 + 0 -1
0
>> 100 > 10*10
false
>> 700 > (100+40)
true
>> (2 > 3 + 2)
false
>> true + 1
null
>> 2 > 3+2
false
>> 2 > 5
false
>> 100 * 2 > 500 /100
true
```

## 条件分岐

Monkeyでは条件分岐のconsequence部の条件が「truthy」つまりnullでもfalseでもない時に実行される。
テストを書いていく。

