---
name: backlog-worklog
description: Liệt kê các issue Backlog mà người dùng đã log Actual Hours trong một ngày, kèm số giờ log trong ngày và trạng thái hiện tại; cộng lịch họp cố định và tự gợi ý việc bù cho đủ giờ làm; sẵn sàng copy dán vào sheet báo cáo hằng ngày. Dùng khi người dùng hỏi hôm nay/ngày X đã làm gì, đã log giờ vào ticket nào, hoặc muốn xem/sửa lịch họp cố định hay danh sách việc gợi ý.
argument-hint: "[ngày: YYYY-MM-DD | DD/MM] (mặc định: hôm nay)"
---

# backlog-worklog

"Làm task trong ngày" = người dùng đã thay đổi **Actual Hours** của issue đó trong ngày. Ngoài ra không tính gì khác (comment, đổi trạng thái, đổi người phụ trách mà không đổi giờ đều bỏ qua). Chỉ **đọc** Backlog. Chỉ **ghi vào sheet** ở bước 9, sau khi người dùng xác nhận. Trước khi ghi luôn hiện bảng xem trước đầu việc trong ngày (bước 8) để người dùng quyết định.

Người dùng có thể không rành kỹ thuật. Luôn trả lời bằng tiếng Việt.

## Config

Đọc `config.default.json` cạnh file này (giá trị chung của team), sau đó đọc `~/.claude/sd-daily-tools/backlog-worklog.json` nếu có (giá trị cá nhân, không bắt buộc); gộp nông, key cá nhân được ưu tiên. Mọi key đều có giá trị mặc định dùng được; `backlog_user_id` là null nghĩa là "lấy qua `get_myself`".

Các key cho phần bù giờ (đặt trong file cá nhân, vì mỗi người một lịch):

```json
{
  "working_hours": 8,
  "recurring_meetings": [
    { "task": "【MTG Daily】Project", "project": "Project", "hours": 0.5, "weekdays": ["mon", "tue", "wed", "thu", "fri"] }
  ],
  "filler_tasks": [
    { "task": "Support team", "project": "Others", "max_hours": 4 }
  ]
}
```

- `recurring_meetings`: họp cố định. `weekdays` dùng `mon`..`sun`. `task`/`project` ghi đúng như sẽ nhập vào sheet.
- `filler_tasks`: việc dùng để gợi ý bù giờ, **thứ tự trong danh sách = thứ tự ưu tiên**. `max_hours` không có → không giới hạn. `weekdays` không bắt buộc; có thì chỉ gợi ý vào các thứ đó.
- Danh sách trống hoặc không có → bỏ qua phần tương ứng.

Các key cho việc ghi sheet (bước 9):
- `sheet_project_list_range` (mặc định `Category!B3:B100`): vùng chứa danh sách PROJECT hợp lệ của sheet (cột PROJECT có data validation theo danh sách này).
- `project_map`: ánh xạ đã học `{"<projectKey Backlog>": "<tên PROJECT trong sheet>"}`, lưu trong file cá nhân.
- Vị trí sheet (spreadsheet, tab, tên người dùng, email) dùng chung config của skill `daily-report`: `config.default.json` trong thư mục `../daily-report/` (cạnh thư mục skill này) gộp với `~/.claude/sd-daily-tools/daily-report.json`.

## Xem / sửa lịch họp và việc gợi ý

Người dùng nói bằng lời (ví dụ "thêm họp X thứ 5 30 phút", "bỏ việc Y", "xem lịch họp của tôi") → đọc file cá nhân, sửa đúng key liên quan, giữ nguyên các key khác, ghi lại file. Đổi phút sang giờ (30 phút = 0.5). Thiếu thông tin (thứ mấy, bao lâu, project) → hỏi bằng `AskUserQuestion`. Sau khi ghi, in lại danh sách hiện tại dạng bảng để người dùng kiểm tra. Không bắt người dùng tự sửa file.

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
   - Không có issue nào → ghi "Không có Actual Hours nào được log ngày <ngày>" và **vẫn làm tiếp** bước 7b (họp cố định và gợi ý vẫn có ích).

