---
name: daily-report
description: Tạo báo cáo cuối ngày để dán vào Slack, lấy dữ liệu từ Google Sheet báo cáo hằng ngày của tháng.
disable-model-invocation: true
argument-hint: "[ngày: YYYY-MM-DD | DD/MM] (mặc định: hôm nay)"
---

# /daily-report

Sheet là nguồn dữ liệu duy nhất; skill này chỉ **đọc**, không bao giờ ghi vào sheet. Kết quả là đoạn text người dùng copy dán vào Slack; khi `slack_dm` bật thì gửi thêm vào tin nhắn Slack cá nhân (DM cho chính mình), giữ in đậm và link ticket, để người dùng xem lại rồi chuyển tiếp.

Người dùng có thể không rành kỹ thuật: không bao giờ bắt họ sửa file hay chạy lệnh. Luôn trả lời bằng tiếng Việt.

## Config

Đọc cả hai file rồi gộp nông (shallow-merge), key trong file cá nhân được ưu tiên:
- `config.default.json` cạnh file này: giá trị mặc định đi kèm plugin. Plugin public nên file này **không** chứa thông tin riêng của team.
- `~/.claude/sd-daily-tools/daily-report.json`: giá trị cá nhân, nằm ngoài plugin nên update plugin không bị ghi đè. Có thể chưa tồn tại.

Các tên như `member_name`, `slack_header`... ở dưới đều là key của config đã gộp.

## Công cụ Google

Dùng loại nào đang kết nối, theo thứ tự ưu tiên:
1. Connector của claude.ai: Google Drive (tìm file) + Google Sheets (`get_values`). Không cần email.
2. MCP `google-sheets` local (`search_drive_files`, `read_sheet_values`): cần `google_email`.

Không có loại nào → dừng lại, hướng dẫn người dùng: gõ `/mcp`, chọn `claude.ai Google Drive` và `claude.ai Google Sheets`, đăng nhập bằng tài khoản Google công ty, rồi chạy lại lệnh.

## Lần chạy đầu (thiếu giá trị cá nhân)

Tự lấy giá trị rồi lưu lại, không bắt người dùng sửa file.

**Bắt buộc hỏi bằng tool `AskUserQuestion`** (form có lựa chọn và ô "Other" để gõ). Không hỏi bằng bảng hay danh sách trong tin nhắn: người dùng không nhận ra đó là câu hỏi cần trả lời. Mỗi câu hỏi có 2–4 lựa chọn; giá trị tự do thì hướng dẫn gõ vào ô "Other" ngay trong nội dung câu hỏi. Chỉ khi tool này không có mới hỏi bằng tin nhắn, và khi đó đánh số từng câu, ghi rõ "Hãy trả lời theo số thứ tự".

**Đợt 1: trước bước 2**, gộp các câu còn thiếu vào **một** lần gọi `AskUserQuestion`:
- `google_email` (chỉ khi phải dùng MCP local): lựa chọn đầu là email của người dùng nếu context phiên có sẵn; lựa chọn còn lại "Email khác (gõ vào ô Other)".
- `sheet_name_pattern`: nếu có công cụ tìm file Drive, tìm trước các spreadsheet có tên chứa `YYYYMM` của tháng này và đưa tối đa 4 tên tìm được làm lựa chọn. Không tìm được thì hỏi người dùng gõ tên file vào ô "Other" (ví dụ `Daily Report (202611)`). Sau khi có tên: thay phần `YYYYMM` trong tên bằng `{YYYYMM}`. Tên không chứa `YYYYMM` của tháng này → hỏi lại.
- `slack_header`: lựa chọn "Không dùng header" và "Nhập header (gõ vào ô Other)". Ghi rõ trong câu hỏi: header là phần mở đầu team dùng chung, chỗ cần xuống dòng thì gõ `\n`. Lưu `\n` thành ký tự xuống dòng thật. Chọn "Không dùng header" → lưu chuỗi rỗng `""`.

