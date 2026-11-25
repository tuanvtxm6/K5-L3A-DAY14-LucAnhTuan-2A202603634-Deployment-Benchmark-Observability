# Day 14 Lab - Mission Answers

> Môi trường chạy: Windows 11 + Docker Desktop 29.9.3; phần cần SIGTERM chạy trong WSL2 Ubuntu. Mọi output bên dưới là output thật chạy ngày 2026-11-25 (token/key đã che).

## Part 1: Localhost vs Production

### Exercise 1.1: Anti-patterns found

Trong `01-localhost-vs-production/develop/app.py`:

1. **Hardcode secret** (dòng 17–18): `OPENAI_API_KEY = "sk-hardcoded-..."` và `DATABASE_URL` chứa password `password123`. Push lên GitHub là lộ key ngay, và muốn đổi key phải sửa code rồi deploy lại.
2. **Log ra secret** (dòng 34): `print(f"[DEBUG] Using key: {OPENAI_API_KEY}")`. Key nằm trong log của mọi request, ai đọc được log đều thấy (đã xác nhận ở Exercise 1.2).
3. **Config cứng trong code** (dòng 21–22): `DEBUG = True`, `MAX_TOKENS = 500`, không đọc từ biến môi trường, nên dev, staging và production không thể khác nhau.
4. **`print()` thay vì logging có cấu trúc** (dòng 33, 38): không có level, timestamp, không parse được, và in nguyên câu hỏi lẫn câu trả lời của user (dữ liệu nhạy cảm).
5. **Không có health check**: `GET /health` trả **404** (đã thử). Platform không biết khi nào app treo để restart.
6. **Bind `host="localhost"`** (dòng 51): trong container chỉ nhận kết nối từ chính container, nên từ ngoài không gọi được.
7. **Port cố định 8000** (dòng 52): bỏ qua biến `PORT` mà Railway hay Render inject.
8. **`reload=True` khi chạy thật** (dòng 53): tốn CPU theo dõi file và chạy thêm một process con. Khi tắt process cha, process con reload vẫn giữ cổng 8000 (gặp thật khi làm bài, phải dừng tay PID con).
9. **Không xử lý shutdown**: không có lifespan hay xử lý SIGTERM, nên request đang chạy có thể bị cắt ngang khi platform tắt container.
10. **`question` nhận qua query string**: gửi JSON body thì trả 422. Câu hỏi bị ghi vào URL và access log, và không có giới hạn độ dài.

### Exercise 1.2: Chạy basic version

```text
$ curl -X POST "http://localhost:8000/ask?question=Hello"
{"answer":"Đây là câu trả lời từ AI agent (mock). Trong production, đây sẽ là response từ OpenAI/Anthropic."}

$ curl http://localhost:8000/health
{"detail":"Not Found"}            ← HTTP 404, không có health check

$ curl -X POST http://localhost:8000/ask -H "Content-Type: application/json" -d '{"question":"Hello"}'
{"detail":[{"type":"missing","loc":["query","question"],"msg":"Field required","input":null}]}   ← HTTP 422

Console của server:
[DEBUG] Got question: Hello
[DEBUG] Using key: sk-hardcoded-fake-key-never-do-this      ← secret lộ trong log
[DEBUG] Response: Đây là câu trả lời từ AI agent (mock). ...
```

App chạy được nhưng chưa production-ready: lộ secret, không có health check, không chạy được trong container.

### Exercise 1.3: Comparison table

Output bản production:

```text
$ curl http://localhost:8000/health
{"status":"ok","uptime_seconds":0.6,"version":"1.0.0","environment":"development","timestamp":"2026-11-25T08:03:06.831740+00:00"}
$ curl http://localhost:8000/ready
{"ready":true}
$ curl -X POST http://localhost:8000/ask -H "Content-Type: application/json" -d '{"question":"Hello"}'
{"question":"Hello","answer":"Đây là câu trả lời từ AI agent (mock). ...","model":"gpt-4o-mini"}
$ curl -X POST http://localhost:8000/ask -H "Content-Type: application/json" -d '{}'
{"detail":"question field is required"}      ← HTTP 422
```

