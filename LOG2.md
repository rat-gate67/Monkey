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