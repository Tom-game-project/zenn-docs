## きっかけ

SedLispというプロジェクトとの出会いをきっかけに、私もSedというランタイム上で動く何かを作ってみたいと考えました。
以下、コミットログを振り返りつつ、sed-compilerの開発を自分で振り返って行きます。

## init commit: b0daa87..

もともとこのリポジトリは、Sedを練習する目的で作られたものでした。リポジトリが作成された9月頃は、私自身まだSedの文法をあまりわかっていないという状況だったので。お試し感覚で加算機を作ってみたのが最初のコミットです。

ちなみにSedは,加減乗除、余りの計算はもちろん、関数や変数などもありません。その代わりに、ジャンプラベル、HoldスペースPatternスペースのような特徴的な機能があります。今回の試みはそんなSedの機能を使って一般的なプログラミング言語がデフォルトでサポートするような機能を実現してみようというモチベーションでやっていきます。

### 最低限のSedの知識

- ジャンプラベル

if節や関数の実装、更にはループ処理などで重要になります。

```sed
: label <-+
...       |
b label --+
```

- HoldスペースPatternスペース

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

## commit: b0daa87..a766d81

Sedで関数の再帰的な呼び出しを実装するための試行錯誤をしています。
特にa766d81:labo.sedは、以下のようなプログラムをSedで表現しています。

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

## commit: a766d81..00b132a

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

## commit: 00b132a..

## まとめ

作った言語の名前は、Se(e)dを埋め込めるのでSoilにしました