| Feature | Develop | Production | Why Important? |
|---------|---------|------------|----------------|
| Config | Hardcode trong code (`DEBUG = True`, key, DB URL) | `config.py` đọc env (`os.getenv`), có `validate()`: thiếu `AGENT_API_KEY` khi `ENVIRONMENT=production` thì dừng ngay lúc khởi động | Cùng một image chạy được mọi môi trường; đổi config không cần sửa code; secret không vào Git; thiếu config quan trọng thì lỗi sớm thay vì chạy sai |
| Secrets | Ghi thẳng trong code và in ra log | Lấy từ env (`.env` nằm trong `.gitignore`), log chỉ ghi `question_length`, không ghi secret | Log thường được gửi sang hệ thống khác (Datadog, Loki) và nhiều người đọc được; secret trong log là lộ |
| Health check | Không có (`/health` 404) | `/health` (liveness: uptime, version) và `/ready` (readiness: cờ `is_ready`, 503 khi đang khởi động hoặc tắt) | Platform dùng `/health` để restart container treo, load balancer dùng `/ready` để chỉ route vào instance sẵn sàng |
| Logging | `print()` | `logging` với format JSON, có level | Log có cấu trúc thì lọc, tìm, đếm được theo field |
| Shutdown | Đột ngột | Lifespan: tắt cờ `is_ready`, dọn tài nguyên; uvicorn xử lý SIGTERM | Không cắt ngang request của user khi deploy hoặc scale down |
| Host / Port | `localhost:8000` cố định | `HOST=0.0.0.0`, `PORT` từ env | Container nhận được kết nối từ ngoài; cloud tự cấp port |
| Reload | `reload=True` luôn bật | Chỉ bật khi `DEBUG=true` | Production không cần theo dõi file, ít process hơn |
| Input | Query param, không validate | JSON body, thiếu `question` thì 422 | Câu hỏi không lộ trong URL và access log; lỗi đầu vào rõ ràng |

**Nhận xét thêm khi chạy thật:** dòng log JSON `agent_request` / `agent_response` của bản production **không hề xuất hiện** trên console, chỉ thấy `WARNING:root:OPENAI_API_KEY not set` ở dạng text thường. Nguyên nhân: `config.py` gọi `logging.warning()` lúc import, trước `logging.basicConfig(...)` trong `app.py`. Lệnh đó tự cấu hình root logger với format mặc định và level WARNING, nên `basicConfig` sau đó không có tác dụng, log INFO bị bỏ và format JSON không được áp dụng. Muốn đúng thì phải cấu hình logging trước mọi import, hoặc dùng `basicConfig(force=True)`. Ngoài ra format `'{"msg":"%(message)s"}'` cũng không ra JSON hợp lệ khi message chứa dấu `"` (xem Part 2.4).

## Part 2: Docker

### Exercise 2.1: Dockerfile questions

`02-docker/develop/Dockerfile`:

1. **Base image:** `python:3.11`, bản Debian đầy đủ có Python, compiler và nhiều tool, nên image rất nặng (1,67 GB khi build xong, xem 2.3).
2. **Working directory:** `/app` (`WORKDIR /app`). Các lệnh `COPY`, `RUN`, `CMD` sau đó chạy tương đối từ thư mục này.
3. **Vì sao COPY requirements.txt trước:** Docker cache theo từng layer. Copy `requirements.txt` rồi `pip install` trước khi copy code thì khi chỉ sửa `app.py`, layer cài dependencies vẫn lấy từ cache. Build lại chỉ mất vài giây thay vì cài lại toàn bộ package. Nếu `COPY . .` trước, mọi thay đổi code đều làm mất cache của bước `pip install`.
4. **CMD vs ENTRYPOINT:** `ENTRYPOINT` là chương trình luôn chạy khi container start; tham số `docker run image <args>` được nối thêm vào nó, và chỉ đổi được bằng `--entrypoint`. `CMD` là lệnh hoặc tham số mặc định, bị thay hoàn toàn khi truyền lệnh khác vào `docker run image <cmd>`. Hay dùng chung: `ENTRYPOINT ["uvicorn"]` + `CMD ["main:app", "--port", "8000"]`. Dockerfile này chỉ dùng `CMD ["python", "app.py"]` (exec form), nên python là PID 1 và nhận SIGTERM trực tiếp.

### Exercise 2.2: Build và run

```text
$ docker build -f 02-docker/develop/Dockerfile -t my-agent:develop .
$ docker run -p 8000:8000 my-agent:develop
$ curl -X POST "http://localhost:8000/ask?question=What%20is%20Docker?"
{"answer":"Container là cách đóng gói app để chạy ở mọi nơi. Build once, run anywhere!"}
$ curl http://localhost:8000/health
{"status":"ok","uptime_seconds":1.2,"container":true}
```

(Bản develop nhận `question` qua query, như lưu ý số 3 trong README.)

### Exercise 2.3: Image size comparison

`02-docker/production/Dockerfile`:

- **Stage 1 (`builder`):** `python:3.11-slim` + `apt-get install gcc libpq-dev` (build tools cho package cần compile), rồi `pip install --user -r requirements.txt` vào `/root/.local`. Stage này không được deploy.
- **Stage 2 (`runtime`):** `python:3.11-slim` mới tinh, tạo user non-root `appuser`, chỉ `COPY --from=builder /root/.local` (package đã cài), `main.py` và `utils/`. Chạy `USER appuser`, có `HEALTHCHECK`, `CMD uvicorn ... --workers 2`.
- **Vì sao nhỏ hơn:** image cuối chỉ chứa layer của stage cuối. gcc, header, apt cache và pip cache nằm lại ở stage builder và bị bỏ. Base `slim` cũng không có compiler, tài liệu và các lib không dùng như bản `python:3.11` đầy đủ.

