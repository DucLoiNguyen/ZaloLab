# Setup Hermes Agent + 9router trên VPS

Các lệnh dưới đây lấy từ tài liệu công khai của từng dự án; **kiểm tra lại với tài liệu chính thức** trước khi chạy vì phiên bản có thể thay đổi.
- 9router: https://github.com/decolua/9router (docs: https://docs.9router.com/)
- Hermes Agent (Nous Research): script cài đặt chính thức `https://hermes-agent.nousresearch.com/install.sh`

Kiến trúc đích:

```
Zalo / Teams / Fanpage ──► (bridge: bot.js hoặc Hermes gateway) ──► Hermes ──► 9router ──► ChatGPT Plus / Claude
```

## 1. Chuẩn bị VPS

- Ubuntu 22.04+ , ≥ 2 vCPU / 4 GB RAM, người dùng không phải root, firewall chỉ mở SSH (và 443 nếu cần webhook).
- Cài Node.js LTS (cho 9router và bot.js).

## 2. Cài 9router

```bash
npm install -g 9router
9router            # mặc định mở dashboard + endpoint OpenAI-compatible tại http://localhost:20128/v1
```

- Giữ 9router **chỉ lắng nghe localhost** (không mở cổng 20128 ra internet). Cần truy cập dashboard từ xa thì dùng SSH tunnel: `ssh -L 20128:localhost:20128 user@vps`.
- Chạy nền bằng systemd (`/etc/systemd/system/9router.service`, `ExecStart=$(which 9router)`, `Restart=always`) hoặc pm2.

## 3. Thêm tài khoản ChatGPT Plus / Claude vào 9router

1. Mở dashboard 9router (qua SSH tunnel) → mục Providers.
2. Chọn provider kiểu subscription (Codex/ChatGPT hoặc Claude Code) → đăng nhập OAuth bằng tài khoản của bạn.
3. Tạo một API key trong dashboard cho Hermes / bot dùng; tạo **một combo/model alias** (ví dụ `zalolab-chat`) với thứ tự fallback mong muốn.
4. Thử: 
   ```bash
   curl http://localhost:20128/v1/chat/completions \
     -H "Authorization: Bearer <key-9router>" -H "content-type: application/json" \
     -d '{"model":"zalolab-chat","messages":[{"role":"user","content":"ping"}]}'
   ```

Lưu ý: dùng gói subscription cá nhân qua router có thể vi phạm điều khoản của nhà cung cấp hoặc bị giới hạn lưu lượng; chatbot chăm sóc khách hàng lưu lượng cao nên cân nhắc API key chính thức.

## 4. Cài Hermes Agent và trỏ về 9router

```bash
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash
hermes setup       # chọn provider "custom / OpenAI-compatible"
```

Trong wizard (hoặc file cấu hình của Hermes) nhập:
- Base URL: `http://localhost:20128/v1`
- API key: key tạo ở bước 3
- Model: `zalolab-chat` (alias trong 9router). Hermes cần model có context ≥ 64k token.

Chạy gateway nhắn tin 24/7: `hermes gateway setup`, rồi chạy dưới systemd.

## 5. Danh sách kênh cần chuẩn bị

| Kênh | Việc cần chuẩn bị | Cách nối |
|---|---|---|
| Zalo | Nick phụ, ID nhóm/người (chế độ `discoveryMode` của bot.js) | `bot.js` (zca-js, không chính thức, có rủi ro khóa nick) |
| Microsoft Teams | Đăng ký Azure Bot / app registration, tenant ID, App ID/secret | Hermes gateway nếu có hỗ trợ Teams, nếu không viết bridge riêng |
| Facebook Fanpage | Meta app, Page access token, webhook (cần HTTPS + domain), quyền `pages_messaging` | Webhook → Hermes/bridge |

Hermes gateway hỗ trợ sẵn một số nền tảng (Telegram, Discord, Slack, WhatsApp, Signal, email…); **Zalo, Teams, Fanpage cần kiểm tra trong tài liệu hiện hành**. Phương án chắc chắn: giữ mỗi kênh là một bridge mỏng (như `bot.js`) gọi chung một endpoint OpenAI-compatible.

## 6. Dùng 9router từ bot.js ngay (chưa cần Hermes)

Trong [config.json](../config.json):

```json
"llm": { "baseUrl": "http://localhost:20128/v1", "apiKeyEnv": "NINEROUTER_API_KEY", "model": "zalolab-chat" }
```

```bash
export NINEROUTER_API_KEY=<key>   # PowerShell: $env:NINEROUTER_API_KEY="<key>"
node bot.js
```

Nếu `llm.baseUrl` rỗng, bot quay lại gọi trực tiếp API Anthropic bằng `ANTHROPIC_API_KEY`.

Guardrails: xem [GUARDRAILS.md](GUARDRAILS.md).
