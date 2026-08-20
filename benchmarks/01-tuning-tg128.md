# 01 - Tune: thread-count sweep

Model `Qwen3.5-0.8B-Q4_K_M.gguf` � host `Windows-AMD64` � llama.cpp `b10488`
CPU: **2 physical � 4 logical** cores � `ngl=99` � metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 12.8 | 95% |
| 2 | 12.5 | 92% |
| 4 | 13.5 | 100% |

**Best**: `-t 4` at 13.5 tok/s
**Slowest tested**: `-t 2` at 12.5 tok/s (1.08x spread)
**Against the physical-core default** (`-t 2`, 12.5 tok/s): 1.08x

Use this in your run:

```bash
LAB_N_THREADS=4 make bench
```

## Your explanation

Đường cong gần như phẳng trong khoảng 12.5–13.5 tok/s và không có knee rõ rệt. Kết quả tốt nhất là 4 threads, đạt 13.5 tok/s, nhanh hơn mặc định 2 threads khoảng 1.08×. Do 25/25 layers đã offload lên Intel UHD qua Vulkan, decode chủ yếu bị giới hạn bởi GPU và shared memory thay vì CPU. Bốn logical threads có thể giúp host scheduling và cấp việc cho GPU tốt hơn một chút. Việc 1 thread nhanh hơn 2 threads cho thấy chênh lệch nhỏ này cũng có thể chứa run-to-run noise.
