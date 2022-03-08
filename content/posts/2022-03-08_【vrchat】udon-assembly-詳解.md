---
title: 【VRChat】Udon Assembly 詳解
date: 2022-03-07T23:31:50.830Z
draft: false
categories:
  - blog
tags:
  - Unity
  - VRChat
  - Udon
---
## Udonについて

VRChatのワールド用実行基盤として、Udon VMという仮想マシンが存在します。
このUdon VMは、Udon Assemblyから生成されたバイトコードを実行します。

Udon Assemblyは、現状では公式のUdonGraphやUdonSharpなどからコンパイルにより生成されますが、Udon Assemblyを直接手書きすることも可能です。
本記事では、Udon Assemblyを記述する方法についてご紹介します。

## Udon Assembly Program Asset

Udon Assemblyを直接記述できるアセットである**Udon Assembly Program Asset**を作成するには、VRChat World SDKをインポートした上で、以下のようにします。

Projectタブ > 右クリック > Create > VRChat > Udon > Udon Assembly Program Asset

![](/images/uploads/udonassemblycontextmenu.png)

これでアセットファイルが生成されます。
アセットファイルを選択すると、Inspectorに設定画面が表示されます。

![](/images/uploads/udonassemblyinspector.png)

テキストエリアが空欄なので、分かりやすいようにアセンブリを入力してみるとこうなります。

![](/images/uploads/udonassemblyinspector2.png)

各部を説明します。

①: アセンブリをクリップボードにコピーするボタン
②: アセンブリの入力欄
③: パブリック変数
④: コンパイル後のバイトコードを逆アセンブルした結果（またはコンパイルエラーメッセージ）

この②にアセンブリを入力してゆくことになりますが、アセンブリのコードにエラーがあると**フレームごとにエラーメッセージが出力され続ける**というトンデモ仕様があります。
これは、コードの入力途中にも適用され、入力途中のコードは当然エラーと解釈されるので、コードを書いている最はずっとエラーが出続けます。
ログファイルを圧迫することもありますので、コードの記述はテキストエディタで行ない、コピーしたものを入力欄にペーストする方法が良いと思います。
ただ、これを気にしないというのであれば直接入力することもできます。

この方法でもコードにミスがあれば大量のエラーが出てしまいますが、**Projectタブからアセットファイルの選択を解除してInspectorが閉じられるとエラーは止まります**ので、覚えておきましょう。

![](/images/uploads/udonassemblyerror.png)

![](/images/uploads/udonassemblyerror2.png)

## Udon Assemblyの構造

Udon Assemblyは大きく分けて2つの部分により構成されます。

* データ部
* コード部

これらを以下のように記述します。

```text
.data_start
    # データ部の記述
.data_end

.code_start
    # コード部の記述
.code_end
```

`.data_start`から`.data_end`の間がデータ部、`.code_start`から`.code_end`の間がコード部です。
なお、コードは行単位で解釈されますが、途中に空行を入れたり、命令文の途中に空白を入れたりすることは自由です。
また、`#`はコメントです。

## データ部の記述

まずデータ部の記述例を示します。

```text
.data_start

    .export _name
    .sync _name, none
    _name: %SystemString, "Akane"
    _value: %SystemSingle, 12.345

.data_end
```

この例では`_name`変数は`string`型で、初期値は`"Akane"`であると定義されています。
また、`_value`変数は`float`型で、初期値は`12.345`であると定義されています。
さらに、`_name`はパブリックで同期されています。

順を追って見ていきましょう。
まずは、`.export`の行と`.sync`の行を無視して、`_name`と`_value`の行に注目します。

### 変数定義

```text
_name: %SystemString, "Akane"
```

変数定義は3つの部位により構成されます。

* 変数名
* 型
* 初期値

`<変数名>: %<型>, <初期値>`
のように記述します。

#### 変数名

アルファベットと数字とアンダースコア`_`を使えます。
加えて、角括弧`[]`と三角括弧`<>`の文字も使用できます。
ただし、最初の文字はアルファベットとアンダースコアのみが許可されます。

```text
【有効】
x
X
abc123
_Number<0>
__this_is_a_variable[][]__>>>

【無効】
1
222x
$abc
```

また、後述するコード部で使用するラベル名と同じ名前をつけても全く問題ありません（文脈で区別可能だからです）。

**注意**

詳細は不明ですが、Udonには**予約変数**が存在するようです。
予約名を回避するために、**変数名の先頭をアンダースコアにすると良い**といわれています。

#### 型

C#（.NET）における型を独自の記法で記述します。
ただし、VRChatが許可する型のみが有効です。

例えば`int`型は、フルネームで`System.Int32`なので、`SystemInt32`と記述します（ドットが消去されていることに注意してください）。
Udonでは配列も使用できますが、例えば`int[]`型の場合、`SystemInt32Array`と最後に`Array`を付加します。