Đo bằng `docker images` / `docker image inspect --format '{{.Size}}'`:

```text
REPOSITORY:TAG      SIZE
my-agent:advanced   240MB      (239,815,277 bytes)
my-agent:develop    1.67GB     (1,667,784,083 bytes)
```

- Develop: **1667.8 MB**
- Production: **239.8 MB**
- Difference: **giảm 85.6%** (image production chỉ bằng 14,4% image develop)

### Exercise 2.4: Docker Compose stack

Chạy `touch .env.local` (file nằm trong `.gitignore`), rồi `docker compose -p k5-day14-docker up -d --build` trong `02-docker/production/`. 4 service được start:

```text
SERVICE   STATUS                                     PORTS
agent     Up (health: starting)                      8000/tcp
nginx     Up                                         0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp
qdrant    Up (healthy)                               6333-6334/tcp
redis     Up (healthy)                               6379/tcp
```

```text
                 ┌──────────────── network "internal" (bridge) ────────────────┐
 Client ──:80──▶ │  nginx  ──proxy_pass http://agent:8000──▶  agent (FastAPI)   │
  (host)         │  (rate limit 10r/s,                         │        │       │
                 │   security headers)                         ▼        ▼       │
                 │                               redis:6379 (cache)  qdrant:6333│
                 │                                                  (vector DB) │
                 └──────────────────────────────────────────────────────────────┘
```

- Chỉ **nginx** mở port ra host (80/443). agent, redis và qdrant chỉ có trong network `internal`, nên từ ngoài không gọi thẳng được.
- Các service gọi nhau bằng **tên service** qua DNS nội bộ của Docker: nginx tới `agent:8000`, agent tới `redis://redis:6379/0` và `http://qdrant:6333`.
- `depends_on` + `condition: service_healthy`: agent chỉ start khi redis và qdrant đã healthy; nginx start sau agent.

```text
$ curl -i http://localhost/health
HTTP/1.1 200 OK
X-Frame-Options: DENY
X-Content-Type-Options: nosniff
X-XSS-Protection: 1; mode=block
{"status":"ok","uptime_seconds":11.6,"version":"2.0.0","timestamp":"2026-11-25T08:07:24.646526"}

$ curl http://localhost/ask -X POST -H "Content-Type: application/json" -d '{"question": "Explain microservices"}'
{"answer":"Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận."}     ← HTTP 200
```

Security header do Nginx thêm vào, chứng tỏ request đi qua reverse proxy. Log của agent:

```text
{"time":"2026-11-25 08:07:24,711","level":"INFO","msg":"{"event": "request", "q_len": 21}"}
```

Dòng này **không phải JSON hợp lệ**: message đã là một chuỗi JSON lại bị bọc thêm trong `"msg":"..."`, nên dấu `"` bên trong làm hỏng chuỗi. Log aggregator sẽ không parse được. Agent Part 6 tránh lỗi này bằng một `logging.Formatter` tự viết, dùng `json.dumps` cho cả dòng.

## Part 3: Cloud Deployment

### Exercise 3.1: Railway deployment

- URL: https://agent-production-8a6fa083.up.railway.app
- Agent được deploy là agent Part 6 (`06-lab-complete/`), trong Railway project `k5-day14-agent`, có service Redis riêng, `REDIS_URL=${{Redis.REDIS_URL}}`.
- Kết quả 4 lệnh tự kiểm CP6: `/health` 200, `/ready` 200, `/ask` không key 401, `/ask` có key 200. Chi tiết trong [DEPLOYMENT.md](DEPLOYMENT.md).
- Screenshot: [dashboard](screenshots/dashboard.png) · [running](screenshots/running.png) · [test](screenshots/test.png)

### Exercise 3.2: So sánh `railway.toml` và `render.yaml`

| Khía cạnh | `railway.toml` | `render.yaml` |
|-----------|----------------|---------------|
| Phạm vi | Cấu hình **một service** (build + deploy) | **Blueprint cả hạ tầng**: service web và Key Value (Redis) trong một file |
| Build | `builder = "DOCKERFILE"`, `dockerfilePath` | `runtime: docker`, `dockerfilePath`, `dockerContext`, `rootDir` |
| Health check | `healthcheckPath = "/health"`, `healthcheckTimeout` | `healthCheckPath: /health` |
| Restart | `restartPolicyType = "ON_FAILURE"`, số lần thử lại | Render tự quản lý, không khai báo trong file |
| Redis | Không khai báo được: tạo bằng `railway add --database redis`, nối bằng biến `${{Redis.REDIS_URL}}` | Khai báo `type: keyvalue`, nối bằng `fromService ... property: connectionString` |
| Biến môi trường và secret | Không nằm trong file: đặt bằng CLI hoặc dashboard | Nằm trong file; secret dùng `sync: false` (nhập trên dashboard) hoặc `generateValue: true`, không ghi giá trị |
| Thư mục con (monorepo) | Chạy `railway up` ngay trong `06-lab-complete/` | Phải khai báo `rootDir: 06-lab-complete` |

