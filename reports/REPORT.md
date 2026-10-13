# Lab 25B — Report

**Họ tên / nhóm:** Phan Hoàng Vũ  **MSV:** 2A202602450  **Commit:** a3f9ce7078c14d183ed1e95d311a2df59bc86855  **Điểm tự động (`make grade`):** 84 / 88 (84/84 public + 4 điểm dữ liệu ẩn của giảng viên)

> Mọi nhận định phải có số. Dán số từ `reports/gameday/*.json`, `reports/workflows.json`, `reports/slo.json`.

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
| G5 | naive | 0.3699 | 5.359 | 17.030 | 16.030 | 2.124 | N/A | FAIL |
| G5 | yours | 0.8179 | 5.477 | 1.003 | 0.002 | 2.504 | N/A | PASS |
| G6 | naive | 0.5170 | 4.836 | 12.344 | 11.344 | 2.009 | 0.0 | FAIL |
| G6 | yours | 1.0000 | 5.961 | 1.055 | 0.019 | 2.189 | 0.0 | PASS |

## 2. Cấu hình client và lý do

| Tham số | Giá trị | Vì sao (dựa trên băng nào, số liệu nào) |
|---|---:|---|
| `attempt_timeout_s` | 8.0 | Tránh để call treo quá lâu; ở G6 nếu đặt 30s thì availability giảm xuống 0.9488 do nghẽn deadline. 8s đủ lớn cho p99 bình thường (~2-3s) và đủ nhỏ để failover kịp trước deadline 30s. |
| `max_attempts_per_provider` | 2 | Cho phép thử lại 1 lần duy nhất trên cùng provider rồi failover ngay. Giúp G3 giảm `retry_ratio` xuống 0.034 (so với naive 0.164) và G2 không tạo bão retry. |
| `backoff_base_s` / `cap_s` | 0.25 / 4.0 | Full jitter với base 0.25s tránh thundering herd; cap 4.0s đảm bảo một lần retry không ngốn quá nhiều thời gian trong deadline 30s. |
| `retry_budget_ratio` | 0.1 | Giới hạn retry tối đa ~10% tổng lưu lượng, triệt tiêu nguy cơ retry storm ở G1/G2/G3 (G3 `retry_ratio` chỉ 0.034 <= 0.10). |
| `breaker.window_size` / `min_calls` | 20 / 10 | Đủ mẫu để tính tỷ lệ lỗi thống kê chính xác, tránh trip nhầm vào ban đêm khi lưu lượng thấp (cần tối thiểu 10 mẫu). |
| `breaker.failure_rate_threshold` | 0.5 | Trip mở breaker khi tỷ lệ lỗi >= 50% trong window, kịp thời ngắt tải khi alpha bị overload (G2) hoặc outage (G6). |
| `breaker.slow_call_s` | 5.0 | Bắt suy thoái hiệu năng khi server chưa báo lỗi HTTP. Ở G4, khi latency tăng thêm 5s (lên ~6.2s), breaker trip mở giúp p95 giảm từ 13.764s (naive) xuống 5.562s (yours). |
| `breaker.open_duration_s` | 10.0 | Giữ mạch mở 10s cho provider phục hồi tải (hàng queue xả bớt) trước khi cho probe thử lại ở HALF_OPEN. |
| `breaker.half_open_max_probes` | 2 | Chỉ cho tối đa 2 request thăm dò đồng thời ở HALF_OPEN, ngăn chặn thundering herd làm sập lại provider vừa mới hồi phục. |

Sơ đồ quyết định của `on_error`:

```
                    +---------------------------+
                    |    Nhận ProviderError     |
                    +-------------+-------------+
                                  |
                                  v
                      +-----------------------+
                      |       classify()      |
                      +-----------+-----------+
                                  |
         +------------------------+------------------------+
         |                        |                        |
         v                        v                        v
  [CLIENT_ERROR]          [RATE_LIMITED /           [OVERLOADED /
  (400, 401, 422)         QUOTA_EXHAUSTED]          SERVER_ERROR / TIMEOUT]
         |                        |                        |
         v                        v                        v
  breaker.release()        breaker.release()        breaker.on_failure()
         |                        |                        |
         v                        v                        v
  GiveUp("client_error")   Đặt cooldown_until       tries < max_attempts &
  (Không retry,            fail over ngay sang      budget.try_acquire() &
   không failover)         provider kế tiếp         kịp deadline?
                           trong ladder             /             \
                                                  Có              Không
                                                  /                 \
                                                 v                   v
                                          Retry SAME provider   fail over sang
                                          với backoff delay     provider kế tiếp
```

