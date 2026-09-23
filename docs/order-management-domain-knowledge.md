# Domain Knowledge — Order Management (Business Rules Only)

**Mục đích:** File này chỉ chứa nghiệp vụ thuần túy (business rules), dùng làm input cho bước viết `requirements.md` trong Claude Code. Không chứa diagram, ERD, hay SQL schema — những phần đó thuộc bước Design, đưa vào sau.

---

## 1. Overview

Module Order Management chịu trách nhiệm cho toàn bộ vòng đời của 1 lệnh giao dịch: từ lúc trader đặt lệnh, hệ thống kiểm tra điều kiện (buying power, thời gian, loại tài sản), tới khi lệnh được khớp, hủy, hoặc hết hạn.

---

## 2. Actors

| Actor | Vai trò |
|---|---|
| **Trader** | Người dùng cuối, đặt/sửa/hủy lệnh |
| **Risk Manager** | Giám sát exposure, có quyền can thiệp/chặn lệnh vượt hạn mức |
| **System Admin** | Xử lý sự cố, can thiệp thủ công khi lệnh bị treo bất thường |
| **Market Data Feed** | Hệ thống ngoài, cung cấp giá Bid/Ask/Last Trade real-time |
| **Matching Engine** | Hệ thống nội bộ, thực hiện khớp lệnh |

---

## 3. Use Cases

1. Đặt lệnh mới
2. Sửa lệnh đang chờ
3. Hủy lệnh
4. Xem trạng thái lệnh
5. Xem lịch sử lệnh
6. Hệ thống tự động hủy lệnh hết hạn TIF
7. Risk Manager chặn lệnh vượt hạn mức

---

## 4. Order Types

### 4.1 Market Order
Lệnh khớp ngay theo giá tốt nhất hiện có, không chỉ định giá. Khớp gần như ngay lập tức; đánh đổi là có thể bị trượt giá (slippage).

### 4.2 Limit Order
Lệnh mua/bán ở 1 mức giá chỉ định hoặc tốt hơn. Buy limit khớp ở giá limit hoặc thấp hơn; sell limit khớp ở giá limit hoặc cao hơn. Nếu giá bằng đúng limit, có thể khớp ngay (marketable) hoặc chờ trên sổ lệnh (non-marketable).

Ràng buộc sub-penny: giá ≥ $1.00 → tối đa 2 chữ số thập phân; giá < $1.00 → tối đa 4 chữ số.

### 4.3 Stop Order
Lệnh chờ kích hoạt — khi giá chạm stop price, tự động biến thành market order. Không đảm bảo giá khớp vì mang đầy đủ rủi ro trượt giá của market order sau khi kích hoạt.

### 4.4 Stop-Limit Order
Kết hợp stop price + limit price. Khi giá chạm stop price, lệnh "elected" và trở thành limit order ở mức limit price.

Edge case: nếu giá gap qua khỏi cả stop lẫn limit price, lệnh vẫn elected nhưng KHÔNG executed — tồn tại như limit order đang chờ, không tự hủy.

### 4.5 Bracket Order
Chuỗi 3 lệnh: entry + take-profit (limit) + stop-loss (stop/stop-limit). 2 lệnh thoát chỉ active sau khi entry khớp hoàn toàn. Chỉ 1 trong 2 lệnh thoát được khớp — lệnh còn lại tự hủy. Nếu take-profit khớp 1 phần, stop-loss tự điều chỉnh số lượng xuống đúng phần còn lại.

### 4.6 OCO Order (One-Cancels-Other)
Giống 2 lệnh thoát của Bracket, nhưng dùng khi vị thế đã có sẵn (không cần entry).

### 4.7 OTO Order (One-Triggers-Other)
Biến thể Bracket, chỉ có 1 trong 2 lệnh thoát (take-profit HOẶC stop-loss).

### 4.8 Trailing Stop Order
Stop price tự động dịch chuyển theo hướng có lợi, dựa trên High Water Mark (HWM). Lệnh bán: HWM = giá cao nhất đã đạt, stop_price = HWM − trail. Lệnh mua: HWM = giá thấp nhất đã đạt, stop_price = HWM + trail. HWM chỉ di chuyển 1 chiều có lợi, không lùi lại.

---

## 5. Time in Force (TIF)

