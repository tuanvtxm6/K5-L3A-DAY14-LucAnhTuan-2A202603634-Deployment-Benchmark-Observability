# Benchmark & Observability Report — Day 14

**Họ tên:** Luc Anh Tuan
**Ngày chạy:** 2026-11-25
**Môi trường:** Windows 11, Intel Core i5-13500HX (14 nhân / 20 luồng), RAM 32 GB. Python 3.13 venv (fastapi 0.165, uvicorn 0.54, httpx 0.27.2, prometheus-client 0.26.0). Agent (1 process uvicorn) và load generator chạy **trên cùng một máy**. Chạy với `PYTHONUTF8=1` vì trên Windows `run_eval.py` ghi JSON bằng encoding mặc định cp1252 và lỗi ở ký tự tiếng Việt.

Mục 4–5 (production stack) chạy trên Docker Desktop 29.9.3 (WSL2 backend), cùng máy.

File gốc của mọi con số bên dưới: `reports/` (copy từ `06-lab-complete/benchmark-observability/develop/reports/`) và `benchmark-results/` (copy từ `.../production/benchmark-results/`).

---

## 1. SLO của agent

| SLI | SLO | Lý do chọn ngưỡng này |
|---|---|---|
| p95 latency `/ask` | < 500 ms | Một lần gọi mock LLM mất khoảng 200 ms (p50 ≈ 203 ms ở mọi mức tải). 500 ms chừa khoảng 2,5 lần cho queueing và retrieval, mà người dùng chat vẫn thấy "tức thì". Dùng p95 chứ không dùng average để không bị tail latency che mất (mục 2.2) |
| Error rate `/ask` | < 2 % | Upstream LLM tự lỗi khoảng 1% (`MOCK_ERROR_RATE=0.01`). SLO 2% cho error budget gấp đôi mức nền, nên lỗi do code hay hạ tầng của mình sẽ lộ ra ngay |
| Accuracy trên golden dataset | ≥ 100 % (23/23), không regression | Golden dataset là bộ regression test: chỉ 23 câu, mỗi câu sai là một hành vi đã từng đúng bị hỏng, nên CI chặn mọi câu sai (`--min-accuracy 1.0`) |

---

## 2. Benchmark hiệu năng (develop, `load_test.py`)

### 2.1 Concurrency vs latency (agent v2)

`python load_test.py --url http://localhost:8002 --requests 300 --concurrency <c> --out reports/load_v2_c<c>.json`

| Concurrency | RPS | p50 (ms) | p95 (ms) | p99 (ms) | Error rate | SLO |
|---|---|---|---|---|---|---|
| 1 | 4.19 | 202.9 | 234.8 | 1682.6 | 1.33% | ✅ PASS |
| 10 | 35.71 | 202.4 | 467.8 | 1726.3 | 1.33% | ✅ PASS |
| 50 | 96.01 | 203.8 | 506.9 | 1741.9 | 0.67% | ❌ FAIL (p95) |
| 100 | 46.93 | 647.1 | 4077.9 | 4434.5 | 1.33% | ❌ FAIL (p95) |

**Nhận xét:** RPS chỉ tăng tuyến tính ở mức tải thấp, sau đó chững lại rồi **giảm**.

- **c = 1 → 10:** RPS tăng 8,5 lần (4,19 → 35,71). Đúng theo định luật Little, RPS ≈ concurrency / latency trung bình: 10 / 0,275 s ≈ 36. Agent chủ yếu chờ I/O (`asyncio.sleep` của mock LLM), nên một event loop xử lý song song được nhiều request.
- **c = 50:** nếu không có queueing thì RPS lý tưởng là 50 / 0,294 ≈ 170, nhưng thực tế chỉ đạt 96. p90 nhảy lên 487 ms vì request bắt đầu phải xếp hàng.
- **c = 100:** RPS giảm còn 46,9, p50 tăng lên 647 ms, p95 lên 4,1 s. Server đã bão hoà. Chỉ có **một process uvicorn với một event loop**, phải lo cả parse JSON, pydantic, ghi log JSON, cập nhật Prometheus cho mọi request. Load generator lại chạy cùng máy và tranh CPU với server. Thêm concurrency lúc này chỉ làm hàng đợi dài ra.
- Điểm gãy nằm giữa c = 10 và c = 50. Muốn tăng throughput phải scale ngang (nhiều worker hoặc instance sau load balancer, như phần 5) chứ không phải tăng concurrency.
- Error rate giữ ở khoảng 1% mọi mức tải. Đó là lỗi 503 do mock upstream tự sinh, không phải do quá tải.