## 3. Ba thí nghiệm (mỗi cái 3–5 câu + số)

1. **Retry budget là lớp phòng thủ cuối:**
   Khi giả lập breaker bị vô hiệu hóa (`min_calls: 1000000000`) và đặt `max_attempts_per_provider: 5`:
   - Khi **có retry budget**: trên G2, amplification giữ ở **1.380**, retry_ratio là **0.058**, recovery đạt **0.0s**, availability đạt **0.9987**. Trên G3, amplification là **1.136**, retry_ratio là **0.068**.
   - Khi **tắt retry budget**: trên G2, amplification tăng vọt lên **1.675**, retry_ratio lên tới **0.495** (gần 50% traffic là retry!), recovery bị trễ mất **25.0s**, availability tụt xuống **0.9666**. Trên G3, retry_ratio tăng lên **0.159**.
   - Khi breaker hoạt động đúng, nó ngắt mạch theo cụm lỗi. Tuy nhiên retry budget vẫn tối quan trọng vì là chốt chặn cuối cùng ngăn chặn khuếch đại lưu lượng ở mức toàn cục trong kịch bản breaker chưa kịp trip hoặc cấu hình ngưỡng quá lỏng.

2. **`attempt_timeout_s: 30`:**
   - Với G6 khi timeout = 8s: availability đạt **1.0000**, latency p95 = **5.961s**, give_up_reasons = `{}`. Khi tăng timeout = 30s: availability tụt xuống **0.9488**, phát sinh 109 lượt give up vì `deadline` và 13 lượt `no_provider_available`, trong khi p95 lại giảm còn **5.315s**. Đây là hiện tượng *survivorship bias*: p95 chỉ tính trên lượt thành công, các lượt treo 30s rồi chết không được tính vào p95!
   - Với G4 khi timeout = 30s: p95 đạt **5.608s** (so với 8s là 5.562s), availability vẫn giữ 1.0. G4 gần như không đổi vì latency thực tế của alpha tăng thêm 5s (tổng ~6.2s) vẫn được circuit breaker bắt qua cơ chế `slow_call_s: 5.0` để failover sang `alpha-eu`.
   - Timeout "đúng" phải dựa trên p99 bình thường cộng thêm một khoảng đệm nhỏ (~2–3× median), và phải đủ nhỏ để cho phép ít nhất 1–2 lần failover trước khi chạm deadline người dùng (30s).

3. **Bỏ lọc tool calling khỏi `ladder()`:**
   - Khi bỏ lọc `supports_tools` trong ladder ở kịch bản G6, số lượng `incompatible_routes` tăng vọt lên **258**, kéo availability từ 1.0000 rớt xuống **0.8918**.
   - Ngôn ngữ Product Manager: Có 258 khách hàng yêu cầu xử lý hoàn tiền (chức năng bắt buộc phải gọi tool tra cứu hóa đơn và giao dịch ngân hàng), nhưng hệ thống lại điều hướng nhầm cuộc gọi của họ sang model giá rẻ (gamma-mini) vốn không có khả năng dùng tool. Hậu quả là 258 khách hàng này nhận được câu trả lời lỗi/từ chối dịch vụ, gây sụt giảm nghiêm trọng trải nghiệm người dùng và tỷ lệ giữ chân khách hàng.

## 4. Workflow hoàn tiền

| Executor | completion | crashes | duplicate refunds | duplicate emails | lost refunds | overpaid | wasted LLM / crash |
|---|---:|---:|---:|---:|---:|---:|---:|
| naive | 1.0000 | 81 | 128 | 81 | 0 | 14,967.00 $ | 1.123 |
| yours | 1.0000 | 66 | 0 | 0 | 0 | 0.00 $ | 0.303 |

**Vì sao chỉ idempotency key là chưa đủ, và chỉ checkpoint cũng chưa đủ?**
- Nếu chỉ có idempotency key mà không có checkpoint: khi worker crash, worker mới khởi động lại sẽ phải chạy lại từ bước 1 (`classify_request`), gây lãng phí chi phí token LLM (`wasted_llm_per_crash` cao) và tăng độ trễ xử lý.
- Nếu chỉ có checkpoint mà không có idempotency key: khi gặp lỗi `ServiceTimeout` (tiền đã trừ bên ngân hàng nhưng response bị drop trên mạng), việc retry bước `issue_refund` sẽ tạo một giao dịch mới khiến khách hàng bị hoàn tiền 2 lần (duplicate refunds).
- **Nếu ghi checkpoint trước khi gọi payment:** Nếu worker bị crash ngay sau khi ghi checkpoint nhưng TRƯỚC khi gọi API payment thành công, worker phục hồi sau đó sẽ thấy checkpoint đã có và bỏ qua bước thanh toán, dẫn đến tiền không bao giờ được hoàn (lost refunds)! Do đó, checkpoint bắt buộc phải ghi SAU khi bước thực thi thành công.

