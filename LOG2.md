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

```go
// evaluator_test.go
func TestIfElseExpressions(t *testing.T) {
	tests := []struct {
		input    string
		expect	 interface{}
	}{
		{"if (true) { 10 }", 10},
		{"if (false) { 10 }", nil},
		{"if (1>2) { 10 } else { 20 }", 20},
	}

	for _, tt := range tests {
		evaluated := testEval(tt.input)
		integer, ok := tt.expect.(int)
		if ok {
			testIntegerObject(t, evaluated, int64(integer))
		} else {
			testNullObject(t, evaluated)
		}
	
	}
}

func testNullObject(t *testing.T, obj object.Object) bool {
	if obj != NULL {
		t.Errorf("object is not NULL. got=%T (%+v)", obj, obj)
		return false
	}

	return true
}

```

例えば
```if (false) { 10 }
```
は何も値を返さないのでNULLになるという処理もはいっている。

```
=== RUN   TestIfElseExpressions
    /Users/kubotadaichi/Desktop/PLP/Monkey/evaluator/evaluator_test.go:146: object is not Integer. got=<nil> (<nil>)
    /Users/kubotadaichi/Desktop/PLP/Monkey/evaluator/evaluator_test.go:160: object is not NULL. got=<nil> (<nil>)
    /Users/kubotadaichi/Desktop/PLP/Monkey/evaluator/evaluator_test.go:146: object is not Integer. got=<nil> (<nil>)
--- FAIL: TestIfElseExpressions (0.00s)
FAIL
FAIL    Monkey/evaluator        0.268s
```

わずかなコードを追加してテストを成功させる。

```go
// evaluator,go

...
	case *ast.BlockStatement:
		return evalStatements(node.Statements)
	case *ast.IfExpression:
		return evalIfExpression(node)
...

func evalIfExpression(ie *ast.IfExpression) object.Object {
	condition := Eval(ie.Condition)

	if isTruthy(condition) {
		return Eval(ie.Consequence)
	}else if ie.Alternative != nil {
		return Eval(ie.Alternative)
	} else {
		return NULL
	}
}

func isTruthy(obj object.Object) bool {
	switch obj {
	case NULL:
		return false
	case TRUE:
		return true
	case FALSE:
		return false
	default:
		return true
	}
}

```

```
=== RUN   TestIfElseExpressions
--- PASS: TestIfElseExpressions (0.00s)
PASS
ok      Monkey/evaluator        0.283s
```

## return文

return文に遭遇するたびに、変えるべき値をあるオブジェクロの内側にラップしておいて追跡する方法を取る。

そのオブジェクトを作成する。

```go
// object/object.go


const (
	INTEGER_OBJ = "INTEGER"
	BOOLEAN_OBJ = "BOOLEAN"
	NULL_OBJ    = "NULL"
	RETURN_VALUE_OBJ = "RETURN_VALUE"
)

...


type ReturnValue struct {
	Value Object
}

func (rv *ReturnValue) Type() ObjectType { return RETURN_VALUE_OBJ }
func (rv *ReturnValue) Inspect() string  { return rv.Value.Inspect() }

```

テストは次のようなものを用意する。

```go
// evaluator_test.go

func TestReturnStatements(t *testing.T) {
	tests := []struct {
		input  string
		expect int64
	}{
		{"return 10;", 10},
		{"return 10;9;", 10},
		{"9; return 10; 8;", 10},
	}

	for _, tt := range tests {
		evaluated := testEval(tt.input)
		testIntegerObject(t, evaluated, tt.expect)
	}
}
```

これらのテストを通すようにするには、既存のevalStatement関数を変更し\*ast.ReturnStatementのためのcase分岐をEvalに追加しなければいけない。

```go
...
	case *ast.ReturnStatement:
		val := Eval(node.ReturnValue)
		return &object.ReturnValue{Value: val}
...

```

```
=== RUN   TestReturnStatements
--- PASS: TestReturnStatements (0.00s)
PASS
ok      Monkey/evaluator        0.363s
```



evalStatementsはevalProgramStatementsとevalBlockStatementsで一連の文を表現するのに使う。
しかし以下のようなテストでは問題がある。

```go
// evaluator_test.go

func TestReturnStatements(t *testing.T) {
	tests := []struct {
		input  string
		expect int64
	}{
		{`
		if (10 > 1) {
			if (10 > 1) {
				return 10;
			}
		}

		return 1;
		`, 10},
	}

	for _, tt := range tests {
		evaluated := testEval(tt.input)
		testIntegerObject(t, evaluated, tt.expect)
	}
}

```

