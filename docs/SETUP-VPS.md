# Cài đặt Hermes Agent + 9router + plugin Zalo (WSL Ubuntu)

Đây là các bước đã chạy thực tế trên Windows 11 + WSL2 Ubuntu 24.04. Trên VPS Ubuntu thì làm tương tự, bỏ phần Task Scheduler và dùng systemd.

- 9router: https://github.com/decolua/9router
- Hermes Agent: https://hermes-agent.nousresearch.com
- Plugin Zalo: https://github.com/moken627-hub/hermes-plugin-zalo (catalog Hermes, ghim SHA)

```
Zalo ──► zalo-platform ──► Hermes (profile cskh) ──► 9router ──► Claude
```

## 1. WSL và Node

```powershell
wsl --install -d Ubuntu-24.04
```

Trong Ubuntu, cài Node bằng nvm. **Không dùng Node của Windows** (WSL mặc định thấy `npm` Windows qua `/mnt/...` và sẽ lỗi):

```bash
curl -fsSL https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
source ~/.nvm/nvm.sh && nvm install --lts
sudo apt-get install -y libatomic1   # Hermes cần thư viện này cho Node của nó
```

## 2. 9router

```bash
npm install -g --allow-scripts=9router 9router
9router -H 127.0.0.1 -n -l --skip-update
```

- Bind `127.0.0.1`: mặc định nó bind `0.0.0.0` và lộ ra LAN.
- Không có `-n -l` thì nó thoát ngay khi chạy không có terminal.
- Mở `http://localhost:20128` → Providers → đăng nhập OAuth ChatGPT Plus hoặc Claude, tạo API key và combo `ZaloLab`.
- Thử: `curl http://localhost:20128/v1/models -H "Authorization: Bearer <key>"`.
- Provider `cc/` (Claude Code) tự chèn system prompt riêng, khoảng 2000 token mỗi lượt.
- Dùng gói subscription cá nhân qua router có thể vi phạm điều khoản nhà cung cấp; chatbot lưu lượng cao nên cân nhắc API key chính thức.

## 3. Hermes

```bash
curl -fsSL https://hermes-agent.nousresearch.com/install.sh -o ~/hermes-install.sh
less ~/hermes-install.sh          # đọc trước khi chạy
bash ~/hermes-install.sh --non-interactive
```

Trỏ về 9router (profile mặc định, rồi nhân bản sang profile `cskh`):

```bash
hermes config set model.provider custom
hermes config set model.base_url http://localhost:20128/v1
hermes config set model.default ZaloLab
hermes config set model.api_key <key-9router>
hermes profile create cskh --clone --description "Chatbot cham soc khach hang"
```

`provider: custom` không đọc `OPENAI_API_KEY`; key phải ở `model.api_key`.

## 4. Persona và khóa tool

```bash
cp hermes/cskh/SOUL.md ~/.hermes/profiles/cskh/SOUL.md
```

Rồi thêm phần `platform_toolsets` ở [hermes/cskh/config.zalo.example.yaml](../hermes/cskh/config.zalo.example.yaml) vào `~/.hermes/profiles/cskh/config.yaml`, và đặt **mọi** nền tảng thành `[]`, kể cả `zalo`. Kiểm tra: `hermes -p cskh tools list --platform zalo` phải không còn tool nào `enabled`.

## 5. Plugin Zalo

1. Tạo bot tại https://bot.zaloplatforms.com và lấy token `numeric_id:secret`.
2. Cài và bật:
   ```bash
   hermes -p cskh plugins install zalo-platform
   hermes -p cskh plugins enable zalo-platform
   echo 'ZALO_BOT_TOKEN=<token>' >> ~/.hermes/profiles/cskh/.env
   chmod 600 ~/.hermes/profiles/cskh/.env
   ```
3. Thêm khối `platforms.zalo` ở [hermes/cskh/config.zalo.example.yaml](../hermes/cskh/config.zalo.example.yaml) vào `config.yaml` của profile `cskh`. Chính sách truy cập và allowlist **phải đặt ở đây**, không phải trong `.env`, vì gateway chung không nạp `.env` của profile.
4. Chạy gateway từ profile **mặc định** (Hermes chỉ cho một gateway cho cả máy):
   ```bash
   hermes gateway run
   ```
5. Nhắn thử cho bot. Log: `~/.hermes/profiles/cskh/logs/gateway.log`.

Chính sách `dm_policy`: `allowlist` (chỉ ID trong `allowed_users`), `open` (ai cũng nhắn được), `pairing` (captcha, nhưng trạng thái đã qua captcha mất khi restart gateway).

## 6. Chạy tự động (Windows)

Hermes gateway và 9router cần một tiến trình giữ WSL sống. Đăng ký hai task Task Scheduler chạy khi đăng nhập:

```powershell
$s = New-ScheduledTaskSettingsSet -ExecutionTimeLimit ([TimeSpan]::Zero) -RestartCount 5 -RestartInterval (New-TimeSpan -Minutes 1) -StartWhenAvailable -AllowStartIfOnBatteries -DontStopIfGoingOnBatteries

$a = New-ScheduledTaskAction -Execute "wsl.exe" -Argument '-d Ubuntu-24.04 -e bash -lc "source ~/.nvm/nvm.sh; exec 9router -H 127.0.0.1 -n -l --skip-update"'
Register-ScheduledTask -TaskName "9router" -Action $a -Trigger (New-ScheduledTaskTrigger -AtLogOn -User $env:USERNAME) -Settings $s

$a = New-ScheduledTaskAction -Execute "wsl.exe" -Argument '-d Ubuntu-24.04 -e bash -lc "export PATH=$HOME/.local/bin:$PATH; exec hermes gateway run"'
$t = New-ScheduledTaskTrigger -AtLogOn -User $env:USERNAME; $t.Delay = "PT30S"
Register-ScheduledTask -TaskName "HermesGateway" -Action $a -Trigger $t -Settings $s
```

Gateway trễ 30 giây để 9router lên trước. Dừng một task: `Stop-ScheduledTask <tên>`; gỡ: `Unregister-ScheduledTask <tên> -Confirm:$false`.

Guardrails: xem [GUARDRAILS.md](GUARDRAILS.md).
