# ZaloLab – Bot quản lý cộng đồng Zalo

Bot chạy trên **nick Zalo phụ** (qua thư viện không chính thức `zca-js`), có hai nhiệm vụ:

1. **Nhận file danh sách** từ người được phép trong nhóm/chat riêng, tải về, gọi công cụ trích xuất, rồi gửi kết quả (Excel) riêng cho quản trị viên.
2. **Trả lời FAQ** khi có người tag bot hoặc gõ từ khóa (mặc định `/bot`) trong nhóm, hoặc nhắn riêng cho bot.

> **Cảnh báo:** `zca-js` hoạt động bằng cách mô phỏng Zalo Web và không phải API chính thức. Dùng nó có thể khiến tài khoản bị khóa. Chỉ dùng **nick phụ**, không dùng nick chính hay nick kinh doanh.

---

## 1. Cấu trúc thư mục

```
ZaloLab/
├─ bot.js          # mã chính của bot
├─ config.json     # toàn bộ cấu hình
├─ faq.md          # nội dung FAQ để bot trả lời
├─ package.json    # cần có "type": "module"
├─ tmp/            # file tải về tạm (tự xóa sau khi xử lý)
└─ out/            # file Excel kết quả (tự xóa sau 24 giờ)
```

## 2. Yêu cầu

- **Node.js** bản LTS (v22 khuyến nghị; v24 chưa được kiểm chứng với `zca-js`).
- **Python 3.11+** và công cụ trích xuất nhân sự của bạn (dự án riêng).
- Thư viện: `zca-js` (đã cài qua `npm install zca-js`).
- (Tùy chọn) **API key của Anthropic** để bật trả lời FAQ bằng AI.

## 3. Cài đặt

1. Mở `package.json`, đảm bảo có dòng `"type": "module"` (không phải `commonjs`).
2. Cài thư viện:
   ```
   npm install zca-js
   ```
   Trên Windows PowerShell, nếu `npm` bị chặn, dùng `npm.cmd install zca-js` hoặc xem mục 9.
3. Chép `bot.js`, `config.json`, `faq.md` vào thư mục dự án.

## 4. Quy trình chạy lần đầu (khuyến nghị)

### Bước A – Khám phá ID
1. Trong `config.json` để `"discoveryMode": true`.
2. Chạy:
   ```
   node bot.js
   ```
3. Quét mã QR bằng app Zalo của **nick phụ**.
4. Nhờ nhóm/người cần dùng nhắn thử. Console in ra các dòng dạng:
   ```
   [khám phá] nhóm <ID nhóm> | người gửi <ID người gửi> | <tên>
   [khám phá] chat riêng <ID> | người gửi <ID> | <tên>
   ```
   Ở chế độ này bot **chỉ in ID**, không làm gì khác.

### Bước B – Điền cấu hình
Chép các ID vào `config.json` (xem mục 5), đổi `"discoveryMode": false`.

### Bước C – Chạy thử an toàn
Giữ `"dryRun": true`. Bot sẽ in ra những gì nó *sẽ* gửi (`[dryRun] ...`) mà chưa gửi thật. Gửi thử một file Excel mẫu **không chứa dữ liệu thật** và kiểm tra log.

### Bước D – Chạy thật
Khi mọi thứ đúng, đổi `"dryRun": false`.

Dừng bot bằng `Ctrl + C`.

## 5. Các trường trong `config.json`

