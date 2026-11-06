---
name: backlog-worklog
description: Liệt kê các issue Backlog mà người dùng đã log Actual Hours trong một ngày, kèm số giờ log trong ngày và trạng thái hiện tại, sẵn sàng copy dán vào sheet báo cáo hằng ngày. Dùng khi người dùng hỏi hôm nay/ngày X đã làm gì, đã log giờ vào ticket nào.
argument-hint: "[ngày: YYYY-MM-DD | DD/MM] (mặc định: hôm nay)"
---

# backlog-worklog

"Làm task trong ngày" = người dùng đã thay đổi **Actual Hours** của issue đó trong ngày. Ngoài ra không tính gì khác (comment, đổi trạng thái, đổi người phụ trách mà không đổi giờ đều bỏ qua). Chỉ **đọc** Backlog; không bao giờ ghi vào sheet. Người dùng tự copy kết quả vào sheet.

Người dùng có thể không rành kỹ thuật. Luôn trả lời bằng tiếng Việt.

## Config

Đọc `config.default.json` cạnh file này (giá trị chung của team), sau đó đọc `~/.claude/sd-daily-tools/backlog-worklog.json` nếu có (giá trị cá nhân, không bắt buộc); gộp nông, key cá nhân được ưu tiên. Mọi key đều có giá trị mặc định dùng được; `backlog_user_id` là null nghĩa là "lấy qua `get_myself`".

## Chưa có MCP Backlog

Không thấy tool `backlog` nào (ví dụ `get_myself`) → dừng lại, chỉ người dùng sang **Bước 3** trong README của plugin. Không bao giờ xin, nhận hay lặp lại API key trong chat: người dùng tự gõ key vào terminal của họ. Nếu họ lỡ dán key vào chat, không nhắc lại key đó và bảo họ vào Backlog xoá key, tạo key mới.

## Các bước

1. **Ngày cần xem.** Đọc `$ARGUMENTS` (`YYYY-MM-DD` hoặc `DD/MM`, năm hiện tại). Trống → hôm nay. Tạo khoảng thời gian theo UTC từ `utc_offset`: `[ngày 00:00 giờ địa phương, ngày+1 00:00 giờ địa phương)`. Ví dụ `+07:00`, 2026-11-03 → `[2026-11-02T17:00:00Z, 2026-11-03T17:00:00Z)`. Mốc `created` của activity là UTC; so sánh theo UTC.

2. **Người dùng.** `backlog_user_id` là null → gọi `get_myself`, dùng `id`.

3. **Lấy activity.** `get_user_recent_updates` với `userId`, `activityTypeId: [2, 14]` (2 = cập nhật issue, 14 = cập nhật hàng loạt), `count: 20`, `order: desc`. Giữ `count` = 20: mỗi activity chứa toàn bộ mô tả issue, lấy 100 một trang sẽ vượt giới hạn output của tool.
   - Phân trang bằng `maxId = <id nhỏ nhất của trang trước> - 1` cho đến khi activity cũ nhất của trang cũ hơn mốc bắt đầu, hoặc trang trả về ít hơn 20, hoặc chạm `max_pages`.
   - Chạm `max_pages` mà vẫn còn trong khoảng thời gian → cảnh báo "kết quả có thể chưa đầy đủ".

4. **Tính giờ.** Với mỗi activity trong khoảng thời gian, với mỗi phần tử trong `content.changes` có `field == "actualHours"`:
   - `old = old_value` (thiếu/trống → 0), `new = new_value` (thiếu/trống → 0). Chênh lệch = `new - old`.
   - Mã issue = `project.projectKey + "-" + content.key_id`. Giữ lại `content.summary`, `project.name`.
   - Cộng dồn chênh lệch theo mã issue trong cả khoảng thời gian (người dùng có thể log nhiều lần trong ngày).
   - Activity loại 14: nếu `changes` có `actualHours` nhưng payload không chỉ rõ từng issue, ghi vào cảnh báo kèm id activity; không đoán.

5. **Bỏ issue bằng 0.** Issue có tổng chênh lệch bằng 0 thì bỏ. Tổng âm (sửa giờ của ngày trước) → giữ lại và ghi cảnh báo.

6. **Trạng thái hiện tại.** Với mỗi issue còn lại, gọi `get_issue` theo mã → `status.name`, `issueType.name`, `actualHours` (luỹ kế). Mỗi issue một lần gọi; số issue thường ít.

7. **Kết quả.** Sắp theo thời điểm activity đầu tiên trong ngày (tăng dần). Bảng Markdown, sau đó là tổng giờ và cảnh báo:

```
Backlog worklog — <YYYY-MM-DD> (<utc_offset>)

| # | Mã | Tiêu đề | Loại | Giờ (trong ngày) | Tổng giờ | Trạng thái |
|---|----|---------|------|------------------|----------|------------|
| 1 | PROJ-101 | 【PROJ-100】... | InternalBug | 1 | 1 | DevelopDeployed |

Tổng: <tổng Giờ (trong ngày)>h
```

   - Ô mã: có `ticket_url_template` → `[MÃ](url)`.
   - Không có issue nào → báo "Không có Actual Hours nào được log ngày <ngày>" và dừng.

8. **Khối để copy.** Sau bảng, một khối code duy nhất, phân cách bằng tab, không có tiêu đề, cùng thứ tự dòng, khớp với các cột sheet `TASK | PROJECT | HOURS` để dán một lần điền được 3 ô liền nhau mỗi dòng:

```
PROJ-101 【PROJ-100】...<tiêu đề> (DevelopDeployed)	PROJ	1
```

   - TASK = `<MÃ> <tiêu đề> (<trạng thái>)`, giữ nguyên văn tiêu đề. PROJECT = `projectKey`. HOURS = số giờ log trong ngày.
   - Dùng ký tự tab thật, không dùng dấu cách.

## Giới hạn đã biết (chỉ nêu khi liên quan)

- Giờ được tính vào ngày **log**, không phải ngày làm thực tế. Sáng nay mới log giờ của hôm qua thì tính vào hôm nay.
- Giờ nhập ngay lúc **tạo** issue (activity loại 1) không có `changes` nên không phát hiện được.
- Giờ do người khác sửa trên issue của người dùng không phải activity của người dùng nên không được tính.
