# LUX.OpenEXR

[English](../README.md) | [日本語](README.md)

LUX コレクションにおける OpenEXR ライブラリの枠。**現時点で本リポジトリが含むのは `AcademySoftwareFoundation/openexr` [2] の上流ソースを同梱したものだけであり、Delphi ユニットは 1 つも存在しない**。したがって EXR 高ダイナミックレンジ画像フォーマット [1] の Object Pascal バインディングは未着手である。本文書は、実際に存在するものと、将来のバインディングが従うべき技術的事実を記録する。

## 利用ライブラリ

* [**LUX**](https://github.com/LUXOPHIA/LUX) ：予定される依存 —— 本リポジトリのコードはまだ利用していないが、将来の Delphi バインディングはその binary16 型 `THalf` の上に構築される想定である（§2.1・§4）。

## 1. 概要

作業ツリー全体は `.gitattributes`、`.gitignore`、および 1 つの同梱ディレクトリから成る。

* **`：AcademySoftwareFoundation/openexr/`** — 上流の OpenEXR リポジトリを `git subtree` として取り込んだもの。`src/lib/OpenEXRCore/openexr_version.h` はバージョン **4.0.0**（開発スナップショット）を報告する。上流の 5 ライブラリ — `OpenEXRCore`（C API）・`OpenEXR`（C++ API）・`OpenEXRUtil`・`Iex`（例外）・`IlmThread`（スレッド） — に加え、コマンドラインツール（`exrheader`・`exrinfo`・`exrmaketiled`・`exrmultipart`・`exrmetrics` ほか）、Python ラッパー、CMake および Bazel のビルドシステム、同梱サードパーティコーデック `external/deflate`（libdeflate）と `external/OpenJPH`（HTJ2K）を含む。
* **Delphi 由来の成果物は皆無** — リポジトリのどこにも `*.pas`・`*.dpr`・`*.dproj`・`*.inc` は存在しない。

確認された帰結として、半精度ピクセル型はこのツリーの一部では *ない*。上流 OpenEXR は `half` を **Imath** から得ており（`cmake/OpenEXRSetup.cmake` の `find_package(Imath 3.1)`。無い場合は Imath リポジトリ [5] から自動取得される）、ゆえに Delphi バインディングは独自の binary16 型を用意しなければならない。それが [LUX](https://github.com/LUXOPHIA/LUX) 基底ライブラリにおける `THalf` の役割である。

## 2. 技術的背景

### 2.1. 半精度浮動小数点数

EXR 本来のピクセル型は 16 ビットの *half* 浮動小数点数、すなわち IEEE 754 binary16 [6] である。符号 1 ビット $s$、指数 5 ビット $e$、仮数 10 ビット $m$。正規化数（$1 \le e \le 30$）では

```math
v = (-1)^s \, 2^{\,e-15} \left( 1 + \frac{m}{1024} \right)
\qquad \text{(2.1)}
```

非正規化数（$e = 0$, $m \ne 0$）では

```math
v = (-1)^s \, 2^{-14} \, \frac{m}{1024}
\qquad \text{(2.2)}
```

であり、ここから、あらゆる変換ルーチンが再現しなければならない限界値が導かれる。

| 量 | 値 | 意味 |
|:---|:---|:---|
| $\varepsilon$ | $2^{-10} \approx 9.7656 \times 10^{-4}$ | $1$ と次の表現可能な値との差 |
| 最小の正規化数 | $2^{-14} \approx 6.1035 \times 10^{-5}$ | (2.1) の $e = 1$, $m = 0$ |
| 最小の非正規化数 | $2^{-24} \approx 5.9605 \times 10^{-8}$ | (2.2) の $m = 1$ |
| 最大の有限値 | $65504$ | (2.1) の $e = 30$, $m = 1023$ |

binary16 から binary32 への拡張は無損失であるため、算術は通常 `Single` で行い、最近接偶数丸めで戻し、$\pm\infty$ へ飽和させる。これは上流の参照実装 `half` クラス、および LUX の `THalf` が採る方式である。

### 2.2. EXR ファイルフォーマット

EXR ファイルは任意個の名前付きチャンネル（典型的には `R`・`G`・`B`・`A`）を格納し、その標本型は `HALF`・`FLOAT`（binary32）・`UINT` のいずれかであり、レイアウトは **スキャンライン** か **タイル** のいずれかである。タイル形式のファイルはさらにミップマップまたはリップマップのピラミッドを持ちうるほか、1 ファイルが複数パートおよびディープ（ピクセルごとの標本数を持つ）データを保持できる [3]。ピクセルブロックはフォーマットのコーデックのいずれかで独立に圧縮される — 可逆（`RLE`・`ZIP`・`ZIPS`・`PIZ`）または非可逆（`PXR24`・`B44`・`B44A`・`DWAA`・`DWAB`、および近年のバージョンでは `HTJ2K`）[1]。

### 2.3. どの API を束ねるべきか

`OpenEXRCore` は純粋な C ライブラリであり、その公開面は 15 個のヘッダ `openexr.h`・`openexr_context.h`・`openexr_part.h`・`openexr_attr.h`・`openexr_std_attr.h`・`openexr_chunkio.h`・`openexr_decode.h`・`openexr_encode.h`・`openexr_coding.h`・`openexr_compression.h`・`openexr_errors.h`・`openexr_base.h`・`openexr_debug.h`・`openexr_config.h`・`openexr_version.h` である。明示的なコンテキストハンドルとエラーコード返却の規約を持ち、例外もテンプレートも用いない C であるため、これが Delphi にとって現実的なバインディング対象となる。C++ の `OpenEXR` ライブラリは Object Pascal から直接呼び出せない。

## 3. アーキテクチャ

上流のモジュール階層と、Delphi 側が入るべき空の枠：

```
：AcademySoftwareFoundation/openexr のモジュール階層
（親は、その下に並ぶ子の上に構築される）

・exrheader / exrinfo / exrmaketiled / exrmultipart / … ･･･ (src/bin)
  ┗・OpenEXRUtil                                        ･･･ C++ ヘルパ
     ┗・OpenEXR                                         ･･･ C++ API
        ┗・OpenEXRCore                                  ･･･ C API
           ┣・Iex
           ┣・IlmThread
           ┗・external/deflate, external/OpenJPH        ･･･ libdeflate・HTJ2K

・外部依存 — ここには同梱されていない
  ┗・Imath ≥ 3.1                                       ･･･ half, Box2i, V3f

・Delphi バインディングユニット — 本リポジトリには未だ存在しない
  ┗・binary16（THalf）とタイル画像（TLuxImage）層は LUX 基底ライブラリにある。
```

ファイル構成：

```
・LUX.OpenEXR/
  ┣・.gitattributes                    ･･･ Git LFS 規則：*.png, *.dll, *.exr
  ┣・.gitignore                        ･･･ Delphi のビルド生成物
  ┗・：AcademySoftwareFoundation/      ･･･ 上流（git subtree）— 編集禁止
     ┗・openexr/                       ･･･ 4.0.0 開発スナップショット
        ┣・src/lib/OpenEXRCore/        ･･･ C API（バインディング対象・§2.3）
        ┣・src/lib/OpenEXR/            ･･･ C++ API
        ┣・src/lib/{OpenEXRUtil,Iex,IlmThread}/
        ┣・src/bin/                    ･･･ exrheader, exrinfo, exrmaketiled, …
        ┣・src/wrappers/python/        ･･･ Python バインディング
        ┣・external/{deflate,OpenJPH}/ ･･･ 同梱コーデック
        ┣・cmake/ , CMakeLists.txt , MODULE.bazel , BUILD.bazel
        ┗・docs/ , CHANGES.md , LICENSE.md（BSD-3-Clause）
```

## 4. 現状と使い方

本リポジトリから `uses` できるものは未だ何もない。Delphi プロジェクトへコンパイル単位を一切提供しないためである。現在あるのは参照実装のソースそのものであり、フォーマットとその C API を作業空間から離れることなく参照し — やがて移植 — できるよう、ツリー内に保持されている。

バインディングが載る Delphi 側の土台は、§2.1 をそのまま実装する `THalf` として、[LUX](https://github.com/LUXOPHIA/LUX) 基底ライブラリに既に用意されている。

```pascal
uses LUX.D1.Half;

var
   H :THalf;
begin
     H := 1.5;                                             // Single → binary16（暗黙）
     H := H * H + 0.25;                                    // Single へ拡張し丸めて戻す
     Assert( HalfToSingle( SingleToHalf( 2.5 ) ) = 2.5 );   // 厳密に表現可能
     Assert( HALF_MAX = 65504.0 );                          // 最大の有限値・§2.1
end;
```

[OpenEXR](https://github.com/LUXOPHIA/OpenEXR) ワークスペースリポジトリは、本ライブラリを LUX とサンプル `.exr` 画像とともにまとめており、読み書き機能の開発はそこで進められる。

## 5. ビルドと必要環境

* **Delphi 側** — コンパイルするものが無く、したがって要件も無い。
* **同梱した C++ 側** — 本コレクションのいかなる Delphi プロジェクトにおいても、これをビルドする必要は *ない*。もしビルドする場合、上流は CMake ≥ 3.14（または Bazel）、C++17 コンパイラ、および **Imath ≥ 3.1** を要求する。Imath は未インストールであれば `cmake/OpenEXRSetup.cmake` が Imath リポジトリ [5] から取得する。libdeflate と OpenJPH は `external/` に同梱されている。
* **Git LFS** — `.gitattributes` が `*.png`・`*.dll`・`*.exr` を Git LFS 経由に振り分けるため、Git LFS なしのチェックアウトではそれらがポインタファイルになる。
* **同梱ツリーを編集しないこと。** `：AcademySoftwareFoundation/openexr/` は上流の `git subtree` であり、ローカルの改変は次回取り込み時に失われ、subtree のマージを壊す。ライセンスは Contributors to the OpenEXR Project による BSD-3-Clause（`LICENSE.md`）である。

## 6. 参考文献

1. [*OpenEXR*](https://openexr.com/) — フォーマットと参照実装の公式サイト。
2. [*AcademySoftwareFoundation/openexr*](https://github.com/AcademySoftwareFoundation/openexr) — ここに同梱した上流リポジトリ。
3. [*OpenEXR File Layout*](https://openexr.com/en/latest/OpenEXRFileLayout.html) — ディスク上の構造の技術文書。
4. [*Reading and Writing Image Files with the C-language API*](https://openexr.com/en/latest/OpenEXRCoreAPI.html) — Delphi バインディングが移植すべき C API。
5. [*AcademySoftwareFoundation/Imath*](https://github.com/AcademySoftwareFoundation/Imath) — 上流の `half` 型の提供元。
6. [*Half-precision floating-point format*](https://en.wikipedia.org/wiki/Half-precision_floating-point_format) — IEEE 754 binary16。

## 💖 [Embarcadero](https://www.embarcadero.com/jp/) [**Delphi**](https://www.embarcadero.com/jp/products/delphi)
ネイティブなクロスプラットフォームアプリを開発するための統合開発環境（ＩＤＥ）。
### Free Download: [**Delphi** Community Edition](https://www.embarcadero.com/jp/products/delphi/starter)
