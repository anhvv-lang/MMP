# Xin duyệt & làm việc với đối tác

> Tài liệu bổ sung cho [PLAN.md](PLAN.md), mục 3.5, 3.8 và 4.
> ⚠️ Chính sách của từng nền tảng thay đổi thường xuyên. Mọi điều kiện ghi bên dưới cần **xác nhận lại trực tiếp với account manager hoặc tài liệu chính thức** tại thời điểm làm việc.

---

## 1. Bức tranh tổng thể: có 4 loại "duyệt"

| Loại | Đối tác | Mục đích | Độ khó |
|---|---|---|---|
| **A. Ad network chấp nhận MMP nội bộ** | AppLovin, Unity Ads, ironSource, Mintegral, Moloco, Liftoff/Vungle, Chartboost, InMobi, Pangle… | Gắn tracking link của mình, nhận postback từ mình để tối ưu campaign | Trung bình – phụ thuộc từng network |
| **B. Chương trình MMP chính thức của SAN** | Meta MMP, Google App Attribution Partner, TikTok MMP, Snap MMP, X | Nhận dữ liệu attribution cấp user | **Rất khó / gần như không khả thi** cho MMP nội bộ |
| **C. Quyền truy cập API** (không cần làm MMP) | Meta Marketing API, Google Ads API, TikTok Marketing API, Apple Search Ads API, Google Play Developer API, App Store Server API, AdMob/MAX/LevelPlay reporting | Lấy cost, số liệu tổng hợp, doanh thu, verify receipt, gửi conversion | Dễ → trung bình, chủ yếu thủ tục |
| **D. Tuân thủ store & pháp lý** | Apple App Review, Google Play review, DPA với đối tác | Đưa SDK lên app thật, hợp đồng xử lý dữ liệu | Dễ nếu chuẩn bị đúng |

---

## 2. Loại A — Ad network chấp nhận MMP nội bộ

### 2.1 Vì sao cần duyệt
Ad network tối ưu bằng dữ liệu install/event mà MMP trả về. Khi bỏ AF/Adjust, họ cần:
1. Chấp nhận tracking link **không thuộc** danh sách MMP tích hợp sẵn.
2. Có endpoint nhận postback từ bạn (định dạng, macro, cách xác thực).
3. Tin vào số liệu của bạn (vì đây là cơ sở **tính tiền** với campaign CPI/CPA).

Thực tế, nhiều network lớn **có** hỗ trợ "in-house / custom attribution" cho advertiser đủ lớn, nhưng điều kiện khác nhau: có network bật ngay, có network yêu cầu mức spend tối thiểu, có network chỉ cho chạy CPM/tROAS chứ không cho CPI, có network yêu cầu chia sẻ dữ liệu fraud hoặc ký phụ lục hợp đồng.

### 2.2 Quy trình chuẩn cho mỗi network (2–8 tuần/network)

```
Bước 1  Liên hệ AM / support ──► Bước 2  Gửi Integration Spec ──► Bước 3  Network review (kỹ thuật + pháp lý)
                                                                          │
Bước 6  Go-live & đối soát ◄── Bước 5  Test campaign nhỏ ◄── Bước 4  Cấu hình link + postback trên 2 phía
```

1. **Liên hệ account manager** (qua email/Slack chung). Nói rõ: advertiser, các app, mức spend tháng, muốn chuyển sang in-house attribution, mốc thời gian.
2. **Gửi bộ tài liệu tích hợp (Integration Spec)** – xem mục 2.3.
3. **Network review:** họ kiểm tra khả năng truyền macro (click ID, device ID, campaign/creative ID…), định dạng postback, có thể yêu cầu IP whitelist, chữ ký, hoặc ký phụ lục (addendum) về dữ liệu & billing.
4. **Cấu hình 2 phía:**
   - Phía network: khai báo click URL / impression URL template, endpoint nhận postback (hoặc họ cung cấp postback URL để bạn gọi).
   - Phía mình: tạo network trong Admin, postback template, event mapping, attribution window thống nhất.