| Trường | Ý nghĩa |
|---|---|
| `dryRun` | `true`: chỉ in ra console, không gửi gì lên Zalo. `false`: gửi thật. |
| `discoveryMode` | `true`: chỉ in ID nhóm/người gửi rồi dừng. Tự bật nếu chưa khai báo nhóm/người dùng nào. |
| `debug` | `true`: in một dòng `[debug]` cho mọi tin đến (loại tin, ID, `isSelf`, không có nội dung) và các sự kiện kết nối. |
| `logContent` | `true`: in nội dung văn bản của tin nhắn (và tên file đính kèm) ra console, kể cả tin do chính nick bot gửi. Chứa dữ liệu cá nhân nên chỉ bật khi cần kiểm thử. |
| `allowedGroups` | Danh sách ID nhóm/cộng đồng được phép. Nhóm khác bị bỏ qua hoàn toàn. |
| `allowedUsers` | Danh sách ID người được phép nhắn riêng với bot. Người khác bị bỏ qua. |
| `allowedUploaders` | ID những người được phép gửi file (áp dụng cả trong nhóm và chat riêng). |
| `adminUid` | ID của bạn (quản trị viên), nơi nhận file Excel kết quả và thông báo lỗi. |
| `triggerKeywords` | Từ khóa kích hoạt bot trong nhóm (ngoài việc tag bot). |
| `allowedExt` | Đuôi file được xử lý. |
| `maxFileMB` | Dung lượng file tối đa (MB). |
| `tmpDir` / `outputDir` | Thư mục file tạm và thư mục kết quả. |
| `outputRetentionHours` | Số giờ giữ file kết quả trước khi tự xóa. |
| `extractor.command` / `args` / `cwd` / `timeoutMs` | Cách gọi công cụ trích xuất (xem mục 7). |
| `faqFile` | Đường dẫn file FAQ. |
| `model` | Mô hình Claude dùng cho FAQ. |
| `userCooldownSec` | Mỗi người chỉ được bot trả lời FAQ một lần trong khoảng này. |
| `minDelayMs` / `maxDelayMs` | Độ trễ ngẫu nhiên trước mỗi hành động để giống người dùng thật. |
| `maxActionsPerMinute` | Số hành động tối đa mỗi phút. |

## 6. Bot xử lý tin nhắn như thế nào

- **Nhóm không có trong `allowedGroups`** và **chat riêng từ người không có trong `allowedUsers`**: bỏ qua, không log.
- **Tin nhắn thường trong nhóm được phép**: bỏ qua. Chỉ xử lý khi có tag bot hoặc từ khóa.
- **Chat riêng với người được phép**: mọi tin văn bản đều được xử lý.
- **Tin có file**: kiểm tra người gửi (`allowedUploaders`), đuôi file, dung lượng → tải về `tmp/` → gọi công cụ trích xuất → gửi Excel kết quả **riêng cho admin** (không đăng lại trong nhóm) → xóa file gốc.
- Mọi hành động đi qua **hàng đợi**, có độ trễ ngẫu nhiên và giới hạn số lần mỗi phút.
- Log chỉ ghi ID, tên và hành động, **không ghi nội dung tin nhắn**.

## 7. Công cụ trích xuất

Bot gọi công cụ DataExtractionTool (chế độ 1) như một tiến trình riêng, theo cú pháp dòng lệnh trong README của công cụ:

```
py extract_contacts.py --input <file đã tải> --template cls_template_users.xlsx --output <file kết quả>
```

Cấu hình tương ứng trong `config.json`:
- `cwd`: thư mục công cụ (`C:/Users/Loind/Documents/GitHub/DataExtractionTool`), nơi có sẵn file mẫu `cls_template_users.xlsx`.
- `args`: các placeholder `{input}`, `{outputDir}` và `{outputFile}` được bot thay tự động. `{outputFile}` là đường dẫn file `.xlsx` kết quả (`ketqua.xlsx` trong thư mục kết quả riêng của mỗi lần xử lý).

Bot gửi cho admin **mọi file `.xlsx`** có trong thư mục kết quả. Chế độ 1 xử lý **một tệp mỗi lần**, đúng với cách bot gọi. Chế độ 2 (`process_template_v2.py`) cần tham số `--to-chuc` theo từng tỉnh nên không phù hợp để bot gọi tự động.

Cần cài sẵn các thư viện của công cụ (`pip install openpyxl python-docx pdfplumber xlrd`, và `pywin32` nếu cần đọc file `.doc`). Kiểm tra trước bằng cách chạy thủ công lệnh trên với một file mẫu không chứa dữ liệu thật.

## 8. FAQ