型のUdon名称を得るための規則を以下に示します。

1. 対象の型の名前空間名と型名をピリオド無しで結合する（`System.String` → `SystemString`）
2. 子クラスの場合は、親クラスの1に対して型名を付加する（`Cinemachine.CinemachinePathBase+Appearance` → `CinemachineCinemachinePathBaseAppearance`）
3. ジェネリック型の場合は、型引数に本規則を適用した名称を列挙する（`System.Collections.Generic.List<int>` → `SystemCollectionsGenericListSystemInt`）
4. 配列型の場合は、`[]`を書かず、`Array`を付加する（`System.DateTime[]` → `SystemDateTimeArray`）

変数定義において、この規則により得た名称の先頭に`%`を付加して表記します。

#### 初期値

限られたリテラルのみ使用可能です。

| 型          | 表記         | 補足                                               |
| ---------- | ---------- | ------------------------------------------------ |
| object     | null       | nullのみ指定可能、struct型はdefault値に初期化される               |
| uint       | 0xFFFFFFFF | 16進数表記                                           |
| int        | 1234567890 | 整数                                               |
| float      | 12.345     | 小数                                               |
| string     | "abcdefgh" | ""で囲った文字列                                        |
| GameObject | this       | thisを指定すると、このUdonBehaviourを所有するGameObjectに初期化される |

当然ながら、`object`や`GameObject`はそれを継承する型にも使用可能です。

以上で変数定義は完了です。

### 属性

データ部に記述できるのは、変数定義と、それに付随する属性の指定です。
指定できる属性は2種類です。

* export属性
* sync属性

#### export属性

対象の変数をPublicに指定します。
Public変数はUnityのInspectorで値を指定することができます。

以下のように記述します。

```text
.export _name
_name: %SystemString, null
```

#### sync属性

対象の変数を同期します。
同期された変数は他のクライアントとの通信で値が更新されます。
同期については本記事では詳しく説明しません。

同期にはモードが3つあります。

* none
* linear
* smooth

以下のように記述します。

```text
.sync _value, none
_value: %SystemSingle, 0.0
```

データ部は以上です。

## コード部の記述

コード部の例を示します。

```text
.code_start
    .export _start
    _start:
        PUSH, _message
        PUSH, _str
        COPY
        PUSH, _str
        EXTERN, "UnityEngineDebug.__Log__SystemObject__SystemVoid"
        JUMP, 0xFFFFFFFC
    .export _custom
    _custom:
        PUSH, _message
        EXTERN, "UnityEngineDebug.__Log__SystemObject__SystemVoid"
        JUMP, 0xFFFFFFFC
.code_end
```

Udon Assemblyのコード部は以下の3要素から成っています。

* 命令
* ラベル
* 属性

命令の`PUSH`という語が示すように、Udon VMはスタックマシンです。
スタックマシンは、スタックに値を積んだり消費したりしながら計算を行なう機械のことです。

### スタックマシンのイメージ

スタックはデータ構造の一種で、**後入れ先出し**（**LIFO**）という特徴があります。
データを順番にスタックに入れてゆくと、取り出すときは、後に入れたものを先に取り出す、ということです。

![](/images/uploads/stackmachinepush.png)

![](/images/uploads/stackmachinepop.png)

図を見て分かるように、数列`1`、`2`をこの順にスタックへプッシュすると、ポップしたときに`2`、`1`と返されます。
これがスタックの挙動です。

以下の命令列について考えてみましょう。
命令列は上の行から順に実行されます。

```text
PUSH, a
PUSH, b
COPY
```

`PUSH`は、変数をスタックに積む命令です。
つまり、

- 変数`a`を積む
- 変数`b`を積む
- コピーする

です。
`COPY`は、スタックから2つ取り出し、1つ目の変数に2つ目の変数の内容をコピーする命令です。
先ほどの例の通り、`a`、`b`と積んだので、取り出したときに`b`、`a`となります。
従って、`b`に`a`をコピーし、この命令列は終了します。

別の例を見てみます。

```text
PUSH, a
PUSH, b
PUSH, c
COPY
PUSH, d
COPY
```

まず、`a`、`b`、`c`がスタックに積まれます。
次にコピーなので2つ取り出すと、`c`、`b`となりますので、`c` ← `b`とコピーされます。
`a`は取り出されずにスタックの底に残りました。
そこに`d`を積み、コピーです。
`d`、`a`と取り出されるので、`d` ← `a`とコピーされます。
まとめると、この命令列は`c` ← `b`、`d` ← `a`という動作になりました。

スタックマシンは以上のように動作します。

### 命令の種類