### 2.2 Tail latency

Bật lại v2 với từng giá trị `MOCK_SLOW_RATE`, chạy `--requests 300 --concurrency 20` (`reports/load_v2_slow_<rate>.json`).

| `MOCK_SLOW_RATE` | avg (ms) | p95 (ms) | p99 (ms) |
|---|---|---|---|
| 0 | 218.0 | 452.5 | 483.0 |
| 0.03 (mặc định) | 252.9 | 482.5 | 1696.0 |
| 0.1 | 384.2 | 1698.7 | 1979.0 |

**Nhận xét:**

- **p99 nhạy nhất.** Chỉ cần 3% request chậm là p99 tăng 3,5 lần (483 → 1696 ms), trong khi avg chỉ tăng 16% (218 → 253 ms) và p95 gần như không đổi. Lý do: 3% < 5% nên request chậm nằm ngoài p95 nhưng lọt vào p99. Lên 10% thì p95 cũng vỡ (1698,7 ms), vì đã có hơn 5% request chậm.
- p50 không đổi (khoảng 203 ms ở cả ba mức), nên request "bình thường" không bị ảnh hưởng.
- Throughput cũng giảm (89,0 → 58,5 → 43,5 RPS). Load test dùng số worker cố định (closed loop), nên mỗi request chậm giữ một slot gần 1,7 s.
- **Vì sao không đặt SLO theo average:** ở mức 0,03, avg 253 ms trông rất ổn nhưng cứ 100 người thì 1 người chờ khoảng 1,7 s. Average trộn chung số đông nhanh với số ít rất chậm, nên che mất trải nghiệm tệ nhất. Người dùng nhớ lần chờ lâu nhất chứ không nhớ giá trị trung bình, nên SLO đặt theo p95/p99.
- Ngay cả khi `MOCK_SLOW_RATE=0`, p95 vẫn là 452,5 ms, cao hơn hẳn p50 khoảng 203 ms. Đây là thời gian xếp hàng trong một event loop ở concurrency 20, khớp với p95 = 467,8 ms ở c = 10 trong bảng 2.1.

---

## 3. Benchmark chất lượng (develop, `eval/run_eval.py`)

| Chỉ số | Agent v1 | Agent v2 | Agent v2 (sau khi sửa KB) |
|---|---|---|---|
| Accuracy | 4.3% (1/23) | 91.3% (21/23) | 100.0% (23/23) |
| Avg tokens / câu | 57.0 | 109.1 | 116.6 |
| Tổng cost (USD) | 0.000402 | 0.000762 | 0.000816 |
| Cost / câu đúng (USD) | 0.00040215 | 0.00003627 | 0.00003547 |
| Quality gate | FAIL (gate 80%) | PASS (gate 80%, so với v1) | PASS (gate 100%, so với v2, 0 regression) |

File: `reports/eval_v1.json`, `reports/eval_v2.json`, `reports/eval_v2_fixed.json`.

**Những câu v2 trả lời sai ban đầu và cách bạn sửa:**

