# ZaloLab – Chatbot Zalo chạy bằng Hermes Agent

Chatbot trả lời câu hỏi thường gặp (FAQ) trên Zalo. Bot dùng **Zalo Bot API chính thức**, không còn dùng nick phụ.

```
Zalo ──► plugin zalo-platform ──► Hermes (profile cskh) ──► 9router ──► Claude
```

- **Hermes Agent** (Nous Research): chạy trong WSL Ubuntu, profile `cskh`.
- **Plugin `zalo-platform`** (cộng đồng, ghim SHA trong catalog của Hermes): nối Zalo Bot API với Hermes.
- **9router**: cổng OpenAI-compatible tại `http://localhost:20128/v1`, combo model `ZaloLab`.

> Bot cũ (`bot.js`, thư viện không chính thức `zca-js`) đã bị gỡ. Bản cuối nằm ở commit `03e5478`. Bot Zalo chính thức **không nhận được file**, nên chức năng nhận file danh sách và gọi công cụ trích xuất `.xlsx` của bot cũ không còn.

## Cấu trúc repo

```
ZaloLab/
├─ hermes/cskh/
│  ├─ SOUL.md                    # persona + guardrails + nội dung FAQ (nguồn chuẩn)
│  └─ config.zalo.example.yaml   # mẫu cấu hình Zalo và tắt tool
└─ docs/
   ├─ SETUP-VPS.md               # cài 9router, Hermes, plugin Zalo, chạy tự động
   └─ GUARDRAILS.md              # quy tắc trả lời và chuyển người
```

Bí mật (key 9router, token Zalo) **không** nằm trong repo. Chúng ở `~/.hermes/profiles/cskh/` trong WSL.

## Vận hành

Xem [docs/SETUP-VPS.md](docs/SETUP-VPS.md) cho toàn bộ cách cài và chạy.

- **Sửa FAQ:** sửa [hermes/cskh/SOUL.md](hermes/cskh/SOUL.md), chép vào `~/.hermes/profiles/cskh/SOUL.md` trong WSL rồi khởi động lại gateway.
- **Xem log:** `~/.hermes/profiles/cskh/logs/gateway.log` và `agent.log`.
- **Khởi động lại gateway:** `hermes gateway stop` rồi `hermes gateway run` (chạy từ profile mặc định, gateway này phục vụ luôn `cskh`).

## Bảo mật

- Bot không có tool nào (terminal, file, web...). Chỉ trả lời văn bản từ FAQ; câu ngoài phạm vi hoặc chưa có nội dung thì hướng người dùng liên hệ quản trị viên, kèm nhãn `[Chuyển: CS|Product|Pháp lý]`.
- Chính sách truy cập là `allowlist`: chỉ ID trong `allowed_users` được nhắn. Danh sách này cũng cấp quyền cho lệnh quản trị của plugin (`/kick`, `/warn`...), nên chỉ đưa tài khoản admin vào.
- Chỉ đưa vào FAQ nội dung được phép công khai.
- Không chia sẻ key 9router hay token Zalo. Key đã từng xuất hiện trong cuộc trò chuyện thì nên tạo lại.

## Giới hạn đã biết

- Bot không báo được cho admin hay đơn vị phụ trách; nó chỉ gắn nhãn `[Chuyển: ...]` trong câu trả lời.
- FAQ mới chỉ có 1 mục trả lời được (FAQ-006); 5 mục còn lại chờ nội dung (xem cuối [SOUL.md](hermes/cskh/SOUL.md)).
- Chưa có lọc theo tag (`requiredTags`) và từ khóa nhạy cảm (`escalateKeywords`) như bot cũ; hiện chỉ dựa vào persona.
- Bot chỉ chạy khi cả 9router và gateway đang chạy.