Vì sao khác: Railway coi file config là phần bổ sung cho project đã tạo bằng CLI hoặc dashboard, nên hạ tầng và biến môi trường nằm ngoài file. Render Blueprint theo hướng Infrastructure as Code: một file mô tả đủ để dựng lại toàn bộ stack, nên phải có cả Redis và danh sách biến, đồng thời có cơ chế riêng (`sync: false`) để secret không bị ghi vào repo.

### Exercise 3.3: Pipeline Cloud Run (`cloudbuild.yaml` + `service.yaml`)

1. **Trigger:** push lên `main` thì Cloud Build chạy `cloudbuild.yaml`.
2. **test:** trong `python:3.11-slim`, cài `requirements.txt` và chạy `pytest tests/`. Test fail thì pipeline dừng, không build.
3. **build và push:** build image, gắn tag `$COMMIT_SHA` và `latest`, dùng `--cache-from latest` để tận dụng layer cache, rồi push lên Container Registry `gcr.io/$PROJECT_ID/ai-agent`.
4. **deploy:** `gcloud run deploy` image đúng `$COMMIT_SHA` lên `asia-southeast1` (1–10 instance, 512Mi, timeout 60s). Secret lấy từ Secret Manager bằng `--set-secrets`, không ghi trong code.
5. **`service.yaml`:** khai báo service ở dạng IaC (Knative). `minScale 1` tránh cold start, `containerConcurrency 80`, `livenessProbe /health`, `startupProbe /ready`, secret đọc qua `secretKeyRef`. Áp dụng bằng `gcloud run services replace`.

## Part 4: API Security

### Exercise 4.1: API Key authentication

`04-api-gateway/develop/app.py`:

- **Check ở đâu:** hàm dependency `verify_api_key` (dòng 39–54) đọc header bằng `APIKeyHeader(name="X-API-Key")` và được gắn vào `/ask` qua `Depends(verify_api_key)` (dòng 70). Endpoint `/` không gắn nên vẫn public.
- **Sai key:** thiếu header thì **401**, có header nhưng sai thì **403**. Script chấm Part 6 yêu cầu cả hai trường hợp đều 401; agent Part 6 trả 401 cho cả hai, so sánh bằng `secrets.compare_digest` để tránh timing attack.
- **Rotate key:** key đọc từ env `AGENT_API_KEY` lúc khởi động, nên rotate bằng cách đặt giá trị mới trên platform rồi restart hoặc redeploy, không phải sửa code. Nhược điểm: chỉ có một key hợp lệ tại một thời điểm, nên client dùng key cũ lỗi ngay khi đổi. Cách tốt hơn là chấp nhận một danh sách key (key cũ và key mới) trong thời gian chuyển tiếp, rồi xoá key cũ.

Chạy với `AGENT_API_KEY=secret-key-123 python app.py`:

```text
$ curl -i http://localhost:8000/ask -X POST -H "Content-Type: application/json" -d '{"question": "Hello"}'
HTTP/1.1 401 Unauthorized
{"detail":"Missing API key. Include header: X-API-Key: <your-key>"}

$ curl -i http://localhost:8000/ask -X POST -H "X-API-Key: wrong-key" -H "Content-Type: application/json" -d '{"question": "Hello"}'
HTTP/1.1 403 Forbidden
{"detail":"Invalid API key."}

$ curl -i http://localhost:8000/ask -X POST -H "X-API-Key: secret-key-123" -H "Content-Type: application/json" -d '{"question": "Hello"}'
HTTP/1.1 200 OK
{"question":"Hello","answer":"Agent đang hoạt động tốt! (mock response) Hỏi thêm câu hỏi đi nhé."}
```

### Exercise 4.2: JWT authentication

JWT flow trong `production/auth.py`:

1. `POST /auth/token` với username và password. `authenticate_user` kiểm tra `DEMO_USERS`, rồi `create_token` ký payload `{sub, role, iat, exp = +60 phút}` bằng HS256 với `JWT_SECRET`.
2. Client gửi `Authorization: Bearer <token>`.
3. `verify_token` gọi `jwt.decode`, kiểm tra chữ ký và hạn dùng. Server lấy `username` và `role` từ token mà không cần tra database. Token hết hạn thì 401, token giả hoặc sai chữ ký thì 403.