| ID | Keyword bị thiếu | Bạn đã sửa gì |
|---|---|---|
| `security-02` ("JWT khác gì API key trong xác thực?") | `token`, `hết hạn`, `chữ ký` | `KNOWLEDGE_BASE` không có tài liệu nào match nên v2 trả câu chung chung. Thêm entry trigger `["jwt", "json web token"]`: JWT là token có chữ ký, có thời điểm hết hạn (`exp`); API key là chuỗi cố định, không tự hết hạn |
| `scaling-03` ("Kubernetes HPA scale dựa trên những chỉ số nào?") | `cpu`, `memory`, `custom metrics` | Cùng nguyên nhân. Thêm entry trigger `["kubernetes", "hpa", "horizontal pod autoscaler", "autoscal"]`: HPA scale theo CPU, memory hoặc custom metrics |

Trước khi thêm, đã kiểm tra các trigger mới không xuất hiện trong câu hỏi nào khác của golden dataset. Entry được thêm vào **cuối** danh sách: khi số keyword match bằng nhau, retrieval sort ổn định nên giữ tài liệu cũ, không gây regression. Bản sửa áp dụng cho cả `develop/utils/mock_llm.py` và `production/utils/mock_llm.py`.

Lệnh cuối: `python eval/run_eval.py --url http://localhost:8002 --min-accuracy 1.0 --baseline reports/eval_v2.json --out reports/eval_v2_fixed.json` in ra `Quality gate (accuracy ≥ 100%, không regression): ✅ PASS`, exit code 0, 2 câu được cải thiện, 0 regression.

> p95 latency trong `eval_v2_fixed.json` (1537.9 ms) cao hơn `eval_v2.json` (237.3 ms) là do mẫu quá nhỏ: 23 request mà có 2 request trúng `MOCK_SLOW_RATE`. Không phải do sửa KB, vì retrieval vẫn là keyword match như cũ. Latency phải đo bằng load test (mục 2).

**Nhận xét về trade-off chất lượng / chi phí / latency giữa v1 và v2:**

- v2 tốn gấp đôi token mỗi câu (57,0 → 109,1, vì prompt có thêm context retrieval) và tổng cost tăng 1,9 lần.
- Nhưng accuracy tăng từ 4,3% lên 91,3%, nên **cost / câu đúng giảm khoảng 11 lần** (0,00040215 → 0,00003627 USD). Sau khi sửa KB, chỉ số này còn giảm tiếp (0,00003547), dù token tăng nhẹ.
- Latency tăng khoảng 30 ms (p50 170,9 → 203,0 ms trong eval) do thêm bước retrieval và prompt dài hơn. Vẫn nằm rất xa dưới SLO 500 ms.
- Kết luận: v2 đáng tiền. "Cost / câu" khiến v1 trông rẻ hơn, nhưng một câu trả lời sai là chi phí vô ích. "Cost / câu đúng" đo đúng thứ người dùng thật sự nhận được.

**Quality gate trong CI (Exercise 7.3, bài 4):** chạy eval v1 với `--baseline reports/eval_v2.json` thì quality gate ❌ FAIL, **exit code 1**, 20 câu bị regression. `run_eval.py` trả exit code khác 0 khi accuracy dưới `--min-accuracy` hoặc có regression so với baseline. Vì vậy chỉ cần cho job CI chạy lệnh này: step fail thì job fail, và branch protection (required status check) chặn nút merge.

```yaml
# .github/workflows/eval-gate.yml
name: eval-gate
on: pull_request
jobs:
  eval:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: 06-lab-complete/benchmark-observability/develop
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.11" }
      - run: pip install -r requirements.txt
      - name: Start agent (code của PR)
        run: |
          AGENT_VERSION=v2 PORT=8002 python app.py &
          for i in $(seq 1 30); do curl -sf localhost:8002/health && break; sleep 1; done
      - name: Quality gate — exit code ≠ 0 thì job fail → PR không merge được
        run: python eval/run_eval.py --url http://localhost:8002 --min-accuracy 1.0 --baseline reports/eval_v2_fixed.json
```

---

## 4. Observability stack (production)

Stack trong `06-lab-complete/benchmark-observability/production/`, chạy bằng `docker compose -p k5-day14-observability ...` (Docker Desktop 29.9.3), ngày 2026-11-25. Giờ trong bài là UTC.

