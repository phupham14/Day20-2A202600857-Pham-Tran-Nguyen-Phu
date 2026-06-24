# Bonus — Thread sweep

Model: `Qwen2.5-7B-Instruct-Q4_K_M.gguf`  ·  GPU layers: `99`

| threads | tg128 (tok/s) |
|---:|---:|
| 1 | 0.0 |
| 2 | 0.0 |
| 6 | 0.0 |
| 12 | 0.0 |
| 14 | 0.0 |
| 28 | 0.0 |

**Best**: `-t 1` at 0.0 tok/s.

Look at the curve. If it peaks around your **physical** core count and drops as you go higher, that's the memory-bandwidth ceiling: extra threads fight over the same memory channels and slow each other down.
