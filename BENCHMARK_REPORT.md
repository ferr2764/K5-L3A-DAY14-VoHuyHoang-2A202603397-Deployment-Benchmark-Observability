# Benchmark & Observability Report — Day 14

> Copy file này thành `BENCHMARK_REPORT.md` trong repo của bạn và điền đầy đủ.
> Mọi con số phải lấy từ report JSON / CSV / screenshot thật, không ước lượng.

**Họ tên:** Võ Huy Hoàng
**Ngày chạy:** 2026-11-25
**Môi trường:** Windows + Docker Desktop (WSL2)

---

## 1. SLO của agent

| SLI | SLO | Lý do chọn ngưỡng này |
|---|---|---|
| p95 latency `/ask` | < 500 ms | Đảm bảo 95% người dùng nhận phản hồi dưới 0.5 giây |
| Error rate `/ask` | < 2.0 % | Chấp nhận lỗi ngẫu nhiên nhỏ của mock LLM nhưng không để sập dịch vụ |
| Accuracy trên golden dataset | ≥ 80 % | Ngưỡng tối thiểu cho quality gate trước khi merge/deploy |

---

## 2. Benchmark hiệu năng (develop, `load_test.py`)

### 2.1 Concurrency vs latency (agent v2)

| Concurrency | RPS | p50 (ms) | p95 (ms) | p99 (ms) | Error rate | SLO |
|---|---|---|---|---|---|---|
| 1 | 3.9 | 201.7 | 237.6 | 1696.9 | 0.67% | PASS |
| 10 | 42.55 | 192.9 | 241.9 | 1694.2 | 0.33% | PASS |
| 50 | 95.16 | 202.3 | 497.5 | 1711.1 | 1.00% | PASS |
| 100 | 59.67 | 563.1 | 3033.7 | 3252.7 | 2.00% | FAIL |

**Nhận xét:** RPS không tăng tuyến tính theo concurrency. Từ 1 → 50 RPS tăng mạnh (3.9 → 95.16), nhưng tới 100 thì RPS giảm xuống 59.67 và p95 vọt lên 3033.7ms do 1 process bị bão hòa (CPU/event-loop contention).

### 2.2 Tail latency

| `MOCK_SLOW_RATE` | avg (ms) | p95 (ms) | p99 (ms) |
|---|---|---|---|
| 0 | 205.3 | 446.4 | 478.8 |
| 0.03 (mặc định) | 247.3 | 461.3 | 1711.9 |
| 0.1 | 385.1 | 1708.2 | 1954.5 |

**Nhận xét:** `p95` và `p99` nhạy nhất với request chậm. Không nên đặt SLO theo `avg` vì trung bình che giấu tail latency — người dùng vẫn gặp lag nặng ở p95/p99 dù `avg` trông thấp.

---

## 3. Benchmark chất lượng (develop, `eval/run_eval.py`)

| Chỉ số | Agent v1 | Agent v2 | Agent v2 (sau khi sửa KB) |
|---|---|---|---|
| Accuracy | 4.3% (1/23) | 91.3% (21/23) | 100.0% (23/23) |
| Avg tokens / câu | 57.0 | 109.1 | 112.4 |
| Tổng cost (USD) | $0.000403 | $0.000762 | $0.000780 |
| Cost / câu đúng (USD) | $0.00040275 | $0.00003627 | $0.00003391 |
| Quality gate | FAIL | PASS | PASS |

**Những câu v2 trả lời sai ban đầu và cách bạn sửa:**

| ID | Keyword bị thiếu | Bạn đã sửa gì |
|---|---|---|
| `security-02` | `token`, `hết hạn`, `chữ ký` | Bổ sung entry JWT vs API key vào `KNOWLEDGE_BASE` trong `develop/utils/mock_llm.py` |
| `scaling-03` | `cpu`, `memory`, `custom metrics` | Bổ sung entry Kubernetes HPA vào `KNOWLEDGE_BASE` trong `develop/utils/mock_llm.py` |

**Nhận xét về trade-off chất lượng / chi phí / latency giữa v1 và v2:** v2 tốn nhiều token hơn (~2x) và thêm bước retrieval nhưng accuracy tăng từ 4.3% lên 100%, giúp **cost / câu đúng** giảm hơn 10 lần ($0.00040275 → $0.00003391). Trong CI, dùng exit code khác 0 của `run_eval.py` (khi dưới ngưỡng `--min-accuracy` hoặc có regression) để chặn merge PR.

---

## 4. Observability stack (production)

### 4.1 Screenshot

- [ ] Dashboard Grafana khi đang có tải (thấy rõ Golden Signals)
- [ ] Một trace chậm trong Jaeger (thấy span nào chiếm thời gian)
- [ ] Một trace lỗi trong Jaeger (có event `exception`)
- [ ] Trang Alerts của Prometheus với ít nhất 1 alert đang **firing**

### 4.2 Thí nghiệm "phá hệ thống"

| # | Thay đổi | Alert nào firing | Thời gian từ lúc đổi → firing | Panel nào phản ánh rõ nhất |
|---|---|---|---|---|
| A | `MOCK_ERROR_RATE=0.2` | | | |
| B | `MOCK_SLOW_RATE=0.3` | | | |
| C | `LLM_MODEL=mock-llm-large` | | | |
| D | `docker pause` 1 instance | | | |
| E | `AGENT_VERSION=v1` | | | |

**Thí nghiệm E không làm alert nào firing. Bạn sẽ phát hiện loại sự cố này trong production bằng cách nào?**

...

### 4.3 Debug bằng trace

Chọn 1 trace chậm nhất bạn tìm được:

- `trace_id`: ...
- Tổng thời gian: ... ms
- Span chiếm nhiều thời gian nhất: ... (... ms, ...%)
- Dòng log tương ứng (grep theo `trace_id`):

```json

```

---

## 5. Benchmark scaling (production, Locust headless, 100 users, 2 phút)

| Số instance | RPS | p50 (ms) | p95 (ms) | p99 (ms) | Failures |
|---|---|---|---|---|---|
| 1 | | | | | |
| 3 | | | | | |

**Giải thích kết quả:** Scale có cải thiện p95 không? Nút thắt nằm ở đâu?

...

---

## 6. Kết luận

- 3 điều bạn học được về benchmark & observability:
  1. ...
  2. ...
  3. ...
- Nếu đưa agent này lên production thật, bạn sẽ thêm metric / alert nào nữa?

...
