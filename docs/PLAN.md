# Kế hoạch A → Z: Xây dựng MMP tự thân thay thế AppsFlyer / Adjust

> Phiên bản: 1.0 · Ngày: 2026-10-31
> Phạm vi: mobile app/game (Android + iOS, Unity là engine chính), chạy UA trên Google, Meta, TikTok, Apple Search Ads và các ad network (AppLovin, Unity Ads, ironSource, Mintegral, Liftoff/Vungle, Moloco…), doanh thu từ IAP + quảng cáo (MAX / LevelPlay / AdMob).

---

## 0. TL;DR

1. **Không phải mọi thứ của AF/Adjust đều tự làm được.** Phần lõi (SDK, thu event, ad revenue, cohort/ROAS, deep link, postback cho ad network, SKAN, Apple Search Ads, export raw data) **làm được hoàn toàn**. Phần khó là **Self-Attributing Networks (Meta, Google, TikTok, Snap)** — các nền tảng này chỉ chia sẻ dữ liệu attribution cấp user với MMP đối tác chính thức. Phải dùng đường vòng (Conversions API, Firebase/GA4 link, API báo cáo, mô hình hoá) → đây là **rủi ro số 1**, cần chốt ngay từ Phase 0.
2. **Chiến lược khuyến nghị: thay thế theo lớp, chạy song song, cắt dần.** Không "big bang". Giữ AF/Adjust ở gói tối thiểu trong giai đoạn chuyển đổi để đối soát, rồi giảm dần đến khi chỉ còn (hoặc không còn) phần SAN.
3. **Timeline thực tế:** ~9–12 tháng cho team 6–8 người để đạt mức "đủ thay thế" cho nhu cầu nội bộ; MVP có giá trị (event + ad revenue + ROAS cohort + raw data) sau ~3 tháng.
4. **Tiêu chí cutover:** chênh lệch installs non-organic so với AF ≤ 5–10% theo từng network trong 4 tuần liên tục, chênh lệch revenue ≤ 3%, uptime ingest ≥ 99.9%.

---

## 1. Mục tiêu & phạm vi

### 1.1 Mục tiêu kinh doanh
- Cắt chi phí MMP (thường tính theo số conversion/attribution, tăng tuyến tính theo scale).
- Sở hữu 100% raw data, không giới hạn retention, không phí export.
- Linh hoạt logic attribution / cohort / LTV theo đặc thù game của mình.
- Nối thẳng dữ liệu vào data warehouse (BigQuery) và hệ thống tối ưu UA nội bộ.

### 1.2 Tính năng cần thay thế (map với AF/Adjust)

| # | Năng lực | AF / Adjust | Tự làm được? | Ưu tiên |
|---|---|---|---|---|
| 1 | SDK thu install/session/event | Có | ✅ Hoàn toàn | P0 |
| 2 | In-app event + IAP revenue (kèm verify receipt) | Có | ✅ | P0 |
| 3 | Ad revenue (ILRD từ MAX/LevelPlay/AdMob) | Có | ✅ | P0 |
| 4 | Raw data export (BigQuery/S3) | Có (trả phí) | ✅ (thế mạnh) | P0 |
| 5 | Attribution qua click/tracking link (ad network) | Có | ✅ nếu network hỗ trợ custom tracking link + postback | P1 |
| 6 | Postback S2S về ad network (install/event/revenue) | Có | ✅ cần làm việc với từng network | P1 |
| 7 | Google Play Install Referrer | Có | ✅ | P1 |
| 8 | Apple Search Ads (AdServices token) | Có | ✅ API mở cho mọi app | P1 |
| 9 | SKAdNetwork / AdAttributionKit (conversion value, postback) | Có | ✅ (Apple cho phép nhận bản copy postback) | P1 |
| 10 | Cost / spend ingestion, ROAS, cohort, LTV | Có | ✅ qua API báo cáo của từng network | P1 |
| 11 | Deep link / deferred deep link / OneLink | Có | ✅ (Universal Links, App Links, deferred matching) | P2 |
| 12 | Meta / Google / TikTok / Snap attribution cấp user | Có (đối tác chính thức) | ⚠️ **Hạn chế** — xem mục 4 | P1 (quyết định sớm) |
| 13 | Fraud protection (click injection, SDK spoofing, bot) | Có (Protect360 / Adjust FPS) | ✅ ở mức cơ bản → trung bình | P2 |
| 14 | Audiences / segment đẩy ra network | Có | ✅ (Customer Match, Custom Audience API) | P3 |
| 15 | Uninstall tracking | Có | ✅ qua silent push FCM/APNs | P3 |
| 16 | Dashboard, report, pivot | Có | ✅ (Metabase/Superset trước, UI riêng sau) | P1 |
| 17 | Privacy: GDPR/CCPA/ND13, opt-out, xoá dữ liệu | Có | ✅ bắt buộc | P0 |

### 1.3 Ngoài phạm vi (giai đoạn đầu)
- Bán MMP cho bên thứ ba (đó là bài toán hoàn toàn khác: certification với Meta/Google, SOC2, multi-tenant…).
- Web-to-app, CTV, PC/console attribution.
- Incrementality testing, MMM (có thể làm ở Phase sau trên cùng data).

---

## 2. Kiến trúc tổng thể

