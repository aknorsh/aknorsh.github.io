+++ 
draft = true
date = 2023-12-17T20:34:02+09:00
title = "(WIP) interface を testify/mock でモック化する"
description = ""
slug = ""
authors = []
tags = ["golang"]
categories = []
externalLink = ""
series = []
+++

[公式 Document](https://pkg.go.dev/github.com/stretchr/testify@v1.8.0/mock)

## TL;DR

[Example usage](https://pkg.go.dev/github.com/stretchr/testify/mock#hdr-Example_Usage) の通りに実装するのみ。以下テンプレ:

```go
/*
*  mockパッケージ内: mockを定義する
*/

// モックがインタフェースを満たしてないときにコンパイルを落とす呪文
var _ hoge.HogeInterface = (*HogeMock)(nil)

type HogeMock struct {
  mock.Mock // mock.Mock型を埋め込む
}

func (m *HogeMock) FugaMethod(s string, num int) (string, error) {
  // Calledメソッドで、「この引数でこのメソッドが呼ばれた」という情報をmockに伝える
  args := m.Called(s, num)

  // Calledが返した値から、後述のReturnメソッドで定義したものを取り出せる (Arguments型)
  return args.String(0), args.Error(1)
}

/*
* テスト側: mockを使用する
*/

func (s *SomeSuite) TestCase1() {
  hogeMock := new(mock.HogeMock)

  // モックの振る舞いを定義する
  hogeMock.On(
    "FugaMethod",
    "someStr1",
    3,
  ).Return(
    "result@example.com",
    nil,
  )

  // 想定通りの呼び出しが行われたか検証する
  hogeMock.AssertExpectations(s.T())
}
```

※ args.Error が golangci-lint の wrapcheck に引っかかるので、package 宣言前に `//nolint:wrapcheck` するといいかもしれない。

## 詳説

mock ライブラリは次の 3 つの型を中心に構成されている。

- Mock 型: すべての基本。モック化したい構造体に埋め込んで使用する。
- Call 型: Mock 型の On メソッドが返す型。メソッドの挙動を定義する。
- Arguments 型: 定義したメソッドの挙動をもとに返り値を管理する型。

### Mock 型

モックの本体。重要なメソッドをピックアップすると以下

- AssertExpectations
  - Call 型で定義した通りの呼び出しがあったか検証するメソッド
- Called
  - モックに特定のメソッドが呼ばれたことを伝達するメソッド。
  - モックに対して対象のメソッドを定義した上で、そのメソッド内で呼ぶという使い方をする。
    - 内部的には MethodCalled のラッパー。モックに該当メソッドを生やしその中から Called を呼ぶことでメソッド名を自動補完した上で MethodCalled を呼んでくれている
  - Arguments 型を返す。そのため、以下の使い方が基本
    1. Called 型で呼び出しがあったことを伝達
    2. 返り値から Return メソッドで定義した返り値を取得
    3. それらを返す
- On
  - Call 型を返す。メソッドの挙動定義を行う。

### Call 型

モックに対して「このメソッドがこういう引数で呼ばれたらこういう値を返してくれ」と定義するやつ。Mock.AssertExpectations で定義した通りに呼ばれたか検証する。

- メソッド

  ```markdown
  # 基本

  - On
    「このメソッドがこういう引数で呼ばれたら」の部分
  - Return
    「こういう値を返してくれ」の部分
  - Panic
    呼び出されたら Panic して欲しいときに使用

  # 呼び出し回数

  - Times
    メソッドが呼び出された回数を Assert する
  - Once, Twice
    Times のエイリアス

  # その他アサーション

  - Maybe
    呼ばれなくてもいいときに使う
  - NotBefore
    メソッドの呼び出し順序を Assert するときに使う

  # その他

  - Unset
    メソッド呼び出し履歴をリセットする。同じテスト内でモックを使い回すときに使用。
  - Run
    callback を渡す。ある意味、最も汎用。
  ```

- Placeholders 機能

  - On の引数が何でも OK なときに使えるメソッド群

    ```go
    m.On(
        "someMethod",
        mock.Anything,              // 引数に何が来てもOK
        mock.AnythingOfType("int"), // int型なら何が来てもOK
        mock.IsType(""),            // string型なら何が来てもOK。 対象の型のゼロ値を渡す。

        // callbackがtrueを返すなら何が来てもOK。下の例はjwt.Token型なら何でもOKを擬似的に表現した例。
        mock.MatchedBy(func(j jwt.Token) bool { return true }),
    )...
    ```

### Arguments 型

Call 型で定義した対象メソッドの返り値を管理する。主な用途はモックに定義した対象メソッドで返り値を用意すること。

- Get メソッド
  - Return 呼び出し時に index 番目に渡した引数を取り出すメソッド。
  - 型に対するアサーションを持たないので、Get を使うなら自前の型アサーションをつけるのが好ましい（定義型を返すときなど）
  - また、対象 index に値がない場合は panic するので nil チェックもつけるとよい。
- Bool, Error, Int, String メソッド
  - Get + 型アサーション。