```
=== RUN   TestReturnStatements
    /Users/kubotadaichi/Desktop/PLP/Monkey/evaluator/evaluator_test.go:173: object is not Integer. got=1, want=10
--- FAIL: TestReturnStatements (0.00s)
FAIL
FAIL    Monkey/evaluator        0.269s
```

以下のような修正を加える。

```go
// evaluator.go

func Eval(node ast.Node) object.Object {
	switch node := node.(type) {
	case *ast.Program:
		return evalProgram(node.Statements)
...
	case *ast.BlockStatement:
		return evalBlockStatement(node)
...
func evalProgram(program *ast.Program) object.Object {
	var result object.Object

	for _, statement := range program.Statements {
		result = Eval(statement)

		if returnValue, ok := result.(*object.ReturnValue); ok {
			return returnValue.Value;
		}
	}

	return result
}

func evalBlockStatement(block *ast.BlockStatement) object.Object {
	var result

	for _, statement := range block.Statements {
		result = Eval(statement)

		if result != nil && result.Type() == object.RETURN_VALUE_OBJ {
			return result
		}
	}

	return result
}
```

```
=== RUN   TestReturnStatements
--- PASS: TestReturnStatements (0.00s)
PASS
ok      Monkey/evaluator        0.283s
```

## エラー処理

エラオブジェクトを作成する。


```go
// objext.go
const (
	INTEGER_OBJ      = "INTEGER"
	BOOLEAN_OBJ      = "BOOLEAN"
	NULL_OBJ         = "NULL"
	RETURN_VALUE_OBJ = "RETURN_VALUE"
	ERROR_OBJ		= "ERROR"
)

type Error struct {
	Message string
}

func (e *Error) Type() ObjectType { return ERROR_OBJ }
func (e *Error) Inspect() string  { return "ERROR: " + e.Message }
```

テストを用意する。

```go
// evlutor_test.go


func TestErrorHandling(t *testing.T) {
	tests := []struct {
		input           string
		expectedMessage string
	}{
		{
			"5 + true;",
			"type mismatch: INTEGER + BOOLEAN",
		},
		{
			"5 + true; 5;",
			"type mismatch: INTEGER + BOOLEAN",
		},
		{
			"-true",
			"unknown operator: -BOOLEAN",
		},
		{
			"true + false;",
			"unknown operator: BOOLEAN + BOOLEAN",
		},
		{
			"true + false + true + false;",
			"unknown operator: BOOLEAN + BOOLEAN",
		},
		{
			"5; true + false; 5",
			"unknown operator: BOOLEAN + BOOLEAN",
		},
		{
			"if (10 > 1) { true + false; }",
			"unknown operator: BOOLEAN + BOOLEAN",
		},
// 		{
// 			`
// if (10 > 1) {
//   if (10 > 1) {
//     return true + false;
//   }

//   return 1;
// }
// `,
// 			"unknown operator: BOOLEAN + BOOLEAN",
// 		},
// 		{
// 			"foobar",
// 			"identifier not found: foobar",
// 		},
	}

	for _, tt := range tests {
		evaluated := testEval(tt.input)

		errObj, ok := evaluated.(*object.Error)
		if !ok {
			t.Errorf("no error object returned. got=%T(%+v)",
				evaluated, evaluated)
			continue
		}

		if errObj.Message != tt.expectedMessage {
			t.Errorf("wrong error message. expected=%q, got=%q",
				tt.expectedMessage, errObj.Message)
		}
	}
}

```
```
=== RUN   TestErrorHandling
    /Users/kubotadaichi/Desktop/PLP/Monkey/evaluator/evaluator_test.go:245: no error object returned. got=*object.Null(&{})
    /Users/kubotadaichi/Desktop/PLP/Monkey/evaluator/evaluator_test.go:245: no error object returned. got=*object.Integer(&{Value:5})
    /Users/kubotadaichi/Desktop/PLP/Monkey/evaluator/evaluator_test.go:245: no error object returned. got=*object.Null(&{})
    /Users/kubotadaichi/Desktop/PLP/Monkey/evaluator/evaluator_test.go:245: no error object returned. got=*object.Null(&{})
    /Users/kubotadaichi/Desktop/PLP/Monkey/evaluator/evaluator_test.go:245: no error object returned. got=*object.Null(&{})
    /Users/kubotadaichi/Desktop/PLP/Monkey/evaluator/evaluator_test.go:245: no error object returned. got=*object.Integer(&{Value:5})
    /Users/kubotadaichi/Desktop/PLP/Monkey/evaluator/evaluator_test.go:245: no error object returned. got=*object.Null(&{})
