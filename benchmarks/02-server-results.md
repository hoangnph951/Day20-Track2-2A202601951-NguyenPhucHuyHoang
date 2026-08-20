# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=2` ·
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 10 | 0.18 | 34000 | 54000 | 54000 | 5.4 | 0.0% |
| 50 | 13 | 0.25 | 29000 | 52000 | 52000 | 8.4 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **1.38x** (28% of linear) |
| P95 latency | **0.96x** |
| Effective concurrency at 50 users | 8.4 vs `--parallel 4` slots (occupancy/slot ratio 2.09) |

**Saturated.** Throughput delivered only 1.38x for 5x the offered load, and effective concurrency (8.4) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

P95 grew no faster than throughput (0.96x vs 1.38x), so this server still has headroom at 50 users.

> **Small sample.** Only 10 requests completed in the
> shorter run, so these percentiles are indicative rather than solid. Note also that
> locust averages only *completed* requests: when the run ends with requests still
> queued, effective concurrency is an **under**-estimate. Trust the throughput-scaling
> row over the concurrency row here, and run longer (`-t 3m`) if you want firmer numbers.

## Your reading

Server đã bão hòa từ khoảng 10 users: effective concurrency 5.4 đã vượt
4 slots. Khi tải tăng 5×, RPS chỉ tăng 1.38×; ở 50 users có 46 request
deferred và busy slots đạt 3.76/4. P95 giảm nhẹ do mẫu nhỏ và chỉ tính
request hoàn tất, không phải còn headroom. Tôi sẽ thử `--parallel 4 → 8`
để giảm queue rồi đo lại RPS/P95, đồng thời theo dõi RAM và context mỗi slot.
