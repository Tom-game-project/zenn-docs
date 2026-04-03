---
title: "エラーを合成するライブラリを作った"
emoji: "📌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["rust"]
published: false
---

crates.ioで公開しています

https://crates.io/crates/error-combinator

リポジトリはこちら

https://github.com/Tom-game-project/error-combinator

## これは何か？

Configファイルやフォーム設定が正しい形式であるか、設定の辻褄があっているかというバリデーションをする際のチェッカー作成を助けるライブラリです.

ここでのバリデーションとは、プログラム実行時に与えられたデータを見たとき、それを変更することなくある形式に従っているかどうかをチェックすることとします.

ある一つのチェック項目について満たしているか？を確かめる関数`f`を考えるとき以下のように設計するでしょう.

```rs
fn f(data: T) -> Result<(), E>
```

たくさんチェックする項目があったらどうでしょう.

```rs
fn f0(data: T) -> Result<(), E0> // 返るerrorも若干違う
fn f1(data: T) -> Result<(), E1>
fn f2(data: T) -> Result<(), E2>
fn f3(data: T) -> Result<(), E3>
fn f4(data: T) -> Result<(), E4>
fn f5(data: T) -> Result<(), E5>
fn f6(data: T) -> Result<(), E6>
fn f7(data: T) -> Result<(), E7>
fn f8(data: T) -> Result<(), E8>
fn f9(data: T) -> Result<(), E9>
```

更にこの関数同士の処理として`f0`,`f1`関数が行う処理の結果によっては後続のチェックはできないが`f2`から`f9`の関数はエラーが起こっても処理自体は進めることができて、且つそこで起きたエラーすべてをユーザーに報告したいという複雑なシチュエーションがあるかもしれません。

どうしてもユーザーの入力がindexの意味合いを持ち、範囲チェックをしなければならない場合。その後後続するフォーム入力については各々の入力制約があるが、ユーザーの入力したデータ形式のミスが次の入力項目の形式の判定に何ら影響を及ぼさないなど、バリデーションの要件は以外と複雑になりがちです。

また、ユーザー側も報告された形式のミスについて修正してもう一度実行したら、今度は（一回の実行によって確かめられたはずの）別のミスを指摘されるとストレスとなります。

エラーコンビネータはそんなケースにぴったりです。

## 使い方

error-combinatorではデータがある項目についてチェックされたか否かを型で表現します。
先程の関数`f`のみでそれは表現できていないため`lift`関数を使ってerror-combinator用の新しいチェッカーに格上げします.

```rs
/// fという項目について、まだチェックされていません
struct Unchecked;
/// fという項目について、すでにチェックが完了しています
struct Checked;

fn f(data: T) -> Result<(), E> {
    ...
}

let checker_f = lift::<_,_,Unchecked/* 事前条件 */,Checked/* 事後条件 */,_,_>(f);
```

型でチェック状態を表現しています. これはユーザーが自由に登録できます.ユースケースによっては、チェック前後で状態を変えないというのもありです.

`lift`関数は下のように設計されています.
```rs
pub fn lift<V, T, Pre, Post, E, F>(
    f: F
) -> impl Check<V, Pre, PostState = Post, Error = E>
where
    T: ?Sized,
    V: Borrow<T>,
    F: Fn(&T) -> Result<(), E>,
```

`impl Check<...>`が`checker_f`に束縛されます。

Checkトレイトを実装したもの同士は合成という操作によって新しいCheckトレイトを実装することができます.

合成はCheckトレイト自身のメソッド`.and(checker_f)` `.or(checker_f)`を通じてできます。

```rs
struct BothUnchecked;
struct Checked0;
struct Checked1;

fn f0(data: T) -> Result<(), E0> { ... }
fn f1(data: T) -> Result<(), E1> { ... }

let checker0 = lift::<_,_,BothUnchecked/* 事前条件 */, Checked0/* 事後条件 */,_,_>(f0);
let checker1 = lift::<_,_,Checked0/* 事前条件 */,Checked1/* 事後条件 */,_,_>(f1);

// 合成によって新しいチェッカーを作成
let checker0_and_checker1 = // checker0でErrだったら即終了
    checker0.and(checker1);
let checker0_or_checker1 =  // checker0でErrでもCheck処理を進める
    checker0.or(checker1);

// このままではエラーの合成のためのルールが足りない
```

ただ、なんのヒントもなく関数をよしなに合成することはできません.

２つのチェッカー関数を合成してできた新しいチェッカー関数の返り値は自明ではないからです。

エラーデータの合成の仕方をerror-combinatorにヒントとして伝える必要があります.

```rs
checker0.or<_, Combine /*ここでエラーの合成のやり方をerror-combinatorに教える*/>(checker1)
```

Combineは以下のトレイトを実装している必要があります. 合成の方法とともに、合成後の返り値をCombineErrorがCheckトレイトに教えます.

```rs
// checker0.or<_, Combine>(checker1)
// left   -(処理の進み)->  right

// Combineは以下のトレイトを実装している必要がある


/// ```rs
/// left_checker.or<...>(right_checker)
/// left_checker.and<...>(right_checker)
/// ```
pub trait CombineError<EA, EB> {
    type Out;

    fn new() -> Self;
    /// left_checkerが失敗したときの処理
    fn left(&mut self, ea: EA);
    /// right_checkerが失敗したときの処理
    fn right(&mut self, eb: EB);
    /// ２つの関数の終了したときに返すエラーの構築
    fn finish(self) -> Self::Out;
}

```

軽く実際のCombineを実装してみましょう

```rs
// ここでは、E0、E1という型から構成される新しいエラー表現用の型NewErrorを考える
enum NewError {
    e0(E0),
    e1(E1),
}

struct Combine {
    data: Vec<NewError>
}

impl CombineError<E0, E1> for Combine {
    type Out = Vec<NewError>;

    fn new() -> Self {
        Self {
            data: Vec::new()
        }
    }
    fn left(&mut self, ea: E0) {
        self.data.push(NewError::e0(ea));
    }
    fn right(&mut self, eb: E1) {
        self.data.push(NewError::e1(eb));
    }
    fn finish(self) -> Self::Out {
        self.data
    }
}
```

処理中に見つかったエラーすべてをVecに格納する実装の例です.


## まとめ

このライブラリを作ろうと思ったきっかけはいくつかあります。

一つは、実際にこのようなエラー処理ケースをどうすればうまく扱えるか模索していたということ.もう一つはchumskyというパーサーコンビネータとの出会いがあります.

表現力の高さに驚かされました.特に関心した点は、ユーザーが触るAPI部分に露出したマクロが少ないにもかかわらず、コードの見た目がそのまま文法の定義のように見えることです.