5. **Test campaign** ngân sách nhỏ (vài trăm USD), 1 app, 1–2 geo: kiểm tra click → install → postback → số trên dashboard network.
6. **Go-live & đối soát:** chạy song song với AF 2–4 tuần, so installs giữa (a) network report (b) AF (c) MMP nội bộ; lệch ≤ 10% mới chuyển toàn bộ.

### 2.3 Bộ tài liệu cần chuẩn bị sẵn (dùng chung cho mọi network)

- **Mô tả công ty & MMP nội bộ:** ai vận hành, phạm vi (chỉ dùng cho app của mình, không bán cho bên khác).
- **Click / impression URL template** + danh sách macro cần network điền:
  `click_id, campaign_id, campaign_name, adset/ad group, creative_id, site_id/sub_publisher, gaid/idfa (nếu có), ip, ua, country, timestamp`.
- **Postback spec:** URL network cung cấp hoặc mình gọi; event nào gửi (install, event đã map, revenue); format (GET/POST JSON); retry; timeout; xác thực (token/HMAC); có gửi organic/“all installs” không.
- **Attribution rules:** click window, view-through window, last-click, thứ tự ưu tiên, re-attribution window, cách xử lý reinstall.
- **Chống gian lận:** rule đang áp dụng (CTIT, click injection…), cơ chế thông báo installs bị reject, quy trình **dispute/hoàn tiền** hằng tháng.
- **Privacy & bảo mật:** tuân thủ ATT, GDPR/consent, lưu trữ & hash ID, DPA mẫu.
- **SLA:** uptime link service, độ trễ postback, đầu mối kỹ thuật & on-call.
- **Báo cáo đối soát:** cam kết cung cấp export installs/postback log khi có tranh chấp.

### 2.4 Email mẫu gửi account manager

> **Subject:** [Piontech] Moving to in-house attribution – integration request
>
> Hi <AM name>,
>
> We're <Company>, currently running ~<X> USD/month with <Network> across <N> apps (<app ids>), measured via AppsFlyer/Adjust.
> We're building an in-house attribution platform (used only for our own apps) and plan to move <Network> to it from <month>.
>
> Could you please confirm:
> 1. Does <Network> support custom/in-house attribution for advertisers? Any requirements (minimum spend, supported bid types – CPI/CPA/tROAS, contract addendum)?
> 2. Your click & impression URL macro list and postback specification (install, in-app events, revenue).
> 3. Requirements for fraud reporting and the billing dispute process.
> 4. Whether we can run a small test campaign in parallel with our current MMP.
>
> Attached is our integration spec. Happy to set up a call with our tech team.
>
> Best regards,
> <Name>

### 2.5 Bảng theo dõi (điền ở Phase 0)

| Network | % spend | Hỗ trợ in-house? | Điều kiện | Bid type cho phép | Spec đã nhận | Test | Go-live | Đầu mối |
|---|---|---|---|---|---|---|---|---|
| AppLovin | | ☐ | | | ☐ | ☐ | ☐ | |
| Unity Ads | | ☐ | | | ☐ | ☐ | ☐ | |
| ironSource/LevelPlay | | ☐ | | | ☐ | ☐ | ☐ | |
| Mintegral | | ☐ | | | ☐ | ☐ | ☐ | |
| Moloco | | ☐ | | | ☐ | ☐ | ☐ | |
| Liftoff/Vungle | | ☐ | | | ☐ | ☐ | ☐ | |
| … | | | | | | | | |

**Thứ tự đàm phán khuyến nghị:** network nhỏ / ít rủi ro trước để hoàn thiện quy trình, network chiếm nhiều spend nhất làm sau cùng khi mọi thứ đã ổn định.

### 2.6 Nếu network từ chối
- Giữ AF/Adjust riêng cho network đó (theo dõi bằng phương án hybrid), hoặc
- Chuyển campaign sang bid type không cần postback cấp install (vd. CPM tối ưu bằng dữ liệu SDK của chính network), hoặc
- Giảm/ngừng spend nếu không đo được đáng tin cậy.

---

## 3. Loại B — Chương trình MMP chính thức của SAN (Meta, Google, TikTok, Snap)