```
                ┌──────────────────────────── Ad Networks / SAN ───────────────────────────┐
                │  Click/impression redirect      Postback nhận về     Cost API   SKAN copy  │
                └───────┬──────────────────────────────▲───────────────────┬──────────┬─────┘
                        │                              │                   │          │
   User click ──► [Link Service] ──► Store / App   [Postback Service]  [Cost Ingest] [SKAN Endpoint]
                   (redirect, deep link,               ▲                   │          │
                    lưu click vào Redis/CH)            │                   │          │
                        │                              │                   ▼          ▼
 [SDK Unity/Android/iOS] ──HTTPS──► [Edge Collector] ──► Kafka/Redpanda ──► [Stream Processor]
   install/session/event,           (Go, stateless,     topics: raw_events,   ├─ Attribution Engine
   referrer, ASA token, ILRD        validate, sign)     clicks, postbacks     ├─ Enrichment (geo, device)
                                                                              ├─ Fraud rules
                                                                              └─ Revenue normalizer
                                                                                     │
                       ┌─────────────────────────────────────────────────────────────┤
                       ▼                              ▼                              ▼
                 ClickHouse (OLAP)            Postgres (config:               BigQuery / GCS
                 events, installs,            apps, networks, links,          (raw export cho
                 attributions, cost,          postback templates, users)      team data)
                 ad_revenue, skan
                       │
                       ▼
          [Query API] ──► [Dashboard (Next.js) / Metabase] ──► UA team
```

### 2.1 Lựa chọn công nghệ (khuyến nghị, có thể đổi)

| Lớp | Lựa chọn | Lý do |
|---|---|---|
| SDK native | Kotlin (Android), Swift (iOS) | Nhẹ, không phụ thuộc |
| SDK Unity | C# wrapper gọi native qua JNI/DllImport | Engine chính của studio |
| Edge collector | Go | Throughput cao, chi phí thấp, dễ scale ngang |
| Message queue | Redpanda (hoặc Kafka / GCP Pub/Sub) | Replay, buffer khi downstream lỗi |
| Stream processing | Go consumer (MVP) → Flink nếu cần stateful phức tạp | Giữ đơn giản trước |
| Click store | Redis (TTL theo attribution window) + ClickHouse (lịch sử) | Lookup nhanh khi install |
| OLAP | ClickHouse | Aggregate tỷ dòng rẻ, nhanh |
| Config DB | PostgreSQL | Quan hệ, transaction |
| Warehouse | BigQuery (export hằng giờ/ngày) | Team data đã quen |
| Orchestration | Airflow / Dagster / Cloud Scheduler | Cost ingest, export, đối soát |
| Hạ tầng | Kubernetes (GKE) + Terraform | IaC, autoscale |
| Observability | Prometheus + Grafana + Loki/Sentry | SLO ingest, lag, lỗi SDK |
| Dashboard | Metabase/Superset (Phase 1) → Next.js riêng (Phase 3) | Ra giá trị nhanh |

### 2.2 Nguyên tắc thiết kế
- **Ingest không bao giờ được mất dữ liệu:** collector chỉ validate + ghi vào queue, mọi xử lý nặng ở downstream; SDK có retry + hàng đợi offline.
- **Idempotent mọi nơi:** mỗi event có `event_id` (UUID v7) sinh ở client → dedup ở downstream.
- **Attribution re-computable:** lưu raw clicks + raw installs, cho phép chạy lại attribution với rule mới (backfill).
- **Tách PII:** device ID và IP được hash/mã hoá, lưu tách bảng, có TTL.

---

## 3. Chi tiết từng module

### 3.1 SDK (Android / iOS / Unity)

**Chức năng:**
- Sinh và lưu `mmp_device_id` (UUID cài đặt), đọc GAID/App Set ID (Android), IDFV + IDFA (khi ATT = authorized) (iOS).
- Event tự động: `first_open/install`, `session_start`, `session_end` (heartbeat), `app_update`.
- Event thủ công: `trackEvent(name, params)`, `trackRevenue(amount, currency, productId, receipt)`, `trackAdRevenue(network, adUnit, placement, revenue, precision)`.
- Thu attribution signals lúc install:
  - Android: Google Play Install Referrer API (referrer string, click/install timestamp), (tuỳ chọn) Huawei/Samsung referrer.
  - iOS: `AAAttribution.attributionToken()` (Apple Search Ads), cập nhật SKAN/AdAttributionKit conversion value.
- Deep link: nhận Universal Link / App Link / custom scheme, callback `onDeepLink` + deferred deep link (trả từ server ở response install).
- Privacy: `setConsent(gdpr, adPersonalization)`, `optOut()`, đọc TCF string, không thu IDFA khi chưa có ATT.
- Kỹ thuật: hàng đợi offline (SQLite/file), batch gửi, retry exponential backoff, ký request (HMAC với app secret + timestamp) chống spoofing, kích thước SDK < 300 KB, không block main thread.
- Tích hợp sẵn với MAX (`onAdRevenuePaid`), LevelPlay (ImpressionData), AdMob (`OnPaidEvent`) qua adapter.

