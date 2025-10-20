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
---

## 8) Serve an OpenAI-Compatible API

```powershell
mlc_llm serve HF://mlc-ai/Qwen2.5-0.5B-Instruct-q4f16_1-MLC --device vulkan --host 0.0.0.0 --port 8000
```

**PowerShell API call (recommended):**

```powershell
$body = @{
  model = "HF://mlc-ai/Qwen2.5-0.5B-Instruct-q4f16_1-MLC"
  messages = @(@{ role="user"; content="Say hi in one short sentence." })
  max_tokens = 128
} | ConvertTo-Json -Depth 10

$resp = Invoke-RestMethod -Uri "http://127.0.0.1:8000/v1/chat/completions" `
  -Method POST -ContentType "application/json" -Body $body -TimeoutSec 1800

$resp.choices[0].message.content
```

**Python (OpenAI SDK):**

```python
from openai import OpenAI
client = OpenAI(base_url="http://127.0.0.1:8000/v1", api_key="mlc-llm")

resp = client.chat.completions.create(
    model="HF://mlc-ai/Qwen2.5-0.5B-Instruct-q4f16_1-MLC",
    messages=[{"role": "user", "content": "Summarize MLC-LLM in one line."}],
    max_tokens=200,
)
print(resp.choices[0].message.content)
```

**Node.js (OpenAI SDK):**

```js
import OpenAI from "openai";
const client = new OpenAI({ baseURL: "http://127.0.0.1:8000/v1", apiKey: "mlc-llm" });

const resp = await client.chat.completions.create({
  model: "HF://mlc-ai/Qwen2.5-0.5B-Instruct-q4f16_1-MLC",
  messages: [{ role: "user", content: "Explain this server in one sentence." }],
  max_tokens: 200,
});
console.log(resp.choices[0].message.content);
```

**Streaming (curl.exe, PowerShell):**

```powershell
$body = @{
  model = "HF://mlc-ai/Qwen2.5-0.5B-Instruct-q4f16_1-MLC"
  stream = $true
  messages = @(@{ role="user"; content="Stream a 10-word greeting." })
  max_tokens = 200
} | ConvertTo-Json -Depth 10

curl.exe -N -X POST "http://127.0.0.1:8000/v1/chat/completions" ^
  -H "Content-Type: application/json" ^
  --data-binary "$body"
```

---

## 9) Switch to a Bigger Model (optional)

Re-enable downloads for first run:

```powershell
# session-only:
$env:HF_HUB_OFFLINE = "0"

# compile + chat once (downloads & JIT)
mlc_llm chat HF://mlc-ai/Llama-3-8B-Instruct-q4f16_1-MLC --device vulkan

# then serve:
mlc_llm serve HF://mlc-ai/Llama-3-8B-Instruct-q4f16_1-MLC --device vulkan --host 0.0.0.0 --port 8000
```

---

## 10) Offline

After you’ve downloaded/compiled what you need:

```powershell
# Never download from HF again
setx HF_HUB_OFFLINE 1

# Never JIT-compile new binaries; only use cache
setx MLC_JIT_POLICY READONLY
```

(These apply to **new** terminals.)

Run fully local by pointing to a local model folder:

```powershell
mlc_llm chat "C:\Users\<you>\AppData\Local\mlc_llm\model_weights\hf\mlc-ai\Qwen2.5-0.5B-Instruct-q4f16_1-MLC" --device vulkan
```
---

## 11) One-Click Start Script

Save as `start-mlc-api.bat`:

```bat
@echo off
CALL %USERPROFILE%\miniconda3\Scripts\activate.bat
CALL conda activate mlc

REM ensure compilers are visible
SET "PATH=C:\Program Files\LLVM\bin;C:\msys64\ucrt64\bin;%PATH%"

REM offline? set to 1 after first run
SET HF_HUB_OFFLINE=1

REM pick your model here
SET MLC_MODEL=HF://mlc-ai\Qwen2.5-0.5B-Instruct-q4f16_1-MLC
REM SET MLC_MODEL=HF://mlc-ai\Llama-3-8B-Instruct-q4f16_1-MLC

echo Serving %MLC_MODEL% at http://0.0.0.0:8000
mlc_llm serve %MLC_MODEL% --device vulkan --host 0.0.0.0 --port 8000
```

---

## 12) Troubleshooting

**PowerShell says scripts disabled:**
`Set-ExecutionPolicy -Scope CurrentUser RemoteSigned -Force`

**`conda` not recognized after install:**
Close & reopen PowerShell. Or:
`& "$env:USERPROFILE\miniconda3\Scripts\conda.exe" init powershell`

**TOS error creating env:**
Run the three `conda tos accept ...` commands (see above).

**`mlc_llm chat` fails with “clang not found”:**
Install LLVM and ensure `C:\Program Files\LLVM\bin` is on PATH (User PATH & current session). `where clang` must show it.

**“linker (via gcc) failed” or “ld not found”:**
Install MSYS2 UCRT64 toolchain and ensure `C:\msys64\ucrt64\bin` is on PATH. `where gcc`, `where ld` must show them.

**“Vulkan not found” or no GPU devices:**
Update AMD driver; ensure `vulkan-loader` is installed in the `mlc` env. Reopen terminal.

**Prevent any downloads:**
`setx HF_HUB_OFFLINE 1`

**Prevent any new JIT compiles:**
`setx MLC_JIT_POLICY READONLY`

**OOM or slow on first run:**
Start with a tiny model (0.5B–3B). Close apps using VRAM (browsers/games).
Use quantized builds (e.g., `q4f16_1`).

**Where are caches?**
Weights: `C:\Users\<you>\AppData\Local\mlc_llm\model_weights\hf\...`
Compiled libs: `C:\Users\<you>\AppData\Local\mlc_llm\model_lib\...`

**Open to LAN:**

```powershell
netsh advfirewall firewall add rule name="MLC LLM API 8000" dir=in action=allow protocol=TCP localport=8000
```

Then use `http://<your-ip>:8000/v1/...` from other devices.

---

## 13) Longer Replies (avoid truncation)

Set a high `max_tokens` and a generous timeout:

```powershell
$body = @{
  model      = "HF://mlc-ai/Qwen2.5-0.5B-Instruct-q4f16_1-MLC"
  messages   = @(@{ role="user"; content="Tell me a very long story and finish naturally." })
  max_tokens = 2000
} | ConvertTo-Json -Depth 10

$resp = Invoke-RestMethod -Uri "http://127.0.0.1:8000/v1/chat/completions" `
  -Method POST -ContentType "application/json" -Body $body -TimeoutSec 1800

$resp.choices[0].message.content
$resp.choices[0].finish_reason  # "stop" is ideal; "length" means raise max_tokens
```

---

### That’s it!




