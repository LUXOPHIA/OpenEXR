# LUX.OpenEXR

[English](README.md) | [日本語](ja/README.md)

The OpenEXR library slot of the LUX collection. **At present this repository contains only the vendored upstream sources of `AcademySoftwareFoundation/openexr` [2] — there are no Delphi units in it yet**, so the Object Pascal binding for the EXR high-dynamic-range image format [1] is still to be written. This document records what is actually present and the technical facts a future binding has to respect.

## 利用ライブラリ

* [**LUX**](https://github.com/LUXOPHIA/LUX) ：A planned dependency — no code in this repository uses it yet, but the future Delphi binding is intended to build on its binary16 type `THalf` (§2.1, §4).

## 1. Overview

The entire working tree consists of `.gitattributes`, `.gitignore`, and one vendored directory:

* **`：AcademySoftwareFoundation/openexr/`** — the upstream OpenEXR repository, imported as a `git subtree`. `src/lib/OpenEXRCore/openexr_version.h` reports version **4.0.0** (development snapshot). It contains the five upstream libraries — `OpenEXRCore` (the C API), `OpenEXR` (the C++ API), `OpenEXRUtil`, `Iex` (exceptions) and `IlmThread` (threading) — plus the command-line tools (`exrheader`, `exrinfo`, `exrmaketiled`, `exrmultipart`, `exrmetrics`, …), the Python wrapper, the CMake and Bazel build systems, and the bundled third-party codecs `external/deflate` (libdeflate) and `external/OpenJPH` (HTJ2K).
* **No Delphi artifacts** — no `*.pas`, `*.dpr`, `*.dproj` or `*.inc` file exists anywhere in the repository.

A verified consequence: the half-precision pixel type is *not* part of this tree. Upstream OpenEXR takes `half` from **Imath** (`find_package(Imath 3.1)` in `cmake/OpenEXRSetup.cmake`, auto-fetched from the Imath repository [5] when absent), so a Delphi binding must supply its own binary16 type — the role of `THalf` in the [LUX](https://github.com/LUXOPHIA/LUX) base library.

## 2. Technical Background

### 2.1. Half-precision floating point

The native pixel type of EXR is the 16-bit *half* float, IEEE 754 binary16 [6]: one sign bit $s$, five exponent bits $e$, ten mantissa bits $m$. For normalized values ($1 \le e \le 30$)

```math
v = (-1)^s \, 2^{\,e-15} \left( 1 + \frac{m}{1024} \right)
\qquad \text{(2.1)}
```

and for subnormal values ($e = 0$, $m \ne 0$)

```math
v = (-1)^s \, 2^{-14} \, \frac{m}{1024}
\qquad \text{(2.2)}
```

whence the limits that any conversion routine must reproduce:

| Quantity | Value | Meaning |
|:---|:---|:---|
| $\varepsilon$ | $2^{-10} \approx 9.7656 \times 10^{-4}$ | gap between $1$ and the next representable value |
| smallest normal | $2^{-14} \approx 6.1035 \times 10^{-5}$ | from (2.1) with $e = 1$, $m = 0$ |
| smallest subnormal | $2^{-24} \approx 5.9605 \times 10^{-8}$ | from (2.2) with $m = 1$ |
| largest finite | $65504$ | from (2.1) with $e = 30$, $m = 1023$ |

Widening binary16 to binary32 is lossless, so arithmetic is normally performed in `Single` and rounded back with round-to-nearest-even, saturating to $\pm\infty$ — the strategy of the upstream reference `half` class, and of `THalf` in LUX.

### 2.2. The EXR file format

An EXR file stores an arbitrary set of named channels (typically `R`, `G`, `B`, `A`) whose samples are `HALF`, `FLOAT` (binary32) or `UINT`, in either **scanline** or **tiled** layout; tiled files may additionally carry a mip-map or rip-map pyramid, and a file may hold multiple parts and deep (per-pixel sample-count) data [3]. Pixel blocks are compressed independently with one of the format's codecs — lossless (`RLE`, `ZIP`, `ZIPS`, `PIZ`) or lossy (`PXR24`, `B44`, `B44A`, `DWAA`, `DWAB`, and `HTJ2K` in recent versions) [1].

### 2.3. Which API to bind

`OpenEXRCore` is a pure C library: its public surface is the fifteen headers `openexr.h`, `openexr_context.h`, `openexr_part.h`, `openexr_attr.h`, `openexr_std_attr.h`, `openexr_chunkio.h`, `openexr_decode.h`, `openexr_encode.h`, `openexr_coding.h`, `openexr_compression.h`, `openexr_errors.h`, `openexr_base.h`, `openexr_debug.h`, `openexr_config.h` and `openexr_version.h`. Being C with an explicit context handle, an error-code return convention and no exceptions or templates, it is the practical binding target for Delphi — the C++ `OpenEXR` library is not directly callable from Object Pascal.

## 3. Architecture

Upstream module layering, and the empty slot where the Delphi side belongs:

```
Module layering of                                       ･･･ ASWF/openexr
(a parent is built on the children listed under it)

・exrheader / exrinfo / exrmaketiled / exrmultipart / … ･･･ (src/bin)
  ┗・OpenEXRUtil                                        ･･･ C++ helpers
     ┗・OpenEXR                                         ･･･ C++ API
        ┗・OpenEXRCore                                  ･･･ C API
           ┣・Iex
           ┣・IlmThread
           ┗・external/deflate, external/OpenJPH        ･･･ libdeflate, HTJ2K

・External dependency — NOT vendored here
  ┗・Imath ≥ 3.1                                       ･･･ half, Box2i, V3f

・Delphi binding units — not present in this repository yet
  ┗・THalf (binary16) and TLuxImage layers live in the LUX base library.
```

File layout:

```
・LUX.OpenEXR/
  ┣・.gitattributes                    ･･･ Git LFS rules: *.png, *.dll, *.exr
  ┣・.gitignore                        ･･･ Delphi build artifacts
  ┗・：AcademySoftwareFoundation/      ･･･ upstream git subtree — do not edit
     ┗・openexr/                       ･･･ version 4.0.0 development snapshot
        ┣・src/lib/OpenEXRCore/        ･･･ C API (the binding target, §2.3)
        ┣・src/lib/OpenEXR/            ･･･ C++ API
        ┣・src/lib/{OpenEXRUtil,Iex,IlmThread}/
        ┣・src/bin/                    ･･･ exrheader, exrinfo, exrmaketiled, …
        ┣・src/wrappers/python/        ･･･ Python bindings
        ┣・external/{deflate,OpenJPH}/ ･･･ bundled codecs
        ┣・cmake/ , CMakeLists.txt , MODULE.bazel , BUILD.bazel
        ┗・docs/ , CHANGES.md , LICENSE.md (BSD-3-Clause)
```

## 4. Current Status and Usage

There is nothing to `uses` from this repository yet: it contributes no compilation units to a Delphi project. What exists today is the reference implementation as source, kept in-tree so that the format and its C API can be consulted — and eventually translated — without leaving the workspace.

The Delphi-side foundation the binding will sit on is already in place in the [LUX](https://github.com/LUXOPHIA/LUX) base library, whose `THalf` implements exactly §2.1:

```pascal
uses LUX.D1.Half;

var
   H :THalf;
begin
     H := 1.5;                                             // Single → binary16 (implicit)
     H := H * H + 0.25;                                    // widened to Single, rounded back
     Assert( HalfToSingle( SingleToHalf( 2.5 ) ) = 2.5 );   // exactly representable
     Assert( HALF_MAX = 65504.0 );                          // largest finite value, §2.1
end;
```

The [OpenEXR](https://github.com/LUXOPHIA/OpenEXR) workspace repository packages this library together with LUX and a sample `.exr` image, and is the place where the reader/writer is being developed.

## 5. Building and Requirements

* **Delphi side** — nothing to compile, and therefore no requirement.
* **Vendored C++ side** — building it is *not* necessary for any Delphi project in this collection. Should it be built, upstream requires CMake ≥ 3.14 (or Bazel), a C++17 compiler, and **Imath ≥ 3.1**, which `cmake/OpenEXRSetup.cmake` fetches from the Imath repository [5] when it is not already installed; libdeflate and OpenJPH are bundled under `external/`.
* **Git LFS** — `.gitattributes` routes `*.png`, `*.dll` and `*.exr` through Git LFS, so a checkout without Git LFS yields pointer files for those.
* **Do not edit the vendored tree.** `：AcademySoftwareFoundation/openexr/` is a `git subtree` of upstream; local modifications would be lost on the next import and would break subtree merges. It is licensed BSD-3-Clause (`LICENSE.md`) by the Contributors to the OpenEXR Project.

## 6. References

1. [*OpenEXR*](https://openexr.com/) — official site of the format and its reference implementation.
2. [*AcademySoftwareFoundation/openexr*](https://github.com/AcademySoftwareFoundation/openexr) — upstream repository, vendored here.
3. [*OpenEXR File Layout*](https://openexr.com/en/latest/OpenEXRFileLayout.html) — technical documentation of the on-disk structure.
4. [*Reading and Writing Image Files with the C-language API*](https://openexr.com/en/latest/OpenEXRCoreAPI.html) — the C API that a Delphi binding would translate.
5. [*AcademySoftwareFoundation/Imath*](https://github.com/AcademySoftwareFoundation/Imath) — provider of the upstream `half` type.
6. [*Half-precision floating-point format*](https://en.wikipedia.org/wiki/Half-precision_floating-point_format) — IEEE 754 binary16.

## 💖 [Embarcadero](https://www.embarcadero.com/) [**Delphi**](https://www.embarcadero.com/products/delphi)
Integrated Development Environment (IDE) for Creating Native Cross-Platform Apps.
### Free Download: [**Delphi** Community Edition](https://www.embarcadero.com/products/delphi/starter)
