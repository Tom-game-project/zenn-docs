---
title: "GAS(Google Apps Script)上でRustを動かす"
emoji: "📦"
type: "tech"
topics:
  - "wasm"
  - "gas"
  - "wasi"
published: false
---

今回は、**W**asm **I**nterface **T**ype(WIT)の導入によって,GASへRustプログラムの埋め込みを楽にできるようにします。

本日の成果物は以下のレポジトリから見ることができます。

https://github.com/Tom-game-project/GoogleAppsScript-in-rust-example

## 背景

WIT登場以前、wasm-pack,wasm-bindgenを使えばweb向けのグルーコードなどは簡単に生成できました。
これはWasmランタイムとそれをホストする言語間のやり取りに統一的なルールが存在しなかった時代に非常にありがたいツールでした。

rs <-> js間のやり取りに優れている一方で、それは暗黙的に「特定のホスト環境」（＝モダンなブラウザやNode.js）を前提としていました。

この問題は、wasm-bindgen のようなツールが想定する標準的なJS環境から外れた場合（今回のよう）に、極めて現実的な課題として表面化します。

私がまだWITを知らない時代に特に大変だったことは、GAS側の関数をWasmに提供する方法の問題でした。開発者はある関数を使いたい、渡したいと思うたびに引数や返り値のデータが具体的にどう渡されなければいけないのかを考える必要がありました。

WITはホスト側とゲスト側(Wasm側)双方にとっての「契約」となって抽象化をもたらします。
ホスト、ゲスト双方にとってWITはデータの取引の相手が何を提供し、また自分が相手に何を提供しなければいけないのかの誓約書のようになって、うまく抽象化されています。WITの標準化によってグルーコードは自動生成できるようになり、開発者はより本質的な実装に集中できるようになりました。

実際に、コード中でWITがどのように使われるか確かめてみましょう。

https://github.com/Tom-game-project/GoogleAppsScript-in-rust-example/blob/master/wit/world.wit

上はWasm(Rust)側のWITファイルです、`import`はWasm(Rust)側にGAS(V8)側から提供されるインターフェイス、また、`export`はWasm(Rust)側がGAS(V8)側に提供しなければいけない関数となります。

実装は`src/`ディレクトリ以下にあります。

逆にGAS側も見てみましょう。

https://github.com/Tom-game-project/GoogleAppsScript-in-rust-example/blob/master/wit/deps/google-apps-script-logger-0.1.0-alpha/package.wit

Loggerのインターフェイスが具体的に提供いなければいけない関数が書いてあります。

実装は`js/`ディレクトリ以下にあります。

## 少々面倒な問題の数々

### BigInt



GASは2025年11月現在V8ランタイムによって動いています。具体的にどのバージョンで動いているのかはわからなかったので、少し独特の環境に泥臭く手探りで対処し、ほとんど自動でGAS向けjsを出力できる環境を作ってみました。動作させることに焦点を置いて書いているので間違っている、古い、などあると思いますが、詳しい方は優しく教えてくれると嬉しいです。

Rustが好きなので、作成したリポジトリはRustのビルドの流れをMakefileに含んでいますが、ゲスト側のプログラムはjcoのページにかかれているものであればすべてが動作させられるはずです。


## まとめと感想

APIをWITに変換するのは面白い作業ではありましたが、同時にかなり骨の折れる作業でもありました。