**Payload mẫu:**
```json
{
  "event_id": "0192f3a1-...",
  "app_id": "com.piontech.game1",
  "event_name": "af_purchase",
  "event_time": "2026-10-31T09:12:44.120Z",
  "mmp_device_id": "6b1c...",
  "ids": { "gaid": "...", "app_set_id": "...", "idfv": null, "idfa": null },
  "device": { "os": "android", "os_ver": "14", "model": "SM-A546E", "lang": "vi", "tz": "Asia/Ho_Chi_Minh" },
  "app": { "ver": "1.4.2", "build": 142, "sdk_ver": "0.3.0" },
  "consent": { "gdpr_applies": false, "ad_personalization": true, "att": "n/a" },
  "params": { "revenue": 1.99, "currency": "USD", "product_id": "gem_pack_1" },
  "sig": "hmac-sha256..."
}
```

### 3.2 Edge Collector
- Endpoint: `POST /v1/events` (batch), `POST /v1/install` (đồng bộ, trả về attribution + deferred deep link), `GET /v1/config` (remote config SDK: bật/tắt event, SKAN schema).
- Validate schema, verify HMAC, chặn replay (timestamp ± 5 phút + nonce), rate limit theo device.
- Enrich nhẹ: IP → country/region (MaxMind GeoIP2), User-Agent.
- Ghi vào Kafka topic `raw_events` (key = `app_id:mmp_device_id` để giữ thứ tự theo device).
- Mục tiêu: p99 < 50 ms, 99.95% availability, multi-zone, autoscale; đặt sau CDN/LB (Cloud Load Balancing / Cloudflare).

### 3.3 Link Service (tracking link + deep link — thay OneLink / Adjust link)
- Domain riêng: `go.<brand>.com` (click), `link.<brand>.com` (deep link) + file `apple-app-site-association` và `assetlinks.json`.
- Cấu trúc link: `https://go.brand.com/{app_slug}/{link_id}?pid=applovin_int&c={campaign}&af_siteid={site}&clickid={clickid}&gaid={gaid}&idfa={idfa}&...` (dùng macro của từng network).
- Xử lý click: sinh `click_id`, lưu (network, campaign, adset, ad, site, device ids, IP, UA, timestamp) vào Redis (TTL = click window) + ClickHouse; redirect 302 đến store (gắn `referrer=mmp_click_id%3D...` cho Google Play).
- Impression (view-through) endpoint: `GET /imp/...` trả 1x1/204.
- Deferred deep link: lưu `deep_link_value` theo click; khi install được match → trả về SDK.
- Mục tiêu: redirect p99 < 30 ms, chịu được burst hàng chục nghìn req/s (ad network bắn click S2S).

### 3.4 Attribution Engine (trái tim hệ thống)

**Thứ tự ưu tiên (waterfall) khi nhận install:**

1. **Apple Search Ads** — token AdServices → gọi `api-adservices.apple.com` → nếu `attribution=true` → gán ASA (campaign/adgroup/keyword).
2. **Google Play Install Referrer** chứa `mmp_click_id` của link mình → match chính xác 100% (Android).
3. **Device ID matching (deterministic)** — GAID (Android) hoặc IDFA (iOS, chỉ khi user đồng ý ATT ở cả app nguồn và app đích) khớp với click/impression trong window.
4. **Referrer có gclid/gbraid** (Google Ads) → gán Google (chi tiết campaign bổ sung qua API, xem 4.2).
5. **SAN claim** (nếu có kênh, xem mục 4).
6. **Probabilistic matching** — **chỉ Android**, IP + UA + timestamp, window ngắn (≤ 1–3 giờ), đánh dấu `match_type=probabilistic`. **Không áp dụng cho iOS** (vi phạm chính sách ATT của Apple).
7. Không có gì → **organic**.
8. iOS không ATT → attribution aggregate qua **SKAN/AdAttributionKit** (mục 3.6).

**Quy tắc:**
- Last-click thắng; click ưu tiên hơn impression. Click window mặc định 7 ngày, view-through 24 giờ, cấu hình được theo network.
- Dedup install: (`app_id`, `mmp_device_id`) + GAID/IDFV để nhận diện reinstall.
- Re-attribution window (mặc định 90 ngày): user cài lại sau khi lapsed và có click mới → install mới được attribution lại.
- Re-engagement (retargeting): click + mở app qua deep link trong window → ghi attribution re-engagement riêng.
- Mọi event sau install kế thừa attribution (bảng `user_attribution` ánh xạ device → install attribution hiện hành).
- Lưu `attribution_reason` + toàn bộ candidate để debug và để đối soát với network.

**Pseudo-code:**
```
on_install(i):
  if asa = resolve_asa(i.asa_token): return attribute(i, asa, "asa")
  if ref = parse_referrer(i.referrer):
      if ref.mmp_click_id and (c := clicks.get(ref.mmp_click_id)): return attribute(i, c, "referrer")
      if ref.gclid or ref.gbraid: return attribute(i, google_from(ref), "google_referrer")
  for id in [i.gaid, i.idfa] where id:
      if c := clicks.last_by_device(id, within=click_window): return attribute(i, c, "device_id")
      if v := imps.last_by_device(id, within=view_window): return attribute(i, v, "view_through")
  if i.os == "android" and (c := clicks.probabilistic(i.ip, i.ua, within=1h)) and unique(c):
      return attribute(i, c, "probabilistic")
  return attribute(i, ORGANIC, "none")
```

