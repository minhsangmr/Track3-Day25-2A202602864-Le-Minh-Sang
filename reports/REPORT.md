# Lab 25B — Report

**Họ tên / nhóm:** Lê Minh Sang (MSSV: 2A202602864)  **Commit:** a3f9ce7078c14d183ed1e95d311a2df59bc86855  **Điểm tự động (`make grade`):** 84.0 / 88 (Đạt điểm tuyệt đối mọi test public, 4 điểm còn lại thuộc bộ test ẩn của GV)

> Mọi nhận định có đầy đủ số liệu thực chứng trích xuất từ `reports/gameday/*.json`, `reports/workflows.json`, `reports/slo.json`, `reports/cache/*.json`.

---

## 1. Kết quả Game Day (naive vs của bạn)

| Băng | Client | availability | p95 (s) | amplification | retry_ratio | cost/turn | recovery (s) | PASS? |
|---|---|---:|---:|---:|---:|---:|---:|:---:|
| G1 | naive | 0.7194 | 7.157 | 7.738 | 6.738 | 2.163 | 4.0 | FAIL |
| G1 | yours | 1.0000 | 4.776 | 1.062 | 0.002 | 2.101 | 0.0 | PASS |
| G2 | naive | 0.4560 | 15.034 | 9.904 | 8.904 | 4.627 | 155.0 | FAIL |
| G2 | yours | 1.0000 | 10.920 | 1.041 | 0.007 | 2.119 | 0.0 | PASS |
| G3 | naive | 1.0000 | 4.768 | 1.164 | 0.164 | 2.328 | 0.0 | FAIL |
| G3 | yours | 1.0000 | 4.660 | 1.060 | 0.034 | 2.145 | 0.0 | PASS |
| G4 | naive | 1.0000 | 13.764 | 1.004 | 0.004 | 2.008 | 0.0 | FAIL |
| G4 | yours | 1.0000 | 5.562 | 1.015 | 0.002 | 2.072 | 0.0 | PASS |
| G5 | naive | 0.3699 | 5.359 | 17.030 | 16.030 | 2.124 | None | FAIL |
| G5 | yours | 0.8179 | 5.477 | 1.003 | 0.002 | 2.504 | None | PASS |
| G6 | naive | 0.5170 | 4.836 | 12.344 | 11.344 | 2.009 | 0.0 | FAIL |
| G6 | yours | 1.0000 | 5.961 | 1.055 | 0.019 | 2.189 | 0.0 | PASS |

---

## 2. Cấu hình client và lý do

| Tham số | Giá trị | Vì sao (dựa trên băng nào, số liệu nào) |
|---|---:|---|
| `attempt_timeout_s` | 8.0s | Dựa trên G6 và G4: service median của alpha là 1.2s, p99 bình thường ~3.5s. Đặt 8.0s đủ bao quát 99% request bình thường, đồng thời khi alpha-eu bị treo (G6), client timeout sau 8s và kịp thời failover sang beta trong giới hạn turn deadline 30s. Nếu đặt 30s thì availability ở G6 tụt xuống 94.88%. |
| `max_attempts_per_provider` | 2 | Dựa trên G1, G2, G3: Cho phép 1 lần retry trên cùng provider để vượt qua nhiễu transient (500/timeout ngẫu nhiên). Nếu lỗi tiếp thì failover sang deployment khác thay vì đập liên tục làm nghẽn hàng đợi (G2 amplification từ 9.904 giảm xuống 1.041). |
| `backoff_base_s` / `cap_s` | 0.25s / 4.0s | Dựa trên G1: Base 0.25s kèm full jitter phân tán đều các đợt thử lại, kết hợp tôn trọng `retry_after` giúp vi phạm `retry_after_violations` ở G1 giảm từ 36,027 về đúng 0. Cap 4.0s giữ độ trễ không vượt quá deadline lượt gọi. |
| `retry_budget_ratio` | 0.1 | Dựa trên G3: Giới hạn lượng retry không vượt quá 10% tổng lưu lượng. Kết quả ở G3 retry_ratio giảm từ 0.164 (16.4%) xuống 0.034 (3.4%), vượt qua ngưỡng kiểm tra tối đa 0.10. |
| `breaker.window_size` / `min_calls` | 20 / 10 | Dựa trên G2 và G4: Cửa sổ trượt 20 cuộc gọi gần nhất giúp breaker phản ứng nhanh khi có sự cố, yêu cầu tối thiểu 10 cuộc gọi để tránh trip nhầm vào ban đêm (cold start / low traffic). |
| `breaker.failure_rate_threshold` | 0.5 | Dựa trên G2 và G6: Khi 50% cuộc gọi trong cửa sổ bị lỗi/timeout, breaker lập tức OPEN để chuyển tải sang alpha-eu/beta, bảo vệ alpha không bị overload storm. |
| `breaker.slow_call_s` | 5.0s | Dựa trên G4: alpha bình thường xử lý 1.2s, khi bị chậm thêm 5s (thành ~6.2s) thì vượt ngưỡng 5.0s. Breaker nhận diện là slow-call để cắt tải sớm, đưa p95 ở G4 từ 13.764s xuống 5.562s. |
| `breaker.open_duration_s` | 10.0s | Dựa trên G2: Giữ breaker OPEN trong 10s để hàng đợi của alpha tiêu hóa hết các request cũ, giúp recovery_s ở G2 giảm từ 155s về 0s. |
| `breaker.half_open_max_probes` | 2 | Dựa trên G2: Chỉ cho phép tối đa 2 request probe bay đồng thời trong HALF-OPEN, ngăn chặn hiện tượng thundering herd đè bẹp provider vừa mới hồi phục. |

