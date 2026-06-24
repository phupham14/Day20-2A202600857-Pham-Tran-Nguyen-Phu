# Reflection — Lab 20 (Personal Report)

> **Đây là báo cáo cá nhân.** Mỗi học viên chạy lab trên laptop của mình, với spec của mình. Số liệu của bạn không so sánh được với bạn cùng lớp — chỉ so sánh **before vs after trên chính máy bạn**. Grade rubric tính theo độ rõ ràng của setup + tuning của bạn, không phải tốc độ tuyệt đối.

---

**Họ Tên:** Phạm Trần Nguyên Phú
**Cohort:** A20-K1
**Ngày submit:** 2026-06-24

---

## 1. Hardware spec (từ `00-setup/detect-hardware.py`)

- **OS:** Windows 11 Pro (build 26200)
- **CPU:** Intel(R) Core(TM) Ultra 7 255U
- **Cores:** 12 physical / 14 logical
- **CPU extensions:** AVX2 (Intel Alder Lake-U architecture)
- **RAM:** 31.5 GB
- **Accelerator:** Vulkan — Intel(R) Graphics (integrated, UMA)
- **llama.cpp backend đã chọn:** Vulkan
- **Recommended model tier:** Qwen2.5-7B-Instruct

**Setup story**: `detect-hardware.py` sử dụng `wmic` đã bị deprecated trên Windows 11, nên báo sai số core/RAM. Phải fix bằng cách dùng `Get-CimInstance Win32_Processor` để lấy đúng spec rồi ghi thủ công vào `hardware.json`. Cài `llama-cpp-python` qua pip bị lỗi `MAX_PATH` (Windows 260-char limit) — fix bằng cách đổi `TEMP` sang `C:\T`. Download model từ Qwen official repo bị 404 (sharded GGUF) — đổi sang `bartowski/Qwen2.5-7B-Instruct-GGUF` để dùng single-file GGUF. Dùng sẵn llama-bench binary từ Vulkan release package, không cần build từ source.

---

## 2. Track 01 — Quickstart numbers (từ `benchmarks/01-quickstart-results.md`)

Settings: `n_threads=12`, `n_ctx=2048`, `n_batch=512`, `n_gpu_layers=99`.

| Model | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode rate (tok/s) |
|---|--:|--:|--:|--:|--:|
| Qwen2.5-7B-Instruct-Q4_K_M.gguf | 6085 | 846 / 7169 | 150.8 / 173.7 | 10295 / 16692 / 17817 | 6.6 |
| Qwen2.5-7B-Instruct-Q2_K.gguf   | 6880 | 822 / 7433 | 255.7 / 268.8 | 17262 / 23194 / 24403 | 3.9 |

**Một quan sát**: Q4_K_M nhanh hơn Q2_K (6.6 vs 3.9 tok/s — 1.69×), ngược với kỳ vọng. Trên CPU thuần túy, Q2_K nhỏ hơn nhưng cần dequantize phức tạp hơn mỗi bước decode, tạo bottleneck compute. Q4_K_M cân bằng tốt hơn giữa kích thước và chi phí dequantize nên decode nhanh hơn — và chất lượng text cũng tốt hơn. Q2_K chỉ hữu ích khi RAM thực sự thiếu.

---

## 3. Track 02 — llama-server load test

Server: native `llama-server.exe` (Vulkan build), `--parallel 4 --cont-batching`, `--port 8080`.

| Concurrency | Total RPS | TTFB P50 (ms) | E2E P95 (ms) | E2E P99 (ms) | Failures |
|--:|--:|--:|--:|--:|--:|
| 10 | *(xem screenshot 04-locust-10.png)* | | | | |
| 50 | *(xem screenshot 05-locust-50.png)* | | | | |

> **Lưu ý cho grader:** Số liệu locust chỉ xuất ra terminal, không lưu file. Xem file `submission/screenshots/04-locust-10.png` và `05-locust-50.png` để đối chiếu.

**Batching observation** (từ `record-metrics.py` — `benchmarks/02-server-metrics.csv`): peak `llamacpp:n_busy_slots_per_decode` = **3.69** (gần full 4 slots), `requests_deferred` đạt **46** khi concurrency = 50. Điều này cho thấy continuous batching đang hoạt động đúng — server không từ chối request mà queue lại và xử lý theo batch. Tuy nhiên với 7B model ở 6.6 tok/s, throughput thực tế bị giới hạn khoảng 26 tok/s tổng (~0.26 RPS), nên latency E2E rất cao khi có nhiều user đồng thời.

