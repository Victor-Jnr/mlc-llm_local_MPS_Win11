# MLC-LLM on Windows (eg. AMD RX 6800) — Setup & API Guide

This README walks you from a **fresh Windows 11** machine to a **local OpenAI-compatible API** running on your **AMD Radeon RX 6800** via **Vulkan**. It includes one-time setup, running small/large models, offline controls, and API usage from PowerShell, Python, and Node.js.

> Works great for: Windows 10/11 + AMD (e.g., RX 6800) + Conda.
> GPU path: **Vulkan** (no ROCm on Windows).
> Tested models: `Qwen2.5-0.5B-Instruct-q4f16_1-MLC` (tiny), `Llama-3-8B-Instruct-q4f16_1-MLC` (bigger).


## 1) Prerequisites

* **Windows 11/10** with latest **AMD Adrenalin** driver.
* **Git** (CLI):

  ```powershell
  winget install -e --id Git.Git
  ```
* **PowerShell** (recommended shell).
* Internet for first model download (you can lock things offline later).

---

## 2) Install Conda & Initialize PowerShell

```powershell
winget install -e --id Anaconda.Miniconda3
& "$env:USERPROFILE\miniconda3\Scripts\conda.exe" init powershell
Set-ExecutionPolicy -Scope CurrentUser RemoteSigned -Force
# Close & reopen PowerShell
```

---

## 3) Create the Conda Environment

```powershell
conda create -n mlc python=3.11 -y
conda activate mlc

# Accept Anaconda channels TOS if prompted
conda tos accept --override-channels --channel https://repo.anaconda.com/pkgs/main
conda tos accept --override-channels --channel https://repo.anaconda.com/pkgs/r
conda tos accept --override-channels --channel https://repo.anaconda.com/pkgs/msys2
```

---

## 4) Install Vulkan Loader & Git LFS

```powershell
conda install -c conda-forge vulkan-loader git-lfs -y
git lfs install
```

> This ensures TVM/MLC can locate the Vulkan runtime and handle HF LFS model shards.

---

## 5) Install Windows Toolchains (Clang + GCC/ld)

MLC’s TVM JIT compiles a small GPU runtime. On Windows, you’ll need:

**LLVM/Clang**

```powershell
winget install -e --id LLVM.LLVM
```

**MSYS2 UCRT64 toolchain (gcc/ld)**

```powershell
winget install -e --id MSYS2.MSYS2
# Start Menu → MSYS2 UCRT64 (first run initializes)
# In UCRT64 shell:
pacman -Syu --noconfirm
# If asked to close/reopen after core update, do it, then:
pacman -Syu --noconfirm
pacman -S --noconfirm mingw-w64-ucrt-x86_64-gcc mingw-w64-ucrt-x86_64-binutils mingw-w64-ucrt-x86_64-gcc-libs
```

Add them to **User PATH** (persistent):

```powershell
$llvm  = "C:\Program Files\LLVM\bin"
$mingw = "C:\msys64\ucrt64\bin"
$user  = [Environment]::GetEnvironmentVariable('Path','User')
if ($user -notlike "*$llvm*")  { $user += ";" + $llvm }
if ($user -notlike "*$mingw*") { $user += ";" + $mingw }
[Environment]::SetEnvironmentVariable('Path',$user,'User')

# New PowerShell
where clang
where gcc
where ld
```

> Optional (inside the `mlc` env):
>
> ```powershell
> conda activate mlc
> conda env config vars set CC=clang CXX=clang++
> conda deactivate
> conda activate mlc
> ```

---

## 6) Install MLC-LLM

Nightly builds have the newest device backends and models:

```powershell
python -m pip install --pre -U -f https://mlc.ai/wheels mlc-llm-nightly mlc-ai-nightly
```

(You can switch to stable later: `pip install mlc-llm mlc-ai`)

---

## 7) First Run (Tiny Model)

Use a small model to validate download + JIT:

```powershell
mlc_llm chat HF://mlc-ai/Qwen2.5-0.5B-Instruct-q4f16_1-MLC --device vulkan
```

You should see tokens streaming. Cache directories:

* Weights: `C:\Users\<you>\AppData\Local\mlc_llm\model_weights\hf\...`
* Compiled DLLs: `C:\Users\<you>\AppData\Local\mlc_llm\model_lib\...`