### 3.1 Thực tế
- Các chương trình này dành cho **công ty MMP thương mại** phục vụ nhiều advertiser. Yêu cầu thường gồm: là sản phẩm thương mại có nhiều khách hàng, đáp ứng tiêu chuẩn bảo mật/kiểm toán (vd. SOC 2, ISO 27001), ký thoả thuận dữ liệu riêng, qua review kỹ thuật & pháp lý của nền tảng, đôi khi phải có quan hệ đối tác chiến lược.
- Một MMP **nội bộ, chỉ phục vụ app của chính mình**, gần như **không đủ điều kiện** – không nên đưa vào kế hoạch như một mốc có thể đạt được.
- **Kết luận:** không xin làm MMP partner. Thay vào đó dùng các API mở cho advertiser (Loại C) + SKAN + mô hình hoá (PLAN.md mục 4).

### 3.2 Việc nên làm thay thế
| Nền tảng | Thay thế cho dữ liệu MMP partner | Cần duyệt gì |
|---|---|---|
| Meta | Facebook SDK hoặc Conversions API for App Events để gửi event cho tối ưu; Marketing API Insights để lấy số tổng hợp; SKAN | Meta App + Business Verification + App Review cho quyền `ads_read` (và `ads_management` nếu cần) |
| Google Ads | Link Google Ads ↔ Firebase/GA4; Install Referrer `gclid/gbraid`; Google Ads API lấy cost & conversion | Developer token Google Ads API (xin access level phù hợp), OAuth app verification nếu cần |
| TikTok | TikTok SDK hoặc Events API; Marketing API báo cáo; SKAN | Tạo developer app trên TikTok for Business, chờ duyệt quyền Reporting |
| Snap / X | Tương tự: Conversions API + Reporting API | Developer app + duyệt quyền |

> Nếu tỷ trọng spend SAN lớn và UA team bắt buộc cần dữ liệu cấp user → chọn **phương án B** trong PLAN.md (giữ AF/Adjust gói tối thiểu chỉ cho SAN).

---

## 4. Loại C — Xin quyền truy cập API (checklist)

| API | Dùng để | Thủ tục chính | Thời gian ước |
|---|---|---|---|
| **Apple AdServices** (Search Ads attribution) | Gán install từ ASA | **Không cần duyệt** – token từ thiết bị gửi tới API công khai của Apple | 0 |
| **Apple Search Ads Campaign Management API** | Cost, campaign/keyword | Trong tài khoản ASA: tạo API user với role phù hợp, upload public key, lấy client ID | 1 ngày |
| **SKAdNetwork / AdAttributionKit postback copy** | Nhận bản copy postback | **Không cần duyệt** – khai báo `NSAdvertisingAttributionReportEndpoint` trong Info.plist, domain HTTPS hợp lệ | 0 (theo release app) |
| **App Store Server API + Server Notifications V2** | Verify IAP, refund, subscription | App Store Connect: tạo In-App Purchase key, cấu hình URL notifications | 1 ngày |
| **Google Play Developer API** | Verify purchase, voided purchases, RTDN | Google Cloud service account + cấp quyền trong Play Console; Pub/Sub cho RTDN | 1–2 ngày |
| **Google Play Install Referrer** | Attribution Android | **Không cần duyệt** – thư viện trong SDK | 0 |
| **Google Ads API** | Cost, conversion | Xin **developer token** trong MCC; mức truy cập cơ bản thường đủ cho nội bộ, cần điền form mô tả mục đích; có thể phải xác minh OAuth app | 1–4 tuần |
| **Meta Marketing API** | Cost, insights | Meta for Developers app, **Business Verification**, App Review cho quyền ads | 1–4 tuần |
| **TikTok Marketing API** | Cost, báo cáo | Đăng ký developer app TikTok for Business, chờ duyệt | 1–3 tuần |
| **AppLovin / Unity / ironSource / Mintegral… reporting API** | Cost advertiser + doanh thu monetization | Lấy API/report key trong dashboard; một số cần AM bật | 1–7 ngày |
| **AppLovin MAX / LevelPlay / AdMob revenue API** | Đối soát ad revenue | Report key (MAX), secret key (LevelPlay), AdMob API qua Google Cloud OAuth | 1–7 ngày |