**Hai lỗi gặp khi dựng stack và cách xử lý:**

1. **Agent crash liên tục** với `ModuleNotFoundError: No module named 'uvicorn'`. Nguyên nhân: Dockerfile tạo user bằng `useradd -d /app agent`, nên `HOME=/app` và Python tìm user site-packages ở `/app/.local/...`. Trong khi đó package được `COPY` vào `/home/agent/.local`. Đã kiểm chứng: `site.getusersitepackages()` in ra `/app/.local/lib/python3.11/site-packages`. Sửa bằng một dòng `ENV PYTHONUSERBASE=/home/agent/.local` trong `production/Dockerfile`.
2. **Lệnh benchmark scaling trong CP8 không scale thật.** `docker compose run --rm locust ...` tự khởi động lại các service phụ thuộc (locust → nginx → agent) theo `deploy.replicas: 2`, nên lần chạy đầu cả `scale_1` lẫn `scale_3` đều chạy **2 instance**. Phát hiện nhờ panel "Instances up" = 2 và `agent-2` khởi động đúng lúc Locust chạy. Đã chạy lại với `docker compose run --no-deps --rm locust ...` và in số container cùng số target `up` của Prometheus trước và sau mỗi lần (xem mục 5).

Unit test alert: `promtool test rules /p/alert_rules_test.yml` cho kết quả `SUCCESS`.

### 4.0 Exercise 7.4: đọc dashboard khi có tải

Truy vấn Prometheus lúc 08:43:58, khi Locust chạy 100 user trên 3 instance (cùng thời điểm với ảnh Grafana):

| Chỉ số | Giá trị |
|---|---|
| Traffic `/ask` | 61,6 req/s (ảnh Grafana); 20,56 / 20,89 / 20,60 req/s cho 3 instance |
| p50 / **p95** / p99 | 174 ms / **294 ms** / 1839 ms |
| **avg** | **235 ms** |
| Tokens/s | input 4861,8, output 2584,5 |
| Cost burn rate | $6,33/giờ |

1. **p95 = 294 ms, avg = 235 ms.** Hai số gần nhau vì 95% request chỉ tốn khoảng thời gian một lần gọi mock LLM (~200 ms). Khoảng cách thật nằm giữa **p95 và p99 (1839 ms)**: khoảng 3% request bị `MOCK_SLOW_RATE` làm chậm thêm 1,5 s. Avg không cho thấy đuôi này, chỉ p99 mới thấy.
2. **Nginx chia đều traffic:** 3 instance nhận 20,56 / 20,89 / 20,60 req/s, lệch nhau dưới 2%, nhờ round-robin qua `server agent:8000 resolve`. Panel "Requests/s theo instance" trong ảnh thấy rõ: lần chạy 1 instance một đường khoảng 60 req/s, sau đó 3 đường khoảng 20 req/s.
3. **Input tokens/s lớn hơn output khoảng 1,9 lần** (4861,8 so với 2584,5). Prompt của v2 gồm system prompt, tài liệu retrieval và câu hỏi (trace mẫu: 72 input token), còn câu trả lời chỉ khoảng một đoạn ngắn (40 output token). Tuy vậy, output token đắt gấp 4 lần input ($0.0006 so với $0.00015 / 1K), nên phần lớn chi phí vẫn đến từ output.
4. **`curl http://localhost:8080/metrics` trả 404** (trang lỗi mặc định của `nginx/1.28.3`). `nginx.conf` có `location = /metrics { return 404; }`: metric chỉ dành cho Prometheus, scrape trực tiếp từng container trong mạng nội bộ `obs_net`. Lý do không public: metric lộ route, lưu lượng, chi phí, phiên bản và các thông tin nội bộ khác.

### 4.1 Screenshot

Ảnh chụp từ stack đang chạy bằng trình duyệt headless (Playwright).

