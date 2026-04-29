---
title: "Local Coding Agent"
date: 2026-04-29T00:00:00Z
---

# Running a Local Coding Agent on Consumer Hardware: pi + Qwen3 on an RTX 4060 Ti
 
Most writeups about local LLM inference start with a disclaimer about needing expensive
hardware. This one does not. What follows is a practical account of getting a capable
coding agent running on a desktop with an RTX 4060 Ti 8GB, an i5-13400F, and 32GB of
DDR4. No cloud. No API keys. No managed endpoints.
 
The stack is pi (badlogic/pi-mono), Qwen3-30B-A3B served by llama.cpp, running on
WSL2 Ubuntu.
 
---
 
## Why This Model on This Hardware
 
Qwen3-30B-A3B is a mixture-of-experts model. The headline number is 30 billion
parameters, but only about 3.6 billion activate per token. That distinction matters
on constrained hardware: the full model weights still need to live somewhere, but
per-token inference cost is closer to a small dense model than a large one.
 
At Q4_K_XL quantization the model weighs in at 17.7GB. An RTX 4060 Ti has 8GB of
VRAM. On paper that does not work. In practice, WSL2 unified memory allows the CUDA
buffer to draw from system RAM beyond the physical VRAM limit. The result at load
time with a 32k context window:
 
```
load_tensors: offloaded 49/49 layers to GPU
load_tensors:   CPU_Mapped model buffer size =   166.92 MiB
load_tensors:        CUDA0 model buffer size = 16722.37 MiB
```
 
All 49 layers on GPU. The 167MB CPU-mapped buffer is negligible. Generation runs at
7.3 tok/s. For a coding agent where you are reading output, not streaming video,
that is workable.
 
---
 
## The Build
 
Building llama.cpp with CUDA on WSL2 Ubuntu has one non-obvious step. The CUDA 12.8
toolkit installs its libraries under a versioned targets path that cmake does not
search by default. The cmake command that actually works:
 
```bash
cmake -B build \
  -DGGML_CUDA=ON \
  -DCMAKE_CUDA_ARCHITECTURES=89 \
  -DCMAKE_CUDA_COMPILER=/usr/local/cuda-12.8/bin/nvcc \
  -DCMAKE_LIBRARY_PATH=/usr/local/cuda-12.8/targets/x86_64-linux/lib
```
 
`-DCMAKE_CUDA_ARCHITECTURES=89` targets Ada Lovelace (sm_89), which is the RTX 4060
Ti's compute capability. Getting this wrong compiles successfully but runs slower.
 
The build produces a wall of `cuda_fp4.hpp` unused parameter warnings. These come
from CUDA 12.8 headers and do not affect the output binary.
 
---
 
## Wiring pi to llama.cpp
 
pi uses an OpenAI-compatible completions API. The non-obvious part for Qwen3 is
thinking mode. There are two separate things here that need to be right:
 
The `api` field in `~/.pi/agent/models.json` must be `openai-completions`, not
`qwen-chat-template`. Setting it to `qwen-chat-template` directly throws:
 
```
No API provider registered for api: qwen-chat-template
```
 
`qwen-chat-template` is a `thinkingFormat` value inside the model's `compat` block,
not a top-level API type. The correct configuration:
 
```json
{
  "providers": {
    "llama-cpp": {
      "baseUrl": "http://localhost:8001/v1",
      "api": "openai-completions",
      "apiKey": "none",
      "models": [
        {
          "id": "qwen3.6-35b-a3b",
          "name": "Qwen3-30B-A3B (local)",
          "reasoning": true,
          "input": ["text"],
          "contextWindow": 32768,
          "maxTokens": 8192,
          "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 },
          "compat": {
            "supportsDeveloperRole": false,
            "supportsReasoningEffort": false,
            "thinkingFormat": "qwen-chat-template"
          }
        }
      ]
    }
  }
}
```
 
The llama.cpp server also needs `--chat-template-kwargs '{"preserve_thinking": true}'`
at launch, otherwise the thinking block is stripped before pi receives the response.
 
When both sides are configured correctly, the raw API response separates the thinking
block into its own field:
 
```json
{
  "content": "Hello! How can I assist you today?",
  "reasoning_content": "Okay, the user asked me to say hello..."
}
```
 
pi displays these separately in the TUI. The thinking block is visible for debugging
without cluttering the conversation output.
 
---
 
## What Actually Works
 
The end-to-end test was straightforward. Running pi in non-interactive mode:
 
```bash
pi -p "Create a Go file named hello.go that prints Hello from pi" \
  --provider llama-cpp \
  --model qwen3.6-35b-a3b
```
 
The model reasoned about the task, called the write tool, produced a valid Go file,
and the result ran correctly with `go run hello.go`.
 
The math test with thinking enabled:
 
```bash
pi -p "What is 17 multiplied by 43, show your reasoning" \
  --thinking high
```
 
The model worked through the problem correctly with visible chain-of-thought before
the final answer.
 
---
 
## What the Numbers Actually Mean
 
A few things worth noting from this setup that do not show up in benchmarks:
 
**The MoE active parameter count is what matters for speed, not the total.**
Qwen3-30B-A3B runs at roughly the same tok/s as a much smaller dense model would
on this hardware. The 30B figure describes memory requirements, not inference cost.
 
**Context size competes with GPU layers for VRAM.** Running at 131k context would
push layers off the GPU and onto CPU, dramatically reducing tok/s. At 32k context
everything fits on GPU. Most coding sessions do not need 131k tokens. Size the
context window to your actual workload, not the model's theoretical maximum.
 
**WSL2 unified memory is not free.** The CUDA buffer exceeding physical VRAM works
because the Windows driver exposes system RAM to WSL2's CUDA runtime. There is
latency when accessing data that is not in the physical VRAM portion. In this
setup it is not a bottleneck at 32k context, but it would become one at larger
context windows or with more aggressive layer counts.
 
**Windows desktop processes consume VRAM before WSL2 starts.** Observed baseline
consumption was 1785 to 1955 MiB depending on what was running. Check `nvidia-smi`
before starting the inference server to know how much VRAM is actually available.
 
---
 
## The Setup in One Paragraph
 
Build llama.cpp with CUDA against the versioned toolkit path, load Qwen3-30B-A3B
at Q4_K_XL with 32k context and `-ngl 99`, configure pi with `openai-completions`
api and `thinkingFormat: qwen-chat-template` in the compat block, set
`preserve_thinking: true` on the server, run pi from your project directory. The
full setup including model download takes an afternoon. The daily workflow after
that is a single command.