### Sơ đồ quyết định của `on_error`

```
                        +----------------------------+
                        |      Nhận ProviderError     |
                        +----------------------------+
                                      |
                                      v
                        +----------------------------+
                        |  Phân loại lỗi (classify)  |
                        +----------------------------+
                                      |
         +----------------------------+----------------------------+
         |                            |                            |
         v                            v                            v
  [CLIENT_ERROR]              [QUOTA_EXHAUSTED]             [RATE_LIMITED]
  (400, 401, 422)             (429 insufficient)            (429 exceeded)
         |                            |                            |
  Breaker release              Breaker release              Breaker release
         |                            |                            |
  GiveUp("client_error")       Cooldown 300s                Cooldown retry_after
  (Không retry,                Failover ngay sang           Failover ngay sang
   Không failover)             provider tiếp theo           provider tiếp theo
                                      |
                                      +----------------------------+
                                                                   |
                                                                   v
                                                     [OVERLOADED / SERVER_ERROR / TIMEOUT]
                                                     (529, 503, 500, 502, timeout 0)
                                                                   |
                                                            Breaker on_failure
                                                                   |
                                              +--------------------+--------------------+
                                              |                                         |
                                     (tries < 2 AND                             (Hết lượt retry /
                                      budget cho phép AND                        hết budget /
                                      còn đủ deadline)                           hết deadline)
                                              |                                         |
                                              v                                         v
                                      Retry CÙNG provider                     Failover provider tiếp
                                      với FullJitterBackoff                   trong capability ladder
```

---

## 3. Ba thí nghiệm (mỗi cái 3–5 câu + số)

1. **Retry budget là lớp phòng thủ cuối:**
   - Khi giả lập breaker bị vô hiệu hóa (`breaker.min_calls: 1000000000`) và cho phép `max_attempts_per_provider: 5`:
     - Trên **G2_overload_storm**: Khi bật Retry Budget, hệ thống duy trì `availability = 99.87%`, `amplification = 1.380`, `retry_ratio = 0.058`, và `recovery = 0.0s`. Khi tắt Retry Budget, tải bùng nổ làm `amplification` vọt lên **1.675**, `retry_ratio` tăng vọt lên **0.495** (gần 50% request là retry vô ích), và thời gian hồi phục kéo dài lên `recovery = 25.0s`.
     - Trên **G3_brownout**: Bật budget giữ `retry_ratio = 0.068` (đạt chuẩn <= 0.10), trong khi tắt budget đẩy `retry_ratio` lên **0.159** (vi phạm tiêu chuẩn).
   - Ngay cả khi circuit breaker hoạt động hoàn hảo, Retry Budget vẫn là chốt chặn bắt buộc vì nó hoạt động như một van tiết lưu cơ học ở mức global (token bucket). Trong các kịch bản cold-start (chưa tích lũy đủ `min_calls`), hoặc sự cố phân tán trên nhiều provider cùng lúc, Retry Budget là lớp phòng thủ duy nhất ngăn chặn khuếch đại tải dẫn đến metastable failure.