7b. **Họp cố định và gợi ý bù giờ.** Bỏ qua bước này nếu không có cả `recurring_meetings` lẫn `filler_tasks`.
   - Thứ của ngày cần xem tính theo giờ địa phương (`utc_offset`).
   - **Họp trong ngày** = các mục `recurring_meetings` có thứ đó trong `weekdays`.
   - `thiếu = working_hours − tổng giờ Backlog trong ngày − tổng giờ họp`.
   - `thiếu > 0` → duyệt `filler_tasks` theo thứ tự (bỏ mục có `weekdays` không chứa thứ đó). Mỗi mục nhận `min(phần còn thiếu, max_hours)`; hết thiếu thì dừng. Duyệt hết danh sách mà vẫn thiếu → cảnh báo "còn thiếu Xh, danh sách việc gợi ý không đủ".
   - `thiếu < 0` → cảnh báo "tổng giờ vượt `working_hours` Xh", không gợi ý gì.
   - In thêm sau phần tổng của bảng Backlog:

```
### Họp cố định (<thứ>)
| Việc | Project | Giờ |
|------|---------|-----|
| 【MTG Daily】Project | Project | 0.5 |

### Gợi ý bù giờ (còn thiếu <thiếu>h)
| Việc | Project | Giờ |
|------|---------|-----|
| Support team | Others | 2 |

Tổng cả ngày: <Backlog + họp + gợi ý>h / <working_hours>h
```

   - Ghi rõ dưới bảng: phần "Gợi ý" chỉ là đề xuất, người dùng tự kiểm tra lại trước khi nhập sheet. Ngày nghỉ lễ / nghỉ phép / họp bị huỷ thì skill không biết.

8. **Xem trước đầu việc trong ngày.** Sau các bảng ở bước 7/7b, in **một** bảng gộp đúng các dòng sẽ ghi vào sheet, theo thứ tự ghi: họp cố định → issue Backlog (theo thứ tự bảng) → gợi ý bù giờ. Đây là thứ người dùng xem để quyết định, nên phải trùng với dữ liệu sẽ ghi ở bước 9.

```
### Đầu việc ngày <DD/MM> (<thứ>) — xem trước trước khi ghi sheet
| # | TASK | PROJECT | HOURS | Nguồn |
|---|------|---------|-------|-------|
| 1 | 【MTG Daily】Project | Project | 0.5 | Họp |
| 2 | PROJ-101 【PROJ-100】...<tiêu đề> (DevelopDeployed) | Project | 1 | Backlog |
| 3 | Support team | Others | 2 | Gợi ý |

Tổng: <tổng>h / <working_hours>h · Số dòng: <n> / <số dòng task của sheet, nếu đã biết>
```

   - Dòng Backlog: TASK = `<MÃ> <tiêu đề> (<trạng thái>)`, giữ nguyên văn tiêu đề. Dòng họp / gợi ý: TASK = `task` trong config.
   - PROJECT / HOURS: nếu đã có config sheet (mục Config của `daily-report`) và công cụ Google, chuẩn hoá ngay theo bước 9c (chỉ **đọc** sheet: danh sách project, số dòng task) để bảng xem trước là giá trị cuối cùng. Project chưa khớp danh sách → hiện `⚠ <giá trị gốc>`, sẽ hỏi ở bước 9. Chưa có config/công cụ Google → hiện `projectKey` / `project` gốc và ghi chú "PROJECT sẽ được chuẩn hoá khi ghi".
   - Số dòng vượt số dòng task của sheet → cảnh báo ngay dưới bảng.
   - Không in khối copy dạng tab nữa (dán tay hay lỗi format). Chỉ in khi người dùng yêu cầu ("cho tôi khối copy"): một khối code, phân cách bằng tab thật, cột `TASK | PROJECT | HOURS`, không tiêu đề.
   - Không có dòng nào (không Backlog, không họp, không gợi ý) → báo "Không có đầu việc nào cho ngày <ngày>" và dừng, không hỏi ghi sheet.