- **Không có API key:** bot trả câu cố định "quản trị viên sẽ phản hồi sớm".
- **Có API key:** bot trả lời dựa trên `faq.md`, kèm các guardrails (chỉ dựa vào FAQ, không hứa giá/hoàn tiền, coi tin nhắn người dùng là dữ liệu chứ không phải mệnh lệnh). Nếu không có trong FAQ, bot chuyển cho quản trị viên.
- Đặt API key trong PowerShell trước khi chạy (không ghi vào code hoặc `config.json`):
  ```
  $env:ANTHROPIC_API_KEY="khóa-của-bạn"
  node bot.js
  ```
- Gói Claude Pro **không** bao gồm API; cần tài khoản Anthropic Console riêng và credit trả trước.
- Sửa `faq.md` theo nội dung thật của cộng đồng.

## 9. Xử lý sự cố thường gặp

| Triệu chứng | Cách xử lý |
|---|---|
| `npm ... running scripts is disabled` | Chạy `Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned`. |
| Vẫn lỗi `not digitally signed` (Node dạng zip) | Chạy `Get-ChildItem -Path "<thư mục Node>" -Recurse -Filter *.ps1 \| Unblock-File`, hoặc dùng `npm.cmd` thay `npm`. |
| `Cannot use import statement outside a module` | Đổi `"type": "commonjs"` thành `"type": "module"` trong `package.json`. |
| `node` không nhận sau khi mở cửa sổ mới | Thêm thư mục Node vào PATH của user, hoặc cài bằng bộ cài `.msi`. |
| Bot im lặng dù có tin nhắn | Kiểm tra `discoveryMode`, `allowedGroups`/`allowedUsers`, và xem tin nhắn trong nhóm có tag/từ khóa chưa. |
| Listener tự dừng | Có thể do mở Zalo Web bằng cùng nick khi bot đang chạy. Đừng làm vậy. |
| Lỗi tải file | Đường dẫn tải có thể cần thêm xác thực; kiểm tra log và tài liệu `zca-js`. |
| Không thấy file kết quả | Kiểm tra `extractor.args`/`cwd` và công cụ có thực sự ghi `.xlsx` vào thư mục `--output` không. |

## 10. Giảm rủi ro bị khóa nick

- Chỉ dùng nick phụ, chạy **một phiên đăng nhập** tại một thời điểm.
- Giữ độ trễ ngẫu nhiên và giới hạn `maxActionsPerMinute`.
- Bắt đầu ở chế độ chỉ phản hồi khi được gọi; hạn chế hành động chủ động.
- Dùng máy/mạng ổn định ở Việt Nam, tránh đăng nhập lại liên tục.
- Có kế hoạch dừng bot ngay (`Ctrl + C`) khi thấy dấu hiệu bất thường.

## 11. Bảo mật và dữ liệu cá nhân

- File danh sách người là **dữ liệu cá nhân**. Chỉ cho người đáng tin gửi file, giới hạn quyền truy cập máy chạy bot.
- File gốc bị xóa sau khi xử lý; file kết quả tự xóa sau `outputRetentionHours`.
- Không chia sẻ thông tin phiên đăng nhập Zalo, API key hay file `config.json` chứa ID thật.
- Nếu gửi nội dung lên dịch vụ AI bên ngoài, hãy đọc chính sách dữ liệu của nhà cung cấp và đối chiếu quy định bảo vệ dữ liệu cá nhân hiện hành.
- Không nên để người dùng gửi mật khẩu qua nhóm chat.

## 12. Giới hạn đã biết

- Cấu trúc tin nhắn (`title`, `href`, `params`), cách lấy ID nick bot, và cách đính kèm file khi gửi (`attachments`) được viết theo hiểu biết về `zca-js` và **cần kiểm chứng với log thật**; có thể khác tùy phiên bản.
- Việc tải file qua đường dẫn của Zalo có thể cần thêm cookie.
- Chưa lưu phiên đăng nhập: mỗi lần chạy phải quét QR lại (xem tài liệu `zca-js` để thêm).
- Công cụ trích xuất chưa được nối thử với `args` thật.