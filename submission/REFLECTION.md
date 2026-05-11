# Báo Cáo Lab Day 23 — Observability Stack cho AI Service

> Mỗi phần cần được điền đầy đủ. Giám khảo đọc kỹ nhất phần "Thay đổi quan trọng nhất".

**Sinh viên:** AI20k Lab Student
**Ngày nộp bài:** 2026-05-11

---

## 1. Thông số phần cứng và kết quả cài đặt

Kết quả chạy `python 00-setup/verify-docker.py`:

```
Docker:        OK  (phiên bản 29.3.1)
Compose v2:    OK  (phiên bản 5.1.1)
RAM khả dụng:  5.66 GB (đạt yêu cầu)
Cổng đang dùng: [8000, 9090, 9093, 3000, 3100, 16686, 4317, 4318, 8888]
Đã ghi báo cáo: 00-setup/setup-report.json
```

Tất cả 7 container khởi động thành công:

- **day23-app** — FastAPI inference service (cổng 8000)
- **day23-prometheus** — Thu thập metrics (cổng 9090)
- **day23-alertmanager** — Định tuyến cảnh báo → Slack (cổng 9093)
- **day23-grafana** — Hiển thị dashboard (cổng 3000)
- **day23-loki** — Tổng hợp log (cổng 3100)
- **day23-jaeger** — Distributed tracing (cổng 16686)
- **day23-otel-collector** — Nhận và lấy mẫu trace (cổng 4317/4318/8888)

---

## 2. Track 02 — Dashboard và Cảnh báo

### 6 panel chính (ảnh chụp màn hình)

Xem ảnh `submission/screenshots/dashboard-overview.png`.

### Panel tỷ lệ đốt ngân sách lỗi (Burn-rate)

Xem ảnh `submission/screenshots/slo-burn-rate.png`.

### Vòng đời cảnh báo: bật lên và tắt xuống

| Thời điểm | Sự kiện | Bằng chứng |
|---|---|---|
| T0 (11:52:48) | Dừng `day23-app` bằng lệnh `docker stop day23-app` | `docker ps` cho thấy container đã dừng |
| T0+90 giây (11:54:20) | Cảnh báo `ServiceDown` kích hoạt trong Alertmanager | Ảnh `alertmanager-firing.png` — trạng thái=active, mức độ=critical |
| T1 (11:54:30) | Khôi phục app bằng lệnh `docker start day23-app` | App trở lại trạng thái healthy sau 10 giây |
| T1+90 giây (11:56:00) | Cảnh báo giải quyết, Slack nhận thông báo | Ảnh `slack-resolved.png` |

### Điều bất ngờ về Prometheus / Grafana

Điều gây bất ngờ nhất là Grafana **không tự động giữ UID cố định cho datasource** khi không khai báo tường minh trong file provisioning. Sau mỗi lần khởi động lại container, Grafana tự sinh một UID ngẫu nhiên mới (ví dụ `PBFA97CFB590B2093`), trong khi các file JSON dashboard đang hard-code `"uid": "prometheus"`. Hậu quả là tất cả dashboard hiển thị "No data" một cách âm thầm — không có thông báo lỗi nào, không có cảnh báo nào. Chỉ cần thêm dòng `uid: prometheus` vào `datasources.yml` là khắc phục được ngay. Đây là bài học thực tế về tính **idempotent** của cấu hình: cho dù khởi tạo lại container bao nhiêu lần, kết quả phải luôn như nhau.

---

## 3. Track 03 — Tracing và Log

### Một ảnh chụp trace từ Jaeger

Xem ảnh `submission/screenshots/jaeger-trace.png` — hiển thị chuỗi span `embed-text → vector-search → generate-tokens`.

Trace `bf1327271d62ff87779165a11cd58cb1` từ Jaeger xác nhận:

- Span gốc: `inference-api: predict` — tổng thời gian 199,29 ms
- Span con 1: `embed-text` — 5,59 ms
- Span con 2: `vector-search` — 10,53 ms
- Span con 3: `generate-tokens` — 168,2 ms

### Dòng log tương quan với trace

```json
{
  "model": "llama3-mock",
  "input_tokens": 6,
  "output_tokens": 33,
  "quality": 0.926,
  "duration_seconds": 0.1897,
  "trace_id": "bf1327271d62ff87779165a11cd58cb1",
  "event": "prediction served",
  "level": "info",
  "timestamp": "2026-05-11T11:34:26.037710Z"
}
```