```text
$ curl -i http://localhost:8000/auth/token -X POST -H "Content-Type: application/json" -d '{"username": "student", "password": "demo123"}'
HTTP/1.1 200 OK
{"access_token":"eyJ***(đã che)***","token_type":"bearer","expires_in_minutes":60,"hint":"Include in header: Authorization: Bearer eyJhbGciOiJIUzI1NiIs..."}

$ curl -i http://localhost:8000/ask -X POST -H "Content-Type: application/json" -d '{"question": "Explain JWT"}'
HTTP/1.1 401 Unauthorized
{"detail":"Authentication required. Include: Authorization: Bearer <token>"}

$ curl -i http://localhost:8000/ask -X POST -H "Authorization: Bearer abc.def.ghi" -H "Content-Type: application/json" -d '{"question": "Explain JWT"}'
HTTP/1.1 403 Forbidden
{"detail":"Invalid token."}

$ curl -i http://localhost:8000/ask -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d '{"question": "Explain JWT"}'
HTTP/1.1 200 OK
{"question":"Explain JWT","answer":"Agent đang hoạt động tốt! (mock response) Hỏi thêm câu hỏi đi nhé.","usage":{"requests_remaining":9,"budget_remaining_usd":1.6e-05}}
```

Lưu ý: `JWT_SECRET` có giá trị mặc định `"super-secret-change-in-production-please"` trong code. Nếu quên đặt env thì ai đọc repo cũng ký được token admin. Production nên dừng ngay lúc khởi động nếu thiếu secret.

### Exercise 4.3: Rate limiting

`production/rate_limiter.py`:

- **Thuật toán:** sliding window log. Mỗi user có một `deque` timestamp; mỗi request xoá timestamp cũ hơn 60 giây, đếm số còn lại, nếu ≥ limit thì 429 kèm `Retry-After` và `X-RateLimit-*`.
- **Limit:** user thường **10 request/phút** (`rate_limiter_user`), admin **100 request/phút** (`rate_limiter_admin`).
- **"Bypass" cho admin:** `app.py` chọn limiter theo `role` trong JWT (`rate_limiter_admin if role == "admin"`). Đăng nhập `teacher` / `teach456` (role admin) thì được 100 request/phút. Đây là nâng limit theo role chứ không bỏ hẳn giới hạn, và an toàn hơn bỏ hẳn: token admin bị lộ vẫn không spam vô hạn được.
- **Hạn chế:** state nằm trong RAM của process, nên mỗi instance hay worker có bộ đếm riêng và restart là mất. Agent Part 6 chuyển sang Redis sorted set để dùng chung và giữ qua restart.

Gọi liên tục 20 lần với token `student` (request đầu tiên ở 4.2 đã dùng 1 quota):

```text
 1: {"question":"Test 1","answer":"...","usage":{"requests_remaining":8,...}}          [HTTP 200]
 2: {"question":"Test 2","answer":"...","usage":{"requests_remaining":7,...}}          [HTTP 200]
 ...
 9: {"question":"Test 9","answer":"...","usage":{"requests_remaining":0,...}}          [HTTP 200]
10: {"detail":{"error":"Rate limit exceeded","limit":10,"window_seconds":60,"retry_after_seconds":56}} [HTTP 429]
11: {"detail":{"error":"Rate limit exceeded","limit":10,"window_seconds":60,"retry_after_seconds":56}} [HTTP 429]
 ...
20: {"detail":{"error":"Rate limit exceeded","limit":10,"window_seconds":60,"retry_after_seconds":54}} [HTTP 429]

$ curl -i ... (request tiếp theo)
HTTP/1.1 429 Too Many Requests
x-ratelimit-limit: 10
x-ratelimit-remaining: 0
x-ratelimit-reset: 1795594195
retry-after: 53
```

Request thứ 10 trong vòng lặp (thứ 11 tính cả request ở 4.2) bị chặn. `retry_after_seconds` giảm dần theo thời gian (56, 55, 54) vì cửa sổ trượt theo timestamp của request cũ nhất.

### Exercise 4.4: Cost guard implementation

**Cách mình làm** (agent Part 6, `06-lab-complete/app/cost_guard.py`):

- **Tính chi phí:** chi phí mỗi request = `input_tokens/1000 × 0.00015 + output_tokens/1000 × 0.0006` (giá gpt-4o-mini). Token ước lượng khoảng 4 ký tự / token.
- **Lưu trong Redis:** hai key `cost:daily:YYYY-MM-DD` và `cost:monthly:YYYY-MM` (theo UTC), cộng dồn bằng `INCRBYFLOAT` trong một transaction (`MULTI/EXEC`). Key có TTL 2 ngày / 32 ngày nên tự "reset" khi sang ngày hoặc tháng mới mà không cần cron. State nằm trong Redis nên mọi worker và instance dùng chung, và còn sau khi restart.
- **`check_budget()`** chạy **trước** khi gọi LLM: đọc cả hai key bằng `MGET`; chạm `DAILY_BUDGET_USD` hoặc `MONTHLY_BUDGET_USD` (mặc định $1/ngày, $10/tháng, đọc từ env) thì trả **402 Payment Required**.
- **`record_cost()`** chạy **sau** khi LLM trả lời, ghi chi phí thật của request.
- **Đã kiểm tra:** script chấm đặt budget 0.000001 USD và nhận 402 ngay request đầu tiên (Redis đã có chi phí của các lần chạy trước).