--- FAIL: TestErrorHandling (0.00s)
FAIL
FAIL    Monkey/evaluator        0.293s


```

ところどころにあるIntegerは演算途中でエラーが発生しているということになっている。
エラーを作成してEvalで実行するためにヘルパー関数を用意して返す。

```go
// evaluator.go

import (
	"Monkey/ast"
	"Monkey/object"
	"fmt"
)
...
func newError(format string, a ...interface{}) *object.Error {
	return &object.Error{Message: fmt.Sprintf(format, a...)}
}

```

このnewエラー関数はこれまでどうするべきかわからずNULLを返していたすべての箇所でその代わりに使える.
```go
// evaluator.go


func evalInfixExpression(
	operator string,
	left, right object.Object,
) object.Object {
	switch {
...
	case left.Type() != right.Type():
		return newError("type mismatch; %s %s %s",
			left.Type(), operator, right.Type())
	default:
		return newError("unknown operator: %s %s %s",
left.Type(), operator, right.Type())




func evalIntegerInfixExpression(
	operator string,
	left, right object.Object,
) object.Object {
	leftVal := left.(*object.Integer).Value
	rightVal := right.(*object.Integer).Value

	switch operator {
...
	default:
		return newError("unknow operator: %s %s %s",
			left.Type(), operator, right.Type())
	}
}


func evalPrefixExpression(operator string, right object.Object) object.Object {
	switch operator {
	case "!":
		return evalBangOperatorExpression(right)
	case "-":
		return evalMinusPrefixOperatorExpression(right)
	default:
		return newError("unknown operator: %s %s",operator, right.Type())
	}
}


func evalMinusPrefixOperatorExpression(right object.Object) object.Object {
	if right.Type() != object.INTEGER_OBJ {
		return newError("unknown operator: -%s", right.Type())
	}

...

```

```

=== RUN   TestErrorHandling
    /Users/kubotadaichi/Desktop/PLP/Monkey/evaluator/evaluator_test.go:246: no error object returned. got=*object.Integer(&{Value:5})
    /Users/kubotadaichi/Desktop/PLP/Monkey/evaluator/evaluator_test.go:252: wrong error message. expected="unknown operator: BOOLEAN + BOOLEAN", got="type mismatch: ERROR + BOOLEAN"
    /Users/kubotadaichi/Desktop/PLP/Monkey/evaluator/evaluator_test.go:246: no error object returned. got=*object.Integer(&{Value:5})
--- FAIL: TestErrorHandling (0.00s)
FAIL
FAIL    Monkey/evaluator        0.113s
```

これらはまだエラーで処理が停止していないことを示している。evalProgramとevalBloackStatementにエラー処理を追加する。

```go
// evaluator.go

func evalProgram(program *ast.Program) object.Object {
	var result object.Object

	for _, statement := range program.Statements {
		result = Eval(statement)

		switch result := result.(type) {
		case *object.ReturnValue:
			return result.Value
		case *object.Error:
			return result
		}
	}

	return result
}


func evalBlockStatement(block *ast.BlockStatement) object.Object {
	var result object.Object

	for _, statement := range block.Statements {
		result = Eval(statement)

		if result != nil {
			rt := result.Type()
			if rt == object.RETURN_VALUE_OBJ || rt == object.ERROR_OBJ {
				return result
			}
		}
	}

	return result
}

```
å
```
=== RUN   TestErrorHandling
    /Users/kubotadaichi/Desktop/PLP/Monkey/evaluator/evaluator_test.go:252: wrong error message. expected="unknown operator: BOOLEAN + BOOLEAN", got="type mismatch: ERROR + BOOLEAN"
--- FAIL: TestErrorHandling (0.00s)
FAIL
FAIL    Monkey/evaluator        0.292s


```

最後にもう一つ。Evalの中でEvalを呼び出す際には常にエラーをチェックしてなければならない。

```go

func Eval(node ast.Node) object.Object {
	switch node := node.(type) {
	case *ast.Program:
		return evalProgram(node)
	case *ast.ExpressionStatement:
		return Eval(node.Expression)
	case *ast.PrefixExpression:
		right := Eval(node.Right)
		if isError(right) {
			return right
		}
		return evalPrefixExpression(node.Operator, right)
	case *ast.Boolean:
		// return &object.Boolean{Value: node.Value}
		return nativeBoolToBooleanObject(node.Value)
	case *ast.InfixExpression:
		left := Eval(node.Left)
		if isError(left) {
			return left
		}
		right := Eval(node.Right)
		if isError(right) {
			return right
		}
		return evalInfixExpression(node.Operator, left, right)
	case *ast.BlockStatement:
		return evalBlockStatement(node)
	case *ast.IfExpression:
		return evalIfExpression(node)
	case *ast.ReturnStatement:
		val := Eval(node.ReturnValue)
		if isError(val) {
			return val
		}
		return &object.ReturnValue{Value: val}
	case *ast.IntegerLiteral:
		return &object.Integer{Value: node.Value}
	}

	return nil
}


func evalIfExpression(ie *ast.IfExpression) object.Object {
	condition := Eval(ie.Condition)

	if isError(condition) {
		return condition
	}
	
...
}
```

```

=== RUN   TestErrorHandling
--- PASS: TestErrorHandling (0.00s)
PASS
ok      Monkey/evaluator        0.301s

```

## 束縛と環境

let文を追加して変数束縛を追加する。

```go
//evaluator_test.go
func TestLetStatements(t *testing.T) {
	tests := []struct {
		input    string
		expected int64
	}{
		{"let a = 5; a;", 5},
		{"let a = 5 * 5; a;", 25},
		{"let a = 5; let b = a; b;", 5},
		{"let a = 5; let b = a; let c = a + b + 5; c;", 15},
	}

	for _, tt := range tests {
		testIntegerObject(t, testEval(tt.input), tt.expected)
	}
}

```

それに識別子が束縛されていない時のエラーも検証する。

```go
// evaluator.go

func TestErrorHandling(t *testing.T) {
	tests := []struct {
		input           string
		expectedMessage string
	}{
...
				{
					"foobar",
					"identifier not found: foobar",
				},
	}
...
```

まず*ast.LetStatementのためのcase分岐をEvalに作る。

```go

	case *ast.LetStatement:
		val := Eval(node.Value)
		if isError(val) {
			return val
		}
		// 何すりゃいいんだこれ
		
```

ここで環境が登場。
環境はね前に関連づけられた値を記録しておくために使われるもので、その本汁は文字列とオブジェクトを関連づけたハッシュマップである。

環境関連のオブジェクトを追加する。

```go
// object.go


func NewEnvironment() *Environment {
	s := make(map[string]Object)
	return &Environment{store: s}
}

type Environment struct {
	store map[string]Object
}

func (e *Environment) Get(name string) (Object, bool) {
	obj, ok := e.store[name]
	return obj, ok
}

func (e *Environment) Set(name string, val Object) Object {
	e.store[name] = val
	return val
}

```

全てのEvalの引数にenvを追加する。

REPLの中でもenvを使う。

```go
// repl.go

	"Monkey/evaluator"
	"Monkey/lexer"
	"Monkey/parser"
	"bufio"
	"fmt"
	"io"
	"Monkey/object"
)


const PROMPT = ">> "

func Start(in io.Reader, out io.Writer) {
	scanner := bufio.NewScanner(in)
	env := object.NewEnvironment()

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

		evaluated := evaluator.Eval(program, env)
		if evaluated != nil {
			io.WriteString(out, evaluated.Inspect())
			io.WriteString(out, "\n")
		}

		// io.WriteString(out, program.String())
		// io.WriteString(out, "\n")
	}
}


```

テストのためにEvalが呼ばれるてびに環境が初期化されるように関数を定義する。

```go
// evaluator.go

func TestEval(input string) object.Object {
	l := lexer.New(input)
	p := parser.New(l)
	program := p.ParseProgram()
	env := object.NewEnvironment()

	return Eval(program, env)
}

```

LeteStatementのcaseに、名前と値を現在の環境に保存する処理を書く。
```go
// evaluator.go

case *ast.LetStatement:
		val := Eval(node.Value, env)
		if isError(val) {
			return val
		}
		// 何すりゃいいんだこれ
		env.Set(node.Name.Value, val)
```

そして識別子を表あする時にはこれらの値を取り出す必要もある。

```go
// evaluator.go
	case *ast.Identifier:
		return evalIdentifier(node, env)

...
func evalIdentifier(
	node *ast.Identifier,
	env *object.Environment,
) object.Object {
	val, ok := env.Get(node.Value)
	if !ok {
		return newError("identifier not found: " + node.Value)
	}

	return val
}

```

```
Monkey％go test ./evaluator
ok      Monkey/evaluator        0.306s
```

```
Monkey％go run main/main.go                      
Hello kubotadaichi! This is the Monkey programming language!
Feel free to type in commands
>> let a = 100
>> a
100
>> let b = a
>> b
100
>>
```

