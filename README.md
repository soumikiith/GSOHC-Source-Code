# GSOHC: Global Synchronization Optimization for Heterogeneous Computing

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![LLVM](https://img.shields.io/badge/LLVM-14.x-orange.svg)](https://llvm.org/releases/14.0.0/)
[![DOI](https://img.shields.io/badge/DOI-10.4230%2FLIPIcs.ECOOP.2025.21-green.svg)](https://doi.org/10.4230/LIPIcs.ECOOP.2025.21)
[![Conference](https://img.shields.io/badge/ECOOP-2025-purple.svg)](https://2025.ecoop.org/)
[![Artifact](https://img.shields.io/badge/Artifact-Zenodo-blue.svg)](https://doi.org/10.5281/zenodo.15302892)

**Authors:** Soumik Kumar Basu, Jyothi Vedurada
Department of Computer Science and Engineering, IIT Hyderabad, India

---

## Table of Contents

- [Overview](#overview)
- [Key Results](#key-results)
- [Technical Approach](#technical-approach)
- [Repository Structure](#repository-structure)
- [Requirements](#requirements)
- [Installation](#installation)
  - [Step 1 — Clone this repository](#step-1--clone-this-repository)
  - [Step 2 — Obtain LLVM 14 source](#step-2--obtain-llvm-14-source)
  - [Step 3 — Copy the pass into the LLVM tree](#step-3--copy-the-pass-into-the-llvm-tree)
  - [Step 4 — Register the pass in LLVM's build system](#step-4--register-the-pass-in-llvms-build-system)
  - [Step 5 — Configure and build](#step-5--configure-and-build)
  - [Step 6 — Verify the build](#step-6--verify-the-build)
- [Usage](#usage)
  - [Via the opt tool](#via-the-opt-tool)
  - [Via the Clang plugin interface](#via-the-clang-plugin-interface)
- [Configuration Flags](#configuration-flags)
- [Reproducing Paper Results](#reproducing-paper-results)
- [Citation](#citation)
- [License](#license)
- [Contact](#contact)

---

## Overview

GSOHC is an LLVM 14 compiler optimization pass that improves the performance of CPU-GPU heterogeneous programs by relocating global synchronization barriers and synchronous memory transfers. In CUDA programs, synchronous API calls such as `cudaDeviceSynchronize()` and synchronous `cudaMemcpy()` introduce artificial stalls that prevent the CPU from performing useful computation while the GPU is still executing. GSOHC performs interprocedural dataflow analysis to identify the latest safe program point at which each such barrier can be placed, thereby maximizing CPU-GPU execution overlap.

This repository contains the LLVM pass source code and serves as the artifact for the ECOOP 2025 paper:

> **Global Synchronization Optimization for Heterogeneous Computing**
> Soumik Kumar Basu, Jyothi Vedurada
> ECOOP 2025 — [https://doi.org/10.4230/LIPIcs.ECOOP.2025.21](https://doi.org/10.4230/LIPIcs.ECOOP.2025.21)

---

## Key Results

- Up to **1.9× speedup** over unoptimized code across diverse GPU benchmarks
- Evaluated on **HeCBench** and **PolyBench** GPU benchmark suites
- Validated on three hardware configurations:
  - Ubuntu 22.04 LTS + NVIDIA RTX A4000
  - Ubuntu 22.04 LTS + NVIDIA A100
  - Debian 11 + NVIDIA P100

---

## Technical Approach

GSOHC operates as an LLVM `ModulePass` and proceeds in three phases.

### Phase 1 — Interprocedural Dataflow Analysis

The pass builds a call graph rooted at `main()` and performs a forward dataflow analysis across the entire program. For each instruction, it maintains three sets:

- **Read Set** — variables read from GPU memory after a kernel launch
- **Write Set** — variables written by a GPU kernel
- **Barrier Set** — synchronization instructions (`cudaDeviceSynchronize`, synchronous `cudaMemcpy`) that depend on a preceding GPU computation

GPU kernel invocations are detected by their `_device_stub_` linkage name. The analysis propagates these sets across call boundaries, tracking data dependencies through function parameters using a separate read/write parameter analysis pass.

### Phase 2 — Safe Relocation Point Computation

For each synchronization instruction in the Barrier Set, the pass computes the latest program point at which it can be safely moved. This computation uses:

- **Dominator Tree** and **Post-Dominator Tree** to verify that moving an instruction preserves all-paths reachability guarantees
- **LoopInfo** to prevent unsafe movement across loop boundaries
- A **unification** step that, when multiple control-flow paths converge, selects the earliest dominator of all candidate target points

### Phase 3 — IR Transformation

Once safe relocation points are established, the pass rewrites the LLVM IR:

- **Intra-procedural:** relocates instructions within a function using `moveBefore()`
- **Inter-procedural:** when the optimal relocation point lies in a different function, the pass clones the relevant function, augments its parameter list to carry the operands of the synchronization call, updates all call sites, and inserts the relocated instruction at the target site

---

## Repository Structure

```
GSOHC-Source-Code/
├── CMakeLists.txt          # LLVM plugin build configuration for this pass
├── gsohc_main_analysis.cpp # ModulePass entry point; orchestrates all three phases
├── bb_analysis.h           # Dataflow analysis, safe-point computation, and unification
├── helper_func.h           # Value tracking, set operations, and interprocedural utilities
└── header_files.h          # LLVM framework includes
```

This repository contains only the pass source code. The benchmarks, evaluation scripts, and Docker image used to reproduce the paper results are archived separately on Zenodo — see [Reproducing Paper Results](#reproducing-paper-results).

---

## Requirements

| Dependency | Version |
|---|---|
| LLVM source tree | 14.x (14.0.0 – 14.0.6) |
| CMake | >= 3.20.0 |
| C++ compiler (GCC or Clang) | C++17 or later |
| Ninja (recommended) | any recent version |

> The pass targets the legacy pass manager present in LLVM 14. It is not compatible with the new pass manager or LLVM versions outside the 14.x series without modification.

---

## Installation

### Step 1 — Clone this repository

```bash
git clone https://github.com/soumikiith/GSOHC-Source-Code.git
```

### Step 2 — Obtain LLVM 14 source

If you do not already have an LLVM 14 source tree, clone it and check out the stable release tag:

```bash
git clone https://github.com/llvm/llvm-project.git
cd llvm-project
git checkout llvmorg-14.0.6
cd ..
```

### Step 3 — Copy the pass into the LLVM tree

GSOHC is built as an in-tree LLVM pass. Copy the entire repository into the LLVM Transforms directory:

```bash
cp -r GSOHC-Source-Code llvm-project/llvm/lib/Transforms/gsohc
```

The directory layout inside the LLVM tree should now be:

```
llvm-project/llvm/lib/Transforms/
├── gsohc/
│   ├── CMakeLists.txt
│   ├── gsohc_main_analysis.cpp
│   ├── bb_analysis.h
│   ├── helper_func.h
│   └── header_files.h
├── AggressiveInstCombine/
├── IPO/
└── ...
```

### Step 4 — Register the pass in LLVM's build system

Open `llvm-project/llvm/lib/Transforms/CMakeLists.txt` and add the following line at the end of the file:

```cmake
add_subdirectory(gsohc)
```

### Step 5 — Configure and build

Create a build directory and configure with CMake. The example below builds only the X86 target and uses Ninja for faster compilation:

```bash
cmake -S llvm-project/llvm -B llvm-build \
  -G Ninja \
  -DCMAKE_BUILD_TYPE=Release \
  -DLLVM_TARGETS_TO_BUILD="X86" \
  -DLLVM_ENABLE_PLUGINS=ON

cmake --build llvm-build --target gsohc_opt -j$(nproc)
```

> Building only the `gsohc_opt` target avoids a full LLVM rebuild and is significantly faster. If you need the full LLVM toolchain, omit `--target gsohc_opt`.

The shared library will be produced at:

```
llvm-build/lib/gsohc_opt.so
```

### Step 6 — Verify the build

Confirm the pass is registered and loadable:

```bash
llvm-build/bin/opt -load llvm-build/lib/gsohc_opt.so -help 2>&1 | grep gsohc_llvm
```

Expected output:

```
    --gsohc_llvm                                         - Global Synchronization Optimization in Heterogeneous Computing Pass
```

---

## Usage

### Via the `opt` tool

Run the pass on LLVM bitcode:

```bash
opt -load llvm-build/lib/gsohc_opt.so \
    -gsohc_llvm \
    -o optimized.bc input.bc
```

To emit human-readable IR instead of bitcode:

```bash
opt -load llvm-build/lib/gsohc_opt.so \
    -gsohc_llvm \
    -S -o optimized.ll input.ll
```

To produce LLVM IR from a CUDA source file with Clang before running the pass:

```bash
# Compile CUDA to LLVM IR (host-side only)
clang++ -x cuda --cuda-host-only -emit-llvm -S \
        -o input.ll your_program.cu

# Run the GSOHC pass
opt -load llvm-build/lib/gsohc_opt.so \
    -gsohc_llvm \
    -S -o optimized.ll input.ll
```

### Via the Clang plugin interface

GSOHC registers itself at `PassManagerBuilder::EP_ModuleOptimizerEarly` and runs automatically when the shared library is loaded through Clang. This integrates the pass directly into the standard compilation pipeline:

```bash
clang++ -x cuda --cuda-host-only \
        -Xclang -load -Xclang llvm-build/lib/gsohc_opt.so \
        -O2 -o output your_program.cu
```

**Input requirements:**

- CUDA source compiled with Clang (not nvcc), so that GPU kernel calls appear with `_device_stub_` linkage names in the IR
- Host-side compilation only (`--cuda-host-only`); the pass analyses and transforms the CPU-side IR

**Output:**

- Transformed IR or binary with relocated synchronization barriers
- Diagnostic output on stderr reporting the source-line mapping of each relocated instruction

---

## Configuration Flags

The following preprocessor flags in the source headers control pass behavior. Edit the relevant `#define` directives in `helper_func.h` or `gsohc_main_analysis.cpp` before building:

| Flag | Default | Effect |
|---|---|---|
| `COMPILER_VERBOSE` | off | Print verbose debug output during analysis |
| `FILE_OUTPUT` | off | Write analysis results to CSV files |
| `TRANSFORM` | on | Enable IR transformation (disable for analysis-only mode) |
| `INTER_PROC_TRANSFORM` | on | Enable interprocedural function cloning and parameter augmentation |
| `GPU_READ_WRITE` | off | Enable detailed GPU parameter read/write tracking (experimental) |

After changing any flag, rebuild the target:

```bash
cmake --build llvm-build --target gsohc_opt -j$(nproc)
```

---

## Reproducing Paper Results

The full evaluation artifact — including a pre-built LLVM 14 toolchain, the HeCBench and PolyBench benchmark suites, a Docker image, and all evaluation and plotting scripts — is archived separately on Zenodo:

> **GSOHC Artifact** — [https://doi.org/10.5281/zenodo.15302892](https://doi.org/10.5281/zenodo.15302892)

Refer to the artifact's README for Docker setup instructions and step-by-step benchmark evaluation.

---

## Citation

If you use GSOHC in your research, please cite the following paper:

```bibtex
@inproceedings{basu2025gsohc,
  author    = {Soumik Kumar Basu and Jyothi Vedurada},
  title     = {Global Synchronization Optimization for Heterogeneous Computing},
  booktitle = {39th European Conference on Object-Oriented Programming (ECOOP 2025)},
  series    = {Leibniz International Proceedings in Informatics (LIPIcs)},
  doi       = {10.4230/LIPIcs.ECOOP.2025.21},
  url       = {https://doi.org/10.4230/LIPIcs.ECOOP.2025.21},
  year      = {2025}
}
```

---

## License

This project is distributed under the [MIT License](LICENSE).

---

## Contact

For questions or issues, please open a [GitHub issue](https://github.com/soumikiith/GSOHC-Source-Code/issues) or contact:

**Soumik Kumar Basu** — [cs21resch11004@iith.ac.in](mailto:cs21resch11004@iith.ac.in)
