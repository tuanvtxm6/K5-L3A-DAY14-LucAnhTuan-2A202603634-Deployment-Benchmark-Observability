# Deployment Information

## Public URL

https://agent-production-8a6fa083.up.railway.app

## Platform

Railway. Deploy thư mục `06-lab-complete/` bằng `railway up`, builder `DOCKERFILE` theo [06-lab-complete/railway.toml](06-lab-complete/railway.toml).

Project `k5-day14-agent` gồm 2 service:

| Service | Nguồn | Vai trò |
|---------|-------|---------|
| `agent` | `06-lab-complete/Dockerfile` (multi-stage, `python:3.11-slim`) | FastAPI agent, healthcheck `/health` |
| `Redis` | Railway Redis template | State của rate limit và cost guard |

## Test Commands

```bash
export BASE=https://agent-production-8a6fa083.up.railway.app
export AGENT_API_KEY=<key đã đặt trên Railway>
```

### Health check

```bash
curl -sS -o /dev/null -w '%{http_code}\n' "$BASE/health"     # 200
curl "$BASE/health"
# {"status":"ok","environment":"production"}
```

### Readiness (Redis)

```bash
curl -sS -o /dev/null -w '%{http_code}\n' "$BASE/ready"      # 200
curl "$BASE/ready"
# {"status":"ready","redis":"ok"}
```

### Không có API key → 401

```bash
curl -sS -o /dev/null -w '%{http_code}\n' -X POST "$BASE/ask" \
  -H 'Content-Type: application/json' -d '{"question":"Hello"}'                              # 401
```

### Có API key → 200

```bash
curl -sS -o /dev/null -w '%{http_code}\n' -X POST "$BASE/ask" \
  -H 'Content-Type: application/json' -H "X-API-Key: $AGENT_API_KEY" \
  -d '{"question":"Hello"}'                                                                   # 200

curl -X POST "$BASE/ask" -H 'Content-Type: application/json' -H "X-API-Key: $AGENT_API_KEY" \
  -d '{"user_id":"test","question":"What is deployment?"}'
# {"question":"What is deployment?","answer":"Deployment là quá trình đưa code từ máy bạn lên server để người khác dùng được.","usage":{"input_tokens":4,"output_tokens":19,"cost_usd":1.2e-05}}
```

### Rate limit → 429

```bash
for i in $(seq 1 12); do
  curl -s -o /dev/null -w '%{http_code} ' -X POST "$BASE/ask" \
    -H 'Content-Type: application/json' -H "X-API-Key: $AGENT_API_KEY" -d '{"question":"test"}'
done; echo
# 200 200 200 200 200 200 200 200 429 429 429 429
# (2 request trước đó đã dùng quota, giới hạn 10 request / 60 giây)
```

### Kết quả đo ngày 2026-11-25

| Lệnh | Mong đợi | Thực tế |
|------|---------:|--------:|
| `GET /health` | 200 | 200 |
| `GET /ready` | 200 | 200 |
| `POST /ask` không key | 401 | 401 |
| `POST /ask` có key | 200 | 200 |

`python check_production_ready.py --public-url https://agent-production-8a6fa083.up.railway.app`
→ `✅ Public URL: GET https://agent-production-8a6fa083.up.railway.app/health → 200`

## Environment Variables Set

Đặt trên Railway (service `agent`), không có trong code hay repo:

| Biến | Giá trị |
|------|---------|
| `AGENT_API_KEY` | secret ngẫu nhiên 32 byte, đặt bằng `railway variable set AGENT_API_KEY --stdin` |
| `REDIS_URL` | `${{Redis.REDIS_URL}}` (tham chiếu tới service Redis) |
| `ENVIRONMENT` | `production` |
| `LOG_LEVEL` | `INFO` |
| `RATE_LIMIT_PER_MINUTE` | `10` |
| `DAILY_BUDGET_USD` | `1.0` |
| `MONTHLY_BUDGET_USD` | `10.0` |
| `PORT` | Railway tự cấp (8080); `CMD` của Dockerfile đọc `${PORT:-8000}` |

## Screenshots

- [Deployment dashboard](screenshots/dashboard.png)
- [Service running](screenshots/running.png)
- [Test results](screenshots/test.png)
