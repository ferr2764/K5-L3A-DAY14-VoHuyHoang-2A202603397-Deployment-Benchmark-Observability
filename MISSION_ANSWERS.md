# Day 14 Lab - Mission Answers

## Part 1: Localhost vs Production

### Exercise 1.1: Anti-patterns found
1. **Hardcode secret trong code** (`develop/app.py` dòng 17–18): `OPENAI_API_KEY` và `DATABASE_URL` được gán trực tiếp, dễ lộ nếu push lên GitHub.
2. **Không có config management** (`develop/app.py` dòng 21–22): `DEBUG = True`, `MAX_TOKENS = 500` viết cứng, không đọc từ biến môi trường.
3. **Dùng `print()` thay vì structured logging và log cả secret** (`develop/app.py` dòng 33–34, 38): in thẳng `OPENAI_API_KEY` ra stdout, khó parse và gây rò rỉ thông tin nhạy cảm.
4. **Không có health check endpoint** (`/health`, `/ready`): platform không thể kiểm tra app còn sống hay sẵn sàng nhận traffic để tự restart.
5. **Bind `host="localhost"`, cứng `port=8000` và bật `reload=True`** (`develop/app.py` dòng 51–53): không chạy được trong container/cloud (cần `0.0.0.0` và đọc `PORT` từ env), `reload=True` không an toàn cho production.

### Exercise 1.2: Test develop version
```powershell
curl.exe -X POST "http://localhost:8000/ask?question=Hello"
# Output:
{"answer":"Agent đang hoạt động tốt! (mock response) Hỏi thêm câu hỏi đi nhé."}
```

### Exercise 1.3: Comparison table
| Feature | Develop (`develop/app.py`) | Production (`production/app.py`) | Why Important? |
|---------|----------------------------|----------------------------------|----------------|
| Config & Secrets | Hardcode `OPENAI_API_KEY`, `DATABASE_URL`, `DEBUG`, `MAX_TOKENS` | Đọc qua `settings` từ biến môi trường (`from config import settings`) | Tránh lộ secret lên Git; dễ đổi cấu hình theo từng môi trường (dev/staging/prod) mà không sửa code. |
| Logging | Dùng `print()`, log cả `OPENAI_API_KEY` | Structured JSON logging (`logging` + `json.dumps`), chỉ log metadata (`question_length`, `client_ip`) | Dễ thu thập/tìm kiếm log trên Datadog/Loki; không làm rò rỉ secret. |
| Health Check | Không có | Có `/health` (liveness) và `/ready` (readiness) | Platform biết khi nào app khởi động xong để route traffic và khi nào app treo để restart. |
| Lifecycle / Shutdown | Không có | Dùng `lifespan` (`@asynccontextmanager`) để xử lý startup và graceful shutdown | Tránh cắt ngang request đang xử lý khi deploy/restart container. |
| Network & Server | `host="localhost"`, `port=8000`, `reload=True` | Bind `0.0.0.0`, đọc `PORT` từ env, tắt `reload` | Container mới nhận được traffic từ bên ngoài và tương thích với `PORT` động của Railway/Render. |

**Kết quả test bản production:**
```powershell
curl.exe http://localhost:8000/health
# {"status":"ok","uptime_seconds":11.3,"version":"1.0.0","environment":"development","timestamp":"2026-11-25T07:40:13.610395+00:00"}

curl.exe -X POST http://localhost:8000/ask -H "Content-Type: application/json" -d '{\"question\": \"Hello\"}'
# {"question":"Hello","answer":"Đây là câu trả lời từ AI agent (mock). Trong production, đây sẽ là response từ OpenAI/Anthropic.","model":"gpt-4o-mini"}
```

## Part 2: Docker

### Exercise 2.1: Dockerfile questions
1. **Base image:** `python:3.11` (bản full Python distribution, dung lượng lớn ~1 GB).
2. **Working directory:** `/app` (`WORKDIR /app`).
3. **Tại sao `COPY requirements.txt` đứng trước `COPY app.py`:** Để tận dụng **Docker layer cache** — khi chỉ sửa source code (`app.py`) mà không thay đổi `requirements.txt`, Docker sẽ dùng lại layer đã `pip install` trước đó thay vì cài lại từ đầu.
4. **Port & Command mặc định:** `EXPOSE 8000` và `CMD ["python", "app.py"]`.

### Exercise 2.3: Image size comparison
- **Develop (`my-agent:develop`):** 1.67 GB (1670 MB)
- **Production (`my-agent:advanced`):** 240 MB
- **Difference:** Giảm ~85.6% (nhờ dùng multi-stage build với `python:3.11-slim` ở stage runtime, bỏ lại build tools)

### Exercise 2.4: Docker Compose stack
- **Sơ đồ các service:** `Client` → `nginx` (port 80/443, reverse proxy) → `agent` (port 8000, FastAPI) ↔ `redis` (6379) & `qdrant` (6333–6334) trong mạng `production_internal`.
- **Kết quả test qua Nginx:**
```powershell
curl.exe http://localhost/health
# {"status":"ok","uptime_seconds":33.1,"version":"2.0.0","timestamp":"2026-11-25T08:02:33.727319"}
```

