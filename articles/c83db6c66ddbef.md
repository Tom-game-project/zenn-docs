---
title: "Sedにコンパイルできる言語を作る"
emoji: "🧘‍♂️"
topics:
  - "sed"
type: idea
published: false
---

## きっかけ

[sedlisp](https://github.com/shinh/sedlisp)というプロジェクトとの出会いをきっかけに、sedというエディターに興味が湧きました。
そこで、今回はsedという特殊なランタイムをターゲットにしたコンパイラ及び言語の作成に取り組もうと思います。
以下、個人プロジェクト`sed-compiler`のコミットログを記事執筆時点での進捗まで振り返ろうと思います。

https://github.com/Tom-game-project/sed-compiler

## init commit: [b0daa87](https://github.com/Tom-game-project/sed-compiler/commit/b0daa87)..

元々はsedというエディタがいかなるものかを身をもって体験しようと思って作ったリポジトリでした。
リポジトリが作成された9月頃は、私自身まだsedの文法すらあまりわかっていないという状況だったので、お試し感覚で加算機を作ってみたのが最初のコミットです。

https://github.com/Tom-game-project/sed-compiler/blob/b0daa87b860b4e72371c9460e6819b3eee78d892/add.sed

リポジトリをCloneしている場合は以下のようにして試すことができます

```sh
# ２つの二進数を標準入力で流す
echo 10101 1110 | sed "$(git show  b0daa87:add.sed)"
```

かなり愚直で無駄が多い実装です。

ちなみにsedには、加減乗除、余りの計算はもちろん、関数や変数などもありません。
その代わりに、ジャンプラベル、HoldスペースPatternスペースのような特徴的な機能があります。

### 最低限のsedの知識

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

- `s///` - sedにおける操作

一般的なsedでも一番使われる機能です。
```sed
s/<置換したいパターン>/<置換後のパターン>/
```

といった感じのやつです。丸括弧でパターンを囲むと置換パターンをグループ化できますが、最大で9個しか使えません

```sed
# パターングループを最大限使ったプログラム
s/\(.*\):\(.*\):\(.*\):\(.*\):\(.*\):\(.*\):\(.*\):\(.*\):\(.*\):/\1\2\3\4\5\6\7\8\9/
```

## commit: [b0daa87](https://github.com/Tom-game-project/sed-compiler/commit/b0daa87)..[a766d81](https://github.com/Tom-game-project/sed-compiler/commit/a766d81)

このコミットの範囲では、sedで関数呼び出しを実現するための試行錯誤をしていました。

![](https://storage.googleapis.com/zenn-user-upload/ba1a8858d943-20251118.webp)

上の図では、対応する概念が同じ色で表現されています。

スタックフレームは関数が呼ばれたときにローカル変数や渡された引数などが保持される領域です。(&#x1f7e2;: 緑で囲まれた部分に引数、ローカル変数が入る, &#x1f7e0;: オレンジの部分がスタックフレームの単位)
関数の呼び出しが発生するとスタックにスタックフレームがpushされ、returnされると最上部（最後にpushされたスタックフレーム）がpopされます。(&#x1f7e3;: 紫の矢印はスタックがcallされた際に伸びる向き)
スタックフレームには更にローカル変数、引数情報以外に含めるべき情報があります。
関数から値が返る際、次に実行すべき命令は呼び出した命令の直後である必要があります。そのためには、各スタックフレームごとにその直後のアドレスが記録されている必要があります。(&#x1f534;: 赤の部分に、返るべき命令のアドレスが保存される)

上の画像のデータ構造をもとに、PatternスペースとHoldスペースをうまい具合にメモリとして扱い、関数に渡された引数、ローカル変数を管理していきます。

a766d81:labo.sedは、上のルールでスタックフレームを構成した、簡単な再帰関数です。

やっていることは以下のようなことです。
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

関数を構築しようと思うたびに自分でスタックフレームを作るのはしんどいので、自動で行えるようにRustプログラムを書きます。
sedより一層高級なレイヤーの中間表現(Intermediate Representation:IR)を作って、それをsedに変換する方針にします。

今回、sedで作ったスタックフレームは特定の文字をデリミタとしてスタックフレーム同士を区切っているため(一般的なスタックフレームのようにサイズで区切っていないため)、スタックフレームの長さを自由に変更して問題ありません。(その代わりデータにデリミタ文字は含められない)
そのためIRはForthやWasm Text Format(WAT)のように、値をスタック指向に操作できるデザインにしました。

スタック指向でデータを操作するってどういうことだ?と思った人は、以前私が書いた以下の記事を読んで見てください。

https://zenn.dev/phantom/articles/2245f32683dae9

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


