---
title: "Golang のmock 処理を真似したら型判定が通らなかったので謎を解明する"
emoji: "👋"
type: "tech" # tech: 技術記事 / 
topics: [Go]
published: true
---

# Golang のテストを書く時に、mock を実装したい
自分は基本的にAPIサーバのコードを書く場合、依存先をinterface にして、DIすることでテストが作成しやすい構造を意識するようにしています\
その場合、テストコードの作成時に依存先メソッドの振る舞いを定義するためにmock を作成する必要が出てきます\
今回はGolang を使っていたため、勉強を兼ねて自前でのmock を作成してみました\
初めてmock 自体を作成するため、検索をかけて出てきた記事を参考にし、構造をそのまま作成してみたところ型判定が通らず苦戦したため、記録を残します。


# 結論
SelectedOption 構造体のフィールドがプライベートだった\
ものすごい初歩的なミスでした。\

## Qiita の記事を参考にして実装をおこなってみた

適当なinterface とその実装を作成
```go:calc.go
package calc

type Calculate interface {
	Sum(first, second int) int
	Delta(first, second int) int
}

type SelectedOption struct {
	calc Calculate
}

func (s SelectedOption) Sum(first, second int) int {
	return first + second
}

func (s SelectedOption) Delta(first, second int) int {
	return first - second
}

func NewSelectedOption(calc Calculate) *SelectedOption {
	return &SelectedOption{calc: calc}
}
```

テストファイルを作成
```go:calc_test.go
package test

import (...)

type Mock struct {
}

// interface で定義した関数を実装した
func (m *Mock) Sum(first, second int) int {
	return 999999999
}

func (m *Mock) Delta(first, second int) int {
	return -999999999
}

func TestSum(t *testing.T) {
	target := &calc.SelectedOption{&Mock{}} // なぜか注入できない
}
```
## なぜかMock 構造体を注入したSelectedOption 構造体を作成できない
記事と同じ構成のコードを作成したのに、SelectedOption 構造体を作成することができませんでした

### New 関数を作成してみる
calc.go, calc_test.go にそれぞれ構造体のポインタ型を作成して返却する関数を作成

```go:calc.go
// 追加
func NewSelectedOption(calc Calculate) *SelectedOption {
	return &SelectedOption{calc: calc}
}
```

```go:calc_test.go

// 追加
func NewMock() calc.Calculate {
	return &Mock{}
}

func TestSum(t *testing.T) {
	target := &calc.SelectedOption{&Mock{} // なぜか注入できない

	target := calc.NewSelectedOption(NewMock()) // 注入できる
	
	...
}
```

# 原因
SelectedOption 構造体のcalc フィールドがプライベートだった
```diff go:calc.go
type SelectedOption struct {
	- calc Calculate
	+ Calc Calculate // calc → Calc に変更
}
```
上記の変更をすることで注入できなかったコードが実行できるようになりました。


# 結論
**SelectedOption 構造体のcalc フィールドがプライベートだった**
今回はパッケージを分割しての実装をおこなったため、構造体のフィールドをプライベートで記述したために外部からの値代入を行うことができませんでした\

もちろん言語仕様通りの挙動で、かつ非常に初歩的な点ではあるのですが、技術記事のコードを鵜呑みにしてコピペすることの危うさを学ぶことができました。
