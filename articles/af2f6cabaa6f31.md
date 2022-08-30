---
title: "Go言語で関数の複雑性を計測する"
emoji: "📑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Go"]
published: false
---

# はじめに

最近、プロジェクトのQCDについて考えたり、説明したりする機会があり、知識を整理するため

[質とスピード（2022春版、質疑応答用資料付き） / Quality and Speed 2022 Spring Edition - Speaker Deck](https://speakerdeck.com/twada/quality-and-speed-2022-spring-edition?slide=17)

を読み返しました。本当にスルメ記事です。
内部品質を確認する際、特にコーディングをした後に
定量的なもので可読性を測れないかなと思い、
主な指標と実際にGo言語で測定する方法を調査しました。

## そもそも内部品質とは

### 品質とは

> 品質（Quality）．一連の固有の特性が要求事項を満足している度合い。
> 品質とは、実現されたパフォーマンスや結果であり、「本来備わっている特性の集まりが、要求事項を満たす程度」（ISO9000）のことをいう。
> 品質は誰かにとっての価値である

「正義」や「悪」のように抽象的な言葉であり、主体によっては変わってしまう。
プロジェクト毎に何かしらの尺度で評価する必要がある。

### 内部品質とは

* 外部品質: ユーザが知覚できる部分に関連する品質
* 内部品質: 設計やソースコードなどソフトウェアを構成する中でユーザが知覚しない部分に関する品質

今回は内部品質の中でも特に保守性という品質特性の観点から、作り終えたソフトウェアに対して
評価ができないか調査しました。

保守性という品質特性は

* 解析性
* 変更性
* 安定性
* 試験性
* 適合性

という要素に細かく分類することができます。

[](https://webrage.jp/wp/wp-content/uploads/2016/07/image_00701.jpg)

https://webrage.jp/wp/wp-content/uploads/2016/07/image_00701.jpg -> ローカルだと表示されない?

[外部品質、内部品質とは？ソフトウェア品質特性について | ソフトウェアテスト・第三者検証ならウェブレッジ | ソフトウェアテスト・第三者検証ならウェブレッジ](https://webrage.jp/techblog/external_quality_inner_quality/)

## 保守性という観点からコードを定量的に評価する

### コードの量 (LOC)

コードの行数が増えるに従って可解析が難しくなるため、コードが多いことは解析性が良くないということになる。
ただ、単純なコードの量だけで判別できる事象は少なく、何かしらの単位で見る必要がある。

`find . -name '*.{拡張子}' | xargs wc -l` で測定可能

狭義では特に関数あたりのコード行数を測定する際に使用されるそうで、
golangでの閾値と計測方法を調査中
個人的には 50行を超えると長い 100行を越えると危険と思っています。
http://go.shibu.jp/effective_go.html

### 循環複雑度 (CC)

> 循環的複雑度（サイクロマティック複雑度、Cyclomatic Complexity）とは、ソフトウェア品質を測定するソフトウェアコードメトリクスのひとつで、プログラムの複雑度を測定するものです。

if, switch, whlieなどの個数によって複雑度を測定することができます。

循環的複雑度が増えると網羅性の高いテストを実行する際に作成しないといけないテストケースが膨大になります。
そのため試験性が悪くなります。

おおよその基準は以下の通りです。

| 循環的複雑度 | 複雑さの状態 | バグ混入確率 |
| --- | --- | --- |
| 10以下 | 非常に良い構造 | 25% |
| 30以上 | 構造的なリスクあり | 40% |
| 50以上 | テスト不可能 | 70% |
| 75以上 | いかなる変更も誤修正を生む | 98% |

[循環的複雑度 - MATLAB & Simulink](https://jp.mathworks.com/discovery/cyclomatic-complexity.html)

go言語は以下のツールで測定可能です。

https://github.com/fzipp/gocyclo

```go install github.com/fzipp/gocyclo/cmd/gocyclo@latest```でインストールする事ができます。
内部では`'if', 'for', 'case', '&&' or '||'` の個数を計算しています。

`gocyclo .`で計測する事ができます。

```go
func SumOfPrimes(max int) int {         // +1
    var total int

OUT:
    for i := 1; i < max; i++ {          // +1
        for j := 2; j < i; j++ {        // +1
            if i%j == 0 {               // +1
                continue OUT
            }
        }
        total += i
    }

    return total
} // Cyclomatic complexity = 4
```

このファイルを計測すると

```3 main main main.go:5:1```
左端に 循環的複雑度 = 3 が出力されます。

### 認知的複雑度

循環的複雑度は試験性を定量的に測ることに優れている一方
人間がソースコードを見て複雑かどうか = 解析性 を定量的に測るのに使用することができます。

go言語は以下のツールで測定可能です。

https://github.com/uudashr/gocognit#cyclometic-complexity

```txt
if, else if, else
switch, select
for
goto, break , continue
```

をカウントしていて、さらに
ネストの深さによってポイントが追加されます。
具体的には以下のように計算されます。

```go
func SumOfPrimes(max int) int {
    var total int

OUT:
    for i := 1; i < max; i++ {          // +1
        for j := 2; j < i; j++ {        // +2 (nesting = 1)
            if i%j == 0 {               // +3 (nesting = 2)
                continue OUT            // +1
            }
        }
        total += i
    }

    return total
} // Cognitive complexity = 7
```

同じコードでも循環的複雑度で測定すると低い数値が出てしまいます。

```go
func SumOfPrimes(max int) int {         // +1
    var total int

OUT:
    for i := 1; i < max; i++ {          // +1
        for j := 2; j < i; j++ {        // +1
            if i%j == 0 {               // +1
                continue OUT
            }
        }
        total += i
    }

    return total
} // Cyclomatic complexity = 4
```

### 測定結果を分析する

| 関数あたりのコード行数 | 循環的複雑度 | 認知的複雑度 |

を計測して、プルリクを出す前にリファクタする。コードレビューに活かす、リファクタする関数の優先度をつけるなど
に活用できそうです。
ただ、完全に計測指数を盲信せずにプロジェクトや使用する言語に応じてうまく活用する必要がありそうです。
例えば

* 静的型付け言語なので、関数あたりの行数が長くなってしまう
* 設定ファイルのマッピングや分岐などで循環的複雑度が高くなってしまう
* ネストは深いが短いコード
* テストコード
などは状況に応じて判断を変えた方が良いのかなと思いました。

## むすびに

自分のソースコードを定量的に見て、理論は知らなかったけど、何となく書いているなという印象を持ちました。
自分だけでなく、他の人のソースコードをレビューする際の一つの尺度にできたらなと思いました。