- [x] Dashboard Grafana khi đang có tải: [screenshots/grafana-dashboard.png](screenshots/grafana-dashboard.png). Chụp lúc 08:43, 100 user, 3 instance: Traffic 61,6 req/s, Error rate 1,09%, p95 294,72 ms, Saturation 17 in-flight, Instances up 3, LLM cost (1h) $0,9985.
- [x] Trace chậm trong Jaeger: [screenshots/jaeger-slow-trace.png](screenshots/jaeger-slow-trace.png). Span LLM chiếm 1,7 s trên tổng 1,7 s (mục 4.3).
- [x] Trace lỗi trong Jaeger, có event `exception`: [screenshots/jaeger-error-trace.png](screenshots/jaeger-error-trace.png). Chi tiết bên dưới.
- [x] Trang Alerts của Prometheus với **2 alert cùng firing** (`AgentHighErrorRate` + `AgentHighLatencyP95`): [screenshots/prometheus-alert-firing.png](screenshots/prometheus-alert-firing.png). Chụp lúc 08:53:38, khi đang bật đồng thời thí nghiệm A và B.

**Trace lỗi** `eabbde9fc0fd958aa1c7dcdcce7deb13` (trong thí nghiệm A): `POST /ask` trả **503** sau 113,1 ms. Span `chat mock-llm-small` (87,3 ms) có `error.type = upstream_unavailable` và event `exception`: `exception.type = utils.mock_llm.LLMError`, `exception.message = Upstream LLM error: 503 Service Unavailable (mock)`. Dòng log cùng `trace_id`:

```json
{"ts": "2026-11-25T08:48:35.671022+00:00", "level": "INFO", "service": "observable-agent", "instance": "db7c286c6640", "event": "llm_error", "trace_id": "eabbde9fc0fd958aa1c7dcdcce7deb13", "span_id": "8a78d0512a24c255", "request_id": "3cb7449af49b", "error": "Upstream LLM error: 503 Service Unavailable (mock)"}
{"ts": "2026-11-25T08:48:35.671260+00:00", "level": "INFO", "service": "observable-agent", "instance": "db7c286c6640", "event": "http_request", "trace_id": "eabbde9fc0fd958aa1c7dcdcce7deb13", "span_id": "8a78d0512a24c255", "request_id": "3cb7449af49b", "method": "POST", "route": "/ask", "status": 503, "ms": 112.7}
```

### 4.2 Thí nghiệm "phá hệ thống"

Tải nền: Locust headless 30 user (`--no-deps`, khoảng 18,5 req/s), 2 instance. Mỗi thay đổi được áp bằng `<BIẾN>=<giá trị> docker compose -p k5-day14-observability up -d --scale agent=2 agent`. T0 là lúc lệnh trả về. Thời gian firing đo bằng script poll `/api/v1/alerts` mỗi 2 giây, đối chiếu với `activeAt` và chuỗi `ALERTS{alertstate="firing"}` của Prometheus.

| # | Thay đổi | Alert nào firing | Thời gian từ lúc đổi → firing | Panel nào phản ánh rõ nhất |
|---|---|---|---|---|
| A | `MOCK_ERROR_RATE=0.2` | `AgentHighErrorRate` (giá trị 18,4–18,8%) | T0 08:46:29 → pending +23 s (08:46:52) → **firing +86 s** (08:47:55) | **Error rate** (đỏ, khoảng 19%), "Requests/s theo status code" (đường 503 tăng), "LLM calls theo kết quả" |
| B | `MOCK_SLOW_RATE=0.3` (vẫn giữ A) | `AgentHighLatencyP95` (p95 = 1,897 s) | T0 08:50:17 → pending +21 s → **firing +141 s** | **Latency p50/p95/p99**: p95 1,897 s, p99 1,98 s, nhưng avg chỉ 0,576 s và p50 vẫn 0,191 s |
| C | `LLM_MODEL=mock-llm-large` (A, B trả về mặc định) | `LLMCostBurnRateHigh` | T0 08:54:25 → pending +23 s → **firing +143 s** | **Cost burn rate**: $35,9/giờ (cửa sổ 1 phút), khoảng 13,5 lần baseline $2,66/giờ; p50 cũng tăng 190 → 411 ms (latency factor 2,5) |
| D | `docker pause k5-day14-observability-agent-1` | `AgentInstanceDown` | T0 08:57:33 → pending +15 s → **firing +45 s** | **Instances up** 2 → 1. Error rate của agent vẫn xanh (0,26%), trong khi log Nginx cùng lúc có **92 × 502 và 27 × 504** |
| E | `AGENT_VERSION=v1` | **Không có** | (theo dõi 150 s) | **Retrieval hit ratio** mất (không còn series), Tokens/s giảm khoảng một nửa (input 1423 → 684/s), bảng "Phiên bản đang chạy" đổi `v2` → `v1`; p95 còn *giảm* (295 → 199 ms) |