**So sánh với solution trong CODE_LAB:**

- Solution tính theo từng user ($10/tháng), còn mình tính tổng của cả agent theo ngày và theo tháng. Rubric Part 6 yêu cầu "tổng chi phí".
- Solution `GET` rồi mới `INCRBYFLOAT` ở hai lệnh riêng, nên hai request đồng thời có thể cùng lọt qua ngưỡng. Ngoài ra nó cộng chi phí *ước tính* trước khi gọi LLM, kể cả khi LLM lỗi.
- Mình tách check (trước) và record (sau, theo chi phí thật). Đổi lại, request cuối cùng có thể vượt ngân sách một chút. Đó là đánh đổi chấp nhận được với số tiền rất nhỏ mỗi request.

**Bug phát hiện trong `04-api-gateway/production/app.py`:** response trả `"budget_remaining_usd": usage.total_cost_usd`, tức là trả **số tiền đã tiêu** chứ không phải số còn lại. Output ở 4.2 cho thấy `budget_remaining_usd: 1.6e-05`, trong khi `GET /me/usage` báo `budget_remaining_usd: 0.999819`. Phải sửa thành `cost_guard.daily_budget_usd - usage.total_cost_usd`.

## Part 5: Scaling & Reliability

### Exercise 5.1: Health checks

`05-scaling-reliability/develop/app.py` đã có sẵn hai endpoint:

- `/health` (**liveness**): process còn sống thì 200, kèm uptime, version và check memory (`degraded` khi RAM > 90%). Không phụ thuộc dependency bên ngoài, vì nếu Redis chết mà `/health` fail thì platform sẽ restart container vô ích.
- `/ready` (**readiness**): 503 khi `_is_ready = False` (đang khởi động hoặc đang tắt), 200 kèm số request đang xử lý. Load balancer dùng endpoint này để tạm ngừng route traffic vào instance, không restart nó.

```text
$ curl http://localhost:8000/health
{"status":"ok","uptime_seconds":0.3,"version":"1.0.0","environment":"development","timestamp":"2026-11-25T08:09:44.484150+00:00","checks":{"memory":{"status":"ok","note":"psutil not installed"}}}
$ curl http://localhost:8000/ready
{"ready":true,"in_flight_requests":1}
```

(`in_flight_requests` là 1 vì middleware đếm luôn chính request `/ready`.)

Agent Part 6 làm readiness theo dependency thật: `/ready` ping Redis, Redis mất kết nối thì 503, trong khi `/health` vẫn 200. Script chấm đã kiểm tra cả hai trường hợp.

### Exercise 5.2: Graceful shutdown

Chạy trong WSL2 (Windows không gửi được SIGTERM): gửi request rồi `kill -TERM` sau 30 ms, lúc request còn đang xử lý.

```text
$ python app.py & PID=$!
$ curl -X POST "localhost:8000/ask?question=Long%20task" &  ; kill -TERM $PID
in-flight request result: {"answer":"Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận."} [HTTP 200]
process exit code: 0

--- server log:
INFO:     Shutting down
INFO:     127.0.0.1:52758 - "POST /ask?question=Long%20task HTTP/1.1" 200 OK
INFO:     Waiting for application shutdown.
2026-11-25 15:09:47,620 INFO 🔄 Graceful shutdown initiated...
2026-11-25 15:09:47,620 INFO ✅ Shutdown complete
INFO:     Application shutdown complete.
INFO:     Finished server process [431]
2026-11-25 15:09:47,621 INFO Received signal 15 — uvicorn will handle graceful shutdown
```

Thứ tự trong log cho thấy cách uvicorn xử lý SIGTERM:

1. Ngừng nhận kết nối mới (`Shutting down`).
2. Chờ request đang chạy trả xong (`POST /ask ... 200 OK`).
3. Chạy phần shutdown của lifespan.
4. Thoát với exit 0.

Dòng `Received signal 15` của `handle_sigterm` tự viết lại in ra **cuối cùng**. Lý do: khi `serve()` bắt đầu, uvicorn đã cài handler của chính nó đè lên `signal.signal(...)` của app, và chỉ gọi lại handler cũ sau khi đã tắt xong. Vì vậy handler tự viết không được chặn hay `sys.exit()`, nếu không sẽ phá luồng tắt của uvicorn. Agent Part 6 không tự đăng ký handler mà chỉ dùng lifespan để dọn tài nguyên (đóng Redis).