**Mẹo khi viết đơn xin quyền (Meta, Google, TikTok):**
- Mô tả rõ: công cụ **nội bộ**, chỉ truy cập tài khoản quảng cáo **của chính công ty**, chỉ đọc báo cáo (read-only) trừ khi thật sự cần ghi.
- Chuẩn bị: privacy policy công khai, website công ty, tài khoản business đã xác minh (giấy phép kinh doanh, domain, email theo domain), video/screencast demo luồng OAuth nếu được yêu cầu.
- Xin **quyền tối thiểu** cần thiết – xin rộng dễ bị từ chối và kéo dài review.
- Dùng tài khoản/ứng dụng đứng tên công ty, không tên cá nhân; lưu credential trong secret manager.

---

## 5. Loại D — Store review & pháp lý

### 5.1 Apple App Store
- SDK có **Privacy Manifest** (`PrivacyInfo.xcprivacy`): khai báo dữ liệu thu, required reason API, tracking domains (domain collector khi dùng cho tracking phải khai báo và bị chặn nếu user không đồng ý ATT).
- Cập nhật **App Privacy** trên App Store Connect (Identifiers, Usage Data, Purchases… và mục đích Third-Party Advertising / Analytics).
- Chỉ đọc IDFA sau khi ATT = authorized; không fingerprint.
- Chuẩn bị ghi chú cho App Review nếu có câu hỏi về tracking.

### 5.2 Google Play
- Cập nhật **Data safety form** (device IDs, app activity, purchase history; chia sẻ với bên thứ ba?).
- Khai báo **Advertising ID** usage trong Play Console; tôn trọng opt-out.
- Game hướng tới trẻ em (Families): không thu GAID, không probabilistic matching, có thể phải tắt attribution hoàn toàn.

### 5.3 Hợp đồng & pháp lý
- **DPA** (Data Processing Agreement) với ad network khi trao đổi dữ liệu cá nhân, đặc biệt người dùng EU/UK.
- Phụ lục với network về **billing dựa trên số liệu MMP nội bộ** và quy trình dispute.
- Cập nhật **privacy policy** của các app: nêu rõ việc đo lường, chia sẻ với đối tác quảng cáo, quyền của người dùng.
- Đánh giá theo **Nghị định 13/2023/NĐ-CP** (hồ sơ đánh giá tác động xử lý dữ liệu cá nhân, chuyển dữ liệu ra nước ngoài nếu server ở nước ngoài), GDPR (nếu có user EU), CCPA/CPRA (US).
- Nếu dùng CMP: đăng ký/khai báo đúng vendor trong TCF (nếu có traffic EU).

---

## 6. Lịch xin duyệt gắn với roadmap

| Thời điểm | Việc |
|---|---|
| **Phase 0 (T1–3)** | Gửi email cho toàn bộ ad network (mục 2.4); nộp đơn Google Ads developer token, Meta Business Verification + App Review, TikTok developer app (vì có thể mất vài tuần); tạo key ASA, App Store, Play Developer API; bắt đầu review pháp lý |
| **Phase 1 (T4–12)** | Hoàn tất quyền API revenue (MAX/LevelPlay/AdMob, Play, App Store); cập nhật Privacy Manifest, Data safety, privacy policy cho app pilot |
| **Phase 2 (T10–20)** | Ký phụ lục + test campaign với 3–5 network đầu tiên; cost API cho SAN |
| **Phase 3 (T18–30)** | Khai báo SKAN endpoint trong bản release iOS; mở rộng network |
| **Phase 5 (T32–44)** | Chuyển dần toàn bộ network; chốt các network từ chối (giữ AF / đổi bid type / dừng) |

**Người phụ trách:** 1 PM/UA liaison làm đầu mối đối tác + 1 kỹ sư backend làm đầu mối kỹ thuật + pháp chế review hợp đồng.