**Nhận xét:**

- **Thời gian tới lúc firing** xấp xỉ "thời gian để `rate(...[1m])` vượt ngưỡng" (khoảng 15–25 s, gồm cả chu kỳ scrape và evaluate 5 s) cộng với `for` của rule (A: 1m, B và C: 2m, D: 30s). `for` là cái giá trả để tránh báo động giả, đổi lại phát hiện chậm hơn.
- **Ngưỡng cost alert quá thấp so với traffic.** Ở cấu hình mặc định (model nhỏ, v2), 30 user (18,5 req/s) đã tốn khoảng $2,66/giờ và 100 user khoảng $6,3/giờ, đều trên ngưỡng $2/giờ. Vì vậy `LLMCostBurnRateHigh` **firing ngay ở baseline** (từ 08:34:12, trong lúc benchmark scaling). Ở thí nghiệm C, alert đang `inactive` chỉ vì thí nghiệm A và B làm giảm số request thành công. Ngưỡng cost nên đặt theo ngân sách thật và traffic dự kiến, hoặc dùng tỉ lệ so với baseline, nếu không alert sẽ luôn đỏ và bị bỏ qua (alert fatigue).
- **D cho thấy giới hạn của metric phía server.** Request vào instance bị pause chết ở Nginx (502/504), agent chưa bao giờ xử lý chúng nên không đếm. Cần thêm metric ở load balancer (nginx/ingress 5xx) hoặc synthetic check từ bên ngoài. `pause` chứ không phải `stop`, vì container bị stop biến mất khỏi Docker DNS: Prometheus coi target không tồn tại chứ không phải `up == 0`, nên không có alert.

**Thí nghiệm E không làm alert nào firing. Bạn sẽ phát hiện loại sự cố này trong production bằng cách nào?**

Đây là lỗi **chất lượng**, không phải lỗi hạ tầng: request vẫn 200, latency còn tốt hơn, nhưng câu trả lời thành câu chung chung (`"Tôi là AI agent được deploy lên cloud..."`, `agent_version: v1`). Cách phát hiện:

1. **Trước khi deploy:** quality gate trong CI, tức `run_eval.py --min-accuracy 1.0 --baseline ...` (mục 3). v1 cho accuracy 4,3% và exit code 1, nên đã bị chặn trước khi lên production.
2. **Metric chất lượng online có alert:**
   - **Retrieval:** alert khi retrieval hit ratio giảm mạnh, hoặc khi series `agent_retrieval_total` biến mất (`absent()`).
   - **Token:** alert khi avg output tokens/câu lệch khỏi baseline.
   - **Phiên bản:** alert khi `agent_info{version}` đổi ngoài kế hoạch deploy.
3. **Eval online:** định kỳ chạy golden dataset vào production (synthetic eval), hoặc lấy mẫu khoảng 1% traffic cho LLM-as-judge chấm, rồi đẩy điểm thành metric `agent_quality_score` và đặt alert.
4. **Phản hồi người dùng:** tỉ lệ 👎 / hỏi lại / bỏ cuộc trò chuyện, cộng với canary deploy (so sánh chất lượng v-mới với v-cũ trên một phần traffic trước khi chuyển hết).

