# NeoPixelをランダムに光らせる

NeoPixelというのは、いわゆるテープLEDによく使われている、シリアル通信で色を変更できる
フルカラー（RGB）LEDのことです。この章を書き始めた時点では、筆者はNeoPixel系のLEDを使ったことが
ありませんでした。秋月やらスイッチサイエンスやらで売っていることも知っているし、品種がだんだん増えてきてるし、
各種マイコン用のライブラリがかなり充実しているし、**マトリクス状に配置するためのキットを買って**
作るところまではしましたが、まだちゃんと試してませんでした。

という個人的な反省を兼ねて、最初はこのRGB LEDを光らせます。GreenPAKの規模ではバッファメモリを
持たせたりするのは難しいので、乱数生成器を内蔵させてランダムっぽく光らせます。

*ここでは"WS2812B"のデータシート(秋月のサイト[^ws2818b-aki]から入手)を参照します。*

## 注意

+-----------------------------------------------------------------------------------------------+
| \\Large{実験の際にはLED用に別電源を用意してください。評価ボードの電源を頼りにしてもいいのは   |
| せいぜい4連までです。それ以上はオンボードLDOが燃えそうな熱さになります・した。}\\normalsize{} |
+-----------------------------------------------------------------------------------------------+

[^ws2818b-aki]: http://akizukidenshi.com/catalog/g/gI-07915

## 下調べ：データ転送プロトコル {#sec:study-protocol}

ちゃんとプロトコルから述べているブログが少ないようなので、自分で調べるところからやります
^[ｴｲﾀﾞﾌﾙｰﾄ=ｻﾝのライブラリを{使って|移植して}ハイおしまい、っていうのばかりでした]。

\\xout{とっても読みにくい}データシートによると、GRBの順で各8ビット、全24ビットMSBファーストの
シリアルデータを書き込み、ストローブとして50us以上"L"にするとLED内に取り込まれるようです。
ストローブ期間の上限は定義されていません。

シリアルデータの1/0はH/Lの割合で判定されます([@fig:data-transfer-time])。
1ビットあたり1250nsなので800KHzのシリアルデータをデューティーを変えながら送り込むことになります。
許容差が+/-300nsある[^torelance]ので950〜1550nsの範囲で周期が変わることは許容されそうです。

[^torelance]: データシートには+/-600nsとあるのですがどちらが正しいんでしょうか。