### 3.5 Postback Service (gửi kết quả về ad network)
- Template postback theo network (URL + macro): install, event (đã map tên), revenue; hỗ trợ cả postback cho install organic/khác network nếu network yêu cầu (một số network cần "all installs" để tối ưu).
- Hàng đợi retry (exponential, tối đa 24–72h), log từng postback (status code, latency) để đối soát.
- Cấu hình per app × network: event nào gửi, có gửi revenue không, `af_siteid` mapping…
- **Việc ngoài kỹ thuật:** liên hệ account manager của từng network để (1) xác nhận họ chấp nhận "custom/in-house MMP" (2) lấy spec postback & macro click URL (3) test end-to-end (quy trình, email mẫu, bộ tài liệu: [PARTNER_APPROVALS.md](PARTNER_APPROVALS.md)). Một số network lớn (AppLovin, Unity, Mintegral, ironSource, Moloco, Liftoff) có hỗ trợ in-house attribution nhưng điều kiện khác nhau → phải xác nhận từng nơi ở Phase 0.

### 3.6 SKAdNetwork / AdAttributionKit (iOS)
- Info.plist: `NSAdvertisingAttributionReportEndpoint = https://skan.<brand>.com` → Apple gửi **bản copy** postback winning về endpoint `/.well-known/skadnetwork/report-attribution/` (và AdAttributionKit tương đương).
- Verify chữ ký postback (public key của Apple), lưu ClickHouse bảng `skan_postbacks`.
- **Conversion value schema** (quan trọng nhất cho game): thiết kế theo revenue buckets / engagement trong 24–48h đầu; hỗ trợ 3 postback windows của SKAN 4 (fine value 0–63 + coarse low/medium/high), lock window.
- SDK nhận schema qua remote config, gọi `updatePostbackConversionValue` / AdAttributionKit API.
- Decode campaign ID / source identifier (4 chữ số SKAN 4) ↔ campaign của từng network (mapping lấy từ API network).
- Báo cáo: installs SKAN theo network/campaign, revenue ước tính từ conversion value, null-rate do crowd anonymity.

### 3.7 Revenue
- **IAP:** SDK gửi receipt → server verify với Google Play Developer API (`purchases.products/subscriptions`) và App Store Server API; đánh dấu `validated`, loại refund (Google Voided Purchases API, App Store Server Notifications V2). Chuẩn hoá tiền tệ về USD theo tỷ giá ngày (bảng `fx_rates`).
- **Subscription:** nhận server notifications (RTDN Google, ASSN V2 Apple) → event renew/cancel/refund.
- **Ad revenue:**
  - Realtime: ILRD từ SDK (MAX `revenue`, LevelPlay `ImpressionData.revenue`, AdMob `AdValue`).
  - Đối soát: API báo cáo user-level/ad-unit-level (AppLovin MAX User Revenue API, LevelPlay ad revenue measurement API, AdMob API) → hiệu chỉnh hằng ngày (ILRD thường lệch 5–15% so với số cuối).
- Gross vs net (sau phí store 15/30%) — lưu cả hai.

### 3.8 Cost Ingestion
- Job hằng giờ/ngày kéo spend theo campaign/adset/ad/geo/ngày từ: Google Ads API, Meta Marketing API, TikTok Marketing API, Apple Search Ads Campaign Management API, AppLovin Reporting API, Unity Ads Advertiser Stats API, ironSource/LevelPlay Advertiser API, Mintegral, Moloco, Liftoff, Snap…
- Chuẩn hoá về schema chung: `date, app_id, network, campaign_id, campaign_name, adset_id, ad_id, country, platform, spend_usd, impressions, clicks, installs_reported`.
- Mapping campaign ID giữa cost và attribution (qua macro `{campaign_id}` trong tracking link).
- Bảng `network_reported` riêng để so với số MMP đo → phát hiện lệch.

### 3.9 Analytics / Reporting
- **Bảng tổng hợp (materialized view ClickHouse):** theo ngày × app × network × campaign × country: installs, cost, CPI, revenue D0/D1/D3/D7/D14/D30/D60/D90 (IAP + Ad), ROAS Dn, retention D1/D7/D30, ARPDAU, payer rate.
- **Cohort** theo ngày install, LTV curve, dự báo LTV (Phase 3: mô hình đơn giản power-law/BG-NBD, sau đó ML).
- **Raw data export:** installs, events, ad_revenue, clicks, postbacks → BigQuery partition theo ngày, cập nhật hằng giờ.
- **UI:** Phase 1 dùng Metabase/Superset; Phase 3 build dashboard Next.js riêng (pivot, compare với AF, alert).
- **Alert:** rớt installs/ROAS bất thường, lag pipeline, lỗi postback, SDK version lỗi.

### 3.10 Fraud Protection
- **Click flooding:** CTIT (click-to-install time) phân phối bất thường, tỷ lệ click/install quá cao theo site.
- **Click injection (Android):** dùng timestamp `install_begin` của Install Referrer — click sau thời điểm bắt đầu cài → reject.
- **SDK spoofing:** HMAC signing, Play Integrity API / App Attest (iOS) cho install event.
- **Bot / device farm:** IP datacenter (danh sách ASN), device ID reset rate, hành vi sau install bất thường (không có session thứ 2).
- **Fake purchase:** chỉ đếm revenue đã verify receipt.
- Kết quả: `fraud_status` (clean / suspicious / rejected) + lý do; installs rejected không gửi postback, có báo cáo để đòi lại tiền network.