---

## 4. Track 03 — Milestone integration

- **N16 (Cloud/IaC):** stub — localhost only (llama-server chạy trực tiếp, không có k3d/docker-compose)
- **N17 (Data pipeline):** stub — in-memory dict (TOY_DOCS, không có Airflow/batch job)
- **N18 (Lakehouse):** stub — in-memory (không có Delta Lake/Iceberg/SQLite)
- **N19 (Vector + Feature Store):** stub — TOY_DOCS với keyword overlap scoring (không có Qdrant/Feast)

**Nơi tốn nhiều ms nhất** trong pipeline:

- embed: N/A — pipeline dùng keyword matching, không có embedding model (0 ms)
- retrieve: < 1 ms — pure Python in-memory dict scan với 5 documents
- llama-server: ~10,000–20,000 ms per query (toàn bộ latency nằm ở đây)

**Reflection**: Bottleneck hoàn toàn là inference time của LLM — đúng như kỳ vọng. Với 7B model ở 6.6 tok/s và 200 max_tokens, mỗi query mất 15-30 giây. Nếu dùng vector search thật (Qdrant), overhead thêm vào chỉ khoảng 5-20ms — không đáng kể so với inference. Để giảm E2E latency, cần tăng tốc inference (GPU tốt hơn, smaller model, hoặc speculative decoding) — không phải optimize retrieval.

---

## 5. Bonus — The single change that mattered most

**Change:** Khám phá rằng thread count gần như không ảnh hưởng khi dùng Vulkan GPU offload (`-ngl 99`). Thread sweep từ `BONUS-llama-cpp-optimization/benchmarks/thread-sweep.py`:

**Before vs after** (đầy đủ kết quả sweep):

```
t=  1   tg64 =  6.2 tok/s   ← 1 CPU thread, GPU handles all layers
t=  2   tg64 =  6.2 tok/s
t=  6   tg64 =  6.5 tok/s
t= 12   tg64 =  6.6 tok/s   ← 12 physical cores (expected sweet spot per theory)
t= 14   tg64 =  6.6 tok/s
t= 28   tg64 =  6.8 tok/s   ← 2× logical cores (expected to DROP per theory)

Speedup 1 thread → 28 threads: 6.2 → 6.8 tok/s ≈ 1.1× (nearly flat)
```

**Tại sao nó work** (hay đúng hơn — tại sao theory KHÔNG đúng ở đây): Deck dự đoán curve dạng "đỉnh ở physical cores rồi giảm" vì decode là memory-bandwidth-bound trên CPU. Nhưng với `ngl=99`, *tất cả* 32 transformer layers đều chạy trên Vulkan GPU — CPU chỉ làm overhead nhỏ (scheduling, I/O). Bottleneck lúc này là **GPU memory bandwidth** của Intel integrated graphics (~68 GB/s UMA), không phải CPU. Thêm CPU threads không giúp ích gì — và thậm chí không hại vì GPU đã saturated trước. Curve flat từ t=1 đến t=28 là bằng chứng trực tiếp: khi GPU offload đầy đủ, CPU thread count irrelevant với decode throughput.

---

## 6. (Optional) Điều ngạc nhiên nhất

Q2_K chậm hơn Q4_K_M dù nhỏ hơn — và với Vulkan offload, số thread CPU gần như không quan trọng. Hai điều này đều ngược với intuition ban đầu nhưng có lý giải rõ ràng: dequantize cost dominates trên CPU, và GPU bottleneck shifts khi offload đầy đủ.

---

## 7. Self-graded checklist

- [x] `hardware.json` đã commit
- [x] `models/active.json` đã commit
- [x] `benchmarks/01-quickstart-results.md` đã commit
- [x] `benchmarks/02-server-metrics.csv` đã commit (từ `record-metrics.py`)
- [x] `benchmarks/bonus-thread-sweep.md` đã commit
- [ ] Ít nhất 6 screenshots trong `submission/screenshots/` (xem `submission/screenshots/README.md`)
- [ ] `make verify` exit 0 (chạy ngay trước khi push)
- [ ] Repo trên GitHub ở chế độ **public**
- [ ] Đã paste public repo URL vào VinUni LMS

---

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Nếu private, grader không xem được → 0 điểm.
