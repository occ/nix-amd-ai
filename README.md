# nix-amd-ai

AMD AI inference stack for NixOS — packages XRT, XDNA driver plugin, FastFlowLM, and Lemonade with a NixOS module for NPU + ROCm GPU support.

## Packages

| Package | Description | Source |
|---------|-------------|--------|
| `xrt` | Xilinx Runtime for AMD NPU | Built from [Xilinx/XRT](https://github.com/Xilinx/XRT) |
| `xrt-plugin-amdxdna` | XDNA userspace driver plugin | Built from [amd/xdna-driver](https://github.com/amd/xdna-driver) branch `1.7` |
| `fastflowlm` | NPU-optimized LLM runtime | Built from [FastFlowLM](https://github.com/FastFlowLM/FastFlowLM) |
| `lemonade` | OpenAI-compatible local AI server | [lemonade-sdk/lemonade](https://github.com/lemonade-sdk/lemonade) RPM |
| `llama-cpp-rocm` | ROCm-accelerated llama.cpp backend | Built from [ggerganov/llama.cpp](https://github.com/ggerganov/llama.cpp) |
| `llama-cpp-vulkan` | Vulkan-accelerated llama.cpp backend | Built from [ggerganov/llama.cpp](https://github.com/ggerganov/llama.cpp) |
| `benchmark` | Multi-backend benchmark harness | `nix run .#benchmark` |

## Usage

```nix
# flake.nix
inputs.nix-amd-ai.url = "github:noamsto/nix-amd-ai";

# host configuration
{inputs, ...}: {
  imports = [inputs.nix-amd-ai.nixosModules.default];

  hardware.amd-npu = {
    enable = true;
    enableFastFlowLM = true;  # LLM inference on NPU
    enableLemonade = true;    # OpenAI-compatible API server
    enableROCm = true;        # Declaratively wires ROCm GPU backends for Lemonade
    enableVulkan = true;      # Declaratively wires Vulkan GPU backends for Lemonade
    # rocmGfxOverride = "11.0.2";  # Strix Point gfx1150 → gfx1102 fallback (see below)
    lemonade.user = "youruser";
  };

  users.users.youruser.extraGroups = ["video" "render"];
}
```

## Binary cache

Pre-built packages are available via Cachix:

```nix
# flake.nix nixConfig (or nix.settings in your NixOS config)
substituters = ["https://nix-amd-ai.cachix.org"];
trusted-public-keys = ["nix-amd-ai.cachix.org-1:F4OU4vw/lV2oiG6SBHZ+nqjl4EFJuqI4X9A7pvaBmhQ="];
```

## Requirements

- NixOS with kernel >= 6.14 (has `amdxdna` driver built-in)
- AMD Ryzen AI processor with XDNA 2 NPU (Strix Point / Strix Halo)
- User in `video` and `render` groups

## What the module configures

- Kernel params (`iommu.passthrough=0`) and modules (`amdxdna`)
- Udev rules for NPU device access
- PAM limits (unlimited memlock for NPU buffer allocation)
- XRT + plugin merged tree for runtime plugin discovery
- Lemonade systemd service with XRT/FLM/ROCm/Vulkan environment
- Environment variables (`XILINX_XRT`, `XRT_PATH`)
- Declarative backend wiring (both the `lemond` service and direct CLI usage receive the ROCm/Vulkan backend paths automatically)
- `rocmGfxOverride`: set to `"11.0.2"` on Strix Point (gfx1150) to override the GFX version to gfx1102, enabling ROCm support on hardware not yet natively supported by ROCm. Example: `rocmGfxOverride = "11.0.2";`

### Why `enableROCm` / `enableVulkan` matter on NixOS

Lemonade's RPM ships its own `llama-server` binaries for each backend, but they're linked against Linux FHS paths (`/usr/lib`) for `libvulkan.so.1`, `libstdc++.so.6`, etc. On NixOS those libraries are not on the default loader path, so the bundled binaries fail to dlopen and **lemonade silently falls back to CPU** — the server still responds, it just does so at a fraction of GPU speed.

`enableROCm = true` and `enableVulkan = true` replace the bundled binaries with the `llama-cpp-rocm` / `llama-cpp-vulkan` packages built in this flake (correct RPATH via `autoPatchelfHook`) by exporting `LEMONADE_LLAMACPP_{ROCM,VULKAN}_BIN`. The lemonade wrapper persists those paths into `~/.cache/lemonade/config.json` on every launch so both the `lemond` service and ad-hoc CLI invocations pick them up.

If you see `lemonade backends` reporting a backend as `installed` but benchmarks report <5 t/s decode on a small model, you're on CPU — check that the matching `enable*` option is set and the host has been rebuilt.

## Which backend should I use?

All numbers measured on Strix Point (gfx1150, Radeon 890M iGPU, 64 GiB DDR5-5600). Prompt 256 tokens, generation 128 tokens, 3 iterations after 1 warmup.

### Large, prefill-heavy: Gemma-4-26B-A4B-it-GGUF (~15.7 GB, via `llama-bench`)

| Metric | ROCm | Vulkan | Winner |
| ------ | ---- | ------ | ------ |
| Prefill (pp) | 395 t/s | 265 t/s | ROCm (+49%) |
| Decode (tg)  | 10.4 t/s | 13.6 t/s | Vulkan (+31%) |

### Mid-size, chat-shaped: Qwen3.5-9B (same family on all three backends)

| Backend | Model | TTFT (s) | Decode (t/s) |
| ------- | ----- | -------: | -----------: |
| Vulkan (llamacpp:vulkan) | `Qwen3.5-9B-GGUF` (UD-Q4_K_XL) | 1.36 | 12.9 +/- 0.1 |
| ROCm (llamacpp:rocm)     | `Qwen3.5-9B-GGUF` (UD-Q4_K_XL) | 1.85 | 9.6 +/- 0.1 |
| FLM (flm:npu)            | `qwen3.5-9b-FLM`               | 4.17 | 11.9 +/- 4.5 |

Notes: FLM's TTFT is dominated by a one-off NPU compile-to-cache; steady-state decode is the useful number. FLM's GGUF-vs-proprietary format means quantization isn't bit-identical to the llamacpp row, so treat these as same-family, not same-weights.

**Recommendation:**

- **Interactive chat on mid-size models** (7–14B Q4): use **Vulkan**. Wins both TTFT and decode here — Vulkan's low per-dispatch overhead dominates when prefill batches are small.
- **Prefill-heavy workloads on large models** (long-context, RAG, batch on 20B+): use **ROCm**. The rocBLAS GEMM advantage shows up as model size grows.
- **Power-budget / idle-CPU scenarios**: use **FLM/NPU** — decode is competitive with Vulkan and offloads the GPU, but the compile-on-first-load TTFT is noticeable.
- ROCm numbers will improve when ROCm ≥ 7.1 ships native gfx1150 kernels (currently requires gfx1102 override via `rocmGfxOverride`).

Enable all three and let lemonade pick the recipe per model.

## Validation

You can verify that backends are correctly wired by running:

```bash
lemonade backends
```

The output should include both backends as `ready`:

```
+------------------+-------------------------------------------------------+---------+
|     BACKEND      |                         PATH                          | STATUS  |
+------------------+-------------------------------------------------------+---------+
| llamacpp:rocm    | /nix/store/...-llama-cpp-rocm-.../bin/llama-server    | ready   |
| llamacpp:vulkan  | /nix/store/...-llama-cpp-vulkan-.../bin/llama-server  | ready   |
+------------------+-------------------------------------------------------+---------+
```

To run a multi-backend benchmark and detect silent CPU fallbacks:

```bash
nix run .#benchmark -- Gemma-4-26B-A4B-it-GGUF
```

The benchmark exits non-zero if any backend falls below `--min-decode-tps` (default 5 t/s), which reliably indicates a CPU fallback rather than GPU execution.

To directly compare ROCm vs Vulkan on the same model, pass `--backend`. This rewrites `llamacpp.backend` in `~/.cache/lemonade/config.json`, restarts `lemond.service` (via sudo), runs the benchmark, and restores the original config on exit:

```bash
nix run .#benchmark -- --backend rocm   Phi-4-mini-instruct-GGUF
nix run .#benchmark -- --backend vulkan Phi-4-mini-instruct-GGUF
```

If you've already set the backend manually, pass `--no-restart` to skip the sudo restart step.

## CI

- **Build**: All packages built and cached on every push to `main`
- **Update**: Weekly check for upstream releases, auto-creates PR with version bumps