### 3.11 Admin / Config
- Quản lý app, network, tracking link, postback template, event mapping, SKAN schema, attribution window, user & phân quyền (RBAC: admin / UA manager / viewer / data), API key, audit log.

---

## 4. Bài toán Self-Attributing Networks (SAN) — đọc kỹ trước khi cam kết

Meta, Google, TikTok, Snap, X **không dùng tracking link**; họ tự claim install và chỉ trả dữ liệu cấp user cho **MMP đối tác chính thức** (Meta Mobile Measurement Partner, Google App Attribution Partner…). Chương trình đối tác này yêu cầu điều kiện mà một MMP nội bộ gần như không đáp ứng được. Vì vậy:

### 4.1 Meta (Facebook/Instagram)
- **Có thể làm:** Facebook SDK (hoặc **Conversions API for App Events**) gửi install + event + purchase trực tiếp cho Meta → Meta tối ưu chiến dịch bình thường (không cần MMP để tối ưu).
- **Đo lường phía mình:** lấy kết quả aggregate (installs, events, revenue theo campaign/adset/ad/ngày) qua Marketing API Insights; iOS dùng SKAN postback copy (Meta là SKAN network).
- **Không có:** attribution cấp user (device nào đến từ ad nào) → user-level ROAS cho Meta phải **mô hình hoá**: installs non-organic "chưa giải thích được" trong khung giờ/geo có chạy Meta → phân bổ theo tỷ lệ; hoặc so sánh cohort theo geo/time.

### 4.2 Google Ads (App campaigns / UAC)
- **Có thể làm:** link Google Ads ↔ **Firebase/GA4** (miễn phí) để Google tối ưu theo in-app event; hoặc gửi conversion qua Google Ads API (offline/app conversion import có giới hạn).
- **Đo lường:** Android Install Referrer thường chứa `gclid`/`gbraid` → nhận diện install đến từ Google ở cấp device (độ phủ không 100%); số aggregate qua Google Ads API; iOS qua SKAN + `gbraid`.
- **Giải pháp phổ biến:** dùng Firebase/GA4 export BigQuery làm nguồn attribution Google, join với dữ liệu MMP qua `app_instance_id`/device id.

### 4.3 TikTok, Snap, X…
- TikTok Events API / TikTok SDK cho tối ưu; số aggregate qua Marketing API; SKAN cho iOS. Tương tự Meta.

### 4.4 Ba phương án cần ra quyết định (Phase 0)

| Phương án | Mô tả | Ưu | Nhược |
|---|---|---|---|
| **A. Hybrid (khuyến nghị)** | MMP nội bộ đo toàn bộ ad network + ASA + organic + revenue; SAN dùng SDK/CAPI riêng + số aggregate + mô hình hoá | Tiết kiệm phần lớn chi phí, sở hữu data | User-level ROAS của SAN là ước lượng |
| **B. Giữ AF/Adjust gói tối thiểu chỉ cho SAN** | Chỉ dùng MMP thương mại cho Meta/Google/TikTok, nội bộ lo phần còn lại | Giữ user-level cho SAN | Vẫn tốn phí, phải dedup 2 hệ thống |
| **C. Full in-house, bỏ hẳn user-level SAN** | Chấp nhận đo SAN thuần aggregate + incrementality test | Rẻ nhất | Mất độ chi tiết, UA team phải đổi cách làm |

> 👉 **Hành động Phase 0:** thống kê % spend theo kênh. Nếu Meta + Google + TikTok > 60% spend, cân nhắc nghiêm túc phương án B trong 6–12 tháng đầu.

---

## 5. Mô hình dữ liệu (ClickHouse, rút gọn)

```sql
-- Mọi event thô từ SDK
CREATE TABLE events (
  event_id UUID, app_id LowCardinality(String), event_name LowCardinality(String),
  event_time DateTime64(3), received_at DateTime64(3),
  mmp_device_id String, gaid_hash String, idfv String, idfa_hash String,
  os LowCardinality(String), os_ver String, app_ver String, sdk_ver String,
  country LowCardinality(String), ip_hash String,
  revenue_usd Float64, revenue_type LowCardinality(String),   -- iap / ad / sub
  params String,                                              -- JSON
  install_id UUID                                             -- attribution kế thừa
) ENGINE = ReplacingMergeTree(received_at)
PARTITION BY toYYYYMM(event_time) ORDER BY (app_id, event_name, event_time, event_id);

CREATE TABLE clicks (
  click_id String, app_id String, network String, campaign_id String, campaign_name String,
  adset_id String, ad_id String, site_id String, is_impression UInt8,
  gaid_hash String, idfa_hash String, ip_hash String, ua String, click_time DateTime64(3),
  deep_link_value String
) ENGINE = MergeTree PARTITION BY toYYYYMMDD(click_time) ORDER BY (app_id, click_time) TTL click_time + INTERVAL 120 DAY;

CREATE TABLE installs (
  install_id UUID, app_id String, mmp_device_id String, install_time DateTime64(3),
  attribution_type LowCardinality(String),   -- install / reinstall / reattribution / reengagement
  network LowCardinality(String), campaign_id String, adset_id String, ad_id String, site_id String,
  match_type LowCardinality(String),         -- asa / referrer / device_id / view_through / probabilistic / san_model / organic
  click_id String, ctit_sec Int32, country String, os String,
  fraud_status LowCardinality(String), fraud_reason String,
  attribution_version UInt32                 -- cho phép chạy lại với rule mới
) ENGINE = ReplacingMergeTree(attribution_version) ORDER BY (app_id, install_id);

CREATE TABLE cost (date Date, app_id String, network String, campaign_id String, adset_id String,
  country String, spend_usd Float64, impressions UInt64, clicks UInt64, installs_reported UInt64)
ENGINE = ReplacingMergeTree ORDER BY (date, app_id, network, campaign_id, adset_id, country);

CREATE TABLE skan_postbacks (...);   CREATE TABLE ad_revenue_daily (...);
CREATE TABLE postback_log (...);     CREATE TABLE fx_rates (...);
```

