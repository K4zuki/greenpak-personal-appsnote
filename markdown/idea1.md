# NeoPixelをランダムに光らせる

NeoPixelというのは、いわゆるテープLEDによく使われている、シリアル通信で色を変更できる
フルカラー（RGB）LEDのことです。この章を書き始めた時点では、筆者はNeoPixel系のLEDを使ったことが
ありませんでした。秋月やらスイッチサイエンスやらで売っていることも知っているし、品種がだんだん増えてきてるし、
各種マイコン用のライブラリがかなり充実しているし、**マトリクス状に配置するためのキットを買って**
作るところまではしましたが、まだちゃんと試してませんでした。

という個人的な反省を兼ねて、最初はこのRGB LEDを光らせます。GreenPAKの規模ではバッファメモリを
持たせたりするのは難しいので、乱数生成器を内蔵させてランダムっぽく光らせます。

*ここでは"WS2812B"のデータシート(秋月のサイト[^ws2818b-aki]から入手)を参照します。*

[^ws2818b-aki]: http://akizukidenshi.com/catalog/g/gI-07915

## 下調べ：データ転送プロトコル {#sec:study-protocol}

ちゃんとプロトコルから述べているブログが少ないようなので、自分で調べるところからやります
^[ｴｲﾀﾞﾌﾙｰﾄ=ｻﾝのライブラリを{使って|移植して}ハイおしまい、っていうのばかりでした]。

\\xout{とっても読みにくい}データシートを読んでみると、GRBの順で各8ビット、全24ビットのシリアルデータを
書き込み、ストローブとして50us以上"L"にするとLED内に取り込まれるようです。
シリアルデータの1/0はH/Lの割合で判定されます([@fig:data-transfer-time])。
1ビットあたり1250nsなので800KHzのシリアルデータをデューティーを変えながら送り込むことになります。
許容差が+/-300ns[^torelance]あるので950〜1550nsの範囲で周期が変わることは許容されそうです。

[^torelance]: データシートには+/-600nsとあるのですがどちらが正しいんでしょうか。