| TIF | Ý nghĩa |
|---|---|
| **DAY** | Hết hiệu lực cuối phiên, tự động hủy nếu chưa khớp |
| **GTC** | Có hiệu lực tới khi bị hủy; tự hết hạn sau 90 ngày |
| **OPG** | Chỉ thử khớp trong phiên đấu giá mở cửa; không khớp thì hủy |
| **CLS** | Chỉ thử khớp trong phiên đấu giá đóng cửa; không khớp thì hủy |
| **IOC** | Khớp ngay phần nào có thể, phần còn lại hủy ngay |
| **FOK** | Phải khớp toàn bộ ngay lập tức, không thì hủy toàn bộ |

Ràng buộc: IOC/FOK/OPG/CLS chỉ áp dụng cho Market và Limit — không áp dụng cho Stop/Stop-Limit. Fractional order chỉ chấp nhận DAY. Crypto chỉ chấp nhận GTC hoặc IOC.

---

## 6. Buying Power Rules

**Rule 6.1:** Khi đặt lệnh mở vị thế, hệ thống trừ ngay giá trị lệnh khỏi available buying power (khóa tạm thời, không phải trừ tiền gốc thật).

**Rule 6.2:** Lệnh mua bị hủy trước khi khớp → buying power giải phóng ngay. Lệnh sell long/buy to cover chỉ hoàn buying power khi EXECUTED, không phải khi PLACED.

**Rule 6.3 — Far Side Pricing:** Buy order dùng giá ASK; sell order dùng giá BID (luôn là giá bất lợi hơn, để đảm bảo an toàn).

**Rule 6.4 — Pricing theo khung giờ:**
| Khung giờ | Cách tính |
|---|---|
| Giờ giao dịch chính | Far side của NBBO |
| Extended hours | Midpoint của Bid/Ask |
| Ngoài giờ hoàn toàn | Giá giao dịch gần nhất |

**Rule 6.5 — Short Sell:** `Giá trị lệnh = MAX(limit_price, 1.03 × current_ask) × quantity`. Khoản đệm 3% vì short sell có rủi ro lỗ không giới hạn. Buying power bị khóa được tính lại liên tục theo giá thị trường, không cố định tại mức lúc đặt lệnh.

---

## 7. Order State Machine

**Terminal states:** `filled`, `canceled`, `expired`

**States:** `new`, `partially_filled`, `filled`, `done_for_day`, `canceled`, `expired`, `replaced`, `pending_cancel`, `pending_replace`

**Valid transitions (ví dụ chính):**
- new → partially_filled → filled
- new → canceled (user hủy hoặc TIF hết hạn)
- new → rejected (vi phạm business rule)
- pending_replace → replaced

**Invalid transition — PHẢI ngăn chặn:**
- pending_replace → canceled (yêu cầu hủy bị từ chối khi đang chờ sửa)
- Bất kỳ terminal state nào → trạng thái khác

---

## 8. Edge Cases

1. **Elected but not executed:** Stop-limit có thể kích hoạt nhưng không khớp nếu giá gap qua khỏi limit price.
2. **Partial fill adjustment:** Take-profit khớp 1 phần → stop-loss tự điều chỉnh số lượng.
3. **Race condition ở pending_replace:** Không cho hủy khi đang chờ sửa, tránh 2 yêu cầu xung đột.
4. **IOC partial failure despite liquidity:** IOC có thể bị hủy hoàn toàn dù thị trường có đủ thanh khoản trên giấy.
5. **Buying power tăng động cho short sell:** Giá tăng → mức khóa tăng theo, có thể dẫn tới margin call.
6. **Lỗ vượt mức đã khóa ban đầu:** Lỗ thực tế short sell có thể vượt xa mức đã khóa lúc đặt lệnh.
7. **Lệnh ngoài giờ bị xếp hàng:** Gửi sau 4:00pm ET, không hủy mà xếp hàng gửi vào ngày kế tiếp.
8. **IPO symbols:** Trước khi IPO bắt đầu giao dịch, chỉ chấp nhận limit order.
9. **Extended hours yêu cầu kép:** Cần cả order type = limit VÀ TIF = day/gtc.
10. **Sub-penny rejection:** Vi phạm quy tắc số chữ số thập phân bị từ chối ngay từ input.

---

## 9. Open Questions

- Hệ thống có cần hỗ trợ đầy đủ 8 loại order ngay từ MVP, hay bắt đầu với Market/Limit/Stop/Stop-Limit trước?
- Có cần hỗ trợ extended hours trong giai đoạn đầu?
- Phạm vi tài sản: chỉ equity, hay cần cả crypto/options ngay từ đầu?
- Risk Manager override hoạt động ra sao?
- Compliance/regulatory requirement cụ thể nào cần tuân thủ?

**Giả định:** Toàn bộ business rule dựa trên tham khảo tài liệu API Alpaca Markets — cần xác nhận lại với stakeholder thật.