### 4.3 Debug bằng trace

Trace chậm nhất tìm được (trong 200 trace ≥ 1 s của 4 phút gần nhất, lúc chạy thí nghiệm B):

- `trace_id`: `10f92300df1f146730f98ab08ba50840`
- Tổng thời gian: **1734,2 ms** (`POST /ask`, HTTP 200)
- Span chiếm nhiều thời gian nhất: **`chat mock-llm-small`** (**1688,7 ms, 97,4%**; `gen_ai.usage.input_tokens = 72`, `output_tokens = 40`). Các span khác: `agent.retrieve_context` 43,9 ms (2,5%); `agent.run` bao trùm 1732,8 ms.
- **Kết luận:** request chậm vì upstream LLM chậm (tail latency của provider), không phải do code agent hay retrieval. Cách giảm: timeout và retry/hedging khi gọi LLM, streaming để người dùng thấy token đầu sớm, hoặc fallback sang model khác.
- Dòng log tương ứng (grep theo `trace_id` trong `docker compose -p k5-day14-observability logs agent`):

```json
{"ts": "2026-11-25T08:53:43.276710+00:00", "level": "INFO", "service": "observable-agent", "instance": "e5d98f95d5fc", "event": "llm_call", "trace_id": "10f92300df1f146730f98ab08ba50840", "span_id": "040afee89fdcf797", "request_id": "8a64b4d9a8da", "model": "mock-llm-small", "agent_version": "v2", "input_tokens": 72, "output_tokens": 40, "cost_usd": 3.5e-05, "llm_ms": 1689.3}
{"ts": "2026-11-25T08:53:43.277016+00:00", "level": "INFO", "service": "observable-agent", "instance": "e5d98f95d5fc", "event": "http_request", "trace_id": "10f92300df1f146730f98ab08ba50840", "span_id": "040afee89fdcf797", "request_id": "8a64b4d9a8da", "method": "POST", "route": "/ask", "status": 200, "ms": 1733.8}
```

`llm_ms = 1689.3` trong log khớp với span LLM 1688,7 ms trong Jaeger. Log cho biết *chuyện gì xảy ra* (model, token, cost), trace cho biết *thời gian nằm ở bước nào*.

---

## 5. Benchmark scaling (production, Locust headless, 100 users, 2 phút)

Lệnh: `docker compose -p k5-day14-observability run --no-deps --rm locust -f /mnt/locust/locustfile.py --host http://nginx --headless -u 100 -r 10 -t 2m --csv /mnt/results/scale_<n>`. Trước và sau mỗi lần chạy đã kiểm tra: `scale_1` có 1 container agent và 1 target `up`; `scale_3` có 3 container và 3 target `up`.

Số liệu dòng `/ask` trong `benchmark-results/scale_1_stats.csv` và `scale_3_stats.csv`:

| Số instance | RPS | p50 (ms) | p95 (ms) | p99 (ms) | Failures |
|---|---|---|---|---|---|
| 1 | 60.70 | 190 | 220 | 1700 | 82 / 7272 (1.13%), toàn bộ là 503 |
| 3 | 59.71 | 190 | 220 | 1700 | 81 / 7152 (1.13%), toàn bộ là 503 |

(Tính cả `/health`, dòng Aggregated: 66.52 và 65.99 req/s.)

**Giải thích kết quả:** Scale lên 3 instance **không cải thiện** gì: RPS, p50, p95 và p99 gần như y hệt.