|命令|記法|命令長(byte)|説明|
|:---|:---|:---:|:---|
|NOP|NOP|4|何もしない|
|PUSH|PUSH, var|8|変数をスタックにプッシュする|
|POP|POP|4|スタックから1つ捨てる|
|JUMP|JUMP, label|8|コード部のラベルへ処理を遷移させる|
|JUMP_IF_FALSE|JUMP_IF_FALSE, label|8|スタックから1つ取り出し、値がFalseの場合のみJUMP|
|JUMP_INDIRECT|JUMP_INDIRECT, var|8|varに格納されたアドレス（uint値）にJUMP|
|COPY|COPY|4|スタックから2つ取り出し、1つ目の変数に2つ目の値をコピーする|
|EXTERN|EXTERN, "method"|8|指定された名前のメソッドを実行（パラメータ分スタックを消費）|
|ANNOTATION|?|4|不明|

#### JUMP系命令について

JUMP系命令は3種類あります。

1. JUMP, label
2. JUMP_IF_FALSE, label
3. JUMP_INDIRECT, var

1は問答無用に命令の実行位置を指定したラベルへ遷移します。

```text
PUSH, a
PUSH, b
JUMP, jump
PUSH, c  # 実行されない
jump:  # ラベル
COPY  # ここから実行
```

この例の場合、`a`、`b`がスタックに積まれ、その後`jump`ラベルまで遷移し、`COPY`が実行されます。

2はスタックから1つ取り出し、Falseの場合のみ遷移します。
取り出した値はBooleanのみ許可されます。
Trueであれば素通りして次の命令に行きます。

```text
PUSH, a
PUSH, b
JUMP_IF_FALSE, jump1
PUSH, c  # b == true
JUMP, jump2
jump1:
PUSH, d  # b == false
jump2:
COPY
```

ちょっと複雑ですが、上記のようにすると`if`と`else`の分岐を表現できます。

3は変則的ですが、指定された変数に格納されたuint値のアドレスへ処理を遷移します。
アドレスとは何かというと、コード部には先頭から順にアドレスが割り振られています。
前掲の表にアセンブリの命令長を示しましたが、命令は命令長分のサイズを持ち、アドレス`0`から順にメモリに格納されていると考えてください。

先頭のアドレス`0`に`PUSH, a`が格納されていたら、`PUSH`の命令調は`8`なので、次の命令のアドレスは`8`となります。

命令長は`4`または`8`ですので、16進数で表記すると、最右の数値は必ず`0`、`4`、`8`、`C`のいずれかになります。

このようにして、命令の位置はラベルだけでなく数値のアドレスでも表せるのですが、`JUMP_INDIRECT`命令の場合、指定された変数の値は必ず数値のアドレスとして解釈されますのでご注意ください。

```text
.data_start
    jumpVar: %SystemUInt32, 0x000001F0
.data_end
.code_start
    ...
    JUMP_INDIRECT, jumpVar  # アドレス0x000001F0へ遷移する
    ...
.code_end
```

もちろん、変数の値を変更することで別の位置へ遷移することもできます。
この命令は、関数の呼び出し元へ戻る際などに使用します。

最後に、`0xFFFFFFFC`への遷移は実行を終了する際に使います。

#### EXTERN命令について

.NETのメソッド呼び出しを行ないます。
例えば、`UnityEngine.Debug.Log(obj)`を呼ぶには以下のようにします。

```text
PUSH, obj
EXTERN, "UnityEngineDebug.__Log__SystemObject__SystemVoid"
```

メソッドの名称は、基本的には以下のように決定されます（この規則に準じていないメソッド名もあります）。

- 型名
- ピリオド
- アンダースコアx2
- メソッド名
- アンダースコアx2
- 引数の型（無ければ`SystemVoid`、複数ある場合はアンダースコアx1で区切る）
- アンダースコアx2
- 戻り値の型（無ければ`SystemVoid`）

メソッドを呼ぶには変数の`PUSH`が必要です。
以下のようにします。

```text
PUSH, <インスタンス>  # 静的メソッドの場合は無し
PUSH, <引数1>
PUSH, <引数2>
...
PUSH, <戻り値を受け取る変数>  # SystemVoidの場合は無し
EXTERN, "method"
```

### イベント

コード部のラベルに`export`属性を付加すると、UdonBehaviourのイベントとして認識されます。
既存のイベントを記述する場合、名称に注意してください。
イベント名称をラベル名称に変換するには以下のようにします。

- `Start`イベント → `_start`ラベル
- `Update`イベント → `_update`ラベル
- `OnPlayerJoined`イベント → `_onPlayerJoined`ラベル

```text
.code_start
    .export _start
    _start:
        ...
    .export _update
    _update:
        ...
.code_end
```

## まとめ

ざっとですが、Udon Assemblyについて見てみました。
普段UdonGraphやUdonSharpでコーディングする方も、これらの言語自体のバグに当たることは少なくないと思います。
そんなときには、Udon Assemblyをチェックしてみてください。
アセンブリが読めると解決策が思いつくかもしれません。