**Đợt 2: sau bước 2** (đã có spreadsheet), nếu `member_name` là null: đọc cột `A` của tab, lấy danh sách tên khác nhau (bỏ các dòng tiêu đề như `NAME`, `DEV`, `END`, `PROJECT`...).
- Chọn tối đa 4 tên có khả năng nhất làm lựa chọn: ưu tiên tên khớp với tên hiển thị của tài khoản Google, hoặc khớp với phần trước `@` của email (ví dụ `mannv` ≈ "Mận" + chữ cái đầu họ/tên đệm "N", "V"), bỏ dấu khi so sánh.
- Ghi trong câu hỏi: không thấy tên mình thì gõ tên vào ô "Other". Tên gõ vào → so khớp với danh sách (bỏ dấu, không phân biệt hoa thường); khớp đúng một tên thì dùng tên đó, còn lại thì hỏi lại.
- Luôn lưu tên đúng như trong sheet.

**Lưu:** ghi các giá trị vừa lấy vào `~/.claude/sd-daily-tools/daily-report.json` (tạo thư mục nếu chưa có; giữ nguyên các key đã có). Báo người dùng một dòng rằng đã lưu và lần sau không hỏi lại.

## Các bước

1. **Ngày cần báo cáo.** Đọc `$ARGUMENTS` (`YYYY-MM-DD` hoặc `DD/MM`, năm hiện tại). Trống → hôm nay. `YYYYMM` = tháng của ngày đó.

2. **Xác định spreadsheet.** Xong khi có đúng một spreadsheet ID.
   - Có `sheet_id_overrides[YYYYMM]` → dùng luôn.
   - Không có → tìm trên Drive spreadsheet có tên đúng bằng `<sheet_name_pattern với YYYYMM>` (MCP local: `search_drive_files` với `name = '...'` và mimeType spreadsheet).
   - 0 kết quả → dừng: nhiều khả năng người dùng chưa có quyền xem sheet tháng này; bảo họ xin quyền.
   - Nhiều hơn 1 kết quả → liệt kê (tên, chủ sở hữu, link), hỏi người dùng chọn; lưu lựa chọn vào `sheet_id_overrides["YYYYMM"]` trong file cá nhân.

3. **Xác định vùng ô.** Tab = `tab_name_pattern`. Xong khi có một khối dòng và một khối 5 cột của ngày.
   - Dòng: đọc cột `A`; khối là các dòng liên tiếp có giá trị bằng `member_name`. Dòng đầu tiên của khối là `WORKING HOURS`.
   - Cột: đọc dòng `2` (`A2:EZ2`); ô ngày `YYYY/MM/DD` là cột đầu tiên trong 5 cột `TASK | PROJECT | HOURS | % Progress | NOTES`. Tính chữ cái cột từ vị trí tìm thấy, không bao giờ dùng offset cố định. Kiểm tra lại: ô ở dòng `3` của cột đầu tiên phải là `TASK`.
   - Lưu ý về công cụ: `read_sheet_values` in số thứ tự dòng ở đầu mỗi dòng (trường đó không phải cột A). `get_values` trả về mảng không có số dòng và bỏ các ô/dòng trống ở cuối: luôn tính dòng/cột từ ô bắt đầu của range, không dựa vào độ dài mảng.

4. **Đọc dữ liệu ngày.** Một lần gọi đọc cho khối dòng × 5 cột. Dòng task = các dòng sau `WORKING HOURS` có TASK không trống.
   - Không có dòng task nào → báo người dùng ngày đó chưa nhập gì trong sheet và dừng.

5. **Kiểm tra.** Tổng HOURS của các dòng task so với giá trị `WORKING HOURS`. Lệch → ghi cảnh báo phía trên kết quả (vẫn tạo kết quả).

6. **Dựng các mục.**
   - **1. Công việc hôm nay**: các dòng task trừ những dòng bị loại (`exclude_rules`: PROJECT nằm trong `projects`, hoặc TASK chứa bất kỳ chuỗi nào trong `task_contains`). Giữ thứ tự trong sheet và giữ nguyên văn TASK, chỉ in đậm trạng thái trong ngoặc ở cuối, ví dụ `(Closed)` → `(*Closed*)`.
   - **2. Công việc ngày tiếp theo**: khối cột của ngày làm việc (thứ 2–6) kế tiếp trong cùng tab (logic như bước 3), cùng các dòng, cùng quy tắc loại. Trống hoặc đã qua cuối tháng → chuyển tiếp các task ở mục 1 có trạng thái không thuộc `done_statuses`, chỉ ghi mã ticket (phần trước dấu `【` đầu tiên hoặc trước tiêu đề cách bởi dấu cách). Báo người dùng rằng mục 2 là chuyển tiếp và có thể cần bổ sung.
   - **3. Vấn đề gặp phải**: các ô NOTES không trống của các dòng hôm nay, mỗi ô một gạch đầu dòng. Không có → `N/A`.
   - `ticket_url_template`: lấy từ config của skill này; null → lấy `ticket_url_template` trong config của `backlog-worklog` (`../backlog-worklog/config.default.json` gộp với `~/.claude/sd-daily-tools/backlog-worklog.json`). Có template (ví dụ `https://x.backlog.com/view/{key}`) → bọc mỗi mã ticket thành `<url|key>`. Vẫn null → để mã ticket dạng text thường.