9. **Ghi vào sheet (khi người dùng đồng ý).** Ngay sau bảng xem trước, hỏi bằng `AskUserQuestion`. Hộp thoại có thể che phần chat phía trên, nên câu hỏi phải tự đủ ý: "Ghi <n> dòng (tổng <tổng>h) vào sheet ngày <DD/MM>?". Lựa chọn:
   - "Ghi vào sheet": trường `preview` chứa lại bảng xem trước dạng chữ đơn giản (`#  TASK | PROJECT | HOURS | Nguồn`, cắt TASK dài quá 50 ký tự bằng `…`) để xem lại ngay trong hộp thoại.
   - "Chưa, tôi muốn sửa": người dùng nói thay đổi (bỏ dòng, đổi giờ, đổi project, thêm việc) → sửa danh sách, **in lại bảng xem trước** và hỏi lại. Lặp đến khi người dùng chọn ghi hoặc không ghi.
   - "Không ghi": dừng ở đây; người dùng vẫn có thể nói "ok push lên sheet" sau đó trong cùng phiên để ghi đúng danh sách đang xem.

   **Không bao giờ ghi khi chưa có xác nhận** cho đúng danh sách đang hiển thị gần nhất.

   a. **Config sheet.** Đọc config `daily-report` như mục Config. Thiếu `sheet_name_pattern` / `member_name` (hoặc `google_email` khi chỉ có MCP local) → làm theo mục "Lần chạy đầu" trong `../daily-report/SKILL.md` để hỏi và lưu, rồi làm tiếp.

   b. **Định vị ô.** Làm theo bước 2 và 3 của `../daily-report/SKILL.md` (xác định spreadsheet, khối dòng của `member_name`, khối 5 cột của ngày). Chỉ ghi 3 cột `TASK | PROJECT | HOURS` của các **dòng task** (các dòng sau `WORKING HOURS` trong khối). Không bao giờ ghi vào dòng `WORKING HOURS` (ô giờ của nó là công thức `SUM`), cột `% Progress`, cột `NOTES`, hay ô của người khác / ngày khác.

   c. **Chuẩn hoá giá trị** cho khớp data validation của sheet (đây là nguyên nhân dán tay hay bị lỗi format):
      - PROJECT: đọc danh sách hợp lệ ở `sheet_project_list_range`. Dòng Backlog: dùng `project_map[projectKey]` nếu có; không có thì so khớp `project.name` rồi `projectKey` với danh sách (không phân biệt hoa thường, bỏ khoảng trắng/gạch). Dòng họp/gợi ý: `project` trong config phải có trong danh sách. Không khớp → hỏi bằng `AskUserQuestion` (tối đa 4 tên gần nhất + ô Other), lưu kết quả vào `project_map` trong file cá nhân.
      - HOURS: số (number), không phải chuỗi; làm tròn tới bội số 0.25 gần nhất và cảnh báo nếu có làm tròn.
      - TASK: chuỗi, giữ nguyên văn như bảng xem trước.

   d. **Kiểm tra trước khi ghi.** Đọc giá trị hiện tại của 3 cột ở các dòng task.
      - Số dòng cần ghi > số dòng task của khối → **dừng**, không chèn thêm dòng (chèn dòng sẽ làm lệch dữ liệu của mọi người và mọi ngày). Báo số dòng còn trống và đề nghị người dùng bỏ/gộp bớt.
      - Các dòng đã có dữ liệu → in bảng "Hiện có trên sheet" và hỏi bằng `AskUserQuestion`: "Ghi đè toàn bộ ngày này" / "Chỉ thêm vào dòng trống" / "Huỷ".
        - Ghi đè: dòng mới ghi từ dòng task đầu tiên; dòng thừa phía dưới ghi `""` cho 3 cột. Nhắc người dùng rằng `% Progress`/`NOTES` của các dòng đó giữ nguyên, cần tự xem lại.
        - Chỉ thêm: giữ nguyên dòng cũ; bỏ các dòng mới trùng mã ticket hoặc trùng TASK với dòng cũ (liệt kê các dòng bị bỏ); ghi phần còn lại vào các dòng trống.
      - In bảng xem trước: địa chỉ ô, giá trị cũ → giá trị mới.

   e. **Ghi.** Ngay trước khi ghi, đọc lại vùng đó; khác với lúc xem trước (có người vừa sửa) → dừng, in lại và hỏi lại. Ghi bằng **một** lần gọi cho cả vùng liên tục `TASK..HOURS` của các dòng task (connector: `update_values`; MCP local: `modify_sheet_values` với `value_input_option: USER_ENTERED`). Không ghi từng ô một (tránh ghi dở dang).

   f. **Xác nhận.** Đọc lại vùng vừa ghi và ô `WORKING HOURS`; so với giá trị dự kiến. Khớp → báo "Đã ghi N dòng vào sheet, WORKING HOURS = Xh" kèm link sheet. Không khớp hoặc lỗi API → báo nguyên văn lỗi, nói rõ đã ghi được gì, không tự thử lại quá 1 lần.

## Giới hạn đã biết (chỉ nêu khi liên quan)

- Giờ được tính vào ngày **log**, không phải ngày làm thực tế. Sáng nay mới log giờ của hôm qua thì tính vào hôm nay.
- Giờ nhập ngay lúc **tạo** issue (activity loại 1) không có `changes` nên không phát hiện được.
- Giờ do người khác sửa trên issue của người dùng không phải activity của người dùng nên không được tính.
