# 555タイマー互換回路を実装する

<!--
思いつきです。**この実装に意味はあるのかと言われると正直微妙です**。
-->

555タイマー(NE555)は何十年も昔からクロック生成・PWM信号生成・ワンショット
タイマ生成（XX秒間リレーを動かす、など）に使用されてきた歴史あるICです。
秋月ではCMOSタイプが売られています(<http://akizukidenshi.com/catalog/g/gI-00130/>)。

タイマ時間は最大2個の外付け抵抗とコンデンサ1個で設定できます。
応用の広さの割に内部回路はコンパレータ2個・トランジスタ2個・RSフリップフロップ1個で比較的シンプルです。

これも移植します。

## 下調べ（または復習）：ピンと機能

Wikipedia^[https://en.wikipedia.org/wiki/555_timer_IC]を参考にしました。

抵抗3個で電源を分圧してコンパレータに入力しています。コンパレータの出力はRS-FFの入力に
つながっています。このフリップフロップはリセット入力があるタイプでピンに出されています。
RS反転FFの反転出力が出力段(`OUT`)と放電NPNトランジスタ(`DISCHARGE`)につながっています
([@fig:block-diagram])。

\\newpage

![等価回路図](images/idea2/timer555block.png){#fig:block-diagram}

## 全体図

製作した回路の全体図を示します([@fig:timer555-overview])。コンパレータ2個、DFF1個、OSC1個、
NOT回路1個、3入力LUT1個のシンプルな構成です。出力段と放電NPNトランジスタはIOの機能を使います。

\\newpage

![全体図](images/idea2/overview.png){#fig:timer555-overview}

## コンパレータ

GreenPAKの電圧ソース選択にはいくつかの制限事項があります。
まずコンパレータそれぞれに*固有の*外部入力ピンが設定されています。また、入力ソースに
外部電圧を選択できるのは非反転側だけです。外部リファレンス電圧は反転側に入力することも
できますが、*すべての*コンパレータで共有するため注意が必要です。コンパレータマクロセルによって
非反転入力にVDDを選択できるものとできないものがあります。

非反転入力にはアテネータがついています。入力をそのままにするか1/2、1/3、1/4倍にするかを選択できます。

下調べしたとおり、コンパレータ2個にはそれぞれ1/3VDDと2/3VDDが入力され、
外部電圧(`THRESHOLD`、`nTRIGGER`)と比較されます。

### 2/3VDD vs. `THRESHOLD`

[@fig:block-diagram]の`THRESHOLD`と2/3VDDの比較回路を再現します。

GreenPAKの反転入力の選択肢には2/3VDDが用意されていないので、代わりに非反転入力のゲインを1/2倍(0.5x)にして
1/3VDDと比較します。SLG46826は1/3VDDがリファレンスの選択肢にないので、使う予定のVDDに合わせて調整します。
VDDが3.3Vなら1120mV、5.0Vなら1696mVが1/3VDDに最も近くなります。リファレンス電圧の選択肢は2.0V付近まで
用意されているので、VDDが3.0V未満であれば非反転入力のゲインを変えずに済みます。

コンパレータマクロセルは`CMP1H`を使いました。

Table: `CMP1H`の調整

|   Option   |        Value         |    Note    |
|:----------:|:--------------------:|:----------:|
|  IN+ gain  |         x0.5         |            |
| IN+ source | Buffered PIN19(IO13) |            |
| IN- source |        1120mV        | VDD = 3.3V |

Table: `PIN19(IO13)`の調整

|     Option     |        Value        | Note |
|:--------------:|:-------------------:|:----:|
| I/O selection  | Analog input/output |      |
|    Resistor    |      Pull Down      |      |
| Resistor value |         1M          |      |

### 1/3VDD vs. `nTRIGGER`

[@fig:block-diagram]の`nTRIGGER`と1/3VDDの比較回路を再現します。

1/3VDDはVDDを非反転入力ソースに選択してゲインを1/3(0.33x)に設定します。
反転入力ソースは外部リファレンス入力を選択します。先述の通り反転入力ソースを外部から得るときは
制限がありますが今回はこのまま行きます。

コンパレータマクロセルはCMP0Hを使いました。

Table: `CMP0H`の調整

|   Option   |         Value         | Note |
|:----------:|:---------------------:|:----:|
|  IN+ gain  |         x0.33         |      |
| IN+ source |          VDD          |      |
| IN- source | Ext. Vref (PIN3(IO1)) |      |

Table: `PIN3(IO1)`の調整

|     Option     |        Value        | Note |
|:--------------:|:-------------------:|:----:|
| I/O selection  | Analog input/output |      |
|    Resistor    |      Pull Down      |      |
| Resistor value |         1M          |      |

## RSフリップフロップ

RS-FFのマクロセルはないので、DFFを使った等価回路を作ります。論理式を調べたところ、
$D = S + Q * nR$でした。3入力LUTをDFFの入力につなげ、論理式を満たすように設定します([@tbl:truth-table])。
いずれかのオシレータを常時出力モードにしてDFFのCK入力につなげます。DFFの出力は反転させます。
リセット入力は外部ピン(`PIN2(IO0)`)から得ます。外部ピンはプルアップさせておきます。

Table: `3-bit LUT0`の調整

| Option | Valut | Note |
|:------:|:-----:|:----:|
|  Type  |  LUT  |      |

Table: 真理値表($D = S + Q * nR$) {#tbl:truth-table}

| IN2(**S**) | IN1(**R**) | IN0(**Q**) | OUT(**D**) |
|:----------:|:----------:|:----------:|:----------:|
|     0      |     0      |     0      |   **0**    |
|     0      |     0      |     1      |   **1**    |
|     0      |     1      |     0      |   **0**    |
|     0      |     1      |     1      |   **0**    |
|     1      |     0      |     0      |   **1**    |
|     1      |     0      |     1      |   **1**    |
|     1      |     1      |     0      |   **1**    |
|     1      |     1      |     1      |   **1**    |

Table: `OSC1`の調整

|     Option     |     Value      | Note |
|:--------------:|:--------------:|:----:|
| OSC power mode | Force Power On |      |

Table: `3-bit LUT1`の調整

|      Option       |     Value     | Note |
|:-----------------:|:-------------:|:-----|
|       Type        |   DFF/LATCH   |      |
|       Mode        |      DFF      |      |
| Initial polarity  |     High      |      |
| Q output polarity | Inverted (nQ) |      |

Table: `PIN2(IO0)`の調整

|     Option     |               Value                | Note |
|:--------------:|:----------------------------------:|:----:|
| I/O selection  |           Digital input            |      |
|   Input mode   | Digital input with Schmitt trigger |      |
|    Resistor    |              Pull Up               |      |
| Resistor value |                 1M                 |      |

## 出力段と放電トランジスタ

放電トランジスタはDFFの出力をIOピンにつなげオープンドレインに設定します。
出力段はDFFの出力をNOT回路ごしにIOピンにつなげます。これらはデジタル出力モードを
持つ適当なIOピンで事足ります。今回は`PIN15`を出力段として、`PIN16`を放電トランジスタとして
使いました。

Table: `PIN15(IO9)`の調整

|    Option     |     Value      | Note |
|:-------------:|:--------------:|:----:|
| I/O selection | Digital output |      |
|  Output mode  |  2x push pull  |      |

Table: `PIN16(IO10)`の調整

|     Option     |       Value        | Note |
|:--------------:|:------------------:|:----:|
| I/O selection  |   Digital output   |      |
|  Output mode   | 2x open drain NMOS |      |
|    Resistor    |      Pull Up       |      |
| Resistor value |        10K         |      |


## まとめ

この回路ブロックを作るにはこれだけのリソースが必要です([@tbl:idea2-resources])。
アナログ入力2ピン以外はすべて任意のデジタルIOピンに置き換えができます。

Table: リソースまとめ {#tbl:idea2-resources}

|        Type        | Count |           Note           |
|:------------------:|:-----:|:------------------------:|
|        DFF         |   1   |          RS-FF           |
|    3-input LUT     |   1   |     RS-FF; DFF input     |
|        OSC         |   1   |     RS-FF; DFF clock     |
|    2-input LUT     |   1   |       1x Inverter        |
| Analog comparator  |   2   |                          |
|  Analog Input pin  |   2   |                          |
| Digital Input pin  |   1   |          w/ PU           |
| Digital Output pin |   2   | 1x P-P, 1x OD(Nch w/ PU) |

## おまけ：応用回路例…は探せば見つかるよ

この回路は既存のかなり有名な石の劣化コピーなのでWikiPediaなりを見ていただければ
応用例はすぐに見つかります。モノステーブル（ワンショットタイマ）・アステーブル（オシレータ）
の項目を探してください。

## おまけ（2）：外部リファレンス入力…は無理

秋月で入手できるLMC555のデータシートにはPWM出力する応用例がありますが、
それを含めて外部から`CONTROL`ピンに外部リファレンス入力する方法がありません。

…って考えてたら`nTRIGGER`をコンパレータの＋入力に入れて、LUT調節で行けるかも感が出てきました。
でも読者の皆さんの自習のためにあえて答えを書かないでおきます（大学の教科書風）。
<!--
## ワンショット(モノステーブル)回路
## フリーラン・オシレータ(アステーブル)回路
-->
