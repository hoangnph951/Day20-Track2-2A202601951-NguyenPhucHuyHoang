# 01 - Measure: latency baseline

Model `Qwen3.5 0.8B` � host `Windows-AMD64` � llama.cpp `b10488`
Settings: `threads=2` `ngl=99` `ctx=2048`
`max_tokens=64` � warm-up discarded
Completed requests: `Q4_K_M` 10/10 � `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 6400 | 919 / 1021 | 75.1 / 80.0 | 5627 / 5844 / 5844 | 13.3 |
| UD-Q2_K_XL | 0.39 | 10924 | 2249 / 2347 | 1011.4 / 1053.4 | 65948 / 68710 / 68710 | 1.0 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **13.30x SLOWER** than `Q4_K_M` here, despite being 0.11 GB smaller. That is a real result, not a mistake: fewer bits only buys speed when decode is limited by memory bandwidth. On a machine that is compute-limited instead � few cores, no GPU offload � the extra dequantization work of a heavily-quantized format can cost more than the bytes it saves. Say which case yours is.

## Your observation

UD-Q2_K_XL nhỏ hơn Q4_K_M 22% (0.39 so với 0.50 GB), nhưng TTFT P50 chậm hơn 2.45×, TPOT chậm hơn 13.47× và decode giảm từ 13.3 xuống 1.0 tok/s. Log xác nhận 25/25 layers đã offload lên Intel UHD qua Vulkan, nên overhead kernel/dequantization Q2 có thể là bottleneck. Chất lượng Q2 kém hơn Q4. Vì vậy Q2 không đáng dùng trên máy này.