[neopixel](data/neopixel.yaml){.wavedrom #fig:data-transfer-time}

::: rmnote

Table: Data transfer time(T~H~ + T~L~ = 1250 +/-600ns) {#tbl:data-transfer-time}

| Logic | T~H~ | T~L~ | Torelance | Unit |
|:-----:|:----:|:----:|:---------:|:-----|
|   1   | 800  | 450  |  +/-150   | ns   |
|   0   | 400  | 850  |  +/-150   |      |

:::

\\newpage

## 全体図

作った回路の全体図を示します([@fig:neopixel-overview])。左上がクロック生成器([@sec:clock-generator])、
右側は上段から通信プロトコル生成器([@sec:protocol-generator])、擬似乱数生成器([@sec:lfsr])、
ストローブ生成器([])です。

![全体図](images/idea1/overview.png){#fig:neopixel-overview}

\\newpage

## クロック生成器 {#sec:clock-generator}

この回路では最終的に2つの異なる周波数のクロック源が必要になりました([@tbl:clock-source])。
このうち800KHzはOSC1から生成しました。400/800nsの生成には高周波数が必要なのでOSC2(25MHz)を使いました。

`800KHz`を得るために原発振2MHzを2逓倍したあと5分周します。
OUT0(2MHz)の出力に`P-DLY`マクロセルをつなげ、両エッジ検出モードにして立ち上がり・立ち下がりエッジごとに
パルスを出させることで逓倍できます([@tbl:config-progdelay])。
5分周には`DLY/CNT`マクロセルをカウンタとして1個使います([@tbl:config-cnt1])。

Table: 必要なクロック {#tbl:clock-source}

|   Frequency   |                     Purpose                      |
|:-------------:|:------------------------------------------------:|
| 2MHz[^not-2M] |           LFSR/protcol clock(`800KHz`)           |
|     25MHz     | NeoPixel protocol timing generation(400ns/800ns) |

[^not-2M]: SLG46826だと厳密には2.048MHzですが、LED側がタイミング的にゆるいのでこちらの定義もゆるめで行きます

![OSC1周辺回路](images/idea1/clock-gen.png){#fig:osc1-config}

```table
---
markdown: True
caption: "`OSC1`の調整 {#tbl:config-osc1}"
alignment: CCC
witdh: 
    - 0.4
    - 0.3
    - 0.3
---
"Option","Value","Note"
"OSC power mode","Force Power On",""
”Control pin mode”,"Force on",""
```

```table
---
markdown: True
caption: "`P DLY`の調整 {#tbl:config-progdelay}"
alignment: CCC
witdh: 
    - 0.4
    - 0.3
    - 0.3
---
"Option","Value","Note"
"Mode","Both edge delay",""
"Delay value","125 ns",""
```

```table
---
markdown: True
caption: "`CNT1/DLY1`の調整 {#tbl:config-cnt1}"
alignment: CCC
witdh: 
    - 0.4
    - 0.3
    - 0.3
---
"Option","Value","Note"
"Multi-function mode","CNT/DLY",""
"Mode","Reset counter",""
"counter data","4",""
"Clock","Ext. Clk. (From matrix)",""
```

\\newpage

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

PipeDelayはあえて深さ14段にして外付けDFFを1個追加して15段にします。これによってタップ14はPipeDelayの出力、
タップ15が追加したDFFの出力を使うことができます。2ビットLUT(XOR設定)にこれらを入力し、XOR出力をPipeDelayの入力に戻します。

シフトレジスタのクロックは`800KHz`([@tbl:config-osc1])を使います。

```table
---
markdown: True
caption: PipeDelayの調整 {#tbl:config-pipedelay}
alignment: CCC
witdh: 
    - 0.4
    - 0.3
    - 0.3
---
"Option","Value","Note"
"OUT0 PD num","14",""
```

## NeoPixelの通信プロトコル生成器 {#sec:protocol-generator}

プロトコル生成器は2つのブロックに細分化できます。全体図([@fig:neopixel-overview])の
右上部がロジック-パルス変換器([@sec:logic-to-pulse])、右下部が
ストローブ信号生成器([@sec:strobe-pulser])です。

### ロジック-パルス変換器 {#sec:logic-to-pulse}

生成すべきパルス列の特性については、先の通り調査済です([@sec:study-protocol])。このタイミングを生成するために
筆者は再びディレイマクロセルを使いました。今回はワンショットモードにして
クロック入力を`OSC2`、`DLY IN`を`800KHz`に設定しました([@fig:neopixel-pulser])。これらの回路は
ロジック0または1のパルス幅をもったパルス列を生成し続けます。

このパルス列をマルチプレクサに入力しLFSRの出力で切り替えます。マルチプレクサは
LFSRがHを出力している間800nsパルスを通過させ、L期間中は400nsパルスを出させます。

![ロジック−パルス変換](images/idea1/neopixel-pulse-gen.png){#fig:neopixel-pulser}

### ストローブ信号生成器 {#sec:strobe-pulser}

ストローブ信号生成器は3段のカウンタで構成されます([@fig:strobe-pulser])。1段目でLED1個分のタイミング、
2段目で256個まで多段カスケードのためのタイミングを測り、3段目で実際のストローブ信号を
生成します。1個分のデータ書き込み時間がT、連結数がN個、3段目の設定値がM個のとき
`N個 x M回 x T`の期間"L"を出力したあとで`N個 x 1回 x T`の期間だけ"H"を出します。

`N`や`M`の値が小さすぎるとLEDの発色の変化に目が追いつかず真っ白に見えてしまうので、適宜大きめの値を入れて
`N x (M+1) x T`が20〜30msになるように調整すときれいになると思います。

![ストローブ信号生成器](images/idea1/strobe-pulser.png){#fig:strobe-pulser}

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
"Counter data","10",""
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
"Counter data","10000",""
"Edge select","Falling",""
"Clock","Ext. Clk. (From matrix)",""
```

## まとめ

この回路ブロックを作るにはこれだけのリソースが必要です([@tbl:idea1-resources])。
25MHzオシレータが必要なので移植可能な品種が限られるのと、ディレイ/カウンタマクロセルが
6個必要なのがちょっと重いです。

Table: リソースまとめ {#tbl:idea1-resources}

|       Type       | Count |               Note               |
|:----------------:|:-----:|:--------------------------------:|
|     2MHz OSC     |   1   |         source of 800KHz         |
|  **25MHz OSC**   |   1   | for precise 400/800ns generation |
| Both Edge Detect |   1   |      2MHz->4MHz OSC doubler      |
|     CNT/DLY      | **6** |     2x One-shot, 4x Counter      |
|    Pipe Delay    |   1   |             14 depth             |
|       DFF        |   1   |             for LFSR             |
|   2-input LUT    |   1   |              1x XOR              |
|   3-input LUT    |   2   |            2x 2to1MUX            |

## 応用例として考えられるもの

この回路ブロックを利用したいくつかの派生が考えられます。

### 応用(1)：疑似乱数発生器の変更・拡張

- 疑似乱数発生器にシード値入力ピンをつける
- 疑似乱数発生器のビット長を増やす・減らす

例示した疑似乱数器は15ビット長でしたが、15ビットにしたのは16ビット近辺では必要なXORブロック
とDFFの個数がともに最小だからです。また、15ビット長の場合は引き出すべきタップが最上位とその直下の
組み合わせだったので回路を単純化できました。

この次にシンプルなのは20段です。タップ17と20を使うのでXORは1個のまま、DFFを2段追加し、PipeDelayを16段に設定します。

<!--### 応用(2)：3-pinSPIスレーブデバイス化する-->

<!--- ChipSelect/Data/Clock入力を用意して、ビット列を外部から入力させる-->
    <!--- Pipe Delayブロックは単なるFIFOにする-->
        <!--- 注：**リセット付きDFFが8個必要**-->
    <!--- ストローブパルス生成器の立ち上がりでFIFOをリセットする-->
<!--- CSの立ち上がりでオシレータを起動して自動的にLEDに転送させる-->
    <!--- クロックソースはCSの状態で切り替える-->