2. **`attempt_timeout_s: 30`:**
   - Khi tăng timeout lên 30s trên **G6_provider_outage** (nơi `alpha-eu` bị treo hang không trả lời): `availability` sụt giảm từ **100% xuống 94.88%** (thất bại check), xuất hiện 109 lần bỏ cuộc do `deadline` và 13 lần do `no_provider_available`. Đáng chú ý, `latency_p95_s` tưởng chừng tăng nhưng lại hiển thị **5.315s** (thấp hơn mức 5.961s của bản 8s). Đây là biểu hiện kinh điển của **Survivorship Bias** (thiên lệch kẻ sống sót): các request treo 30s đều bị chết và bị loại khỏi tập đo p95 của lượt thành công.
   - Ngược lại, trên **G4_slow_region**, availability vẫn giữ nguyên 100% và p95 đạt 5.608s (gần như không đổi so với 5.562s) vì breaker đã phát hiện slow calls (latency >= 5.0s) và tự động trip chuyển vùng trước khi chạm timeout.
   - Do đó, giá trị timeout tối ưu **không bao giờ** được đặt bằng turn deadline (30s), mà phải dựa trên bội số p99 lúc bình thường (khoảng 3–4× median ~ 8s) để đảm bảo còn đủ thời gian thực hiện failover trong ladder.

3. **Bỏ lọc tool calling khỏi `ladder()`:**
   - Trên **G6_provider_outage**, khi toàn bộ cụm Provider A sập và Beta bị nghẽn rate limit, việc bỏ lọc capability khiến client định tuyến 234 lượt gọi tool call sang `gamma-mini` (model C-mini không có khả năng gọi tool), làm xuất hiện `incompatible_routes = 234` và availability tụt từ **100% xuống 90.19%**.
   - Dưới góc nhìn của một Product Manager: *"Chúng ta đã gửi 234 yêu cầu hoàn tiền của khách hàng vào một mô hình AI giá rẻ chỉ biết trò chuyện mà không có cổng kết nối ngân hàng. Hậu quả là 234 khách hàng nhận được câu trả lời hứa hẹn suông nhưng tiền không bao giờ về tài khoản, làm tỷ lệ hoàn tất dịch vụ giảm 10% và trực tiếp gây khủng hoảng niềm tin người dùng."*

---

## 4. Workflow hoàn tiền

| Executor | completion | crashes | duplicate refunds | duplicate emails | lost refunds | overpaid | wasted LLM / crash |
|---|---:|---:|---:|---:|---:|---:|---:|
| naive | 1.0000 | 81 | 128 | 81 | 0 | 1,496,700 ₫ | 1.123 |
| yours | 1.0000 | 66 | 0 | 0 | 0 | 0 ₫ | 0.303 |

