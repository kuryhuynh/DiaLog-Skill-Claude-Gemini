# Minecraft PaperMC Native Dialogs Skill (Claude & Gemini)

[![PaperMC](https://img.shields.io/badge/PaperMC-1.21%2B-blue?style=for-the-badge&logo=paper)](https://papermc.io)
[![Folia](https://img.shields.io/badge/Folia-Supported-brightgreen?style=for-the-badge)](https://papermc.io/software/folia)
[![Claude](https://img.shields.io/badge/Anthropic-Claude%20Code-purple?style=for-the-badge&logo=anthropic)](https://claude.ai)
[![Gemini](https://img.shields.io/badge/Google-Antigravity%20%2F%20Gemini-orange?style=for-the-badge&logo=google)](https://deepmind.google)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg?style=for-the-badge)](LICENSE)

Bộ tài liệu hướng dẫn và **Skill chuẩn Production** dành cho **Claude** (Claude Code, Claude Desktop) và **Gemini** (Google Antigravity, Gemini CLI) chuyên sâu về phát triển Minecraft Plugin sử dụng **Paper Native Dialogs API** (PaperMC 1.21+ / Minecraft 1.21.4 - 1.21.7+).

---

## 🎯 Vấn Đề Thực Chiến Được Giải Quyết Triệt Để

Khi phát triển giao diện Native Dialogs trên Minecraft 1.21+, hầu hết lập trình viên và AI thông thường đều gặp phải các lỗi nghiêm trọng sau:

| Vấn đề thường gặp | Nguyên nhân gốc | Giải pháp chuẩn trong Skill này |
| :--- | :--- | :--- |
| **Chuột bị văng về giữa màn hình** | Gọi `player.closeDialog()` hoặc đặt `DialogAfterAction.CLOSE`, khiến client đặt `Screen = null` rồi mở lại. | **Zero-Mouse-Reset**: Bắt buộc dùng `afterAction(DialogAfterAction.NONE)` và gọi thẳng `player.showDialog(...)` đè in-place. |
| **Giật màn hình khi Toggle BẬT/TẮT** | Delay tick `runTaskLater(1)` hoặc close/re-open gây nháy thế giới 3D 1 frame. | Cập nhật nguyên tử ngay trên **Entity Scheduler**, giữ nguyên vị trí nút trong ma trận giao diện. |
| **Drop TPS nghiêm trọng khi spam click** | Chạy logic I/O (Database, Yaml) hoặc gọi `Bukkit.getScheduler()` trực tiếp trong click callback. | **3 Lớp Phòng Thủ**: Rate Limiter 150ms + Bộ nhớ đệm RAM + Điều phối luồng chuẩn **Folia/Paper** (`SchedulerUtil`). |
| **Nghẽn mạng (Network Eater / Ping Spike)** | Packet flood do spam gửi Dialog mỗi tick hoặc rebuild toàn bộ Component không kiểm soát. | **Dirty Checking** (chỉ gửi khi state đổi), giới hạn số lần dùng `ClickCallback.Options.uses(1)`. |
| **Màu sắc lòe loẹt, lỗi font** | Sử dụng bảng màu rainbow (`&e`, `&a`, `&c`) và text tiếng Việt không escape. | **White Typography**: Chữ trắng tối giản phong cách Vanilla thanh lịch, mã hóa Unicode `\uXXXX` chống lỗi font 100%. |
| **Lag server do Player Head** | Gọi đồng bộ `getOfflinePlayer(uuid)` để lấy skin Mojang trên Main Thread. | **Zero-Lag Skin Loading**: Trích xuất textures Base64 từ online profile hoặc cache RAM, không bao giờ block main thread. |

---

## 📂 Cấu Trúc Thư Mục Repository

```text
DiaLog-Skill-Claude-Gemini/
├── .agents/
│   └── skills/
│       └── paper-dialog-plugin/
│           └── SKILL.md            # Skill chuẩn cho Google Antigravity & Gemini CLI
├── .claude/
│   ├── skills/
│   │   └── paper-dialog-plugin/
│   │       └── SKILL.md            # Skill định dạng tối ưu cho Anthropic Claude
├── skills/
│   └── paper-dialog-plugin/
│       └── SKILL.md                # Bản Universal Skill dùng chung
├── CLAUDE.md                       # File chỉ dẫn tự động cho Claude Code CLI
├── LICENSE                         # Giấy phép nguồn mở MIT
└── README.md                       # Hướng dẫn sử dụng chi tiết
```

---

## 🚀 Hướng Dẫn Cài Đặt & Sử Dụng

### 1. Dành cho Anthropic Claude (Claude Code / Claude Desktop)

#### Cách 1: Sử dụng với Claude Code (CLI)
Đặt file [CLAUDE.md](file:///CLAUDE.md) tại thư mục gốc dự án của bạn. Mỗi khi khởi động session Claude Code, Claude sẽ tự động nạp toàn bộ quy chuẩn Dialog:
```bash
claude
```

#### Cách 2: Sử dụng với Claude Projects / Claude Desktop
Copy toàn bộ nội dung của file [.claude/skills/paper-dialog-plugin/SKILL.md](file:///.claude/skills/paper-dialog-plugin/SKILL.md) dán vào ô **Custom Instructions** hoặc **Project Knowledge** của dự án Claude.

---

### 2. Dành cho Google Gemini (Google Antigravity)

#### Cách 1: Cài đặt cho Workspace hiện tại
Clone hoặc copy thư mục `.agents/skills/paper-dialog-plugin/` vào thư mục gốc của repository dự án Minecraft của bạn:
```bash
git clone https://github.com/kuryhuynh/DiaLog-Skill-Claude-Gemini.git
cp -r DiaLog-Skill-Claude-Gemini/.agents /path/to/your/minecraft-plugin/
```

#### Cách 2: Cài đặt Toàn Cầu (Global) cho mọi dự án
Copy file `SKILL.md` vào thư mục global của Antigravity:
- **Windows**: `%USERPROFILE%\.gemini\config\skills\paper-dialog-plugin\SKILL.md`
- **Linux/macOS**: `~/.gemini/config/skills/paper-dialog-plugin/SKILL.md`

---

## 💡 Code Mẫu Tối Ưu Tóm Tắt

### Khởi tạo Dialog BẬT/TẮT Mượt Mà (Không Giật - Không Reset Chuột)

```java
import io.papermc.paper.dialog.Dialog;
import io.papermc.paper.registry.data.dialog.ActionButton;
import io.papermc.paper.registry.data.dialog.DialogBase;
import io.papermc.paper.registry.data.dialog.action.DialogAction;
import io.papermc.paper.registry.data.dialog.body.DialogBody;
import io.papermc.paper.registry.data.dialog.type.DialogType;
import net.kyori.adventure.text.Component;
import net.kyori.adventure.text.format.NamedTextColor;
import net.kyori.adventure.text.format.TextDecoration;

// BẮT BUỘC: afterAction = NONE để client không đóng màn hình khi click
Dialog dialog = Dialog.create(b -> b.empty()
    .base(DialogBase.builder(Component.text("C\u00e0i \u0110\u1eb7t", NamedTextColor.WHITE, TextDecoration.BOLD))
        .canCloseWithEscape(true)
        .pause(false)
        .afterAction(DialogBase.DialogAfterAction.NONE)
        .body(List.of(DialogBody.plainMessage(Component.text("T\u00f9y ch\u1ec9nh t\u00ednh n\u0103ng:", NamedTextColor.WHITE))))
        .build()
    )
    .type(DialogType.multiAction(buttons, exitButton, 2))
);

// Đè trực tiếp trên Entity Scheduler -> Con trỏ chuột đứng im tại chỗ!
player.showDialog(dialog);
```

### Nhúng Sprite Vật Phẩm & 2D Player Head

```java
// Nhúng Sprite 2D Vật Phẩm Minecraft
Component swordSprite = IconAndHeadCatalog.sprite("minecraft:diamond_sword");

// Nhúng Khuôn Mặt 2D Người Chơi Có Lớp Mũ Ngoài (.hat(true))
Component playerHead = IconAndHeadCatalog.playerHead(player.getUniqueId(), player.getName(), null);

Component buttonLabel = Component.empty()
    .append(playerHead)
    .append(Component.text(" "))
    .append(Component.text(player.getName(), NamedTextColor.WHITE));
```

---

## 📜 Tài Liệu Tham Khảo

- [PaperMC Official Dialogs Documentation](https://docs.papermc.io/paper/dev/dialogs/)
- [Adventure Text Component Documentation](https://docs.advntr.dev/)
- [Folia Threading & Regionized Server Model](https://docs.papermc.io/folia/reference/overview)

---

## 📄 License

Dự án phát hành theo giấy phép [MIT License](LICENSE). Đóng góp và pull request luôn được hoan nghênh!
