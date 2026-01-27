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

## 作った動機

ユーザーのフォーム入力やconfigファイルのチェックに、複数の条件について満たしているかをチェックする機能を実装したいと考えていました。具体的なチェック項目がいくつかあり、複数あるチェック関数はおよそ以下のように設計したいです.

```rs
/* 擬似的な例 */
// targetという項目についてdata(formやconfig)が満たしているかを確認する関数check_target
fn check_target ( data ) -> Result<(), E>
```

入力を変更せずにある項目について満たしているかチェックする、いわゆるバリデーションです.

ところで、フォームを入力しているユーザーにとって、可能であれば、一回でできる限りのcheckはされて欲しいでしょう.

具体的にはフォームが以下のような状態だったときに、

```
first name : empty
last name : empty
```

処理はチェック関数が「first nameはemptyではないか？」「last nameはemptyではないか？」とチェックしていきます。
入力にエラーがある(emptyな入力部分が存在する)場合は詳細に、
「first nameが空です」
「last nameが空です」
「fist name, last nameが空です」
とエラーで拾えたすべての情報を出力したいでしょう.

```rs
/* 擬似的な例 */
// dataにはチェック対象のformが入る
// 特定の形式の引数と戻り値を持ったチェック用関数は自動的にcheckトレイトが実装されます。
// 
fn check_first_name_is_not_empty ( data ) -> Result<(), E1>;
fn check_last_name_is_not_empty ( data ) -> Result<(), E2>;
```

これらのチェックはどちらかが失敗していてもお互い独立したチェックなので、後続するチェックができないという事態にはなりません.
そして、どちらでもerrorが起きた場合にはそれらをすべてキャッチし、その情報をすべて返したいです.

error-combinatorはこのような関数を合成して新しい関数を生成するためのライブラリです.
例えば上のケースでは

```rs
/* 擬似的な例 */
let check_all_item = // 新しい関数を合成します
    check_first_name_is_not_empty
    /*orメソッドは前のチェックが失敗したとしても次のチェックを進めるという意味*/
    .or</*返り値は自動的に算出されないのでそのための実装をします*/>(check_last_name_is_not_empty)

// NEW_EはE1、E2の情報を含む
fn check_all_item ( data ) -> Result<(), NEW_E>
```
のようになります

例のケースは、ライブラリの使用が必要になるほど面倒ではないですが、チェックのロジックが複雑な場合には利点があるかもしれません.

上の合成には、単にチェック関数を合成して新しい関数を作るだけでなく、新しい関数の返り値も設計する必要があります.

合成関数の戻り値となるエラーはcheck関数の合成から自動的に導き出せるものではないからです.


