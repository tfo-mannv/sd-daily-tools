# sd-daily-tools

Hai lệnh Claude Code hỗ trợ báo cáo hằng ngày. Không bao giờ sửa Backlog. Chỉ ghi vào sheet khi bạn đồng ý (lệnh `backlog-worklog`).

| Lệnh | Dùng để |
|---|---|
| `/sd-daily-tools:daily-report` | Tạo báo cáo cuối ngày và gửi vào Slack cá nhân của bạn (DM cho chính mình, giữ in đậm và link ticket) để bạn xem lại rồi chuyển tiếp |
| `/sd-daily-tools:backlog-worklog` | Xem hôm nay bạn đã log giờ vào ticket Backlog nào, rồi bấm **Ghi vào sheet** để Claude điền thẳng vào sheet (không cần copy) |

Muốn xem ngày khác thì thêm ngày vào sau lệnh, ví dụ `/sd-daily-tools:daily-report 05/11`.

---

## Bước 1: Cài plugin

Mở **Terminal**, dán lệnh sau rồi Enter:

```bash
claude plugin install sd-daily-tools --marketplace tfo-mannv/sd-daily-tools
```

Hoặc gõ ngay trong Claude Code:

```
/plugin install sd-daily-tools --marketplace tfo-mannv/sd-daily-tools
```

Nếu được hỏi `Add marketplace?` thì gõ `y`; được hỏi scope thì chọn **user** (Enter). Sau đó thoát Claude Code và mở lại.

Cập nhật bản mới: `claude plugin update sd-daily-tools`.

## Bước 2: Kết nối Google (cho `daily-report`)

1. Trong Claude Code, gõ `/mcp` rồi Enter.
2. Chọn **claude.ai Google Drive**, bấm kết nối, đăng nhập bằng tài khoản Google công ty.
3. Làm tương tự với **claude.ai Google Sheets**.
4. (Khuyên dùng) Làm tương tự với **claude.ai Slack** để báo cáo được gửi vào tin nhắn Slack cá nhân. Claude chỉ gửi cho chính bạn, không gửi vào channel. Không muốn thì nói "tắt gửi Slack".

Lần đầu chạy `daily-report`, Claude sẽ hỏi 3 điều (team lead sẽ gửi sẵn 2 điều đầu qua Slack):

1. Tên file Google Sheet báo cáo của tháng này
2. Phần mở đầu báo cáo Slack của team
3. Tên của bạn trong sheet (chọn từ danh sách)

Trả lời một lần, lần sau không hỏi lại.

## Bước 3: Kết nối Backlog (chỉ cần nếu dùng `backlog-worklog`)

1. Cài **Node.js** (bản LTS) từ https://nodejs.org nếu máy chưa có.
2. Lấy API key: vào Backlog → ảnh đại diện góc trên phải → **Cá nhân (Personal settings)** → **API** → nhập ghi chú → **Đăng ký**. Copy key vừa tạo.
3. Mở **Terminal** (không phải trong Claude), dán lệnh dưới, thay `DOMAIN_BACKLOG` bằng domain Backlog của team (phần trước `/` trong địa chỉ web Backlog, ví dụ `abc.backlog.jp`) và `KEY_CUA_BAN` bằng key vừa copy, rồi Enter:

```bash
claude mcp add backlog -s user -e BACKLOG_DOMAIN=DOMAIN_BACKLOG -e BACKLOG_API_KEY=KEY_CUA_BAN -- npx -y backlog-mcp-server
```

4. Thoát Claude Code rồi mở lại.

> ⚠️ API key giống như mật khẩu: **không** dán vào chat với Claude, **không** gửi cho ai. Lỡ để lộ thì vào Backlog xoá key đó và tạo key mới.

## Bước 4 (tuỳ chọn): Lịch họp cố định và việc gợi ý bù giờ

`backlog-worklog` có thể tự cộng các buổi họp cố định, và nếu tổng giờ (Backlog + họp) chưa đủ 8 tiếng thì gợi ý thêm việc để báo cáo cho đủ.

Không cần sửa file, chỉ cần **nói với Claude** sau khi chạy `/sd-daily-tools:backlog-worklog`, ví dụ:

- "Thêm họp cố định: Daily MTG dự án ABC, thứ 2 đến thứ 6, 30 phút, project ABC"
- "Thêm việc gợi ý: Support team, project Others, tối đa 4 tiếng"
- "Xem lịch họp và việc gợi ý của tôi"

Claude sẽ lưu vào file cá nhân và in lại danh sách để bạn kiểm tra. Các dòng gợi ý chỉ là đề xuất, hãy xem lại trước khi dán vào sheet.

---

## Gặp lỗi?

| Hiện tượng | Cách xử lý |
|---|---|
| Claude báo chưa kết nối Google | Làm lại Bước 2 |
| Claude báo không tìm thấy sheet tháng này | Bạn chưa có quyền xem sheet, nhờ người quản lý share |
| Claude báo chưa có Backlog | Làm lại Bước 3, nhớ mở lại Claude Code |
| Trả lời nhầm ở lần đầu | Nói với Claude: "xoá config cá nhân của daily-report", rồi chạy lại lệnh |

## Giới hạn

- `daily-report` dựa vào bố cục sheet hiện tại (ngày ở dòng 2, tên ở cột A). Sheet đổi bố cục thì cần cập nhật plugin.
- `backlog-worklog` tính giờ theo ngày bạn **log**, không phải ngày làm thực tế.