### Exercise 5.3: Stateless design

- **Vấn đề:** `conversation_history = {}` nằm trong RAM của từng process. Khi chạy 3 instance sau load balancer, request 1 vào instance A, request 2 vào instance B, và B không có history. Restart hay scale down cũng làm mất toàn bộ history.
- **Cách sửa trong `05-scaling-reliability/production/app.py`:** session lưu ở Redis (`save_session` dùng `SETEX session:<id>` với TTL 1 giờ; `load_session` dùng `GET`), history giữ tối đa 20 tin nhắn. Instance nào cũng đọc ghi được cùng một session, và có thể thêm hoặc bớt instance tuỳ ý.
- **Điểm cần lưu ý:** code có nhánh fallback `_memory_store` khi không kết nối được Redis (dòng 65, 73). Nhánh này âm thầm quay lại lưu state trong RAM, nên chạy nhiều instance mà Redis lỗi sẽ sinh đúng loại bug vừa sửa, mà không có lỗi nào báo ra. Ở production nên fail rõ ràng (`/ready` 503) thay vì fallback. Agent Part 6 làm như vậy: Redis lỗi thì `/ask` trả 503 và `/ready` trả 503.

### Exercise 5.4: Load balancing

```text
$ cd 05-scaling-reliability/production && touch .env.local
$ docker compose -p k5-day14-scaling up -d --build --scale agent=3
NAME                       STATUS
k5-day14-scaling-agent-1   Up 13 seconds (healthy)
k5-day14-scaling-agent-2   Up 13 seconds (healthy)
k5-day14-scaling-agent-3   Up 13 seconds (healthy)
k5-day14-scaling-nginx-1   Up 13 seconds
k5-day14-scaling-redis-1   Up 24 seconds (healthy)
```

- Nginx (cổng **8080** trên host) có `upstream agent_cluster { server agent:8000; }`. Docker DNS trả về IP của cả 3 container agent, nên Nginx chia request round-robin.
- `proxy_next_upstream error timeout http_503` + `proxy_next_upstream_tries 3`: instance lỗi hoặc trả 503 thì Nginx tự thử instance khác.
- Header `X-Served-By $upstream_addr` cho biết instance nào xử lý.

Số request `/chat` mỗi instance nhận được (đếm từ `docker compose logs agent`, gồm cả bài 5.5): agent-1: 2, agent-2: 3, agent-3: 4. Traffic được chia cho cả 3.

### Exercise 5.5: Test stateless

```text
$ python test_stateless.py
Session ID: e21e6ee8-467c-42bf-bd8f-f4fcbc25a36a

Request 1: [instance-902b19]   Q: What is Docker?
Request 2: [instance-bc009c]   Q: Why do we need containers?
Request 3: [instance-902b19]   Q: What is Kubernetes?
Request 4: [instance-c6cb53]   Q: How does load balancing work?
Request 5: [instance-bc009c]   Q: What is Redis used for?
------------------------------------------------------------
Total requests: 5
Instances used: {'instance-902b19', 'instance-bc009c', 'instance-c6cb53'}
✅ All requests served despite different instances!

--- Conversation History ---
Total messages: 10
✅ Session history preserved across all instances via Redis!
```

5 request liên tiếp trong cùng một session đi qua **cả 3 instance**, nhưng history vẫn đủ 10 tin nhắn (5 câu hỏi + 5 câu trả lời). State nằm trong Redis chứ không nằm trong instance nào.

Script không tự tắt instance, nên mình thử thêm bước "kill instance": dừng hẳn `agent-1` rồi gửi tiếp vào **cùng session**:

```text
$ docker stop k5-day14-scaling-agent-1
served_by=instance-c6cb53 turn=7 [HTTP 200]
served_by=instance-bc009c turn=8 [HTTP 200]
served_by=instance-c6cb53 turn=9 [HTTP 200]
served_by=instance-bc009c turn=10 [HTTP 200]
$ GET /chat/<session_id>/history
count = 18
```

Sau khi một instance chết, mọi request vẫn trả 200: Nginx chuyển traffic sang 2 instance còn lại, và hội thoại tiếp tục đúng lượt (turn 7–10). History tăng lên 18 tin nhắn, tức là không mất gì. Đã dọn stack bằng `docker compose -p k5-day14-scaling down`.

## Part 6: Final Project

`python check_production_ready.py`, chạy **đầy đủ** (không `--skip-docker` / `--static-only`) trong `06-lab-complete/`, trên WSL2 Ubuntu (`os.name == "posix"`) với Docker Desktop 29.9.3 bật WSL integration, ngày 2026-11-25:

