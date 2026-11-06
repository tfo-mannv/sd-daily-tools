---
name: daily-report
description: Tạo báo cáo cuối ngày để dán vào Slack, lấy dữ liệu từ Google Sheet báo cáo hằng ngày của tháng.
disable-model-invocation: true
argument-hint: "[ngày: YYYY-MM-DD | DD/MM] (mặc định: hôm nay)"
---

# /daily-report

Sheet là nguồn dữ liệu duy nhất; skill này chỉ **đọc**, không bao giờ ghi vào sheet. Kết quả là đoạn text người dùng copy dán vào Slack.

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

Tự lấy giá trị rồi lưu lại, không bắt người dùng sửa file. Hỏi gộp các câu hỏi trong một lần nếu được:
- `sheet_name_pattern` là null → trước bước 2, hỏi tên file Google Sheet báo cáo của tháng này (người dùng copy từ tiêu đề file). Thay phần `YYYYMM` của tháng hiện tại trong tên bằng `{YYYYMM}`, ví dụ `Daily Report (202611)` → `Daily Report ({YYYYMM})`. Tên không chứa `YYYYMM` của tháng hiện tại → hỏi lại.
- `slack_header` là null → hỏi phần mở đầu báo cáo Slack mà team dùng chung (người nhắc tên, câu chào; có thể nhiều dòng). Giữ nguyên văn, kể cả xuống dòng.
- `member_name` là null → sau bước 2, đọc cột `A` của tab, liệt kê các tên khác nhau, hỏi người dùng tên nào là của họ (dùng AskUserQuestion nếu có). Giữ nguyên tên đúng như trong sheet.
- `google_email` là null và chỉ có MCP local → hỏi email Google công ty của người dùng.
- Ghi các giá trị vừa lấy vào `~/.claude/sd-daily-tools/daily-report.json` (tạo thư mục nếu chưa có; giữ nguyên các key đã có trong file). Báo người dùng một dòng rằng đã lưu và lần sau sẽ không hỏi lại.

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
   - Có `ticket_url_template` (ví dụ `https://x.backlog.com/view/{key}`) → bọc mỗi mã ticket thành `<url|key>`. Null → để mã ticket dạng text thường.

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

Chỉ hiển thị khối code và các ghi chú ngắn; người dùng copy dán trực tiếp.