## 5. Alerting & error budget

| Sự cố | Mong đợi | Naive alerter | Burn-rate alerter | Độ trễ phát hiện |
|---|---|---|---|---:|
| `checkout_bug` | page | page (phút 3481) | page (phút 3507) | 27 min |
| `provider_outage` | page | page (phút 8760) | page (phút 8764) | 4 min |
| `deploy_blip` | none | page (phút 15240) | none | N/A (không báo sai) |
| `slow_leak` | ticket | none | ticket (phút 22037) | 1337 min |
| `degraded_evening` | page | page (phút 30214) | page (phút 30291) | 291 min |
| `night_spike` | none | page (phút 34740) | none | N/A (không báo sai) |
| `small_bump` | none | page (phút 39542) | none | N/A (không báo sai) |

- **Error budget 30 ngày:** Đã tiêu **42.95%** budget (55,618 lỗi trên 129,508.8 lỗi cho phép), còn dư **57.05%**. Do budget còn lại > 0 nên **không đóng băng release** (`freeze_releases: false`).
- **Vì sao `small_bump` (20 phút ở 12% lỗi) không nên đánh thức ai lúc 11 giờ trưa?**
  Mặc dù tỷ lệ lỗi 12% trong 20 phút có vẻ cao cục bộ, nhưng tổng số lượng request hỏng chỉ chiếm một phần rất nhỏ của error budget tháng (< 1%). Đánh thức kỹ sư on-call cho những sự cố ngắn tự hết (transient blip) gây kiệt sức cảnh báo (alert fatigue). Cơ chế multi-window kết hợp cửa sổ dài (1h) và cửa sổ ngắn (5m) giúp lọc bỏ hoàn toàn các blip này mà không kích hoạt false page.

## 6. Postmortem — Băng G2 với client naive (blameless)

*Hình ảnh phân tích: `reports/gameday/G2_overload_storm.png`*
*Dữ liệu chi tiết: `reports/gameday/naive_G2_overload_storm_timeseries.csv`*

**Tóm tắt:**
Vào giây ảo t=120, deployment chính `alpha` bị mất 75% GPU trong vòng 3 phút, làm giảm khả năng phục vụ và tăng nghẽn hàng đợi. Do client naive thiếu cơ chế circuit breaker, backoff jitter và retry budget, các client đồng loạt gửi retry dồn dập vào `alpha`, làm số lượng attempt tăng vọt gấp 10 lần (từ ~10 lên 254 req/s), dẫn đến 429 rate limit và 529 overload. Khi nhà cung cấp đã khôi phục GPU ở t=300, hệ thống vẫn tiếp tục tê liệt thêm 155 giây và chỉ hồi phục ở t=455 do hàng đợi ngập tràn các request quá hạn.

**Ảnh hưởng:**
- Tổng cộng 1,285 lượt user bị thất bại (availability chỉ đạt 45.60%, 1077/2362 lượt tốt).
- Sự cố kéo dài tổng cộng 335 giây (180 giây sự cố nhà cung cấp + 155 giây tự nghẽn).
- Tiêu tốn 4.627 chi phí trên mỗi lượt thành công (so với 2.119 của bản resilient), vi phạm nặng nề SLO khả dụng tháng.

**Timeline (giây ảo):**

| t | Sự kiện | Bằng chứng (số) |
|---:|---|---|
| 120 | Provider `alpha` mất 75% năng lực GPU; hàng đợi bắt đầu ứ đọng | attempts_alpha = 5, calls_ok = 7 |
| 122 | Latency tăng vượt ngưỡng, request bắt đầu timeout và lỗi | calls_ok giảm về 0, attempts bắt đầu tăng |
| 136 | Retry storm bùng nổ, 3 tầng retry chồng chéo dội tải vào alpha | attempts_alpha tăng vọt lên 53 attempts/s |
| 143 | Đỉnh điểm quá tải: alpha bị rate limit và từ chối toàn diện | attempts_alpha đạt đỉnh 254 attempts/s, calls_ok = 0 |
| 300 | Alpha phục hồi 100% tài nguyên GPU | attempts_alpha = 115, calls_ok vẫn = 0 |
| 305–450 | Alpha tiếp tục xử lý các request tồn đọng mà client đã bỏ qua (client_timeout) | calls_ok duy trì ở mức 0 suốt 150s dù GPU đã đủ |
| 455 | Hàng đợi cũ được giải phóng hoàn toàn; hệ thống bắt đầu phục hồi | calls_ok phục hồi lên 4 ở t=455 và đạt 16 ở t=471 |