- **Nút thắt không nằm ở agent mà ở chính load generator.** Locust chạy vòng kín: 100 user, mỗi user nghỉ `between(0.5, 2)` giây (trung bình 1,25 s) giữa các request. Thông lượng tối đa vì thế khoảng 100 / (1,25 + 0,235) ≈ **67 req/s**, khớp với 66,5 req/s đo được.
- **Một instance đã đủ sức.** p95 của 1 instance (220 ms) bằng đúng latency của một lần gọi mock LLM khi không tải, nên chưa có hàng đợi. Thêm instance chỉ chia nhỏ cùng một lượng tải (khoảng 60 → 3 × 20 req/s, ảnh Grafana).
- **p99 = 1700 ms ở cả hai cấu hình** là tail latency của LLM upstream (`MOCK_SLOW_RATE` 3% cộng 1,5 s). Panel "LLM latency p95 theo model" cho thấy gần như toàn bộ latency nằm trong lời gọi LLM. Scale ngang không làm LLM nhanh hơn, chỉ có thay model/provider, timeout và hedging, hoặc streaming mới cải thiện được. **Failures 1,13%** cũng đến từ upstream (`MOCK_ERROR_RATE` 1%), không phụ thuộc số instance.
- **Muốn thấy lợi ích của scale,** phải tăng tải tới khi 1 instance bão hoà: nhiều user hơn hoặc bỏ thời gian nghỉ. Mỗi container chỉ có `cpus: "0.5"`, và ở develop (mục 2.1) một process đã gãy ở khoảng 50 concurrent request.

---

## 6. Kết luận

- 3 điều bạn học được về benchmark & observability:
  1. **Percentile chứ không phải trung bình.** Ở cả develop (mục 2.2) lẫn production (4.0), avg luôn trông ổn (218–253 ms) trong khi p99 khoảng 1,7 s. SLO và alert phải đặt theo p95/p99, và cần biết đuôi đó đến từ đâu (trace cho thấy 97% thời gian nằm ở span LLM).
  2. **Số đo chỉ đúng khi biết chính xác mình đang đo cái gì.** Lần benchmark scaling đầu tiên âm thầm chạy 2 instance cho cả hai cấu hình (`docker compose run` kéo dependency lên lại). Kết quả 1 vs 3 instance bằng nhau vì Locust là nút thắt chứ không phải agent. Phải kiểm tra điều kiện thí nghiệm (số instance, số target `up`) trước khi tin vào con số.
  3. **Mỗi tầng chỉ thấy phần của mình, và metric hạ tầng không thấy được chất lượng.** Thí nghiệm D: agent báo 0,26% lỗi trong khi Nginx trả 119 lỗi 502/504. Thí nghiệm E: dashboard còn "xanh hơn" khi chất lượng sụp. Cần kết hợp metric (có vấn đề không), trace (ở bước nào), log có `trace_id` (chi tiết gì), cộng với eval cho chất lượng.
- Nếu đưa agent này lên production thật, bạn sẽ thêm metric / alert nào nữa?
  - **Tầng load balancer:** tỉ lệ 5xx của Nginx/ingress, cộng với synthetic check `/ask` từ bên ngoài mỗi phút. Bắt được lỗi kiểu thí nghiệm D.
  - **Chất lượng:** điểm eval online (golden set chạy định kỳ, LLM-as-judge trên mẫu traffic) có alert khi giảm; alert `absent(agent_retrieval_total)` hoặc khi hit ratio giảm; tỉ lệ phản hồi 👎 của người dùng.
  - **Cost:** ngưỡng theo ngân sách thật (vd dự báo vượt budget ngày, dựa trên `predict_linear` của `llm_cost_usd_total`) thay cho ngưỡng cố định $2/giờ đang firing ngay ở baseline. Thêm cost theo API key hoặc tenant (ở log hoặc trace, không ở label Prometheus để tránh bùng nổ cardinality).
  - **Upstream LLM:** latency p99 và error rate theo provider/model, kèm timeout và circuit breaker; tỉ lệ request bị rate limit (429) và tỉ lệ budget (402) ở agent Part 6.
  - **Saturation:** CPU/memory container, độ sâu hàng đợi; alert theo burn rate của error budget (multi-window) thay cho ngưỡng tức thời.