Postgres: `apps`, `networks`, `tracking_links`, `postback_templates`, `event_mappings`, `skan_schemas`, `attribution_settings`, `users`, `roles`, `api_keys`, `audit_log`.

---

## 6. Privacy, pháp lý & bảo mật

- **Apple:** tuân thủ ATT (không fingerprint iOS), Privacy Manifest (`PrivacyInfo.xcprivacy`) cho SDK, khai báo App Privacy nutrition label, tracking domains.
- **Google Play:** Data safety form, chính sách Advertising ID (tôn trọng opt-out → GAID zeroed), Families policy nếu có game cho trẻ em (không thu ID quảng cáo, không probabilistic).
- **GDPR / ePrivacy (EU/UK), CCPA/CPRA (US), Nghị định 13/2023/NĐ-CP (VN):** consent trước khi thu dữ liệu cá nhân ở vùng áp dụng, hỗ trợ TCF v2.2 + Google Consent Mode v2, API xoá/ xuất dữ liệu theo device (DSAR) trong ≤ 30 ngày, DPA với vendor hạ tầng, ghi nhận hoạt động xử lý dữ liệu.
- **Data minimization:** hash (SHA-256 + salt) GAID/IDFA/IP khi lưu dài hạn, raw ID chỉ giữ trong Redis/TTL ngắn phục vụ matching; IP cắt octet cuối sau khi geo.
- **Bảo mật:** TLS 1.2+, HMAC SDK, secret rotation, mã hoá at-rest, RBAC + SSO, audit log, pen-test trước go-live.

---

## 7. Roadmap triển khai (A → Z)

Giả định team: 1 Tech Lead/Architect, 2 Backend (Go), 1 Data Engineer, 1 Mobile/SDK (Android+iOS+Unity), 1 Frontend (bán thời gian từ Phase 3), 1 QA, 1 PM/UA liaison (bán thời gian), DevOps chia sẻ.

### Phase 0 — Khảo sát & quyết định (Tuần 1–3)
- [ ] Thống kê hiện trạng: số app, MAU/DAU, events/ngày, installs/tháng, chi phí AF/Adjust hiện tại, spend theo network.
- [ ] Liệt kê **mọi báo cáo/feature UA team đang dùng** trên AF/Adjust (phỏng vấn UA, data, monetization).
- [ ] Liên hệ từng ad network: xác nhận hỗ trợ in-house MMP, lấy spec click URL macro + postback (quy trình chi tiết: [PARTNER_APPROVALS.md](PARTNER_APPROVALS.md)).
- [ ] Nộp sớm các đơn xin quyền API mất nhiều thời gian: Google Ads developer token, Meta Business Verification + App Review, TikTok developer app.
- [ ] Chốt phương án SAN (A/B/C – mục 4.4).
- [ ] Ước tính tải: ví dụ 50M events/ngày ≈ 600 events/s trung bình, peak ×5–10.
- [ ] Chọn cloud, chốt tech stack, ước lượng chi phí hạ tầng.
- **Deliverable:** tài liệu yêu cầu (PRD), ADR kiến trúc, business case (ROI), danh sách network & trạng thái hỗ trợ.

### Phase 1 — Nền tảng dữ liệu & MVP "Analytics" (Tuần 4–12)
- [ ] Hạ tầng: Terraform, GKE, Redpanda, ClickHouse, Postgres, CI/CD, môi trường dev/staging/prod, monitoring.
- [ ] SDK v0.x Android + iOS + Unity: install/session/event, revenue, ad revenue (MAX/LevelPlay/AdMob), offline queue, HMAC, consent.
- [ ] Edge collector + stream processor (validate, dedup, enrich geo).
- [ ] IAP receipt verification + refund.
- [ ] Organic vs non-organic cơ bản: Install Referrer (Android), ASA (iOS).
- [ ] Export BigQuery + dashboard Metabase: DAU, retention, revenue, ARPDAU theo app/country/version.
- [ ] Tích hợp SDK vào **1 game pilot** (song song AF, chạy cả hai SDK).
- **Exit criteria:** event count lệch ≤ 2% so với AF/Firebase; revenue lệch ≤ 3%; mất dữ liệu = 0 trong test kill-node.