**Trigger vs root cause:**
- **Trigger:** Provider A gặp sự cố phần cứng, mất 75% năng lực xử lý GPU trong khoảng t=120 đến t=300.
- **Root cause:** Kiến trúc client v1 thiếu khả năng bảo vệ: 3 tầng retry nhân số lần gọi (tối đa 36 lần), delay cố định không jitter, không có retry budget và không có circuit breaker.
- **Metastable failure:** Khi client timeout, server vẫn tiếp tục tính toán và xử lý request đó trong hàng đợi FIFO. Client hết kiên nhẫn lại gửi thêm request mới, khiến hàng đợi đầy ắp các "công việc rác" (wasted work). Ngay cả khi GPU đã phục hồi ở t=300, năng lực của server vẫn bị thiêu đốt vào việc xử lý các request đã chết, duy trì vòng lặp quá tải tự sinh cho đến khi toàn bộ hàng đợi cạn kiệt ở t=455.

**Điều gì giúp / điều gì làm tệ hơn:**
- *Làm tệ hơn:* Retry ngay lập tức mà không có backoff jitter; không tôn trọng header `retry-after` khiến 429 chồng 429; thiếu fallback sang các deployment khả dụng (`alpha-eu`, `beta`).
- *Giúp giải quyết:* Cơ chế Circuit Breaker tự động ngắt mạch sang `alpha-eu`/`beta`; Token-bucket Retry Budget giới hạn retry dưới 10%; Full Jitter backoff xóa tan sự cộng hưởng xung đột nhịp gửi.

**Action items:**

| # | Hành động | Loại | Owner | Cách kiểm chứng |
|---|---|---|---|---|
| 1 | Triển khai `ResilientClient` có Sliding Window Breaker và Fallback Ladder | Mitigate | On-call / Platform Team | Chạy G2 qua CI đảm bảo availability >= 95% và recovery <= 30s |
| 2 | Cấu hình Token-bucket Retry Budget (ratio=0.1, cap=20) | Prevent | Core Agent Team | CI test: amplification <= 1.2 dưới mọi lỗi 5xx |
| 3 | Tích hợp cảnh báo đa cửa sổ Multi-window Burn Rate Alerting (SRE Workbook) | Detect | SRE Team | Giả lập outage cảnh báo page trong vòng < 5 phút |

## 7. Một điểm yếu còn lại

- **Điểm yếu:** Khi chuyển vùng (failover) từ `alpha` sang `beta`, mặc dù độ sẵn sàng (availability) được bảo toàn ở mức 1.0, chi phí trên mỗi lượt tăng lên đáng kể (cost của `beta` là 1.4 so với `alpha` là 1.0). Đồng thời, `beta` có rate limit thấp hơn (16 rps so với alpha 20 rps) và latency median cao hơn (1.8s so với 1.2s), dẫn đến nguy cơ làm quá tải domino sang cả cụm `beta` nếu toàn bộ traffic ồ ạt tràn qua.
- **Cách khắc phục:**
  1. Triển khai **weighted load shedding** hoặc **graceful degradation**: khi buộc phải failover sang `beta`, hạ bớt độ dài prompt hoặc chuyển bớt các tác vụ không quan trọng sang model rẻ (`gamma-mini` nếu không cần tool).
  2. Áp dụng cơ chế **rate limiter client-side** (token bucket cho từng provider đích) để điều tiết lưu lượng tràn (spillover) không vượt quá 16 rps của tenant trên `beta`.

## 8. Cache (Part G)

Số liệu từ `reports/cache/*.json` (`make cache`):

| Client | availability | faq_availability | cache_hit_ratio | stale_served | wrong_answers | outdated_answers | cost/turn | PASS? |
|---|---:|---:|---:|---:|---:|---:|---:|:---:|
| resilient (không cache) | 0.8373 | 0.8424 | 0.000 | 0 | 0 | 0 | 1.386 | FAIL |
| cached (của bạn) | 0.9124 | 0.9645 | 0.592 | 192 | 0 | 2 | 0.955 | PASS |

