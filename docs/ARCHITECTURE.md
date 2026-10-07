# Cấu trúc dự án ZaloLab

Sơ đồ Mermaid, xem được trực tiếp trong VS Code (xem trước Markdown có hỗ trợ Mermaid) hoặc GitHub.

## 1. Luồng hoạt động của bot

```mermaid
flowchart LR
    U["Người dùng Zalo<br/>(ID trong allowed_users)"] -->|tin nhắn| Z["Zalo Bot API<br/>(chính thức)"]
    Z --> P["Plugin zalo-platform<br/>ZALO_BOT_TOKEN"]
    P --> H["Hermes Gateway<br/>profile cskh<br/>(không có tool nào)"]
    S["SOUL.md<br/>persona + guardrails + FAQ"] -.nạp.-> H
    H -->|"OpenAI-compatible<br/>localhost:20128/v1"| R["9router<br/>combo ZaloLab"]
    R -->|OAuth| C["Claude / ChatGPT"]
    C --> R --> H --> P --> Z --> U
```

## 2. Quy trình bot xử lý một câu hỏi

```mermaid
flowchart TD
    A[Tin nhắn đến] --> B{"Người gửi có trong<br/>allowed_users?"}
    B -- Không --> X[Bỏ qua]
    B -- Có --> C{Tiêm lệnh /<br/>yêu cầu ngoài FAQ?}
    C -- Có --> D["Trả câu từ chối cố định"]
    C -- Không --> E{Khớp mục nào?}
    E -- "FAQ đã sẵn sàng<br/>(FAQ-006)" --> F[Trả lời theo FAQ]
    E -- "FAQ chưa có nội dung /<br/>pháp lý / khiếu nại /<br/>không khớp" --> G["Mẫu chuyển tiếp<br/>+ nhãn [Chuyển: CS | Product | Pháp lý]"]
    G --> H2[Người dùng tự liên hệ quản trị viên]
```

## 3. Cấu trúc thư mục

```mermaid
flowchart TD
    ROOT["ZaloLab/"] --> README["README.md<br/>tổng quan, vận hành"]
    ROOT --> CLAUDE["CLAUDE.md<br/>hướng dẫn cho Claude Code"]
    ROOT --> GI[".gitignore"]
    ROOT --> HERMES["hermes/cskh/"]
    ROOT --> DOCS["docs/"]
    HERMES --> SOUL["SOUL.md<br/>nguồn chuẩn: persona, guardrails, FAQ"]
    HERMES --> CFG["config.zalo.example.yaml<br/>mẫu: tắt tool + allowlist"]
    DOCS --> SETUP["SETUP-VPS.md<br/>cài 9router, Hermes, plugin, chạy tự động"]
    DOCS --> GUARD["GUARDRAILS.md<br/>quy tắc trả lời, chuyển người"]
    DOCS --> ARCH["ARCHITECTURE.md<br/>(file này)"]
```

## 4. Thành phần chạy ngoài repo (WSL Ubuntu)

```mermaid
flowchart LR
    subgraph REPO["Repo (Git)"]
        SOUL2["hermes/cskh/SOUL.md"]
        CFG2["config.zalo.example.yaml"]
    end
    subgraph WSL["WSL Ubuntu trên máy Windows"]
        subgraph PROFILE["~/.hermes/profiles/cskh/"]
            SOUL3["SOUL.md (bản triển khai)"]
            CONF["config.yaml<br/>platform_toolsets: []<br/>platforms.zalo"]
            ENV[".env<br/>ZALO_BOT_TOKEN"]
            LOGS["logs/gateway.log, agent.log"]
        end
        GW["hermes gateway run<br/>(từ profile mặc định)"]
        NR["9router -H 127.0.0.1 -n -l"]
    end
    subgraph WIN["Windows Task Scheduler"]
        T1["Task 9router (khi đăng nhập)"]
        T2["Task HermesGateway<br/>(chưa đăng ký)"]
    end
    SOUL2 -- "chép tay + restart gateway" --> SOUL3
    CFG2 -. "mẫu, điền vào" .-> CONF
    T1 --> NR
    T2 -.-> GW
    GW --> PROFILE
    GW --> NR
```

## 5. Lịch sử và việc còn dở

| Hạng mục | Trạng thái |
|---|---|
| Bot cũ `bot.js` (`zca-js`, nhận file, trích xuất `.xlsx`) | Đã gỡ, bản cuối ở commit `03e5478` |
| FAQ-006 | Có câu trả lời |
| FAQ-001 đến 004, câu "AI luật ban hành dựa trên công văn nào" | Chỉ chuyển tiếp, chờ nội dung duyệt |
| FAQ-005 | Đã hủy, không dùng |
| Báo admin tự động khi chuyển tiếp | Chưa có (bot không có tool) |
| Task `HermesGateway` tự chạy khi đăng nhập | Chưa đăng ký |
| Hành vi trong nhóm Zalo | Chưa kiểm chứng |