### Phase 2 — Attribution ad network & Postback (Tuần 10–20)
- [ ] Link Service (click/impression, redirect, Redis click store).
- [ ] Attribution engine: waterfall đầy đủ, window, reinstall, re-attribution, debug view.
- [ ] Postback service + template cho 3–5 network lớn nhất (vd. AppLovin, Unity, Mintegral, ironSource, Moloco).
- [ ] Event mapping per network, test end-to-end với từng network (test campaign nhỏ).
- [ ] Cost ingestion cho các network trên + ASA + (aggregate) Meta/Google/TikTok.
- [ ] Báo cáo: installs, CPI, ROAS D0/D7, retention theo network/campaign/country.
- [ ] **Đối soát hằng ngày** với AF/Adjust: script so sánh theo network × ngày × country, dashboard chênh lệch.
- **Exit criteria:** chênh lệch installs non-organic ≤ 10% theo từng network (pilot app), postback success ≥ 99.5%.

### Phase 3 — iOS SKAN, SAN, Cohort & Dashboard riêng (Tuần 18–30)
- [ ] SKAN/AdAttributionKit: endpoint nhận postback copy, verify, conversion value schema + remote config, decode campaign.
- [ ] Kênh SAN theo phương án đã chốt (CAPI/SDK cho Meta & TikTok, Firebase link cho Google, mô hình phân bổ).
- [ ] Cohort LTV D0–D180, dự báo LTV v1, ROAS theo cohort, đường cong payback.
- [ ] Dashboard Next.js: overview, cohort, pivot, compare-with-AF, alert; Query API có cache.
- [ ] Admin UI: app, link, postback, event mapping, SKAN schema, RBAC.
- [ ] Mở rộng pilot lên 3–5 game.

### Phase 4 — Deep link, Fraud, Uninstall (Tuần 26–36)
- [ ] Universal Links / App Links / deferred deep link, link builder UI, QR, short link.
- [ ] Fraud rules (CTIT, click injection, datacenter IP, Play Integrity/App Attest) + báo cáo rejected installs.
- [ ] Uninstall tracking (silent push).
- [ ] Audiences: đẩy segment sang Google Customer Match / Meta Custom Audiences (Phase 4–5).

### Phase 5 — Chạy song song toàn bộ & Cutover (Tuần 32–44)
- [ ] Tích hợp SDK cho toàn bộ portfolio game (Unity package qua UPM nội bộ).
- [ ] Parallel run ≥ 4–6 tuần cho mỗi app: UA team ra quyết định bằng số của MMP nội bộ, AF chỉ để đối chiếu.
- [ ] Chuyển tracking link / postback trên từng network từ AF sang MMP nội bộ (theo đợt, network nhỏ trước).
- [ ] Load test 3× peak, DR drill (khôi phục từ Kafka replay & backup), pen-test, review privacy/pháp lý.
- [ ] Runbook vận hành, on-call, SLO, tài liệu cho UA team, training.
- [ ] Gỡ SDK AF/Adjust (hoặc hạ gói còn SAN nếu chọn phương án B), giữ export lịch sử từ AF vào BigQuery.

### Phase 6 — Vận hành & tối ưu liên tục (sau tuần 44)
- LTV prediction bằng ML, incrementality test (geo-lift), MMM, tối ưu bid tự động dựa ROAS dự báo, mở rộng network, giảm chi phí hạ tầng.

### Gantt tóm tắt

```
Tuần         1   4   8   12  16  20  24  28  32  36  40  44
Phase 0      ███
Phase 1          █████████
Phase 2                 ██████████████
Phase 3                         ██████████████
Phase 4                                 ██████████
Phase 5                                     ████████████
```

---

## 8. Kiểm thử & đối soát

- **Unit/integration:** attribution engine có bộ test case cố định (golden dataset) cho mọi nhánh waterfall, window biên, reinstall, timezone.
- **SDK:** test trên ma trận thiết bị (Android 8–15, iOS 15–19), Firebase Test Lab / BrowserStack, test offline/kill app/đổi giờ hệ thống.
- **E2E với network:** campaign test ngân sách nhỏ, kiểm tra click → install → postback → network dashboard.
- **Load test:** k6/vegeta cho collector & link service ở 3× peak.
- **Chaos:** tắt node ClickHouse/Kafka/consumer → không mất dữ liệu, lag hồi phục < 15 phút.
- **Đối soát liên tục với AF/Adjust** (giai đoạn song song): báo cáo tự động hằng ngày, ngưỡng cảnh báo, điều tra nguyên nhân lệch (window khác, timezone, định nghĩa install, fraud, consent).

**KPI chất lượng:**

| Chỉ số | Mục tiêu |
|---|---|
| Uptime collector / link service | ≥ 99.9% (mục tiêu 99.95%) |
| Độ trễ dữ liệu lên dashboard | ≤ 15 phút (realtime metrics), ≤ 2h (cohort) |
| Mất event | < 0.1% |
| Lệch installs non-organic vs AF (theo network) | ≤ 5–10% |
| Lệch revenue (IAP + ad) | ≤ 3% |
| Postback thành công | ≥ 99.5% |
| Crash do SDK | 0 |

---

## 9. Ước tính chi phí

### 9.1 Hạ tầng (tham khảo, ~50M events/ngày, ~3M installs/tháng)