```text
================================================================
  📋 Checklist Deliverable
     ✅ Dockerfile (multi-stage, < 500 MB)
     ✅ docker-compose.yml (agent + redis)
     ✅ .dockerignore
     ✅ Health check endpoint (GET /health)
     ✅ Readiness endpoint (GET /ready)
     ✅ API Key authentication
     ✅ Rate limiting
     ✅ Cost guard
     ✅ Config từ environment variables
     ✅ Structured logging
     ✅ Graceful shutdown
     ✅ Public URL ready (Railway / Render config)
----------------------------------------------------------------
  Result: 12/12 tiêu chí đạt · 43/43 checks passed
  🎉 PASS 12/12 — PRODUCTION READY! Deploy nào!
================================================================
```

Số đo nổi bật trong lần chạy này:

- Image **273 MB** (build 13s).
- Rate limit: **429 sau 5 request** với `RATE_LIMIT_PER_MINUTE=5`, và vẫn 429 sau khi restart agent.
- Cost guard: **402** ngay request đầu khi budget là 0.000001 USD.
- Log: **17 dòng JSON** hợp lệ.
- SIGTERM khi đang có request: request vẫn **200**, process thoát với **exit 0**.
- `docker stop`: **exit 0 sau 0,5s**.

Public URL: `python check_production_ready.py --public-url https://agent-production-8a6fa083.up.railway.app` cho kết quả `✅ Public URL: GET https://agent-production-8a6fa083.up.railway.app/health → 200`.

---

## AI log

| Phần | Hỏi AI gì | Nhận được gì | Tự sửa và kiểm tra thế nào |
|------|-----------|--------------|----------------------------|
| Đọc đề | Claude Code (Opus 5.5): đọc codebase, tóm tắt yêu cầu và tiêu chí chấm | Bản tóm tắt cấu trúc repo, rubric, các quy ước script chấm cần | Đối chiếu với `README.md`, `RUBRIC.md`, `06-lab-complete/README.md` |
| Part 1, 2, 4, 5 | Claude Code: chạy các ví dụ có sẵn, đo image, trả lời câu hỏi Exercise | Output thật của từng lệnh (dán ở trên), phân tích code mẫu; phát hiện 3 lỗi trong code mẫu (log JSON không hiện ở Part 1, log JSON hỏng ở Part 2, `budget_remaining_usd` sai ở Part 4) | Mọi output và số đo (1667,8 MB / 239,8 MB) lấy từ lệnh chạy thật trong phiên làm bài; SIGTERM chạy trong WSL2 |
| Part 6 | Claude Code: hoàn thành các TODO trong `06-lab-complete/` | Code `app/*.py`, Dockerfile multi-stage, compose, `.dockerignore`, `.env.example`, `railway.toml`, `render.yaml` | `check_production_ready.py` chạy đầy đủ trong WSL2 có Docker: **PASS 12/12, 43/43 checks** (image 273 MB); `docker compose up` test thủ công `/health` 200, `/ready` 200, `/ask` không key 401 |
| Part 3 | Claude Code: deploy agent Part 6 lên Railway, viết `DEPLOYMENT.md` và mục Part 3 | Project Railway (agent + Redis), biến môi trường (key sinh ngẫu nhiên, đặt qua stdin), domain public; bảng so sánh Railway/Render; mô tả pipeline Cloud Run | 4 lệnh tự kiểm CP6 trả 200/200/401/200; vòng lặp rate limit trên cloud ra 429; `--public-url` ✅; quét repo không thấy key |
| Part 7 (CP7) | Claude Code: chạy load test 4 mức concurrency + 3 mức `MOCK_SLOW_RATE`, eval v1/v2, sửa `KNOWLEDGE_BASE`, điền mục 1–3 của `BENCHMARK_REPORT.md` | 10 file JSON trong `reports/` (do lệnh thật sinh ra, không ước lượng); 2 entry KB mới (JWT, Kubernetes HPA); phân tích và snippet GitHub Actions | Eval cuối `--min-accuracy 1.0 --baseline reports/eval_v2.json` ✅ PASS 23/23, 0 regression; số trong report lấy trực tiếp từ các file JSON bằng script đọc file |
| Part 7 (CP8) | Claude Code: dựng stack Prometheus/Grafana/Jaeger/Locust, chạy benchmark scaling, 5 thí nghiệm "phá hệ thống", chụp 4 ảnh, điền mục 4–6 của `BENCHMARK_REPORT.md` | Sửa 1 dòng `production/Dockerfile` (`PYTHONUSERBASE`, agent crash vì thiếu uvicorn); phát hiện `docker compose run` tự scale lại agent nên chạy lại với `--no-deps`; ảnh chụp bằng Playwright từ stack đang chạy | Trước và sau mỗi lần benchmark in số container và số target `up`; thời gian firing đối chiếu với `activeAt` và `ALERTS` của Prometheus; `trace_id` trong ảnh khớp dòng log JSON; số scaling lấy từ `scale_*_stats.csv`; `promtool test rules` SUCCESS |
