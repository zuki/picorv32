
PicoSoC - PicoRV32を使用したSoCの簡単な例
=============================================

![](overview.svg)

コードをSPI Flashから直接実行するPicoRV32の簡単な設計例です。
ASICやFPGAの設計における簡単な制御タスクとしてすぐに使用できます。

Lattice社のiCE40-HX8K Breakoutボードを対象とした実装例が含まれています。

Flashは、0x00000000と0x01000000から始まるメモリ領域にマップされ、SRAMは0x00000000にオーバーレイマッピングされます。 SRAMは単なる小さなスクラッチパッドメモリです（デフォルトは256ワード、すなわち、1kBです）。

リセットベクタは0x00100000であり、Flashメモリ内の1MBの位置に設定されています。

このシステムのファームウェアイメージの構築方法については、添付されている
デモ用ファームウェアとリンカスクリプトを参照してください。

テストベンチを実行するには、`make hx8ksim`または`make icebsim`を実行します
（`testbench.vcd`が作成されます）。

構成ビットストリームとファームウェアイメージの構築、およびiCE40-HX8K Breakout
ボードへのアップロードを行うには、`make hx8kprog`を実行します。

構成ビットストリームとファームウェアイメージの構築、およびiCEBreaker
ボードへのアップロードを行うには、`make icebprog`を実行します。

| ファイル                              | 説明                                                     |
| --------------------------------- | --------------------------------------------------------------- |
| [picosoc.v](picosoc.v)            | 最上位レベルのPicoSoC Verilogモジュール                         |
| [spimemio.v](spimemio.v)          | 外部SPI Flashとインターフェースするメモリコントローラ           |
| [simpleuart.v](simpleuart.v)      | SoCのTX/RXラインに直接接続された簡単なUARTコア                  |
| [start.s](start.s)                | firmware.hex/firmware.bin のアセンブリソース                    |
| [firmware.c](firmware.c)          | firmware.hex/firmware.bin のCソースコード                       |
| [sections.lds](sections.lds)      | firmware.hex/firmware.bin 用のリンカスクリプト                  |
| [hx8kdemo.v](hx8kdemo.v)          | iCE40-HX8K Breakoutボードを対象とするFPGAベースの実装例         |
| [hx8kdemo.pcf](hx8kdemo.pcf)      | iCE40-HX8K Breakoutボード対象の実装におけるPIN制約              |
| [hx8kdemo\_tb.v](hx8kdemo_tb.v)   | iCE40-HX8K Breakoutボード対象の実装用のテストベンチ             |
| [icebreaker.v](hx8kdemo.v)        | iCEBreakerボードを対象とするFPGAベースの実装例                  |
| [icebreaker.pcf](hx8kdemo.pcf)    | iCEBreakerボード対象の実装におけるPIN制約                       |
| [icebreaker\_tb.v](hx8kdemo_tb.v) | iCEBreakerボード対象の実装用のテストベンチ                      |

### メモリマップ:

| アドレス範囲             | 説明                                    |
| ------------------------ | --------------------------------------- |
| 0x00000000 .. 0x00FFFFFF | 内蔵SRAM                                |
| 0x01000000 .. 0x01FFFFFF | 外部シリアルFlash                       |
| 0x02000000 .. 0x02000003 | SPI Flashコントローラ構成レジスタ       |
| 0x02000004 .. 0x02000007 | UARTクロック分周レジスタ                |
| 0x02000008 .. 0x0200000B | UART送受信データレジスタ                |
| 0x03000000 .. 0xFFFFFFFF | メモリマップトユーザペリフェラル        |

実装済み物理SRAMを超える内蔵SRAM領域のアドレスを読み込もうとすると、
該当するシリアルFlashのアドレスから読み込みます。

UART送受信データレジスタの読み込みでは、最後に受信したバイト、または、
受信バッファーが空の場合は-1（32ビットすべてが1）を返します。

UARTクロック分周レジスタには、システムクロック周波数をボーレートで割った
値を設定する必要があります。

設計例（hx8kdemo.v）では、iCE40-HX8K Breakoutボードにある8つのLEDを
使用しており、これらはアドレス0x03000000の32ビットワードの下位バイトに
マップされています。

### SPI Flashコントローラ構成レジスタ:

| ビット位置 | 説明                                               |
| -----: | --------------------------------------------------------- |
|     31 | MEMIO有効化 (reset=1, Bit Bang SPIコマンドを使う場合は0をセット) |
|  30:23 | 予約済み (read 0)                                         |
|     22 | DDR有効化ビット (reset=0): dualSPI                        |
|     21 | QSPI有効化ビット (reset=0): quadSPI                       |
|     20 | CRM有効化ビット (reset=0)                                 |
|  19:16 | Readレイテンシ（ダミー）サイクル (reset=0)                |
|  15:12 | 予約済み (read 0)                                         |
|   11:8 | Bit BangモードにおけるIO出力有効化ビット                  |
|    7:6 | 予約済み (read 0)                                         |
|      5 | Bit Bangモードにおけるチップセレクト (CS) ライン          |
|      4 | Bit Bangモードにおけるシリアルクロックライン              |
|    3:0 | Bit BangモードにおけるIOデータビット                      |

CRM/DDR/QSPIモード用の次の設定が有効:

| CRM | QSPI | DDR | Read Command Byte     | Mode Byte |
| :-: | :--: | :-: | :-------------------- | :-------: |
|   0 |    0 |   0 | 03h Read              | N/A       |
|   0 |    0 |   1 | BBh Dual I/O Read     | FFh       |
|   1 |    0 |   1 | BBh Dual I/O Read     | A5h       |
|   0 |    1 |   0 | EBh Quad I/O Read     | FFh       |
|   1 |    1 |   0 | EBh Quad I/O Read     | A5h       |
|   0 |    1 |   1 | EDh DDR Quad I/O Read | FFh       |
|   1 |    1 |   1 | EDh DDR Quad I/O Read | A5h       |

次のプロット図は各設定の相対パフォーマンスを示しています。

![](performance.png)

使用するSPI Flashのデータシートで、チップがサポートしている構成と
各構成における最大クロック周波数を確認してください。

Quad I/Oモードの場合、SPIマスターでQuad I/Oを有効にする前に、CR1V(Status Register 2)の
QUADフラグ(qe: bit-1)を設定する必要があります。設定は、CR1NVの対応するビットを1回だけ
書き込むか、ブートアップのたびにデバイスファームウェアから書き込むことで
行います（後者の例については、`firmware.c`の`set_flash_qspi_flag()`を
参照してください）。

高速な構成をサポートするには、Lattice iCE40-HX8K Breakoutボードに次の
変更が必要です。(1) 高速読み取りコマンドをサポートするFlashチップに
交換する。(2) Flashチップ上のIO2ピンとIO3ピンをFPGAの（J3の中央付近に
ある）IOピンのT9とT8に接続する。

**MAX1000ではstandard SPIしかサポートしていない(IO2, IO3がプルアップされ、
nSTATSUとDEV CLRnに接続されている)**