| Hạng mục | USD/tháng (ước) |
|---|---|
| K8s (collector, link, consumer, API) | 800 – 1,500 |
| Redpanda/Kafka (managed hoặc tự host) | 400 – 1,000 |
| ClickHouse (ClickHouse Cloud hoặc tự host 3 node) | 800 – 2,000 |
| Redis, Postgres | 200 – 400 |
| BigQuery storage + query | 200 – 800 |
| CDN/LB, egress, GeoIP, monitoring | 200 – 500 |
| **Tổng** | **~2,600 – 6,200** |

### 9.2 Nhân sự
- Xây dựng: 6–8 người × 9–12 tháng.
- Vận hành sau go-live: 2–3 người (backend/data + SDK) duy trì, cập nhật API network, OS mới (iOS/Android hằng năm), SKAN/AdAttributionKit.

### 9.3 Business case
- So sánh: chi phí AF/Adjust hiện tại/năm vs (hạ tầng + nhân sự vận hành)/năm. Thường hoà vốn khi phí MMP > ~150–250K USD/năm; dưới mức này, lợi ích chính là sở hữu dữ liệu & linh hoạt chứ không phải tiết kiệm.

---

## 10. Rủi ro & giảm thiểu

| Rủi ro | Mức | Giảm thiểu |
|---|---|---|
| SAN không chia sẻ user-level data | Cao | Chốt phương án 4.4 sớm; hybrid; mô hình hoá; Firebase cho Google |
| Ad network không hỗ trợ in-house MMP / postback | Trung bình | Xác nhận ở Phase 0; giữ AF cho network đó đến khi có giải pháp |
| Lệch số liệu làm UA team mất niềm tin | Cao | Parallel run dài, dashboard đối soát minh bạch, giải thích rõ định nghĩa |
| Thay đổi chính sách Apple/Google (ATT, Privacy Sandbox Android, SKAN → AdAttributionKit) | Trung bình | Kiến trúc mô-đun hoá signal; người phụ trách theo dõi WWDC/Google I/O |
| SDK gây crash / ảnh hưởng hiệu năng game | Cao | Rollout theo % (remote config kill-switch), test ma trận thiết bị, crash monitoring |
| Mất dữ liệu ingest | Cao | Queue bền vững, offline queue SDK, multi-zone, replay |
| Vi phạm privacy / pháp lý | Cao | Review pháp lý trước go-live, consent, DSAR, data minimization |
| Thiếu nhân sự hiểu ad-tech | Trung bình | Thuê/đào tạo 1 người có kinh nghiệm MMP/ad-tech; tài liệu hoá |
| Chi phí vận hành vượt dự kiến | Thấp–TB | Đo cost/1M events từ Phase 1, TTL dữ liệu thô, tiering storage |

---

## 11. Cấu trúc repo đề xuất

```
MMP/
├── docs/                    # PLAN, PRD, ADR, runbook, spec postback từng network
├── sdk/
│   ├── android/             # Kotlin library (AAR)
│   ├── ios/                 # Swift Package / XCFramework
│   └── unity/               # UPM package (C# wrapper + plugins)
├── services/
│   ├── collector/           # Go – ingest
│   ├── linkservice/         # Go – click/impression/deeplink
│   ├── attribution/         # Go – stream consumer + engine
│   ├── postback/            # Go – gửi postback
│   ├── skan/                # Go – nhận & verify SKAN/AAK
│   ├── revenue/             # Go – verify receipt, ILRD reconcile
│   ├── api/                 # Query API + admin API
│   └── jobs/                # cost ingest, fx, export, reconcile (Airflow/Dagster DAGs)
├── web/                     # Dashboard Next.js
├── schemas/                 # Event schema (JSON Schema/Protobuf), ClickHouse DDL, Postgres migrations
├── infra/                   # Terraform, Helm charts
└── tools/                   # load test, golden datasets, reconcile scripts
```

---

## 12. Checklist go-live (tóm tắt)

- [ ] Toàn bộ app đã tích hợp SDK bản ổn định, crash-free ≥ 99.9% phiên.
- [ ] Mọi network đang chạy đã chuyển tracking link + postback, test E2E pass.
- [ ] Cost ingest đủ 100% spend; ROAS khớp kỳ vọng UA team.
- [ ] SKAN schema đã deploy cho mọi app iOS; postback copy được nhận.
- [ ] Đối soát 4 tuần liên tục đạt ngưỡng ở mục 8.
- [ ] Runbook, on-call, SLO, alert hoạt động; DR drill pass.
- [ ] Review pháp lý & privacy pass; Privacy Manifest, Data safety cập nhật.
- [ ] Export lịch sử AF/Adjust về BigQuery; hợp đồng AF/Adjust: hạ gói/không gia hạn đúng hạn.

---

## 13. Bước tiếp theo ngay (2 tuần tới)

1. Thu thập số liệu hiện trạng (mục Phase 0) + chi phí AF/Adjust → dựng business case.
2. Họp UA team: liệt kê báo cáo bắt buộc phải có ở ngày đầu cutover.
3. Gửi email tới account manager 5 network lớn nhất hỏi về in-house attribution & postback spec.
4. Chốt phương án SAN (A/B/C) và tech stack (ADR-001).
5. Khởi tạo repo theo cấu trúc mục 11, dựng hạ tầng dev và skeleton SDK + collector.