7. **Kết quả.** Một khối code duy nhất, đúng định dạng dưới đây, sau đó là các cảnh báo/ghi chú dạng gạch đầu dòng:

```
<slack_header>
1. Công việc hôm nay
• <task>
2. Công việc ngày tiếp theo
• <task>
3. Vấn đề gặp phải
• N/A
```

**Header Slack (bắt buộc, mọi lần chạy, không chỉ lần đầu):**
- Dòng đầu của khối code luôn là **toàn bộ** `slack_header` đã lưu, nguyên văn từng ký tự: giữ mọi mention (`@...`), dấu câu, khoảng trắng và đủ số dòng. Không rút gọn, không dịch, không thay bằng placeholder, không bỏ dòng nào, không đưa header ra ngoài khối code.
- Ký tự xuống dòng trong header in thành xuống dòng thật. Nếu giá trị lưu đang chứa chuỗi hai ký tự `\n` (gạch chéo + n) thay vì xuống dòng thật → vẫn in thành xuống dòng, đồng thời sửa lại file cá nhân để lưu đúng.
- Trước khi trả kết quả, tự kiểm tra: phần đầu khối code phải trùng khớp hoàn toàn với `slack_header` (sau khi đổi `\n` thành xuống dòng). Không khớp → sửa rồi mới trả.
- `slack_header` là chuỗi rỗng (người dùng chọn "Không dùng header") → bỏ header, khối code bắt đầu từ `1. Công việc hôm nay`.

Chỉ hiển thị khối code và các ghi chú ngắn; người dùng copy dán trực tiếp.

8. **Gửi vào Slack cá nhân.** Chỉ khi `slack_dm` là `true` (mặc định) và có connector `claude.ai Slack` (`slack_send_message`). Chỉ gửi vào **DM của chính người dùng**; không bao giờ gửi vào channel hay cho người khác, kể cả khi header có mention.
   - `channel_id` = Slack user ID của người dùng hiện tại: lấy từ mô tả tool Slack ("current logged in user's user_id is ...") hoặc gọi `slack_read_user_profile` không truyền `user_id`. Không hard-code, không lưu vào config.
   - `message` = cùng nội dung với khối code ở bước 7 (đã qua bước tự kiểm tra header), nhưng **không bọc trong khối code**, để Slack hiển thị định dạng. Tool nhận Markdown chuẩn, không nhận Slack mrkdwn, nên đổi trước khi gửi:
     - In đậm `*Closed*` → `**Closed**`.
     - Link `<url|PROJ-101>` → `[PROJ-101](url)`.
     - Giữ nguyên mọi ký tự khác: header (mention `@...` để dạng chữ, không đổi thành `<@USERID>` để không ai bị ping), `•`, tiêu đề task, xuống dòng. Dòng mục `1. Công việc hôm nay` có thể bị Slack hiểu là danh sách đánh số; nếu số mục hiển thị sai thì thoát thành `1\. Công việc hôm nay`.
   - Trên 5000 ký tự → tách thành nhiều tin theo ranh giới mục (1/2/3), gửi theo thứ tự.
   - Gửi thành công → báo một dòng kèm link tin nhắn. Vẫn in khối code trong chat như bước 7.
   - Chưa kết nối Slack → không dừng; in kết quả như bước 7 và nhắc một dòng: gõ `/mcp`, chọn `claude.ai Slack`, đăng nhập. Muốn tắt gửi thì nói "tắt gửi Slack" → lưu `slack_dm: false` vào file cá nhân.
   - Lỗi API → báo nguyên văn lỗi, không thử lại quá 1 lần. Chạy lại skill sẽ gửi thêm một tin mới (không sửa tin cũ); vì là DM cho chính mình nên chấp nhận được.