`trace_id: bf1327271d62ff87779165a11cd58cb1` — khớp với trace trong Jaeger (xem ảnh `jaeger-trace.png`). Đây là minh chứng cho khả năng **tương quan log-trace**: khi một request gặp vấn đề, ta có thể đi từ log → trace_id → Jaeger để xem toàn bộ luồng xử lý nội bộ.

### Tính toán tỷ lệ lấy mẫu đuôi (Tail-sampling)

**Cấu hình chính sách hiện tại (như được cài sẵn):**

```
decision_wait:       30 giây
keep-errors:         Giữ 100% trace có trạng thái ERROR
keep-slow:           Giữ 100% trace có độ trễ > 2000 ms
probabilistic-1pct:  Giữ 1% các trace khỏe mạnh còn lại
```

**Lưu lượng thực tế trong quá trình load test** (60 giây, 10 người dùng, tốc độ tăng 2/giây):

| Chỉ số | Giá trị |
|---|---|
| Tổng request | ~120 req / 60 giây ≈ **2 req/giây** |
| Độ trễ trung bình | ~200 ms (dưới ngưỡng 2.000 ms) |
| Tỷ lệ lỗi | 0% (không gọi `?fail=true`) |

**Phép tính:**

```
Trace khỏe mạnh/giây  = 2 req/giây
Trace chậm giữ lại   = 0  (tất cả < 2 giây)
Trace lỗi giữ lại    = 0  (không có lỗi được đưa vào)
Giữ theo xác suất    = 2 req/giây × 1% = 0,02 trace/giây

Trong cửa sổ 60 giây:
  Tổng trace tạo ra  = 2 × 60 = 120 trace
  Dự kiến giữ lại    = 120 × 0,01 = 1,2 trace ≈ 1–2 trace được lấy mẫu
  Tỷ lệ loại bỏ      = (120 − 1,2) / 120 ≈ 99%
```

**Xác minh với trace lỗi** (rubric mục 14):
Khi gọi `POST /predict?fail=true`, chính sách `status_code == ERROR` kích hoạt → **100% trace lỗi được giữ lại**, bất kể chính sách xác suất 1%. Điều này đã được xác minh bằng cách gọi endpoint với tham số `fail=true` và xác nhận trace xuất hiện trong Jaeger với `status=ERROR`.

**Điều chỉnh trong lab:** Chính sách tạm thời đặt `sampling_percentage: 100` để minh họa toàn bộ pipeline trace từ đầu đến cuối. Trong môi trường production thực tế với ≥ 100 req/giây, cài đặt 1% sẽ phù hợp để giới hạn lưu trữ Jaeger ở mức ~1 span/giây — tiết kiệm tài nguyên đáng kể mà không mất đi khả năng debug.

---

## 4. Track 04 — Phát hiện Data Drift

### Điểm số PSI

Nội dung file `04-drift-detection/reports/drift-summary.json`:

```json
{
  "prompt_length": {
    "psi": 3.461,
    "kl": 1.7982,
    "ks_stat": 0.702,
    "ks_pvalue": 0.0,
    "drift": "yes"
  },
  "embedding_norm": {
    "psi": 0.0187,
    "kl": 0.0324,
    "ks_stat": 0.052,
    "ks_pvalue": 0.133853,
    "drift": "no"
  },
  "response_length": {
    "psi": 0.0162,
    "kl": 0.0178,
    "ks_stat": 0.056,
    "ks_pvalue": 0.086899,
    "drift": "no"
  },
  "response_quality": {
    "psi": 8.8486,
    "kl": 13.5011,
    "ks_stat": 0.941,
    "ks_pvalue": 0.0,
    "drift": "yes"
  }
}
```

### Test nào phù hợp với feature nào?

