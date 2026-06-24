# Bonus — Thread sweep

Model: `Qwen2.5-7B-Instruct-Q4_K_M.gguf`  ·  GPU layers: `99`

| threads | tg128 (tok/s) |
|---:|---:|
| 1 | 6.2 |
| 2 | 6.2 |
| 6 | 6.5 |
| 12 | 6.6 |
| 14 | 6.6 |
| 28 | 6.8 |

**Best**: `-t 28` at 6.8 tok/s.

Look at the curve. If it peaks around your **physical** core count and drops as you go higher, that's the memory-bandwidth ceiling: extra threads fight over the same memory channels and slow each other down.
