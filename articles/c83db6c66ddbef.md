## きっかけ

[sedlisp](https://github.com/shinh/sedlisp)というプロジェクトとの出会いをきっかけに、私もSedというランタイム上で動く何かを作ってみたいと考えました。
以下、コミットログを振り返りつつ、個人プロジェクト`sed-compiler`の開発を自分で振り返って行きます。

https://github.com/Tom-game-project/sed-compiler

## init commit: [b0daa87](https://github.com/Tom-game-project/sed-compiler/commit/b0daa87)..

もともとこのリポジトリは、Sedを練習する目的で作られたものでした。リポジトリが作成された9月頃は、私自身まだSedの文法すらあまりわかっていないという状況だったので、お試し感覚で加算機を作ってみたのが最初のコミットです。


リポジトリをCloneしている場合は以下のようにして試すことができます

```sh
# ２つの二進数を標準入力で流す
echo 10101 1110 | sed "$(git show  b0daa87:add.sed)"
```

ちなみにSedは、加減乗除、余りの計算はもちろん、関数や変数などもありません。その代わりに、ジャンプラベル、HoldスペースPatternスペースのような特徴的な機能があります。今回の試みはそんなSedの機能を使って一般的なプログラミング言語がデフォルトでサポートするような機能を実現してみようというモチベーションでやっていきます。

### 最低限のSedの知識

- ジャンプラベル - sedにおける制御構文

if節や関数の実装、更にはループ処理などで重要になります。

```sed
: label <-+
...       |
b label --+
```

- HoldスペースPatternスペース - sedにおけるメモリ

今回は、関数のスタック（ローカル変数を保存する領域）の実現のために使います。

```
+-------------------+
|   pattern space   | <= メインの作業スペース
+-------------------+
        ^
        | x,h,H,g,Gなどのコマンドでスイッチしたりコピーしたり追加したり...
        v
+-------------------+
|     hold space    | <= 作業内容を一旦保存する場所
+-------------------+
```

## commit: [b0daa87](https://github.com/Tom-game-project/sed-compiler/commit/b0daa87)..[a766d81](https://github.com/Tom-game-project/sed-compiler/commit/a766d81)

Sedで関数の再帰的な呼び出しを実装するための試行錯誤をしていました。

この段階でおよそ以下のような方針でスタックフレームを構築することにしました。

![](https://storage.googleapis.com/zenn-user-upload/ba1a8858d943-20251118.webp)

上の図は、りあるわーるどの"普通の"スタックフレームと今回作成したSed上でのスタックフレームを比較したものになります。対応する概念は同じ色で表現されています。

より詳しく知りたい場合は、以下の本のpwnの章を読むと良いと思います。CTF自体に興味がなくても、mallocの仕組みや、スタックフレームがどうやって構築されるかとかの興味深い知識を得ることができます。

https://book.mynavi.jp/ec/products/detail/id=122750

上の画像内で定義したようなデータ構造をもとに、PatternスペースとHoldスペースをうまい具合に使って関数内の引数、変数を管理していきます。

a766d81:labo.sedは、以下のようなプログラムをSedで表現しています。

```py
def banana_maker(a):
    if a == "ba":
        return banana_maker("bana")
    elif a == "bana":
        return banana_maker("banana")
    elif a == "banana":
        return a
    else:
        pass

banana_maker("ba")
```

リポジトリをcloneしている場合は以下のコマンドで試すことができます

```sh
# 標準入力にテキトーな文字を入れると実行できます
echo "hello" | sed "$(git show a766d81:labo.sed)"
```

---

ここまででできたこと、

- 加算の実装
- スタックフレームの構築


## commit: [a766d81](https://github.com/Tom-game-project/sed-compiler/commit/a766d81)..[00b132a](https://github.com/Tom-game-project/sed-compiler/commit/00b132a)

関数を構築しようと思うたびに自分でスタックフレームを作るのはしんどいので、自動で行えるようにRustプログラムを書き始めました。
ここで、Sedより一層高級なレイヤーのIR(中間表現)を作るという方針にしました。ここらへんから、Sedの基本を学ぶというよりネタコンパイラのプロジェクトになってきます。

中間表現のデザインはWasm Text Format(WAT)を参考に、全部スタックといった感じにしています。

以下は掛け算の実装のIRです。

```rust
    let mut func_mul = FuncDef::new("mul", 2, 1, 1);
    func_mul.set_proc_contents(vec![
        SedInstruction::Val(Value::Arg(1)),
        SedInstruction::Call(CallFunc::new("is_empty")),
        SedInstruction::IfProc(IfProc::new(
            vec![
                SedInstruction::ConstVal(ConstVal::new("0")),
                SedInstruction::Set(Value::Local(0)), // rstr
            ],
            vec![
                SedInstruction::Val(Value::Arg(1)),
                SedInstruction::Call(CallFunc::new("ends_with_zero")),
                SedInstruction::IfProc(IfProc::new(
                    vec![
                        // rstr = mul(shift_left1(a), shift_right1(b))
                        SedInstruction::Val(Value::Arg(0)), // a
                        SedInstruction::Call(CallFunc::new("shift_left1")),
                        SedInstruction::Val(Value::Arg(1)), // b
                        SedInstruction::Call(CallFunc::new("shift_right1")),
                        SedInstruction::Call(CallFunc::new("mul")),
                        SedInstruction::Set(Value::Local(0)), // rstr
                    ],
                    vec![
                        // rstr = add(a, mul(shift_left1(a), shift_right1(b)))
                        SedInstruction::Val(Value::Arg(0)), // a
                        SedInstruction::Call(CallFunc::new("shift_left1")),
                        SedInstruction::Val(Value::Arg(1)), // b
                        SedInstruction::Call(CallFunc::new("shift_right1")),
                        SedInstruction::Call(CallFunc::new("mul")),
                        SedInstruction::Val(Value::Arg(0)), // a
                        SedInstruction::Call(CallFunc::new("add")),
                        SedInstruction::Set(Value::Local(0)), // rstr
                    ],
                )),
            ],
        )),
        // return rstr;
        SedInstruction::Val(Value::Local(0)),
        SedInstruction::Ret,
    ]);
```

Wasm Text Format(WAT)で書くとするのなら、以下のようになるでしょうか（あくまで例のため動作確認はできていませんが...）

```wasm
(func $mul (param i32) (local i32) (local i64) (result i32)
    local.get 0
    call $is_empty
    (if
      (then
        const.i32 0
        local.set 2
      )
      (else
        local.get 1
        call $ends_with_zero
        (if
          (then
            ;; rstr = mul(shift_left1(a), shift_right1(b))
            local.get 0
            call $shift_left1
            local.get 1
            call $shift_right1
            call $mul
            local.get 2
          )
          (else
            ;; rstr = add(a, mul(shift_left1(a), shift_right1(b)))
            local.get 0
            call $shift_left1
            local.get 1
            call $shift_right1
            call $mul
            local.get 0
            call $add
            local.set 2
          )
        )
      )
    )
    local.get 2
    return
)
```

スタック指向な言語としてWATを書けるという記事を以前書いたので、よかったら参考にしてください。

https://zenn.dev/phantom/articles/2245f32683dae9

この段階でしっかり中間表現のような概念を作ったことが後の実装のスムーズさにつながったと思います。

## commit: [00b132a](https://github.com/Tom-game-project/sed-compiler/commit/00b132a)..


### 余談:二の補数

二進数を負数で表現しようと考えたらまずどうするでしょうか？愚直な考え方としては最上位のビットで正負を表現、というのが普通でしょう。でもこれだと0が±0の二通りで表現できてしまいます(4bit で考えたら0000b == 1000b)。

4bitに制限して考えるとどうでしょう。下のように数が割り当てられたらうまく行きそうですよね！？
こうすれば、0が二通りで表現されることがありません（0000b != 1000b）。

![](https://storage.googleapis.com/zenn-user-upload/73a8cf6a78a4-20251118.png)

上の図で`0110b`の負数を考えたいときは、真ん中の対照線で折ったときに重なる部分を見れば良さそうです。

計算で求めるにはどうすればいいでしょう。オーバーフローを一旦無視して、`10000b - 0110b`を考えたら良いでしょう！

ところで、bitを反転させた後と前を足すと常に(4bitなら)1111bになります。これは当たり前な結果です。

```
1001 --+  1010 --+
       !         !
0110 <-+  0101 <-+ (+
---------------------
1111      1111
(!はbit反転の意味)
```

さっきの`10000b - 0110b`を一般化したもの`f(x)`を考えると、、、

```
x:4bit int
x + x! = 1111b    // bitを反転させた後と前を足すと常に4bit
x!     = 1111b - x

f(x) = 10000b - x
     = 1b + (1111b - x)
     = 1b + x!
```

こうしてめでたくマイナスの数を求めるためにマイナスを用いなくてもよくなりました。

## まとめ

作った言語の名前は、Se(e)dを埋め込めるのでSoilにしました