### Trả lời câu hỏi kiến trúc:
- **Vì sao chỉ idempotency key là chưa đủ?** Idempotency key bảo vệ tính duy nhất ở phía external API (ngân hàng/email provider), nhưng nếu worker bị crash và không có checkpoint, worker mới được hồi sinh sẽ phải thực thi lại toàn bộ workflow từ bước 1. Điều này gây lãng phí chi phí gọi LLM phân loại và tra cứu database (`wasted_llm_per_crash > 1.0`), đồng thời làm tăng latency nghiêm trọng.
- **Vì sao chỉ checkpoint cũng chưa đủ?** Checkpoint chỉ ghi nhận trạng thái sau khi một bước hoàn thành trong database nội bộ. Nếu worker gửi request thanh toán thành công tới cổng ngân hàng, nhưng worker bị crash đúng vào microsecond tiếp theo trước khi kịp ghi checkpoint (hoặc response bị mất do timeout mạng), worker mới khởi động lại sẽ thực thi lại bước thanh toán. Nếu không có idempotency key ổn định theo `(session, step)`, cổng thanh toán sẽ trừ tiền lần 2 (như bản naive đã làm duplicate 128 giao dịch, overpaid gần 1.5 triệu đồng).
- **Điều gì xảy ra nếu ghi checkpoint TRƯỚC KHI gọi payment?** Nếu ghi checkpoint hoàn tiền thành công trước khi gọi cổng thanh toán, và tiến trình bị crash hoặc API thanh toán gặp lỗi thất bại, worker khởi động lại sẽ đọc checkpoint và ngộ nhận rằng tiền đã được hoàn tất, từ đó nhảy qua bước tiếp theo (gửi email chúc mừng). Hậu quả là xuất hiện **Lost Refund** (khách hàng không nhận được tiền nhưng hệ thống báo xong) — một lỗi vi phạm nghiêm trọng trong hệ thống tài chính.

---

## 5. Alerting & error budget

| Sự cố | Mong đợi | Naive alerter | Burn-rate alerter | Độ trễ phát hiện |
|---|---|---|---|---:|
| `checkout_bug` | page | page | page | 27 min |
| `provider_outage` | page | page | page | 4 min |
| `deploy_blip` | none | page (false alert) | none | — |
| `slow_leak` | ticket | none (bỏ sót) | ticket | 1337 min |
| `degraded_evening` | page | page | page | 291 min |
| `night_spike` | none | page (false alert) | none | — |
| `small_bump` | none | page (false alert) | none | — |

### Báo cáo Error Budget 30 ngày:
- Tổng số requests: **25,901,755**
- Lỗi cho phép (0.5% ở SLO 99.5%): **129,508.8**
- Số lỗi thực tế: **55,618**
- Availability thực tế: **99.7853%**
- Ngân sách lỗi đã tiêu: **42.95%**
- Ngân sách lỗi còn lại: **57.05%**
- **Quyết định:** **KHÔNG đóng băng release** (`freeze_releases = False`), vì ngân sách lỗi vẫn còn hơn 57%.
- **Vì sao `small_bump` (20 phút ở 12% lỗi) không nên đánh thức ai lúc 11 giờ trưa?**
  Mặc dù tỷ lệ lỗi 12% trong 20 phút trông có vẻ cao, nhưng tổng số lỗi tích lũy trong sự cố này chỉ tiêu tốn chưa đầy 0.3% error budget của cả tháng. Một kỹ sư on-call chỉ nên bị page (đánh thức / gián đoạn công việc) khi sự cố đe dọa làm cạn kiệt ngân sách lỗi của dịch vụ trong thời gian ngắn (burn rate cao trên cả cửa sổ dài và ngắn). Với `small_bump`, sự cố tự hết sau 20 phút mà không gây nguy hại cho SLO tháng, do đó việc page là một báo động giả (alert fatigue) gây lãng phí năng suất kỹ sư.

---

## 6. Postmortem — Băng G2 với client naive (blameless)

*Tham chiếu: Biểu đồ timeseries `reports/gameday/G2_overload_storm.png` và dữ liệu CSV `reports/gameday/naive_G2_overload_storm_timeseries.csv`.*

### Tóm tắt (3 câu):
Vào giây thứ 120 của mô phỏng, cụm deployment chính `alpha` bị sụt giảm 75% năng lực xử lý phần cứng. Client v1 (NaiveClient) phản ứng bằng cơ chế retry 3 tầng xếp chồng (tối đa 36 lần), không có jitter, không có retry budget và không có circuit breaker, khiến lượng request gửi vào alpha bùng nổ lên 254 attempts/s (vượt gấp 12 lần năng lực). Hệ thống rơi vào trạng thái metastable failure: dù phần cứng alpha đã hồi phục hoàn toàn ở giây thứ 300, hàng đợi vẫn tắc nghẽn bởi các request đã timeout và hệ thống mất thêm 155 giây nữa mới có thể phục hồi.