| Tham số `cache:` | Giá trị | Vì sao (số liệu nào) |
|---|---:|---|
| `similarity_threshold` | 0.92 | Ngưỡng 0.80 quá lỏng gây 181 lỗi `other_question` (trả nhầm câu hỏi khác). Nâng lên 0.92 loại bỏ hoàn toàn wrong answers (`wrong_answers = 0`) mà vẫn duy trì `cache_hit_ratio = 59.2% >= 50%`. |
| `ttl_s` | 30 | Tại t=480 bảng giá ship đổi âm thầm không có webhook. TTL = 30s giúp `outdated_answers = 2` (rất thấp so với ngưỡng tối đa 15), trong khi TTL = 300s khiến outdated answers lên tới 95! |
| `max_stale_s` | 300 | Đợt provider outage ở C1 kéo dài 120s (t=240 đến t=360). `max_stale_s: 300` đảm bảo cache có thể phục vụ 192 lượt stale answer an toàn, cứu `faq_availability` đạt 96.45% (vượt mốc 93%). |

Thí nghiệm:

1. **Ngưỡng similarity:**
   - Khi threshold = 0.80: `cache_hit_ratio` = 0.610, nhưng `wrong_answers` = **181** (tất cả đều do `other_question`), cost = 1.015, faq_avail = 0.8593. Các cặp câu hỏi như "Chính sách hoàn tiền" và "Thời gian hoàn tiền" có cosine similarity khoảng 0.85 bị nhận nhầm là một!
   - Khi threshold = 0.92: `cache_hit_ratio` = **0.592**, `wrong_answers` = **0**, cost = 0.955, faq_avail = 0.9645.
   - Khi threshold = 0.97: `cache_hit_ratio` tụt xuống **0.499** (< 50% yêu cầu), `wrong_answers` = 0, cost tăng lên 1.018 do miss cache nhiều hơn.
2. **TTL:**
   - Với `ttl_s: 30`: `outdated_answers` = **2**, hit ratio = 0.592, cost = 0.955.
   - Với `ttl_s: 60`: `outdated_answers` = **3**, hit ratio = 0.652.
   - Với `ttl_s: 300`: `outdated_answers` tăng vọt lên **95** (vi phạm check <= 15).
   - Vì thay đổi bảng giá ship là thay đổi không báo trước (silent update, carrier không bắn webhook), client không nhận được tín hiệu để gọi `invalidate()`. Cách duy nhất để giới hạn số câu trả lời cũ là để TTL tự hết hạn định kỳ, buộc client revalidate với provider.
3. **Stale-if-error:**
   - Khi tắt stale-if-error (`max_stale_s: 0`): `stale_served` = 0, `faq_availability` tụt từ **96.45%** xuống còn **85.29%** (FAIL vì yêu cầu >= 93%). Trong 2 phút mọi provider sập (t=240–360), toàn bộ các câu hỏi FAQ bị từ chối trả lời vì không được phép dùng câu trả lời stale.

Trả lời câu hỏi:

- **Vì sao câu hỏi cá nhân ("Đơn #48213 của tôi đang ở đâu?") không được cache, kể cả khi cache theo từng user?**
  1. *Tính đúng đắn và thời gian thực:* Trạng thái đơn hàng, điểm tích lũy hay tiền hoàn thay đổi liên tục theo từng giây (đang giao, đã nhận, đã hoàn tiền). Trả về dữ liệu cache cũ có thể khiến khách hàng lầm tưởng đơn hàng chưa được xử lý.
  2. *Rủi ro bảo mật và rò rỉ dữ liệu (Data Privacy):* Embedding ngữ nghĩa không phân biệt được số định danh (order id). Nếu ngưỡng tương đồng nhận diện hai câu hỏi cá nhân tương tự nhau, câu trả lời có thể tiết lộ địa chỉ giao hàng, số điện thoại hoặc mã đơn của người khác. Do đó, các request `personal=True` bắt buộc phải bypass cache.
- **Trong lúc mọi provider sập, dashboard vẫn xanh nhờ cache. Bạn sẽ alert dựa trên số nào để on-call vẫn biết có sự cố?**
  Cần giám sát và alert dựa trên:
  1. **Tỷ lệ phục vụ dữ liệu cũ (`stale_served_ratio = stale_served / total_requests`)**: Khi tỷ lệ này tăng đột biến (> 0%), chứng tỏ các yêu cầu tới provider phía sau đang bị lỗi (give up) và client phải dùng cơ chế cứu hộ stale-if-error.
  2. **Tỷ lệ lỗi của lớp backend provider (Origin Provider Error Rate / Circuit Breaker State)**: Giám sát trực tiếp trạng thái OPEN của các circuit breaker hoặc tỷ lệ 5xx/timeout trả về từ provider, tách biệt hoàn toàn khỏi SLO của người dùng cuối.
