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