| Feature | Test được chọn | Lý do |
|---|---|---|
| `prompt_length` | **PSI** | Dữ liệu liên tục, phân phối có thể thay đổi mạnh giữa các ngày (người dùng gửi prompt dài hơn/ngắn hơn tùy thời điểm). PSI chia histogram thành bins và tính tổng sai lệch tương đối — phù hợp cho deployment monitoring liên tục. PSI = 3,46 vượt xa ngưỡng cảnh báo 0,25, xác nhận drift rõ ràng. |
| `embedding_norm` | **KS** | Dữ liệu liên tục. Kolmogorov-Smirnov test so sánh hai hàm phân phối tích lũy (CDF) thực nghiệm mà không cần chia bin. Do `embedding_norm` có phân phối ổn định (PSI = 0,019), KS nhạy hơn PSI ở vùng đuôi phân phối — phù hợp để phát hiện shift nhỏ trong không gian vector. |
| `response_length` | **KS** | Tương tự `embedding_norm`: phân phối ổn định, KS phù hợp. KS p-value = 0,087 (> 0,05) xác nhận không có drift, nhất quán với PSI = 0,016. |
| `response_quality` | **KL Divergence** | `response_quality` là điểm số thuộc khoảng [0, 1] có dạng phân phối beta. KL divergence đo entropy tương đối — đặc biệt phù hợp khi một phân phối bị thu hẹp hoặc tập trung bất thường. KL = 13,5 là dấu hiệu rõ ràng rằng phân phối hiện tại rất khác với phân phối tham chiếu, ngụ ý chất lượng phản hồi đã thay đổi đáng kể. |

---

## 5. Track 05 — Tích hợp đa nguồn (Cross-Day)

### Metric nào từ ngày trước khó expose nhất? Tại sao?

Metric khó expose nhất là **embedding latency từ Day 19 (Qdrant vector store)**. Nguyên nhân chính: Qdrant có sẵn endpoint `/metrics` nhưng nằm ở cổng riêng (6333) và schema của nó không phân tách rõ ràng "thời gian tính toán embedding" — chỉ có `rest_response_duration_seconds` gộp chung cả network round-trip lẫn CPU compute. Để lấy được latency thực sự của bước embedding, cần bổ sung custom middleware hoặc instrument ở tầng ứng dụng (trong `monitor-day19-vector-store.py`), không thể scrape trực tiếp từ Qdrant mà không có custom exporter.

Ngược lại, `llama.cpp` từ Day 20 đã expose sẵn các metric chuẩn như `llama_decode_time_total` và `llama_tokens_per_second` theo format Prometheus — chỉ cần thêm một scrape job là đủ. Sự chênh lệch này phản ánh nguyên tắc cốt lõi: bất kỳ service nào muốn "observable" đều phải **chủ động thiết kế để expose metric** theo format đúng, thay vì để phía ngoài đoán mò.

---

## 6. Thay đổi duy nhất có tác động lớn nhất

**Khắc phục nguyên nhân gốc rễ: Khai báo tường minh UID của datasource trong Grafana provisioning**

Thay đổi có tác động lớn nhất trong toàn bộ lab không phải là viết code instrumentation, thiết kế dashboard hay cấu hình alert — mà chỉ là thêm hai dòng `uid: prometheus` và `uid: loki` vào file `datasources.yml`. Khi thiếu hai dòng này, cả ba dashboard đều hiển thị "No data" dù Prometheus đang thu thập metrics hoàn hảo. Nguyên nhân: Grafana tự sinh UID ngẫu nhiên cho datasource (ví dụ `PBFA97CFB590B2093`) sau mỗi lần provision, trong khi các file JSON dashboard đang hard-code `"uid": "prometheus"`. Kết quả là một mismatch âm thầm — không có thông báo lỗi, không có cảnh báo, chỉ là màn hình trống "No data" mà không có manh mối nào để debug.

Đây là bài học sâu sắc về **configuration drift** và **tính idempotent** — hai khái niệm trung tâm của observability hiện đại. Nếu cấu hình của chính công cụ monitoring thay đổi mỗi lần restart container (vì container là ephemeral), toàn bộ khả năng nhìn thấy hệ thống sẽ biến mất đúng vào lúc cần thiết nhất. Việc pin explicit UID trong provisioning là biểu hiện của "infrastructure as code" áp dụng cho tầng observability: đảm bảo rằng dù recreate container bao nhiêu lần, dashboards vẫn luôn kết nối đúng datasource, alert vẫn đúng ngưỡng, log vẫn đúng nguồn. Trong thực tế sản xuất, đây chính là ranh giới giữa một stack observability "chạy được" và một stack "đáng tin cậy khi cần nhất".