### Ảnh hưởng:
- **1,285 / 2,362** lượt người dùng thất bại (availability sụt giảm nghiêm trọng xuống còn **45.60%**).
- Thời gian gián đoạn kéo dài **335 giây** (từ t=120 đến t=455).
- Sự cố này đã tiêu tốn **38.8%** toàn bộ error budget tháng của ShopAgent (chỉ trong vòng chưa đầy 6 phút).

### Timeline (giây ảo — trích xuất từ `naive_G2_overload_storm_timeseries.csv`):

| t | Sự kiện | Bằng chứng (số liệu thực tế) |
|---:|---|---|
| 120 | alpha mất 75% GPU | `attempts_alpha = 5/s`, `calls_ok = 7/s`. Bắt đầu nghẽn hàng đợi xử lý. |
| 122 | Tỷ lệ thành công sụp đổ | `calls_ok = 0/s`, request bắt đầu vượt quá `max_queue_wait_s` và bị 529. |
| 136 | Retry storm bùng phát | Các tầng retry app/gateway/SDK kích hoạt đồng loạt: `attempts_alpha = 53/s`. |
| 143 | Tải retry đạt đỉnh cực đại | `attempts_alpha = 254/s`, `calls_ok = 0/s`. Tải gấp 12.7 lần giới hạn 20 rps của tenant. |
| 144–299 | Metastable failure tự duy trì | `attempts_alpha` duy trì liên tục ở mức 140–250/s. Toàn bộ `calls_ok = 0/s`. Xuất hiện 26,298 lỗi 429 và 3,194 lỗi 529. |
| 300 | alpha hồi phục 100% GPU | Năng lực phần cứng trở lại bình thường, nhưng `attempts_alpha` vẫn ở mức 115/s. `calls_ok` vẫn bằng 0! |
| 305–452 | Hàng đợi kẹt request rác | Server bận xử lý các request mà client đã timeout từ trước. Client tiếp tục gửi retry dồn dập. `calls_ok = 0/s` suốt 152 giây sau khi hạ tầng đã xong. |
| 455 | Bắt đầu xử lý được request mới | `calls_ok = 4/s`, hàng đợi bắt đầu thông thoáng. |
| 456–460 | Hệ thống hồi phục hoàn toàn | `calls_ok = 8–10/s` (đạt >= 95% thành công). Độ trễ phục hồi `recovery_s = 155.0s`. |

### Trigger vs Root Cause:
- **Trigger:** Sụt giảm đột ngột 75% năng lực tính toán GPU tại cụm deployment `alpha` tại t=120s.
- **Root Cause:** Kiến trúc client v1 thiếu khả năng kiểm soát độ tin cậy:
  1. Cấu hình 3 tầng retry độc lập (App × Gateway × SDK) nhân số lần gọi tối đa lên 36 attempts/call.
  2. Retry với khoảng cách cố định (không jitter), không có cơ chế token-bucket Retry Budget.
  3. Không có Circuit Breaker để fail fast và chuyển vùng tải sang `alpha-eu` hay `beta`.
- **Cơ chế Metastable Failure:** Khi hàng đợi của provider bị đầy, server vẫn tiếp tục tốn GPU xử lý các request cũ mà client đã timeout từ trước (wasted work). Trong khi đó, client lại liên tục nhồi thêm hàng trăm request retry mới vào hàng đợi. Vòng lặp phản hồi dương này (positive feedback loop) tự duy trì tải > năng lực kể cả khi nguyên nhân kích hoạt ban đầu đã chấm dứt hoàn toàn.

### Điều gì giúp / Điều gì làm tệ hơn:
- **Làm tệ hơn:** Cơ chế timeout phía server không hủy tác vụ khi client đã bỏ cuộc; client thiếu jitter làm các đợt retry đập vào cùng thời điểm; retry liên tục vào provider đang báo lỗi 429 rate limit.
- **Giúp ích:** Turn deadline 30s của người dùng cuối cùng đã giới hạn tổng số request bay lơ lửng, giúp lượng retry hạ dần sau t=305 để hệ thống có cơ hội tự giải phóng sau 155 giây.

### Action Items:

| # | Hành động | Loại | Owner | Cách kiểm chứng |
|---|---|---|---|---|
| 1 | Thay thế NaiveClient bằng ResilientClient: tích hợp FullJitterBackoff, RetryBudget ratio=0.1, SlidingWindowBreaker | Prevent | SRE Team | `make gameday` đạt amplification <= 1.1 và recovery_s = 0.0s trên G2 |
| 2 | Triển khai Fallback Ladder có nhận biết năng lực (alpha -> alpha-eu -> beta) | Mitigate | Core Agent Team | Availability trên G2 đạt 100% trong môi trường staging |
| 3 | Xây dựng hệ thống cảnh báo đa cửa sổ Multi-window Burn-rate Alerting (Google SRE Workbook) | Detect | Monitoring Team | Alert phát hiện sự cố overload storm trong vòng < 5 phút và 0 false page |

---

## 7. Một điểm yếu còn lại

**Điểm yếu:** Chi phí và độ trễ gia tăng khi sự cố kéo dài và toàn bộ tải dồn sang `beta`.
- Trong kịch bản G6 (Provider A sập hoàn toàn), toàn bộ lượt gọi buộc phải chuyển sang `beta`. Chi phí của `beta` cao hơn 40% so với alpha (1.4 vs 1.0), trong khi rate limit của beta chỉ có 16 rps (thấp hơn tổng lưu lượng).
- Nếu sự cố kéo dài nhiều ngày hoặc toàn bộ 4 provider cùng bị suy thoái mạng diện rộng (global latency degradation), hệ thống sẽ đối mặt với tình trạng cạn kiệt slot và chi phí vận hành tăng vọt.

**Giải pháp khắc phục:**
1. **Tiered Model Degradation:** Khi nhận diện failover sang tầng 3 (Beta), client có thể tự động hạ thấp max token output hoặc chuyển phần chat tóm tắt sang mô hình lightweight để tiết kiệm slot.
2. **Dynamic Concurrency Limiter (Adaptive Throttling):** Áp dụng thuật toán Vegas / AIMD (Additive Increase Multiplicative Decrease) tại client để tự động điều chỉnh tốc độ gửi request phù hợp với throughput thực tế của Beta thay vì đẩy tối đa tải.

---

## 8. Cache (Part G)

*Số liệu từ `reports/cache/cached.json` và `reports/cache/resilient.json` (`make cache` trên băng C1).*

| Client | availability | faq_availability | cache_hit_ratio | stale_served | wrong_answers | outdated_answers | cost/turn | PASS? |
|---|---:|---:|---:|---:|---:|---:|---:|:---:|
| resilient (không cache) | 0.8373 | 0.8424 | 0.000 | 0 | 0 | 0 | 1.386 | FAIL |
| cached (của bạn) | 0.9124 | 0.9645 | 0.592 | 192 | 0 | 2 | 0.955 | PASS |

### Cấu hình `cache:` và lý do

| Tham số `cache:` | Giá trị | Vì sao (dựa trên số liệu nào) |
|---|---:|---|
| `similarity_threshold` | 0.92 | Tại 0.80, xuất hiện tới **181 câu trả lời sai** (`wrong_answers: 181`, do nhầm lẫn giữa các câu hỏi khác nhau). Tại 0.97, tỷ lệ hit tụt xuống **49.9%** (dưới ngưỡng yêu cầu 50%). Ngưỡng 0.92 mang lại sự cân bằng hoàn hảo: `wrong_answers = 0` và `cache_hit_ratio = 59.2%`. |
| `ttl_s` | 30.0s | Tại TTL 300s, khi bảng phí ship đổi âm thầm ở t=480, có tới **95 câu trả lời lỗi thời** (vượt trần 15). Giảm TTL về 30s giúp `outdated_answers` giảm chỉ còn **2 câu**, đáp ứng xuất sắc tiêu chí. |
| `max_stale_s` | 300.0s | Khi `max_stale_s = 0`, trong đợt provider outage 120s (t=240-360), `faq_availability` tụt xuống **85.29%** (FAIL). Đặt 300s cho phép stale-if-error phục vụ **192 câu hỏi** và đưa `faq_availability` lên **96.45%** (vượt mức chuẩn 93%). |

### Kết quả 3 thí nghiệm Cache:

