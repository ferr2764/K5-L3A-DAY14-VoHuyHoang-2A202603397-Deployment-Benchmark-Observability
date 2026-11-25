# Deployment Info — Day 14 Final Project

- **Platform:** Railway
- **Public URL:** `https://k5-l3a-day14-vohuyhoang-2a202603397-deployment-b-production.up.railway.app`
- **Root Directory:** `06-lab-complete`
- **Config file:** `06-lab-complete/railway.toml`

## Environment Variables (đã đặt trên Railway, không lưu trong code)
- `AGENT_API_KEY` (đặt trên Railway Variables)
- `REDIS_URL` (`${{Redis.REDIS_URL}}`)
- `RATE_LIMIT_PER_MINUTE=10`
- `DAILY_BUDGET_USD=1.0`

## Lệnh kiểm tra
```bash
BASE="https://k5-l3a-day14-vohuyhoang-2a202603397-deployment-b-production.up.railway.app"
curl -sS -o /dev/null -w '%{http_code}\n' "$BASE/health"     # 200
curl -sS -o /dev/null -w '%{http_code}\n' "$BASE/ready"      # 200
curl -sS -o /dev/null -w '%{http_code}\n' -X POST "$BASE/ask" -H 'Content-Type: application/json' -d '{"question":"Hello"}'   # 401
curl -sS -o /dev/null -w '%{http_code}\n' -X POST "$BASE/ask" -H 'Content-Type: application/json' -H "X-API-Key: $AGENT_API_KEY" -d '{"question":"Hello"}'   # 200
```
