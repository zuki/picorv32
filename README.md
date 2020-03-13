
PicoRV32 - サイズ最適化RISC-V CPU
======================================

PicoRV32は、[RISC-V RV32IMC命令セット](http://riscv.org/)を実装したCPUコア
です。RV32E、RV32I、RV32IC、RV32IM、RV32IMCのいずれかのコアとして構成でき、
オプションとしてビルトイン割り込みコントローラを持つことができます。

ツール（gcc、binutilsなど）は、[RISC-VのWebサイト](https://riscv.org/software-status/)から入手できます。
PicoRV32にバンドルされている例では、さまざまなRV32ツールチェーンが
`/opt/riscv32i[m][c]`にインストールされていると想定されています。
詳細については、以下の[ビルド手順](#RV32Iツールチェインの構築)を
参照してください。

PicoRV32は、（MITライセンスまたは2条項BSDライセンスと類似したライセンスで
ある）[ISCライセンス](http://en.wikipedia.org/wiki/ISC_license)の下で
ライセンスされたフリーのオープンハードウェアです。

#### 目次

- [機能および代表的なアプリケーション](#機能および代表的なアプリケーション)
- [リポジトリのファイル](#リポジトリのファイル)
- [Verilogモジュールパラメタ](#Verilogモジュールパラメタ)
- [CPIパフォーマンス](#CPIパフォーマンス)
- [PicoRV32ネイティブメモリインターフェース](#PicoRV32ネイティブメモリインターフェース)
- [PCPI: Picoコプロセッサインターフェース](#PCPI-Picoコプロセッサインターフェース)
- [IRQ処理用カスタム命令](#IRQ処理用カスタム命令)
- [RV32Iツールチェインの構築](#RV32Iツールチェインの構築)
- [PicoRV32のバイナリとnewlibのリンク](#PicoRV32のバイナリとnewlibのリンク)
- [評価: ザイリンクス7-シリーズFPGAでのタイミングと使用率](#評価-ザイリンクス7-シリーズFPGAでのタイミングと使用率)


機能および代表的なアプリケーション
---------------------------------

- 小面積 (ザイリンクス7シリーズアーキテクチャで750-2000 LUT)
- 高f<sub>max</sub> (ザイリンクス7シリーズFPGAで250-450 MHz)
- ネイティブメモリインターフェースとAXI4-liteマスタで選択可能
- オプションのIRQサポート (簡単なカスタムISAを使用)
- オプションのコプロセッサインターフェース

このCPUは、FPGAデザインおよびASICの補助プロセッサとして使用されることを
意図しています。 f<sub>max</sub>が高いため、CDC（Clock Domain Crossing)なしに
既存のほとんどの設計に統合できます。低周波数で動作させた場合、
タイミングスラックが大きいため、タイミングクロージャを損なうことなく
設計に追加できます。

さらにサイズを小さくするために、命令`RDCYCLE[H]`、`RDTIME[H]`、`RDINSTRET[H]`と
レジスタ`x16`..`x31`のサポートを無効にすることで、プロセッサをRV32Eコアに
変換することができます。

さらに、レジスタファイルの実装でデュアルポートとシングルポートを選択する
ことができます。前者はパフォーマンスを向上させ、後者はコアを小さくします。

*注記: 多くのFPGAのように内部メモリリソースにレジスタファイルを実装する
アーキテクチャでは、16個の上位レジスタを無効にしたり、デュアルポート
レジスタファイルを無効にしても、コアサイズをさらに小さくする
ことはできません。*

コアには3つのバリエーション、`picorv32`、`picorv32_axi`、`picorv32_wb`が
あります。`picorv32`は、シンプルなネイティブメモリインターフェイスを提供
するものであり、シンプルな環境で使いやすいコアです。`picorv32_axi`は、
AXI-4 Liteマスタインターフェイスを提供するものであり、AXI標準を使用している
既存のシステムに容易に組み込むことができます。`picorv32_wb`は、Wishbone
マスタインターフェイスを提供するものです。

ネイティブメモリインターフェイスとAXI4をブリッジするために、独立したコア`picorv32_axi_adapter`が提供されています。このコアは、1つ以上のPicoRV32
コアとローカルRAM、ROM、メモリマップトペリフェラルを持ち、各コンポーネント
間はネイティブインターフェイスで、外部とはAXI4で通信するカスタムコアの
作成に使用することができます。

オプションのIRQ機能は、外部からのイベントに反応したり、フォールト
処理を実装したり、大規模ISAからの命令をキャッチしてソフトウェアで
エミュレートしたりすることに使用することができます。

オプションのPicoコプロセッサインターフェイス（PCPI）は、外部コプロセッサによる
非分岐命令の実装に使用することができます。 このパッケージにはM標準拡張命令である
`MUL[H[SU|U]]`と`DIV[U]/REM[U]`を実装するPCPIコアの実装が含まれています。

リポジトリのファイル
------------------------

#### README.md

あなたが今読んでいるファイルです。

#### picorv32.v

このVerilogファイルには次のVerilogモジュールが含まれています。

| モジュール               | 説明                                                           |
| ------------------------ | -------------------------------------------------------------- |
| `picorv32`               | PicoRV32 CPU                                                   |
| `picorv32_axi`           | AXI4-Liteインターフェースを持つPicoRV32 CPU                    |
| `picorv32_axi_adapter`   | PicoRV32メモリインターフェースからAXI4-Liteへのアダプタ        |
| `picorv32_wb`            | Wishboneマスタインターフェースを持つPicoRV32 CPU               |
| `picorv32_pcpi_mul`      | `MUL[H[SU\|U]]` 命令を実装したPCPIコア                         |
| `picorv32_pcpi_fast_mul` | シングルサイクル乗算器を使用した`picorv32_pcpi_fast_mul`       |
| `picorv32_pcpi_div`      | `DIV[U]/REM[U]`命令を実装したPCPIコア                          |

このファイルをあなたのプロジェクトにコピーしてください。

#### Makefileとテストベンチ

基本的なテスト環境です。`make test`を実行すると、標準構成で標準テストベンチ
（`testbench.v`）を実行します。他にも別のテストベンチと構成があります。
詳細については、Makefileの`test_*`ターゲットを参照してください。

`make test_ez`を実行すると、外部ファームウェアの.hexファイルを必要としない
非常に簡単なテストベンチである`testbench_ez.v`を実行します。これは、RISC-V
コンパイラツールチェーンが利用できない環境で役立つでしょう。

*注記: テストベンチはIcarus Verilogを使用しています。 ただし、Icarus Verilog
0.9.7（本文書執筆時点の最新リリース）には、テストベンチの実行を妨げるバグが
あります。テストベンチは、Icarus Verilogのgithubの最新マスタにアップグレード
して実行してください。*

#### firmware/

簡単なテスト用のファームウエアです。`tests/`配下の基本的なテストとCコードを
実行し、IRQの処理と乗算器PCPIコアをテストします。

`firmware/`にあるすべてのコードはパブリックドメインです。 使用できるものは
何でもコピーして使ってください。

#### tests/

[riscv-tests](https://github.com/riscv/riscv-tests) による
簡単な命令レベルのテストです。

#### dhrystone/

Dhrystoneベンチマークを実行するもう一つの簡単なテスト用ファームウェアです。

#### picosoc/

メモリマップトSPIフラッシュから直接コードを実行できるPicoRV32を使った
簡単なSoCの例です。

#### scripts/

様々な（合成）ツールとハードウェアアーキテクチャ用のスクリプトと例です。

Verilogモジュールパラメタ
-------------------------

以下のVerilogモジュールパラメーターを使用して、PicoRV32コアを
構成することができます。

#### ENABLE_COUNTERS (default = 1)

このパラメータは、`RDCYCLE[H]`, `RDTIME[H]`, `RDINSTRET[H]`の各命令の
サポートを有効にします。`ENABLE_COUNTERS`にゼロを設定した場合、これらの命令は、
（他の未サポート命令と同様に）ハードウェアトラップを引き起こします。

*注記: 厳密に言うと、`RDCYCLE[H]`, `RDTIME[H]`, `RDINSTRET[H]`の各命令は、
RV32Iコアではオプションではありません。 しかし、アプリケーションコードが
デバッグされ、プロファイルされた後に、これらの命令が見逃されることが
ないようにします。 これらの命令は、RV32Eコアではオプションです。*

#### ENABLE_COUNTERS64 (default = 1)

このパラメータは、`RDCYCLEH`, `RDTIMEH`, `RDINSTRETH`の各命令の
サポートを有効にします。 このパラメーターに0を、`ENABLE_COUNTERS`に
1を設定した場合、`RDCYCLE`, `RDTIME`, `RDINSTRET`の各命令しか使用できません。

#### ENABLE_REGS_16_31 (default = 1)

このパラメータは、レジスタ`x16`..`x31`のサポートを有効にします。 RV32E ISAは
このレジスタを除外します。RV32E ISA仕様では、コードがこのレジスタにアクセス
しようとした際にハードウェアトラップの発行を要求していますが、これは
PicoRV32では実装されていません。

#### ENABLE_REGS_DUALPORT (default = 1)

レジスタファイルは、読み取りポートを2つ持つものと1つ持つものを実装できます。
デュアルポートレジスタファイルはパフォーマンスを少し向上させますが、コアの
サイズも増加させる場合があります。

#### LATCHED_MEM_RDATA (default = 0)

`mem_rdata`がトランザクション後にも外部回路によって安定している場合、
これを1に設定します。デフォルトの設定では、PicoRV32コアは、`mem_rdata`入力が
`mem_valid && mem_ready`であるサイクルでのみ有効であると想定し、値を
内部的にラッチします。

このパラメータは、`picorv32`コアでのみ使用可能です。 `picorv32_axi`コアと
`picorv32_wb` コアでは、これは暗黙的に0に設定されます。

#### TWO_STAGE_SHIFT (default = 1)

デフォルトでは、シフト操作は2段階で実行されます。まず、4ビット単位でシフトし、
次に1ビット単位でシフトします。 これによりシフト操作が高速化されますが、
ハードウェアが追加されます。このパラメータを0に設定すると、2段階シフトが
無効になり、コアのサイズがさらに小さくなります。

#### BARREL_SHIFTER (default = 0)

デフォルトでは、シフト操作は少量ずつ連続的にシフトすることにより実行されます
（上の`TWO_STAGE_SHIFT`を参照）。 このオプションを設定すると、代わりに
バレルシフタが使用されます。

#### TWO_CYCLE_COMPARE (default = 0)

FFステージを追加し、条件付き分岐命令のクロックサイクル遅延を追加
するというコストを払うことで、最長のデータパスを少し緩和します。

*注記: 合成フローでリタイミング（別名「レジスタバランシング」）が有効に
なっている場合、このパラメーターを有効にすると効果的です。*

#### TWO_CYCLE_ALU (default = 0)

ALUデータパスにFFステージを追加し、ALUを使用するすべての命令のクロック
サイクルを追加するというコストを払うことでタイミングを改善します。

*注記: 合成フローでリタイミング（別名「レジスタバランシング」）が有効に
なっている場合、このパラメーターを有効にすると効果的です。*

#### COMPRESSED_ISA (default = 0)

RISC-V縮小命令セットのサポートを有効にします。

#### CATCH_MISALIGN (default = 1)

0に設定すると、非アライメントメモリアクセスを捕捉するための回路を無効にします。

#### CATCH_ILLINSN (default = 1)

0に設定すると、不正命令を捕捉するための回路を無効にします。

このオプションを0に設定しても、コアは`EBREAK`命令はトラップします。
IRQが有効な場合、`EBREAK`は通常IRQ 1をトリガします。このオプションを
0に設定すると、`EBREAK`は割り込みをトリガすることなくプロセッサを
トラップします。

#### ENABLE_PCPI (default = 0)

1に設定すると、Picoコプロセッサインターフェース（PCPI）を有効にします。

#### ENABLE_MUL (default = 0)

このパラメタは内部的にPCPIを有効化し、`MUL[H[SU|U]]`を実装する
`picorv32_pcpi_mul`コアをインスタンス化します。外部PCPIインターフェースは
ENABLE_PCPIが同時に設定されている場合にのみ機能します。

#### ENABLE_FAST_MUL (default = 0)

このパラメタは内部的にPCPIを有効化し、`MUL[H[SU|U]]`を実装する
`picorv32_pcpi_fast_mul`コアをインスタンス化します。外部PCPIインターフェースは
ENABLE_PCPIが同時に設定されている場合にのみ機能します。

ENABLE_MULとENABLE_FAST_MULが同時にセットされた場合は、ENABLE_MULは無視され、
高速乗算コアがインスタンス化されます。

#### ENABLE_DIV (default = 0)

このパラメタは内部的にPCPIを有効化し、`DIV[U]/REM[U]`を実装する
`picorv32_pcpi_div`コアをインスタンス化します。外部PCPIインターフェースは
ENABLE_PCPIが同時に設定されている場合にのみ機能します。

#### ENABLE_IRQ (default = 0)

1に設定するとIRQを有効にします（IRQの議論については下の「IRQ処理用
カスタム命令」を参照してください）。

#### ENABLE_IRQ_QREGS (default = 1)

0に設定すると、`getq`と`setq 命令のサポートを無効にします。qレジスタが
ない場合、RISC-V ABIに従い、irqのリターンアドレスはグローバルポインタ
レジスタであるx3（gp）に、IRQビットマスクはスレッドポインタレジスタで
あるx4（tp）に格納されます。通常のCコードから生成されるコードは、これらの
レジスタを扱うことはできません。

ENABLE_IRQを0に設定すると、qレジスタのサポートは常に無効になります。

#### ENABLE_IRQ_TIMER (default = 1)

0に設定すると、`timer`命令のサポートを無効にします。

ENABLE_IRQを0に設定すると、タイマのサポートは常に無効になります。

#### ENABLE_TRACE (default = 0)

`trace_valid`と`trace_data`出力ポートを使用して実行トレースを生成します。
この機能をデモするには、`make test_vcd`を実行してトレースファイルを作成し、
`python3 showtrace.py testbench.trace firmware/firmware.elf`を実行して
トレースファイルをデコードします。

#### REGS_INIT_ZERO (default = 0)

1に設定すると、すべてのレジスタを（Verilogの`initail`ブロックを使用して）
zeroに初期化します。これはformal検証のシミュレーションに便利です。

#### MASKED_IRQ (default = 32'h 0000_0000)

このビットマスクの1のビットは、常に無効であるIRQに対応します。

#### LATCHED_IRQ (default = 32'h ffff_ffff)

このビットマスクの1のビットは、対応するIRQが「ラッチ」されることを示します。
つまり、IRQラインが1サイクルだけhighの場合、割り込みは保留としてマークされ、
割り込みハンドラが呼び出されるまで保留のままになります（「パルス割り込み」
または「エッジトリガ割り込み」と呼ばれます）。

このビットマスクのビットを0に設定すると、割り込みラインを
「レベルセンシティブな」割り込みとして動作するように変換します。

#### PROGADDR_RESET (default = 32'h 0000_0000)

プログラムの開始アドレスです。

#### PROGADDR_IRQ (default = 32'h 0000_0010)

割り込みハンドラの開始アドレスです。

#### STACKADDR (default = 32'h ffff_ffff)

このパラメータの値が0xffffffffと異なる場合、レジスタ`x2`
（スタックポインタ）はリセット時にこの値に初期化されます（他の
すべてのレジスタは初期化されません）。RISC-Vの呼び出し規約では、
スタックポインタは16バイト境界（RV32Iソフトウェア浮動小数点呼び出し規約の
場合は4バイト境界）に揃える必要があることに注意してください。

CPIパフォーマンス
----------------------------------

*手短な注意: このコアは、パフォーマンスではなく、サイズとf<sub>max</sub>に
最適化されています。*

特に明記しない限り、以下の数値は、ENABLE_REGS_DUALPORTが有効で、
1クロックサイクルで要求に対応できるメモリに接続されているPicoRV32に
適用されます。

命令あたりの平均サイクル（CPI）は、コード内の命令の組み合わせに応じて
おおよそ4です。個々の命令のCPI値は、以下の表に記載されています。
「CPI（SP）」カラムは、ENABLE_REGS_DUALPORTを無効にしたコアのCPI値です。

| 命令                 |  CPI | CPI (SP) |
| ---------------------| ----:| --------:|
| direct jump (jal)    |    3 |        3 |
| ALU reg + immediate  |    3 |        3 |
| ALU reg + reg        |    3 |        4 |
| branch (not taken)   |    3 |        4 |
| memory load          |    5 |        5 |
| memory store         |    5 |        6 |
| branch (taken)       |    5 |        6 |
| indirect jump (jalr) |    6 |        6 |
| shift operations     | 4-14 |     4-15 |

`ENABLE_MUL`が有効な場合、`MUL`命令は40サイクルで、`MULH[SU|U]`命令は
72サイクルで実行されます。

`ENABLE_DIV`が有効な場合、`DIV[U]/REM[U]`命令は40サイクルで実行されます。

`BARREL_SHIFTER`を有効にした場合、シフト操作には他のALU操作と同じ時間が
かかります。

以下のdhrystoneベンチマークの結果は、`ENABLE_FAST_MUL`, `ENABLE_DIV`,
`BARREL_SHIFTER`の各オプションを有効にしたコアの値です。

Dhrystoneベンチマーク結果：0.516 DMIPS/MHz（908 Dhrystones/Second/MHz）

Dhrystoneベンチマークの平均CPIは4.100です。

先読みメモリインターフェイスを使用しない場合（通常、最大クロック速度に必要）、
結果は0.305 DMIPS/MHzおよび5.232 CPIに低下します。

PicoRV32ネイティブメモリインターフェース
--------------------------------

PicoRV32のネイティブメモリインターフェイスは、一回に一つのメモリ転送を
実行できるシンプルなvalid-readyインターフェイスです。

    output        mem_valid
    output        mem_instr
    input         mem_ready

    output [31:0] mem_addr
    output [31:0] mem_wdata
    output [ 3:0] mem_wstrb
    input  [31:0] mem_rdata

コアは`mem_valid`をアサートすることでメモリ転送を開始します。
valid信号は、ピアが`mem_ready`をアサートするまでHighを持続します。
コアのすべての出力は、`mem_valid`がhighの間はずっと変化しません。
メモリ転送が命令フェッチの場合、コアは`mem_instr`をアサートします。

#### Read転送

Read転送では、`mem_wstrb`の値は0であり、`mem_wdata`は使用されません。

メモリは、アドレス`mem_addr`を読み取り、`mem_ready`がhighの間
`mem_rdata`でread値を読み取れるようにします。

他にwaitサイクルは必要ありません。メモリ読み取りは、`mem_valid`と同じ
サイクルで`mem_ready`を非同期にhighにする、または、`mem_ready`を定数1と
するような実装をすることができます。

#### Write転送

Write転送では、`mem_wstrb`は0ではなく、`mem_rdata`は使用されません。
メモリは、`mem_wdata`のデータをアドレス`mem_addr`に書き込み、
`mem_ready`をアサートすることで転送を確認します。

`mem_wstrb`の4ビットは、ワードアドレスの4バイトの書き込みイネーブルです。
`0000`, `1111`, `1100`, `0011`, `1000`, `0100`, `0010`, `0001`の8つの値
だけが指定可能です。各々、書き込みなし、32ビット書き込み、上位16ビット書き込み、下位16書き込み、単一バイトの書き込みです。

他にwaitサイクルは必要ありません。メモリは、`mem_valid`と同じサイクルで
`mem_ready`を非同期にhighにする、または、`mem_ready`を定数1と
することで、書き込みを即座に確認できます。

#### 先読みインターフェース

PicoRV32コアは、通常のインターフェイスよりも1クロックサイクル早く次の
メモリ転送に関するすべての情報を提供する「先読みメモリインターフェイス」も
提供します。

    output        mem_la_read
    output        mem_la_write
    output [31:0] mem_la_addr
    output [31:0] mem_la_wdata
    output [ 3:0] mem_la_wstrb

`mem_valid`がhighになる前のクロックサイクルで、このインターフェイスは
`mem_la_read`または`mem_la_write`にパルスを出力し、次のクロックサイクルの
readまたはwriteトランザクションの開始を示します。

*注記: 信号`mem_la_read`, `mem_la_write`, `mem_la_addr`は、PicoRV32コアの
組み合わせ回路で駆動されます。先読みインターフェイスでは先に説明した
通常のメモリインターフェイスよりもタイミングクロージャを達成するのが
難しいかもしれません。*

PCPI: Picoコプロセッサインターフェース
----------------------------------

Picoコプロセッサインターフェース (PCPI)は外部コアによる非分岐命令の
実装に使用することができる。

    output        pcpi_valid
    output [31:0] pcpi_insn
    output [31:0] pcpi_rs1
    output [31:0] pcpi_rs2
    input         pcpi_wr
    input  [31:0] pcpi_rd
    input         pcpi_wait
    input         pcpi_ready

サポートされていない命令が検出された場合、PCPI機能がアクティブになって
いると（上記のENABLE_PCPIを参照）、`pcpi_valid`がアサートされ、命令ワードが
`pcpi_insn`に出力されます。また、`rs1`と`rs2`フィールドがデコードされて
そのレジスタ値が`pcpi_rs1`と`pcpi_rs2`に出力されます。

外部のPCPIコアは、その命令をデコード・実行して、命令の実行が終了したら
に`pcpi_ready`をアサートします。オプションとして、結果値を`pcpi_rd`に
書き込み、`pcpi_wr`をアサートできます。そして、PicoRV32コアは命令の
`rd`フィールドをデコードし、`pcpi_rd`の値を該当するレジスタに書き込みます。

外部のPCPIコアが16クロックサイクル内に命令のackしない場合、不正命令例外が
発生し、対応する割り込みハンドラが呼び出されます。命令の実行にさらに
数サイクルが必要な場合、PCPIコアは、命令を正常にデコードした後、直ちに
`pcpi_wait`をアサートし、それを`pcpi_ready`をアサートするまで継続する
必要があります。これにより、PicoRV32コアが不正命令例外を発生しなくなります。

IRQ処理用カスタム命令
------------------------------------

*注記: PicoRV32のIRQ処理機能は、RISC-Vの特権ISA仕様に準拠していません。
代わりに、少数の非常にシンプルなカスタム命令を使用して、ハードウェア
オーバーヘッドを最小限にするIRQ処理を実装しています。*

以下のカスタム命令は、`ENABLE_IRQ`パラメータ（上記を参照）でIRQが有効に
なっている場合にのみサポートされます。

PicoRV32コアには、32個の割り込み入力を備えた割り込みコントローラーが
組み込まれています。割り込みは、コアの`irq`入力の対応するビットを
アサートすることでトリガできます。

割り込みハンドラーが開始されると、処理される割り込みの割り込み終了
（EOI）信号`eoi`がHighになります。 割り込みハンドラが復帰すると、
`eoi`信号が再びLowになります。

IRQ 0〜2は、次の組み込みの割り込みソースによって内部的にトリガされます。


| IRQ | 割り込みソース                      |
| ---:| ------------------------------------|
|   0 | タイマー割り込み                    |
|   1 | EBREAK/ECALL、または、不正命令       |
|   2 | BUSエラー (非アラインメモリアクセス)|

この割り込みは、PCPIにより接続されているコプロセッサなどの外部ソースからも
トリガされることがあります。

コアには、IRQ処理に使用される4つの32ビットレジスタ`q0 .. q3`が追加されます。
IRQハンドラが呼び出される際、レジスタ`q0`にはリターンアドレスが、
`q1`には処理されるべきすべてのIRQのビットマスクが設定されます。これは、`q1`に
複数のビットが設定されている場合、割り込みハンドラは1回の呼び出しで複数の
IRQを処理する必要があることを意味します。

縮小命令のサポートが有効になっている場合、中断された命令が縮小命令であると、
q0のLSBがセットされます。これは、IRQハンドラが中断された命令をデコードしたい
場合に使用できます。

レジスタ`q2`と`q3`は初期化されず、IRQハンドラでレジスタ値を保存/復元する
際の一時ストレージとして使用できます。

以下のすべての命令は、`custom0`オペコードとしてエンコードされます。
これらすべての命令で、f3とrs2フィールドは無視されます。

これら命令のニーモニックを実装しているGNUアセンブラマクロについては、
[firmware/custom_ops.S](firmware/custom_ops.S)を参照してください。

割り込みハンドラのアセンブララッパーの実装例については
[firmware/start.S](firmware/start.S) を、実際の割り込みハンドラについては
[firmware/irq.c](firmware/irq.c)を各々参照してください。

#### getq rd, qs

この命令はq-レジスタの値を汎用レジスタにコピーします。

    0000000 ----- 000XX --- XXXXX 0001011
    f7      rs2   qs    f3  rd    opcode

例:

    getq x5, q2

#### setq qd, rs

この命令は汎用レジスタの値をq-レジスタにコピーします。

    0000001 ----- XXXXX --- 000XX 0001011
    f7      rs2   rs    f3  qd    opcode

例:

    setq q2, x5

#### retirq

割り込みから復帰します。この命令は`q0`の値をプログラムカウンタにコピーし、
割り込みを再度有効にします。

    0000010 ----- 00000 --- 00000 0001011
    f7      rs2   rs    f3  rd    opcode

例:

    retirq

#### maskirq

"IRQ Mask"レジスタにはマスク（無効化）する割り込みのビットマスクを設定します。
この命令はirqマスクレジスタに新しい値を書き込み、元の値を読み込みます。

    0000011 ----- XXXXX --- XXXXX 0001011
    f7      rs2   rs    f3  rd    opcode

例:

    maskirq x1, x2

プロセッサはすべての割り込みを無効にして処理を開始します。

不正命令割込またはバスエラー割込を無効にしている場合に、不正命令または
バスエラーが発生した場合、プロセッサは停止します。

#### waitirq

割り込みが保留になるまで実行を一時停止します。 保留するIRQの
ビットマスクは`rd`に書き込まれます。

    0000100 ----- 00000 --- XXXXX 0001011
    f7      rs2   rs    f3  rd    opcode

例:

    waitirq x1

#### timer

タイマカウンタを新しい値にリセットします。カウンタはクロックサイクルを
カウントダウンし、1から0に変わる際にタイマ割込をトリガします。カウンタに
0を設定するとタイマは無効になります。カウンタの元の値は`rd`に書き込まれます。

    0000101 ----- XXXXX --- XXXXX 0001011
    f7      rs2   rs    f3  rd    opcode

例:

    timer x1, x2


RV32Iツールチェインの構築
-------------------------------

TL;DR: 次のコマンドを実行して、完全なツールチェーンを構築します。

    make download-tools
    make -j$(nproc) build-tools

[riscv-tools](https://github.com/riscv/riscv-tools)ビルドスクリプトの
デフォルト設定は、任意のRISC-V ISAを対象としたコンパイラ、アセンブラ、
リンカをビルドできますが、ライブラリはRV32GとRV64Gを対象にビルドされます。
以下の手順にしたがい、純粋にRV32I CPUを対象とする完全なツールチェーン
（ライブラリを含む）を構築します。

以下のコマンドは、純粋なRV32Iを対象とするRISC-V GNUツールチェーンと
ライブラリを構築し、`/opt/riscv32i`にインストールします。

    # ビルドに必要なUbuntuパッケージ:
    sudo apt-get install autoconf automake autotools-dev curl libmpc-dev \
            libmpfr-dev libgmp-dev gawk build-essential bison flex texinfo \
	    gperf libtool patchutils bc zlib1g-dev git libexpat1-dev

    sudo mkdir /opt/riscv32i
    sudo chown $USER /opt/riscv32i

    git clone https://github.com/riscv/riscv-gnu-toolchain riscv-gnu-toolchain-rv32i
    cd riscv-gnu-toolchain-rv32i
    git checkout 411d134
    git submodule update --init --recursive

    mkdir build; cd build
    ../configure --with-arch=rv32i --prefix=/opt/riscv32i
    make -j$(nproc)

コマンドはすべて、接頭辞`riscv32-unknown-elf-`を使って命名されます。
これにより、通常のriscv-tools（デフォルトでは、名前接頭辞
`riscv64-unknown-elf-`を使って命名されています）に対応させて簡単に
インストールできます 。

または、PicoRV32のMakefileにあるmakeターゲットの一つを選んで、
`RV32I[M][C]`ツールチェーンを構築することもできます。 上に示したように、
まず、必要なパッケージをすべてインストールする必要があります。次に、
PicoRV32ソースディレクトリで次のコマンドのいずれかを実行します。

| コマンド                          | インストールディレクトリ  | ISA       |
|:---------------------------------------- |:------------------ |:--------  |
| `make -j$(nproc) build-riscv32i-tools`   | `/opt/riscv32i/`   | `RV32I`   |
| `make -j$(nproc) build-riscv32ic-tools`  | `/opt/riscv32ic/`  | `RV32IC`  |
| `make -j$(nproc) build-riscv32im-tools`  | `/opt/riscv32im/`  | `RV32IM`  |
| `make -j$(nproc) build-riscv32imc-tools` | `/opt/riscv32imc/` | `RV32IMC` |

または、単に`make -j$(nproc) build-tools`を実行して、4つすべてのツール
チェーンをビルドおよびインストールします。

デフォルトでは、これらのmakeターゲットを呼び出すたびに、ツールチェーンの
ソースが（再）ダウンロードされます。事前に一回だけ、`make download-tools`を
実行して、ソースを`/var/cache/distfiles/`にダウンロードします。

*注記: これらの手順は、riscv-gnu-toolchainのgit rev 411d134（2018-02-14）用です。*

PicoRV32のバイナリとnewlibのリンク
-----------------------------------------

ツールチェーン（インストール手順については前のセクションを参照）には、
newlib C標準ライブラリが付属しています。

バイナリにnewlibライブラリをリンクするには、リンカースクリプト
[firmware/riscv.ld](firmware/riscv.ld)を使用してください。 このリンカ
スクリプトを使用すると、エントリポイントが0x10000のバイナリが作成されます
（デフォルトのリンカスクリプトには静的なエントリポイントがないため、
プログラムのロード時に、実行時のエントリポイントを決定することができる
適切なELFローダが必要です）。

Newlibには少数のsyscallスタブがあります。これらsyscallの独自実装を作成し、
その実装にプログラムをリンクして、newlibのデフォルトスタブを上書きする
必要があります。その方法については、[scripts/cxxdemo/](scripts/cxxdemo/)の
`syscalls.c`を参照してください。

評価: ザイリンクス7-シリーズFPGAでのタイミングと使用率
-----------------------------------------------------------

以下の評価は、Vivado 2017.3で実行しました。

#### ザイリンクス7シリーズFPGAのタイミング

`TWO_CYCLE_ALU`を有効にした`picorv32_axi`モジュールを、ザイリンクスの
Artix-7T、Kintex-7T、Virtex-7T、Kintex UltraScale、Virtex UltraScaleについて
すべてのスピードグレードで配置・配線しました。バイナリ検索を使用して、
デザインがタイミングを満たす最短クロック周期を見つけました。

[scripts/vivado/](scripts/vivado/)の`make table.txt`を参照してください。

| デバイス                  | デバイス               | 速度グレード | クロック周期（周波数） |
|:------------------------- |:---------------------|:----------:| --------------------:|
| Xilinx Kintex-7T          | xc7k70t-fbg676-2     | -2         |     2.4 ns (416 MHz) |
| Xilinx Kintex-7T          | xc7k70t-fbg676-3     | -3         |     2.2 ns (454 MHz) |
| Xilinx Virtex-7T          | xc7v585t-ffg1761-2   | -2         |     2.3 ns (434 MHz) |
| Xilinx Virtex-7T          | xc7v585t-ffg1761-3   | -3         |     2.2 ns (454 MHz) |
| Xilinx Kintex UltraScale  | xcku035-fbva676-2-e  | -2         |     2.0 ns (500 MHz) |
| Xilinx Kintex UltraScale  | xcku035-fbva676-3-e  | -3         |     1.8 ns (555 MHz) |
| Xilinx Virtex UltraScale  | xcvu065-ffvc1517-2-e | -2         |     2.1 ns (476 MHz) |
| Xilinx Virtex UltraScale  | xcvu065-ffvc1517-3-e | -3         |     2.0 ns (500 MHz) |
| Xilinx Kintex UltraScale+ | xcku3p-ffva676-2-e   | -2         |     1.4 ns (714 MHz) |
| Xilinx Kintex UltraScale+ | xcku3p-ffva676-3-e   | -3         |     1.3 ns (769 MHz) |
| Xilinx Virtex UltraScale+ | xcvu3p-ffvc1517-2-e  | -2         |     1.5 ns (666 MHz) |
| Xilinx Virtex UltraScale+ | xcvu3p-ffvc1517-3-e  | -3         |     1.4 ns (714 MHz) |

#### ザイリンクス7シリーズFPGAの使用率

以下の表に、次の3つのコアを面積最適化で合成した場合のリソース使用率を示します。

- **PicoRV32 (small):** カウンター命令なし、2段シフトなし、外部ラッチの`mem_rdata`、非アラインメモリアクセスと不正命令の捕捉なしの`picorv32`モジュール。

- **PicoRV32 (regular):** デフォルト構成の`picorv32`モジュール。

- **PicoRV32 (large):** PCPI、IRQ、MUL、DIV、BARREL_SHIFTER、COMPRESSED_ISAの各機能を有効にした`picorv32`モジュール。

[scripts/vivado/](scripts/vivado/)の`make area`を参照してください。

| コアの種類         | Slice LUTs | LUTs as Memory | Slice Registers |
|:------------------ | ----------:| --------------:| ---------------:|
| PicoRV32 (small)   |        761 |             48 |             442 |
| PicoRV32 (regular) |        917 |             48 |             583 |
| PicoRV32 (large)   |       2019 |             88 |            1085 |