[Neopixelのシリアルデータ](data/neopixel.yaml){.wavedrom #fig:data-transfer-time}

## 全体図

製作した回路の全体図を示します([@fig:neopixel-overview])。左上がクロック生成器([@sec:clock-generator])、
右側は上段から通信プロトコル生成器([@sec:protocol-generator])、擬似乱数生成器([@sec:lfsr])、
ストローブ生成器([@sec:strobe-pulser])です。

\\newpage

![全体図](images/idea1/overview.png){#fig:neopixel-overview}

## クロック生成器 {#sec:clock-generator}

この回路では最終的に2つの異なる周波数のクロック源が必要になりました([@tbl:clock-source])。
このうち800KHzはOSC1から生成しました([@fig:clock-gen-sch])。OSC2(25MHz)は400/800nsの生成に必要でした。

`800KHz`を得るために原発振2MHzを2逓倍したあと5分周します。
OUT0(2MHz)の出力に`P-DLY`マクロセルをつなげ、両エッジ検出モードにして立ち上がり・立ち下がりエッジごとに
パルスを出させることで逓倍できます([@fig:clock-gen-wave])。
5分周には`DLY/CNT`マクロセルをカウンタとして1個使います([@tbl:config-cnt1])。

Table: 必要なクロック {#tbl:clock-source}

|   Frequency   |                     Purpose                      |
|:-------------:|:------------------------------------------------:|
| 2MHz[^not-2M] |           LFSR/protcol clock(`800KHz`)           |
|     25MHz     | NeoPixel protocol timing generation(400ns/800ns) |

[^not-2M]: SLG46826だと厳密には2.048MHzですが、LED側のタイミング定義がゆるいので
こちらの定義もゆるめで行きます。

![OSC1周辺回路](images/idea1/clock-gen.png){#fig:clock-gen-sch}

[800KHzクロック生成](data/osc-config.yaml){.wavedrom #fig:clock-gen-wave}

Table: `OSC1`の調整 {#tbl:config-osc1}

|      Option      |     Value      | Note |
|:----------------:|:--------------:|:----:|
|  OSC power mode  | Force Power On |      |
| Control pin mode |    Force on    |      |

Table: `P DLY`の調整 {#tbl:config-progdelay}

|   Option    |      Value      | Note |
|:-----------:|:---------------:|:----:|
|    Mode     | Both edge delay |      |
| Delay value |     125 ns      |      |

Table: `CNT1/DLY1`の調整 {#tbl:config-cnt1}

|       Option        |          Value          | Note |
|:-------------------:|:-----------------------:|:----:|
| Multi-function mode |         CNT/DLY         |      |
|        Mode         |      Reset counter      |      |
|    counter data     |            4            |      |
|        Clock        | Ext. Clk. (From matrix) |      |

## 擬似乱数生成器 {#sec:lfsr}

Wikipedia「線形帰還シフトレジスタ(LFSR)」の項目[^lfsr-wikipedia] とその参照先の
アプリケーションノート[^lfsr-appsnote]を参考にして、15ビット長の疑似乱数生成回路
を作りました([@fig:linear-feedback-shift-register])。この回路はランダムビット列を
生成し続けます。

![LFSRブロック](images/idea1/lfsr-gen.png){#fig:linear-feedback-shift-register}

[^lfsr-wikipedia]: https://ja.wikipedia.org/wiki/%E7%B7%9A%E5%BD%A2%E5%B8%B0%E9%82%84%E3%82%B7%E3%83%95%E3%83%88%E3%83%AC%E3%82%B8%E3%82%B9%E3%82%BF
[^lfsr-appsnote]: https://www.xilinx.com/support/documentation/application_notes/xapp052.pdf

WikiPediaより、15ビット長のLSFRの周期を最大化するにはシフトレジスタのタップ15と14をXOR演算してタップ1に戻します。
GreenPAKには最大16ビット長のPipe Delayモジュール(最大深さ8または16段の1ビットFIFO)が搭載されているので、
これを用います([@tbl:config-pipedelay])。

PipeDelayはあえて深さ14段にして外付けDFFを1個追加して15段にします。これによってタップ14にPipeDelayの出力、
タップ15に追加したDFFの出力を使うことができます。2ビットLUT(XOR設定)にこれらを入力し、XOR出力をPipeDelayの入力に戻します。

シフトレジスタのクロックは`800KHz`([@tbl:config-osc1])を使います。

Table: PipeDelayの調整 {#tbl:config-pipedelay}

|   Option    | Value | Note |
|:-----------:|:-----:|:----:|
| OUT0 PD num |  14   |      |

## NeoPixelの通信プロトコル生成器 {#sec:protocol-generator}

プロトコル生成器は2つのブロックに細分化できます。全体図([@fig:neopixel-overview])の
右上部がロジック-パルス変換器([@sec:logic-to-pulse])、右下部が
ストローブ信号生成器([@sec:strobe-pulser])です。

### ロジック-パルス変換器 {#sec:logic-to-pulse}

生成すべきパルス列の特性については、先の通り調査済です([@sec:study-protocol])。このタイミングを
生成するために筆者は再びディレイマクロセルを使いました。今回はワンショットモードにして
クロック入力を`OSC2`、`DLY IN`を`800KHz`に設定します([@fig:neopixel-pulser-sch])。これらの回路は
ロジック0または1のパルス幅をもったパルス列を生成し続けます。

![ロジック−パルス変換](images/idea1/neopixel-pulse-gen.png){#fig:neopixel-pulser-sch}

このパルス列をマルチプレクサに入力しLFSRの出力で切り替えます。マルチプレクサは
LFSRがHを出力している間800nsパルスを通過させ、L期間中は400nsパルスを出させます([@fig:neopixel-pulser-wave])。

[マルチプレクサの動作波形](data/neopixel_pulser.yaml){.wavedrom #fig:neopixel-pulser-wave}

```table
---
markdown: True
caption: "`CNT2/DLY2`の調整 {#tbl:config-dly2}"
alignment: CCC
witdh: 
    - 0.3
    - 0.3
    - 0.4
---
"Option","Value","Note"
"Multi-function mode","CNT/DLY",
"Mode","One shot",
"Counter data","5","Apply Simulation results"
"Edge select","Rising",""
"Clock","OSC2",
```

```table
---
markdown: True
caption: "`CNT3/DLY3`の調整 {#tbl:config-dly3}"
alignment: CCC
witdh: 
    - 0.3
    - 0.3
    - 0.4
---
"Option","Value","Note"
"Multi-function mode","CNT/DLY",
"Mode","One shot",
"Counter data","14","Apply Simulation results"
"Edge select","Rising",""
"Clock","OSC2",
```

\\newpage

### ストローブ信号生成器 {#sec:strobe-pulser}

ストローブ信号生成器は3段のカウンタで構成されます([@fig:strobe-pulser-sch])。1段目でLED1個分の
データ書き込みタイミング、2段目で2多段カスケードのためのタイミングを測り、3段目で
データ更新周期を調整します。

![ストローブ信号生成器](images/idea1/strobe-pulser.png){#fig:strobe-pulser-sch}

それぞれのカウンタは設定値までカウントアップしたあと、クロック1周期分の”H”信号を出力します。
カウンタ出力をクロック源として捉えると、1周期は`<クロック周期> x (<カウンタ設定値> + 1)`になります。
たとえばNeopixelは1個あたり24ビットデータなので1段目の設定値は**23**です([@tbl:config-cnt4])。

一般化して1個分のデータ書き込み時間がT、連結数がN個(2段目の設定値は*(N−1)*)、
3段目の設定値がM回のとき、3段目のカウンタは`N個 x M回 x T`の期間"L"を出力したあとで
`N個 x 1回 x T`の期間だけ"H"を出します([@fig:strobe-pulser-wave])。

<!--\\newpage-->

[各カウンタの動作](data/strobe_pulse_gen.yaml){.wavedrom #fig:strobe-pulser-wave}

`N`や`M`の値が小さすぎるとLEDの発色の変化に目が追いつかず真っ白に見えてしまうので、適宜大きめの値を入れて
`N x (M+1) x T`が200〜300msになるように調整するときれいになると思います。
800KHz、LED1個あたり24ビットデータのとき`T = 24/800[ms] = 30[us]`です。8連なら`N = 7`、`M = 1249`
に設定すると更新周期が300[ms]になります。

```table
---
markdown: True
caption: "`CNT4/DLY4`の調整 {#tbl:config-cnt4}"
alignment: CCC
witdh: 
    - 0.4
    - 0.3
    - 0.3
---
"Option","Value","Note"
"Multi-function mode","CNT/DLY",""
"Mode","Reset counter",""
"Counter data","23",""
"Edge select","Rising",""
"Clock","Ext. Clk. (From matrix)",""
```

```table
---
markdown: True
caption: "`CNT5/DLY5`の調整 {#tbl:config-cnt5}"
alignment: CCC
witdh: 
    - 0.4
    - 0.3
    - 0.3
---
"Option","Value","Note"
"Multi-function mode","CNT/DLY",""
"Mode","Reset counter",""
"Counter data","*(N-1)*","*N = number of LEDs in series*"
"Edge select","Falling",""
"Clock","CNT4/DLY4 (OUT)",""
```

```table
---
markdown: True
caption: "`CNT0/DLY0/FSM0`の調整 {#tbl:config-cnt0}"
alignment: CCC
witdh: 
    - 0.4
    - 0.3
    - 0.3
---
"Option","Value","Note"
"Multi-function mode","CNT/DLY",""
"Type","CNT/DLY",""
"Mode","Counter/FSM",""
"Counter data","*(M-1)*",""
"Edge select","Falling",""
"Clock","Ext. Clk. (From matrix)",""
```

## まとめ

この回路ブロックを作るにはこれだけのリソースが必要です([@tbl:idea1-resources])。
25MHzオシレータが必要なので移植可能な品種が限られるのと、ディレイ/カウンタマクロセルが
6個必要なのがちょっと重いです。

Table: リソースまとめ {#tbl:idea1-resources}

|        Type        | Count |               Note               |
|:------------------:|:-----:|:--------------------------------:|
|      2MHz OSC      |   1   |         source of 800KHz         |
|   **25MHz OSC**    |   1   | for precise 400/800ns generation |
|  Both Edge Detect  |   1   |      2MHz->4MHz OSC doubler      |
|      CNT/DLY       | **6** |     2x One-shot, 4x Counter      |
|     Pipe Delay     |   1   |             14 depth             |
|        DFF         |   1   |             for LFSR             |
|    2-input LUT     |   1   |              1x XOR              |
|    3-input LUT     |   2   |            2x 2to1MUX            |
| Digital Output pin |   1   |              1x P-P              |

## おまけ：応用例として考えられるもの

この回路ブロックを利用したいくつかの派生が考えられます。

### 応用(1)：疑似乱数発生器の変更・拡張

- 疑似乱数発生器のビット長を増やす・減らす

例示した疑似乱数器は15ビット長でしたが、15ビットにしたのは16ビット近辺では必要なXORブロック
とDFFの個数がともに最小だからです。また、15ビット長の場合は引き出すべきタップが最上位とその直下の
組み合わせだったので回路を単純化できました。

この次にシンプルなのは17段です。タップ17と14を使うのでXORは1個のまま、DFFを2段追加します。
以下に16ビット長近辺で追加部品が比較的少ない組み合わせの一覧を示します。

| Bit width | Pipe Delay Depth |      Taps      |  DFF  |    XOR     |
|:---------:|:----------------:|:--------------:|:-----:|:----------:|
|    11     |        9         |    2(11,9)     |   2   | 1x 2-input |
|   (15)    |        14        |    2(15,14)    |   1   | 1x 2-input |
|    17     |        14        |    2(17,14)    |   3   | 1x 2-input |
|    20     |        16        |    2(17,20)    |   4   | 1x 2-input |
|    24     |        16        | 4(24,23,22,17) | **8** | 3x 2-input |

### 応用(2)：通信プロトコル生成器だけ使う

- ChipSelect/Data/Clock入力を用意して、ビット列を外部から入力させる

巷にあふれるNeopixel応用記事では大抵マイコンをタイミング生成にまで使っていますが、
マイコンで400/800/1250nsを精度よく出そうとすると高い動作周波数(10MHzかそれ以上)が必要です。
一方例示した通信プロトコル生成器を利用し、マイコンからは800KHzクロックとそれに同期したデータ列
を入力すれば細かい方のタイミング生成について考えずに済みます。800Kが精度良く出せない場合や
1MHzにしたい場合でもディレイマクロセルの設定分解能は40nsなので微調整可能です。