## Part 3: Cloud Deployment

### Exercise 3.1: Railway deployment
- **URL:** `https://k5-l3a-day14-vohuyhoang-2a202603397-deployment-b-production.up.railway.app`
- **Screenshots:** `screenshots/dashboard.png`, `screenshots/running.png`, `screenshots/test.png`
- **Kết quả 4 lệnh tự kiểm:** `/health` → `200`, `/ready` → `200`, `/ask` không key → `401`, `/ask` có key → `200`.

### So sánh `railway.toml` và `render.yaml`
- **`railway.toml`:** Khai báo ngắn gọn theo từng service (`[build]` chỉ định `DOCKERFILE`, `[deploy]` chỉ định `healthcheckPath`, `restartPolicyType`); Redis và biến môi trường liên kết trực tiếp trên Dashboard (`${{Redis.REDIS_URL}}`).
- **`render.yaml`:** Là Infrastructure-as-Code (Blueprint) khai báo toàn bộ stack trong một file (gồm cả `web` service, `redis` service, `envVars` dùng `fromService` để lấy connection string), phù hợp khi muốn tái tạo toàn bộ hạ tầng bằng Git.

### Exercise 3.3: Cloud Run CI/CD (`cloudbuild.yaml` & `service.yaml`)
Pipeline trong `03-cloud-deployment/production-cloud-run/` đi qua các bước:
1. Build Docker image từ source và gắn tag theo `$COMMIT_SHA` / `latest`.
2. Push image lên Artifact Registry / Container Registry.
3. Deploy image mới lên **Google Cloud Run** (`gcloud run deploy` hoặc áp dụng `service.yaml` với cấu hình `PORT`, `minScale`/`maxScale`, resource limits và readiness/liveness probe).

## Part 4: API Security

### Exercise 4.1-4.3: Test results
**1. API Key (`develop/app.py`) — 401 & 200:**
```powershell
# Không có X-API-Key → 401 Unauthorized
HTTP/1.1 401 Unauthorized
{"detail":"Missing API key. Include header: X-API-Key: <your-key>"}

# Có X-API-Key → 200 OK
HTTP/1.1 200 OK
{"question":"Hello","answer":"Agent đang hoạt động tốt! (mock response) Hỏi thêm câu hỏi đi nhé."}
```

**2. JWT + Rate Limiting (`production/app.py`) — 200 & 429 (tài khoản `student`, đã che token):**
```powershell
# Request 1..10 (Authorization: Bearer <REDACTED_TOKEN>) → 200 OK
{"question":"Hello","answer":"Agent đang hoạt động tốt! (mock response) Hỏi thêm câu hỏi đi nhé.","usage":{"requests_remaining":0,"budget_remaining_usd":0.000181}}

# Request 11 → 429 Too Many Requests
HTTP/1.1 429 Too Many Requests
x-ratelimit-limit: 10
x-ratelimit-remaining: 0
retry-after: 59
{"detail":{"error":"Rate limit exceeded","limit":10,"window_seconds":60,"retry_after_seconds":59}}
```

### Exercise 4.4: Cost guard implementation
- **Rate limit** chỉ giới hạn *tốc độ* (số request/phút) để chống spam/DoS, nhưng một request hợp lệ vẫn có thể gửi prompt rất dài gây tốn nhiều token.
- **Cost guard** bảo vệ *ngân sách*: theo dõi tổng chi phí (hoặc số token) tích lũy theo user/ngày (lưu trong Redis/DB). Trước khi gọi LLM, kiểm tra nếu vượt ngưỡng ngân sách (`budget_remaining_usd <= 0`) thì chặn ngay với mã lỗi `402 Payment Required`. Cần kết hợp cả hai để vừa chống flood vừa chống đội chi phí API.

## Part 5: Scaling & Reliability

### Exercise 5.1-5.5: Implementation notes
- **Liveness (`/health`) vs Readiness (`/ready`):**
  - `/health` kiểm tra process app còn sống hay không (trả `200 OK` kể cả khi Redis tạm mất, để platform không restart oan).
  - `/ready` kiểm tra app đã sẵn sàng nhận traffic chưa (ping Redis; nếu mất kết nối Redis thì trả `503 Service Unavailable` để load balancer ngừng route traffic tới instance đó).
- **Vì sao state trong RAM bị hỏng khi scale nhiều instance:** Nếu lưu session/rate-limit trong RAM của từng container, khi Nginx chia tải sang instance khác thì instance mới không có dữ liệu cũ → mất lịch sử chat hoặc đếm sai rate limit. Lưu state tập trung vào **Redis** giúp mọi instance đều stateless và dùng chung state.
- **Kết quả `test_stateless.py` với `--scale agent=3`:**
  - 5 request được phục vụ bởi cả 3 instance (`instance-af004a`, `instance-3a4ed5`, `instance-0dc80d`).
  - Lịch sử hội thoại vẫn giữ đủ **10 messages** (5 user + 5 assistant) nhờ lưu qua Redis.