1. **Ngưỡng tương đồng (`similarity_threshold`):**
   - Với ngưỡng `0.80`: `cache_hit_ratio = 61.0%`, nhưng có tới `wrong_answers = 181` (tất cả là lỗi `other_question`), chi phí 1.015/turn. Các câu hỏi có ngữ cảnh khác biệt nhưng chung từ khóa (như *"Thời gian hoàn tiền mất bao lâu?"* và *"Điều kiện hoàn tiền là gì?"*) bị trả lời nhầm lẫn tai hại.
   - Với ngưỡng `0.92`: `cache_hit_ratio = 59.2%`, `wrong_answers = 0`, chi phí giảm xuống mức tối ưu **0.955/turn**.
   - Với ngưỡng `0.97`: `wrong_answers = 0` nhưng `cache_hit_ratio = 49.9%` (không đạt chuẩn >= 50%), chi phí tăng lên 1.018/turn do nhiều câu hỏi đồng nghĩa bị miss cache.
2. **TTL (`ttl_s`):**
   - Chạy với `ttl_s = 30s`: `outdated_answers = 2`.
   - Chạy với `ttl_s = 60s`: `outdated_answers = 3`.
   - Chạy với `ttl_s = 300s`: `outdated_answers = 95`.
   - **Giải thích:** Webhook chỉ gửi tín hiệu khi CMS chủ động thông báo (như sự kiện ở t=150s). Đối với những thay đổi âm thầm từ bên thứ ba (silent updates như đối tác vận chuyển đổi bảng giá ở t=480s), không có bất kỳ webhook nào được bắn tới. Do đó, cơ chế Invalidation hoàn toàn bất lực và TTL hết hạn là rào chắn duy nhất giới hạn số câu trả lời cũ được phát ra.
3. **Stale-if-error (`max_stale_s`):**
   - Khi đặt `max_stale_s: 0`: Toàn bộ cơ chế stale-if-error bị tắt, `stale_served = 0`, khiến `faq_availability` tụt từ **96.45% xuống 85.29%** (thất bại so với ngưỡng 93%).
   - Khi đặt `max_stale_s: 300`: Cache đã phục vụ thành công **192 câu trả lời cũ** trong thời gian 2 phút mọi nhà cung cấp đều sập, giữ vững trải nghiệm người dùng và cứu sống độ khả dụng của hệ thống.

### Trả lời câu hỏi phỏng vấn:
- **Vì sao câu hỏi cá nhân ("Đơn #48213 của tôi đang ở đâu?") không được cache, kể cả khi cache theo từng user?**
  1. *Tính đúng đắn (Freshness):* Trạng thái đơn hàng, số dư tài khoản thay đổi liên tục theo thời gian thực (đang giao -> đã giao). Cache sẽ trả về dữ liệu cũ sai lệch.
  2. *An toàn dữ liệu & Privacy:* Rủi ro rò rỉ dữ liệu nhạy cảm (PII) nếu xảy ra bug định danh user hoặc hash collision.
  3. *Hiệu quả chi phí:* Tần suất lặp lại cùng một câu hỏi về một đơn hàng cụ thể của một user là cực kỳ thấp (gần như bằng 0). Cache những dữ liệu này chỉ làm ô nhiễm và phình to bộ nhớ LRU mà không tăng cache hit ratio.
- **Trong lúc mọi provider sập, dashboard vẫn xanh nhờ cache. Bạn sẽ alert dựa trên số nào để on-call vẫn biết có sự cố?**
  - Không được dùng User-facing Availability làm metric duy nhất vì cache đã che giấu sự cố (stale-if-error).
  - On-call phải thiết lập alert dựa trên:
    1. **Tỷ lệ phục vụ dữ liệu cũ (`stale_served_rate` hoặc `stale_ratio > 5%`)**: Cho biết cache đang phải "gồng gánh" cứu lỗi backend.
    2. **Egress Upstream Error Rate / Breaker State**: Giám sát trực tiếp tỷ lệ lỗi tại tầng gọi ra provider (egress call failures) và trạng thái của các Circuit Breakers (cảnh báo ngay khi breaker của Alpha/Beta chuyển sang OPEN).
