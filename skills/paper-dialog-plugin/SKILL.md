---
name: paper-dialog-plugin
description: >-
  Chuyên gia thiết kế, tối ưu và lập trình Minecraft Dialog Plugin (PaperMC 1.21+) theo chuẩn Paper Native Dialogs API.
  Áp dụng kiến trúc chống giật lag (zero-flicker toggle), triệt tiêu ngốn network/packet spam, chống drop TPS (Folia-ready, debounce),
  khắc phục triệt để lỗi reset chuột về giữa màn hình, và chuẩn hóa phong cách giao diện tối giản chữ trắng (white typography).
---

# PaperMC Native Dialogs Plugin Development Skill

Tài liệu hướng dẫn và bộ quy chuẩn toàn diện để phát triển các plugin Minecraft giao diện **Native Dialogs** (theo chuẩn PaperMC 1.21+ Dialog API). Bộ quy chuẩn này được đúc kết từ thực chiến phát triển các hệ thống lớn (`CrystalCore`, `SokiMCPauseMenu`), giải quyết triệt để tất cả các vấn đề nhức nhối: **drop TPS khi spam click, nghẽn mạng do packet flood, giật màn hình khi toggle BẬT/TẮT, lỗi chuột bị văng/reset về giữa màn hình**, và chuẩn hóa phong cách **giao diện tối giản chữ trắng (clean white typography)**.

---

## 📑 Mục Lục
1. [Khái quát Paper Native Dialogs API](#1-khái-quát-paper-native-dialogs-api)
2. [Triệt tiêu Lỗi Reset Chuột về Giữa Màn Hình (Zero-Mouse-Reset)](#2-triệt-tiêu-lỗi-reset-chuột-về-giữa-màn-hình-zero-mouse-reset)
3. [Tối ưu Action BẬT/TẮT Mượt Mà & Không Giật Màn Hình](#3-tối-ưu-action-bậttắt-mượt-mà--không-giật-màn-hình)
4. [Chống Drop TPS & Đa luồng Folia/Paper Native](#4-chống-drop-tps--đa-luồng-foliapaper-native)
5. [Chống Ngốn Băng Thông & Packet Flood (Anti-Network Eater)](#5-chống-ngốn-băng-thông--packet-flood-anti-network-eater)
6. [Quy chuẩn Thẩm mỹ Giao diện: Chữ Trắng Tối Giản (White Typography)](#6-quy-chuẩn-thẩm-mỹ-giao-diện-chữ-trắng-tối-giản-white-typography)
7. [Tương thích Phiên bản & Fallback Client Cũ](#7-tương-thích-phiên-bản--fallback-client-cũ)
8. [Tích Hợp Icon Sprites & Player Head (Đầu Người Chơi)](#8-tích-hợp-icon-sprites--player-head-đầu-người-chơi)
9. [Bộ Code Mẫu Chuẩn Production (Production Codebase Template)](#9-bộ-code-mẫu-chuẩn-production-production-codebase-template)
10. [Bảng Kiểm Thử Checklist (Pre-Flight Verification)](#10-bảng-kiểm-thử-checklist-pre-flight-verification)

---

## 1. Khái quát Paper Native Dialogs API

Native Dialog là cơ chế giao diện modal máy chủ gửi trực tiếp về máy trạm (client-side modal) của Minecraft 1.21+. Khác với Inventory GUI truyền thống (mượn hòm đồ ảo 9x6), Dialog hiển thị dưới dạng cửa sổ giao diện gốc, có hỗ trợ:
- Các loại layout: `DialogType.notice`, `DialogType.confirmation`, `DialogType.multiAction(List<ActionButton> actions, ActionButton exit, int columns)`.
- Các nút hành động: `ActionButton` gắn callback `DialogAction.customClick(...)`.
- Nội dung thân hộp thoại: `DialogBody.plainMessage(Component)`, `DialogBody.item(ItemStack)`.
- Thuộc tính cơ sở: `DialogBase.builder(title).canCloseWithEscape(true).pause(false).afterAction(action)...`.

---

## 2. Triệt tiêu Lỗi Reset Chuột về Giữa Màn Hình (Zero-Mouse-Reset)

### 2.1. Nguyên nhân kỹ thuật gốc trong Client Minecraft
Trong Minecraft Java Edition, con trỏ chuột được điều khiển qua `MouseHandler`:
1. Khi không có menu nào mở (`Screen == null`), chuột bị khóa (grabbed) ở tâm màn hình để xoay góc nhìn camera.
2. Khi một cửa sổ mở lên lần đầu (`setScreen(screen)` khi trước đó `screen == null`), game sẽ giải phóng chuột và **đặt vị trí con trỏ vào chính giữa màn hình**: `(width / 2, height / 2)`.
3. **Lỗi kinh điển của lập trình viên**:
   - Khi người chơi click vào một nút (ví dụ chuyển trang hoặc Bật/Tắt setting), plugin gọi `player.closeDialog()`, sau đó gọi `openDialog(player)`.
   - Lệnh `closeDialog()` gửi packet đóng màn hình (`ClientboundCloseScreenPacket`) -> Client set `screen = null` -> Chuột bị khóa lại vào tâm!
   - Ngay tick sau khi `openDialog()` gửi xuống -> Client mở màn hình mới -> Chuột bị mở khóa và **reset ngay lập tức về chính giữa màn hình**!
   - Thậm chí nếu dùng cấu hình `.afterAction(DialogBase.DialogAfterAction.CLOSE)` trên nút bấm, client sẽ tự động đóng dialog trước khi server kịp gửi dialog mới!

### 2.2. 4 Quy Tắc Vàng Giữ Chuột Tại Chỗ (Stay-In-Place)
1. **QUY TẮC 1: KHÔNG BAO GIỜ gọi `player.closeDialog()` khi cập nhật trạng thái.**
   - Để làm mới dialog hoặc toggle button, hãy gọi thẳng `player.showDialog(newDialog)`. Client Minecraft sẽ thay thế nội dung màn hình đang mở mà KHÔNG hề giải phóng hay dịch chuyển con trỏ chuột!
2. **QUY TẮC 2: BẮT BUỘC đặt `.afterAction(DialogBase.DialogAfterAction.NONE)`.**
   - Áp dụng trên `DialogBase` cho tất cả các dialog có nút tương tác (toggle, next page, tab switch). Trừ nút `Đóng/Exit` duy nhất mới dùng `DialogAfterAction.CLOSE` hoặc gọi `closeDialog()`.
3. **QUY TẮC 3: Cập nhật đồng bộ trên Entity Scheduler không trễ tick.**
   - Không được dùng `runTaskLater(..., 1L)` hay `runTaskLater(..., 2L)`. Việc tạo độ trễ 1-2 ticks sẽ khiến màn hình bị "treo" trạng thái cũ hoặc nháy khung hình. Hãy cập nhật ngay lập tức bên trong tác vụ Entity Scheduler của người chơi.
4. **QUY TẮC 4: Giữ nguyên vị trí nút bấm trong ma trận giao diện.**
   - Trong `DialogType.multiAction`, giữ nguyên tổng số nút và thứ tự vị trí (`index`) của các button. Nếu nút toggle nằm ở vị trí cột 1 dòng 2, khi bấm vào nó chỉ thay đổi nhãn/icon, không được đảo lộn vị trí các nút khác, con trỏ chuột của người chơi sẽ nằm im ngay trên nút đó để họ có thể click liên tục mà không trượt chuột.

---

## 3. Tối ưu Action BẬT/TẮT Mượt Mà & Không Giật Màn Hình

### 3.1. Hiện tượng "Giật Màn Hình" (Screen Stutter/Jitter)
- Màn hình bị chớp đen hoặc nháy thế giới 3D trong 1 frame (1/60s).
- Text bị rung hoặc co giật kích thước do đổi font chữ đột ngột.
- Âm thanh click bị trễ hoặc lặp đè lên nhau gây xé tai.

### 3.2. Giải pháp chuyển đổi State nguyên tử (Atomic State Transition)
- **Đổi nhãn tinh tế không đổi độ rộng nút**: Cố định độ rộng nút (`.width(...)`) hoặc sử dụng các biểu tượng có độ dài tương đương:
  - Bật: `[ BẬT ]` / `[ ON ]` / `[ ✔ ]` / `● BẬT`
  - Tắt: `[ TẮT ]` / `[ OFF ]` / `[ ✖ ]` / `○ TẮT`
- **Âm thanh phản hồi trực quan (Audio Feedback)**:
  - Khi **BẬT**: `p.playSound(p.getLocation(), Sound.BLOCK_NOTE_BLOCK_PLING, 1.0f, 2.0f);` (âm trong trẻo nốt cao).
  - Khi **TẮT**: `p.playSound(p.getLocation(), Sound.UI_BUTTON_CLICK, 0.8f, 0.8f);` (âm trầm nhẹ).
  - Tránh phát các âm thanh hiệu ứng nặng (nổ, sấm sét, wither spawn).

---

## 4. Chống Drop TPS & Đa luồng Folia/Paper Native

### 4.1. Hiểm họa gây Drop TPS từ Dialog
- **Click Callbacks chạy không kiểm soát**: Khi một người chơi dùng macro hoặc auto-clicker spam vào nút Dialog với tốc độ 20-50 clicks/giây:
  - Nếu mỗi click kích hoạt một phép tính database (SQLite/MySQL query), một lệnh đọc file YAML, hay một vòng lặp duyệt toàn bộ người chơi server để tính top leaderboard, Main Thread sẽ bị nghẽn (tick loop overload), đẩy MSPT lên > 100ms và TPS rớt từ 20.0 xuống 5.0.
  - Trên **Folia**, nếu gọi trực tiếp các API Bukkit truyền thống bên ngoài Entity Region của người chơi, server sẽ ném ngoại lệ `IllegalStateException` hoặc gây deadlock giữa các luồng tick region.

### 4.2. Kiến trúc 3 Lớp Chống Drop TPS

```
[ Player Click Event ]
         │
         ▼
[ Lớp 1: Anti-Spam Debounce Guard (Atomic / In-Memory, < 1ms) ]
   ├── Spam < 150ms? ──> Bỏ qua ngay lập tức (Silent drop)
   └── Hợp lệ ──> Đi tiếp
         │
         ▼
[ Lớp 2: Folia/Paper Entity Scheduler Dispatch ]
   ├── SchedulerUtil.runForEntity(player, task)
   └── Đảm bảo chạy đúng Region Thread sở tại của Player
         │
         ▼
[ Lớp 3: Zero-IO State Mutator & Pre-Cached Reader ]
   ├── Đọc/Ghi dữ liệu từ RAM Cache (ConcurrentHashMap / PlayerData)
   ├── Bất đồng bộ hóa (Async) tác vụ lưu Database vào Background
   └── Gửi lại Dialog mới cho Player
```

#### Code mẫu Debounce Cooldown:
```java
public class DialogRateLimiter {
    private static final Map<UUID, Long> LAST_CLICK = new ConcurrentHashMap<>();
    private static final long COOLDOWN_MS = 150; // Tối thiểu 150ms giữa 2 lần tương tác

    public static boolean checkAndRecord(UUID uuid) {
        long now = System.currentTimeMillis();
        Long last = LAST_CLICK.get(uuid);
        if (last != null && (now - last) < COOLDOWN_MS) {
            return false; // Spam click -> Chặn
        }
        LAST_CLICK.put(uuid, now);
        return true;
    }

    public static void clean(UUID uuid) {
        LAST_CLICK.remove(uuid);
    }
}
```

---

## 5. Chống Ngốn Băng Thông & Packet Flood (Anti-Network Eater)

### 5.1. Vấn đề "Network Performance Eater"
Mỗi Dialog của Paper khi gửi cho Client sẽ serialize toàn bộ cây JSON của các Adventure Component (tiêu đề, body, từng button, hover event, click event callback token).
Nếu plugin:
1. Gửi lại Dialog packet liên tục theo chu kỳ tick (`runTaskTimer`).
2. Gửi lại Dialog packet dù người chơi bấm vào trạng thái không thay đổi.
3. Không dọn dẹp các `ClickCallback`, để token tích tụ trong bộ nhớ máy chủ.
Hậu quả: Đường truyền bị nghẽn (buffer saturation), người chơi bị tăng ping đột ngột từ 30ms lên 500ms, mạng chập chờn và client có thể bị `Timed Out`.

### 5.2. Các Quy Tắc Bảo Vệ Băng Thông
1. **Giới hạn số lần dùng Callback (`uses`) & Thời gian sống (`lifetime`)**:
   ```java
   ClickCallback.Options options = ClickCallback.Options.builder()
       .uses(1) // Hoặc số lần cụ thể
       .lifetime(Duration.ofMinutes(5))
       .build();
   ```
2. **Dirty Checking (Kiểm tra thay đổi trước khi gửi packet)**:
   - Trước khi gọi `player.showDialog(...)`, kiểm tra xem trạng thái mới có khác trạng thái đang hiển thị hay không. Nếu người chơi click chọn một tùy chọn mà tùy chọn đó đã đang được chọn rồi, chỉ cần play âm thanh nhỏ hoặc bỏ qua, **tuyệt đối không rebuild và resend toàn bộ Dialog**.
3. **Không bao giờ chạy bộ đếm thời gian (Live Timer/Countdown) bằng cách gửi lại Dialog mỗi giây**:
   - Nếu muốn đếm thời gian trong dialog, chỉ dùng cho các màn hình tĩnh hoặc chấp nhận tần số tối đa 1-2 giây một lần, không bao giờ update ở tần số tick (50ms).

---

## 6. Quy chuẩn Thẩm mỹ Giao diện: Chữ Trắng Tối Giản (White Typography)

Theo yêu cầu nghiêm ngặt: **"dialog khong dung mau, chu su dung trang"**, giao diện Dialog cần loại bỏ hoàn toàn các màu mè lòe loẹt (vàng kim, đỏ chói, xanh neon, hồng tím), thiết lập phong cách tối giản thanh lịch (clean minimalist vanilla look):

### 6.1. Bảng màu Chuẩn (Monochrome Palette)

| Phần tử Giao diện | Mã Màu / Adventure Color | Phong cách / Trang trí |
| :--- | :--- | :--- |
| **Tiêu đề Dialog** | `NamedTextColor.WHITE` (`#FFFFFF`) | `TextDecoration.BOLD` |
| **Thân bài (Body text)** | `NamedTextColor.WHITE` (`#FFFFFF`) | Plain, không nghiêng |
| **Ghi chú / Hướng dẫn phụ** | `NamedTextColor.GRAY` (`#AAAAAA`) | Nhẹ nhàng, dễ đọc |
| **Nhãn nút bấm (Action Label)**| `NamedTextColor.WHITE` (`#FFFFFF`) | Rõ nét, thanh thoát |
| **Nút đang BẬT** | `NamedTextColor.WHITE` + `[ BẬT ]` | Đậm nét hoặc có biểu tượng |
| **Nút đang TẮT** | `NamedTextColor.GRAY` + `[ TẮT ]` | Tone xám dịu |
| **Tooltip (Chú thích nút)** | `NamedTextColor.GRAY` (`#AAAAAA`) | Trực quan khi rê chuột |
| **Nút Đóng / Thoát** | `NamedTextColor.WHITE` (`#FFFFFF`) | Icon barrier hoặc viền tối giản |

### 6.2. Mã hóa Tiếng Việt Chống Lỗi Font (Mojibake-Proof)
Toàn bộ chuỗi hiển thị tiếng Việt có dấu phải được chuẩn hóa bằng mã thoát Unicode (`\uXXXX`) trong code Java để đảm bảo hiển thị đúng 100% trên mọi hệ điều hành (Windows UTF-16, Linux UTF-8) và trên mọi client:
- `Menu Ch\u00ednh` (Menu Chính)
- `C\u00e0i \u0110\u1eb7t` (Cài Đặt)
- `B\u1ea3ng X\u1ebfp H\u1ea1ng` (Bảng Xếp Hạng)
- `\u0110\u00f3ng` (Đóng)
- `Quay L\u1ea1i` (Quay Lại)
- `Trang K\u1ebf` (Trang Kế)

---

## 7. Tương thích Phiên bản & Fallback Client Cũ

Native Dialogs API chỉ hỗ trợ trên Minecraft phiên bản mới có tính năng Dialog. Khi server có cài đặt **ViaVersion/ViaBackwards** hoặc người chơi tham gia từ các phiên bản cũ (1.20.x, 1.21.0 - 1.21.3):
- Nếu server gửi gói Dialog cho client không hỗ trợ, client sẽ bị bỏ qua hoặc ngắt kết nối.
- **Giải pháp**: Xây dựng lớp phát hiện hỗ trợ:

```java
public class ClientCompatUtil {
    public static boolean supportsNativeDialog(Player player) {
        try {
            // Kiểm tra class hoặc method tồn tại trong Paper runtime
            Class.forName("io.papermc.paper.dialog.Dialog");
            
            // Nếu có ViaVersion API:
            // int protocol = com.viaversion.viaversion.api.Via.getAPI().getPlayerVersion(player.getUniqueId());
            // return protocol >= 769; // Protocol phiên bản có Dialog
            return true;
        } catch (Throwable t) {
            return false;
        }
    }
}
```
Khi `supportsNativeDialog(player) == false`, hệ thống tự động mở Menu rương ảo (Inventory GUI) fallback để người chơi vẫn sử dụng được tính năng bình thường.

---

## 8. Tích Hợp Icon Sprites & Player Head (Đầu Người Chơi)

Một trong những ưu điểm vượt trội của Paper Native Dialogs (kết hợp Adventure 4.17+) là khả năng hiển thị trực tiếp hình ảnh 2D của vật phẩm Minecraft và khuôn mặt skin 2D/3D thực tế của người chơi mà **không cần ép client cài đặt resource pack hay mod bên thứ ba**.

---

### 8.1. Cơ Chế Icon Sprites (`ObjectContents.sprite`)

Trong Adventure (Minecraft 1.21+), `Component` hỗ trợ nhúng đối tượng Sprite (`ObjectComponent` / `ObjectContents.sprite`). Client Minecraft sẽ tự động trích xuất texture 2D từ Atlas đồ họa nội bộ của game để render lên nút bấm hoặc văn bản:

#### 1. Hai Atlas Texture Chính trong Minecraft:
- **Items Atlas**: `Key.key("minecraft", "items")` với sprite key dạng `Key.key("minecraft", "item/<id>")` (Ví dụ: `item/diamond_sword`, `item/ender_pearl`, `item/clock`).
- **Blocks Atlas**: `Key.key("minecraft", "blocks")` với sprite key dạng `Key.key("minecraft", "block/<id>")` (Ví dụ: `block/stone`, `block/crafting_table_top`).

#### 2. Các Trường Hợp Ngoại Lệ (Edge Cases):
- Khối đầu lâu (`Material.SKELETON_SKULL` / `WITHER_SKELETON_SKULL`): Vì đầu lâu trong game là mô hình khối 3D, không có sprite 2D trực tiếp trong atlas `item/`. Nếu gọi `item/skeleton_skull`, client sẽ không hiển thị được icon. **Giải pháp**: Tự động chuyển đổi sang `minecraft:skull_banner_pattern` để hiển thị biểu tượng đầu lâu 2D sắc nét.
- Khối tuyết (`Material.SNOW_BLOCK`): Tự động chuyển thành `minecraft:snow`.

#### 3. Bộ Nhớ Đệm Bất Biến (Sprite In-Memory Caching):
Khởi tạo và đối chiếu Key liên tục sẽ sinh rác cho Garbage Collector. Hãy luôn cache các Sprite Component đã render vào một `Map<String, Component> SPRITE_CACHE` dùng chung.

---

### 8.2. Kỹ Thuật Player Head 2D trong Nhãn Nút & Văn Bản (`ObjectContents.playerHead`)

Nhúng đầu người chơi 2D (khuôn mặt skin có lớp mũ/tóc ngoài) trực tiếp vào dòng chữ trước tên người chơi trong Bảng xếp hạng, danh sách bạn bè, hoặc thông tin hồ sơ:

#### 1. Cấu Trúc Khởi Tạo (`PlayerHeadObjectContents`):
```java
var builder = net.kyori.adventure.text.object.ObjectContents.playerHead()
        .hat(true) // BẮT BUỘC: Giữ lớp tóc/mũ ngoài (outer layer) để skin tự nhiên
        .id(uuid); // UUID người chơi

// Nếu có chuỗi Base64 textures (lấy từ PlayerProfile):
if (base64Texture != null && !base64Texture.isBlank()) {
    builder.profileProperty(net.kyori.adventure.text.object.PlayerHeadObjectContents.property("textures", base64Texture));
}

Component headComponent = Component.object(builder.build());
```

#### 2. Ứng Dụng Vào Nút Bấm ActionButton:
```java
Component playerLabel = Component.empty()
        .append(headComponent)
        .append(Component.text(" "))
        .append(Component.text(player.getName(), NamedTextColor.WHITE));

ActionButton playerButton = ActionButton.builder(playerLabel)
        .tooltip(Component.text("Xem chi ti\u1ebft th\u00f4ng s\u1ed1", NamedTextColor.GRAY))
        .action(...)
        .build();
```

---

### 8.3. Kỹ Thuật Player Head 3D trong Thân Hộp Thoại (`DialogBody.item`)

Để hiển thị mô hình đầu 3D quay tròn trong modal hồ sơ cá nhân (Profile Modal):
Paper Dialog API hỗ trợ nhúng trực tiếp một `ItemStack` vào thân hộp thoại qua `DialogBody.item(ItemStack)`. Client sẽ tự động dựng mô hình vật phẩm 3D nổi bật:

```java
public static DialogBody create3DPlayerHeadBody(UUID uuid, String playerName) {
    ItemStack headItem = new ItemStack(Material.PLAYER_HEAD);
    SkullMeta meta = (SkullMeta) headItem.getItemMeta();
    if (meta != null) {
        if (uuid != null) {
            meta.setOwningPlayer(Bukkit.getOfflinePlayer(uuid));
        } else if (playerName != null) {
            meta.setOwner(playerName);
        }
        headItem.setItemMeta(meta);
    }
    return DialogBody.item(headItem);
}
```

---

### 8.4. Cảnh Báo Hiệu Năng: Chống Khóa Luồng Khi Tải Skin (Zero-Lag Skin Loading)

> [!CAUTION]
> **TỬ HUYỆT DROP TPS TỪ SKIN PLAYER HEAD**:
> Tuyệt đối **KHÔNG BAO GIỜ** kích hoạt tác vụ tải skin từ Mojang SessionServer (hoặc gọi các hàm tải skin đồng bộ) trên Main Thread.
> Khi mở bảng xếp hạng Top 10 mà 10 người chơi đó đang offline và chưa có cache skin, nếu truy vấn đồng bộ qua mạng, Main Thread sẽ bị khóa từ **1.000ms đến 3.000ms**, khiến server giật đứng hình (freeze) và tụt sạch TPS!

#### Nguyên Tắc Vàng Xử Lý Skin Đạt Chuẩn 20.0 TPS:
1. **Đối với người chơi Online**:
   Lấy trực tiếp từ RAM (`player.getPlayerProfile()`), trích xuất property `textures` chỉ tốn `0.01ms`.
2. **Đối với người chơi Offline**:
   Lưu sẵn chuỗi Base64 `textures` vào database (SQLite/MySQL) khi người chơi thoát game (`PlayerQuitEvent`). Khi nạp danh sách Top Leaderboard ở luồng ngầm (Async Thread), nạp sẵn Base64 vào RAM Cache.
3. **Fallback Khi Chưa Có Texture**:
   Chỉ truyền UUID thuần túy để client tự fetch ở phía máy trạm, hoặc fallback sang ký tự trắng tối giản `👤 `, tuyệt đối không để server block luồng chờ mạng.

---

### 8.5. Mã Nguồn Tiện Ích Trọn Gói: `IconAndHeadCatalog.java`

Lớp tiện ích chuyên biệt, an toàn với Reflection (tránh lỗi ClassNotFound trên các build Adventure khác nhau) và tự động cache:

```java
package net.yourplugin.dialog.util;

import net.kyori.adventure.key.Key;
import net.kyori.adventure.text.Component;
import org.bukkit.Bukkit;
import org.bukkit.Material;
import org.bukkit.NamespacedKey;
import org.bukkit.entity.Player;
import org.jetbrains.annotations.Nullable;

import java.lang.reflect.Method;
import java.util.Locale;
import java.util.Map;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;

public final class IconAndHeadCatalog {

    private static final Key ATLAS_BLOCKS = Key.key("minecraft", "blocks");
    private static final Key ATLAS_ITEMS = Key.key("minecraft", "items");

    private static final Map<String, Component> SPRITE_CACHE = new ConcurrentHashMap<>();
    private static final Map<UUID, Component> HEAD_CACHE = new ConcurrentHashMap<>();

    private IconAndHeadCatalog() {}

    /**
     * Tạo Component Sprite cho một vật phẩm Minecraft (ví dụ minecraft:diamond_sword).
     */
    public static @Nullable Component sprite(@Nullable String materialKey) {
        if (materialKey == null || materialKey.isBlank()) return null;

        // Xử lý các edge cases không có sprite flat item
        if ("minecraft:skeleton_skull".equalsIgnoreCase(materialKey) || "skeleton_skull".equalsIgnoreCase(materialKey)
                || "minecraft:wither_skeleton_skull".equalsIgnoreCase(materialKey) || "wither_skeleton_skull".equalsIgnoreCase(materialKey)) {
            materialKey = "minecraft:skull_banner_pattern";
        }
        if ("minecraft:snow_block".equalsIgnoreCase(materialKey) || "snow_block".equalsIgnoreCase(materialKey)) {
            materialKey = "minecraft:snow";
        }

        Material m = Material.matchMaterial(materialKey);
        if (m == null) return null;

        String keyStr = m.getKey().toString();
        Component cached = SPRITE_CACHE.get(keyStr);
        if (cached != null) return cached;

        Component built = buildSprite(m);
        if (built != null) {
            SPRITE_CACHE.put(keyStr, built);
        }
        return built;
    }

    private static @Nullable Component buildSprite(Material m) {
        NamespacedKey nk = m.getKey();
        String id = nk.getKey();
        Key atlas = m.isBlock() ? ATLAS_BLOCKS : ATLAS_ITEMS;
        Key spriteKey = Key.key(nk.getNamespace(), (m.isBlock() ? "block/" : "item/") + id);

        try {
            Class<?> ocClass = Class.forName("net.kyori.adventure.text.object.ObjectContents");
            Method spriteMethod = ocClass.getMethod("sprite", Key.class, Key.class);
            Object spriteObj = spriteMethod.invoke(null, atlas, spriteKey);
            Method compObjectMethod = Component.class.getMethod("object", ocClass);
            return (Component) compObjectMethod.invoke(null, spriteObj);
        } catch (Throwable ignored) {
            return Component.translatable(m); // Fallback an toàn
        }
    }

    /**
     * Tạo 2D Player Head Component có lớp tóc/mũ đầy đủ, an toàn tuyệt đối không lag luồng.
     */
    public static Component playerHead(@Nullable UUID uuid, @Nullable String playerName, @Nullable String base64Texture) {
        if (uuid != null) {
            Component cached = HEAD_CACHE.get(uuid);
            if (cached != null) return cached;

            try {
                // Kiểm tra người chơi online để lấy texture tức thời
                Player online = Bukkit.getPlayer(uuid);
                String texture = base64Texture;
                if (texture == null && online != null) {
                    try {
                        var pp = online.getPlayerProfile();
                        for (var prop : pp.getProperties()) {
                            if ("textures".equalsIgnoreCase(prop.getName()) && prop.getValue() != null) {
                                texture = prop.getValue();
                                break;
                            }
                        }
                    } catch (Throwable ignored) {}
                }

                var builder = net.kyori.adventure.text.object.ObjectContents.playerHead()
                        .hat(true)
                        .id(uuid);

                if (texture != null && !texture.isBlank()) {
                    builder.profileProperty(net.kyori.adventure.text.object.PlayerHeadObjectContents.property("textures", texture));
                }

                Component comp = Component.object(builder.build());
                HEAD_CACHE.put(uuid, comp);
                return comp;
            } catch (Throwable ignored) {}
        }

        if (playerName != null && !playerName.isBlank()) {
            try {
                return Component.object(net.kyori.adventure.text.object.ObjectContents.playerHead(playerName));
            } catch (Throwable ignored) {}
        }

        return Component.text("\uD83D\uDC64 "); // Placeholder 👤
    }

    public static void cleanCache(UUID uuid) {
        HEAD_CACHE.remove(uuid);
    }
}
```

---

## 9. Bộ Code Mẫu Chuẩn Production (Production Codebase Template)

### 9.1. Lớp Điều Phối Tác Vụ Đa Luồng (`SchedulerUtil.java`)
Tương thích hoàn hảo cả Paper truyền thống và Paper Folia:

```java
package net.yourplugin.dialog.util;

import org.bukkit.Bukkit;
import org.bukkit.entity.Player;
import org.bukkit.plugin.Plugin;

public class SchedulerUtil {

    private static Boolean isFolia = null;

    public static boolean isFolia() {
        if (isFolia == null) {
            try {
                Class.forName("io.papermc.paper.threadedregions.RegionizedServer");
                isFolia = true;
            } catch (ClassNotFoundException e) {
                isFolia = false;
            }
        }
        return isFolia;
    }

    public static void runForEntity(Plugin plugin, Player player, Runnable runnable) {
        if (player == null || !player.isOnline()) return;

        if (isFolia()) {
            player.getScheduler().run(plugin, task -> runnable.run(), null);
        } else {
            if (Bukkit.isPrimaryThread()) {
                runnable.run();
            } else {
                Bukkit.getScheduler().runTask(plugin, runnable);
            }
        }
    }
}
```

---

### 9.2. Lớp Dịch Vụ Dialog Mẫu Chuẩn (`StandardDialogService.java`)
Triệt tiêu hoàn toàn: Mouse Reset, Drop TPS, Jitter, và Lỗi Màu Sắc:

```java
package net.yourplugin.dialog;

import io.papermc.paper.dialog.Dialog;
import io.papermc.paper.registry.data.dialog.ActionButton;
import io.papermc.paper.registry.data.dialog.DialogBase;
import io.papermc.paper.registry.data.dialog.action.DialogAction;
import io.papermc.paper.registry.data.dialog.body.DialogBody;
import io.papermc.paper.registry.data.dialog.type.DialogType;
import net.kyori.adventure.text.Component;
import net.kyori.adventure.text.event.ClickCallback;
import net.kyori.adventure.text.format.NamedTextColor;
import net.kyori.adventure.text.format.TextDecoration;
import net.yourplugin.YourPlugin;
import net.yourplugin.dialog.util.DialogRateLimiter;
import net.yourplugin.dialog.util.IconAndHeadCatalog;
import net.yourplugin.dialog.util.SchedulerUtil;
import org.bukkit.Sound;
import org.bukkit.entity.Player;

import java.time.Duration;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;

public class StandardDialogService {

    private final YourPlugin plugin;

    // Giả lập cache cài đặt in-memory (RAM)
    private final Map<UUID, Boolean> soundToggleCache = new ConcurrentHashMap<>();
    private final Map<UUID, Boolean> particlesToggleCache = new ConcurrentHashMap<>();

    public StandardDialogService(YourPlugin plugin) {
        this.plugin = plugin;
    }

    public void openSettingsDialog(Player player) {
        if (player == null || !player.isOnline()) return;

        // Đảm bảo chạy trên Entity Scheduler của Player (Folia Safe)
        SchedulerUtil.runForEntity(plugin, player, () -> {
            UUID uuid = player.getUniqueId();
            boolean soundEnabled = soundToggleCache.computeIfAbsent(uuid, k -> true);
            boolean particleEnabled = particlesToggleCache.computeIfAbsent(uuid, k -> true);

            // 1. Tiêu đề và Mô tả: CHỮ TRẮNG THUẦN TÚY (White Typography)
            Component title = Component.text("C\u00e0i \u0110\u1eb7t H\u1ec7 Th\u1ed1ng", NamedTextColor.WHITE, TextDecoration.BOLD);
            Component description = Component.text(
                    "T\u00f9y ch\u1ec9nh c\u00e1c t\u00ednh n\u0103ng v\u00e0 hi\u1ec7u \u1ee9ng c\u00e1 nh\u00e2n.\nNh\u1ea5n v\u00e0o n\u00fat \u0111\u1ec3 B\u1eacT ho\u1eb7c T\u1eaeT tr\u1ef1c ti\u1ebfp.",
                    NamedTextColor.WHITE
            );

            List<ActionButton> buttons = new ArrayList<>();

            // 2. Nút 1: Profile Người Chơi (Hiển thị 2D Player Head thực tế)
            Component headComp = IconAndHeadCatalog.playerHead(uuid, player.getName(), null);
            Component profileLabel = Component.empty()
                    .append(headComp).append(Component.text(" "))
                    .append(Component.text("H\u1ed3 S\u01a1: ", NamedTextColor.GRAY))
                    .append(Component.text(player.getName(), NamedTextColor.WHITE, TextDecoration.BOLD));

            buttons.add(ActionButton.builder(profileLabel)
                    .tooltip(Component.text("Xem th\u00f4ng tin t\u00e0i kho\u1ea3n c\u00e1 nh\u00e2n", NamedTextColor.GRAY))
                    .action(DialogAction.customClick((view, audience) -> {
                        // Click profile
                    }, ClickCallback.Options.builder().uses(1).build()))
                    .build());

            // 3. Nút 2: Toggle Âm Thanh (Icon Sprite + Chữ Trắng, Zero-Mouse-Reset, Zero-Jitter)
            buttons.add(createToggleButton(
                    player,
                    "minecraft:note_block",
                    "\u00c2m Thanh",
                    soundEnabled,
                    "B\u1eadt ho\u1eb7c t\u1eaft \u00e2m thanh hi\u1ec7u \u1ee9ng",
                    () -> {
                        boolean newState = !soundEnabled;
                        soundToggleCache.put(uuid, newState);
                        playToggleFeedback(player, newState);
                    }
            ));

            // 4. Nút 3: Toggle Hạt Hiệu Ứng (Icon Sprite + Chữ Trắng)
            buttons.add(createToggleButton(
                    player,
                    "minecraft:blaze_powder",
                    "H\u1ea1t Hi\u1ec7u \u1ee8ng",
                    particleEnabled,
                    "B\u1eadt ho\u1eb7c t\u1eaft hi\u1ec7u \u1ee9ng h\u1ea1t",
                    () -> {
                        boolean newState = !particleEnabled;
                        particlesToggleCache.put(uuid, newState);
                        playToggleFeedback(player, newState);
                    }
            ));

            // 5. Nút Đóng: Barrier Icon Sprite + Chữ Trắng, dọn dẹp Rate Limiter khi thoát
            Component exitLabel = Component.empty();
            Component barrierSprite = IconAndHeadCatalog.sprite("minecraft:barrier");
            if (barrierSprite != null) {
                exitLabel = exitLabel.append(barrierSprite).append(Component.text(" "));
            }
            exitLabel = exitLabel.append(Component.text("\u0110\u00f3ng", NamedTextColor.WHITE));

            ActionButton exitButton = ActionButton.builder(exitLabel)
                    .tooltip(Component.text("\u0110\u00f3ng b\u1ea3ng c\u00e0i \u0110\u1eb7t", NamedTextColor.GRAY))
                    .action(DialogAction.customClick((view, audience) -> {
                        if (audience instanceof Player p) {
                            DialogRateLimiter.clean(p.getUniqueId());
                            p.closeDialog();
                        }
                    }, ClickCallback.Options.builder().uses(1).build()))
                    .build();

            // 6. Cấu hình DialogBase CỐT LÕI:
            // - afterAction(DialogAfterAction.NONE): Ngăn client tự đóng màn hình khi click nút!
            // - canCloseWithEscape(true): Cho phép bấm phím ESC để thoát an toàn
            Dialog dialog = Dialog.create(b -> b.empty()
                    .base(DialogBase.builder(title)
                            .canCloseWithEscape(true)
                            .pause(false)
                            .afterAction(DialogBase.DialogAfterAction.NONE) // BẮT BUỘC: Giữ chuột & chống giật
                            .body(List.of(DialogBody.plainMessage(description)))
                            .build()
                    )
                    .type(DialogType.multiAction(buttons, exitButton, 2))
            );

            // Cập nhật đè trực tiếp (In-place screen replacement) -> Chuột đứng im tại chỗ!
            player.showDialog(dialog);
        });
    }

    private ActionButton createToggleButton(Player player, String spriteKey, String featureName, boolean isEnabled, String tooltip, Runnable onToggle) {
        Component label = Component.empty();

        // Nhúng Sprite từ IconAndHeadCatalog
        Component sprite = IconAndHeadCatalog.sprite(spriteKey);
        if (sprite != null) {
            label = label.append(sprite).append(Component.text(" "));
        }

        // Nhãn chữ trắng với trạng thái rõ ràng
        Component stateBadge = isEnabled
                ? Component.text("[ B\u1eacT ]", NamedTextColor.WHITE, TextDecoration.BOLD)
                : Component.text("[ T\u1eaeT ]", NamedTextColor.GRAY);

        label = label.append(Component.text(featureName + " ", NamedTextColor.WHITE))
                .append(stateBadge);

        return ActionButton.builder(label)
                .tooltip(Component.text(tooltip, NamedTextColor.GRAY))
                .action(DialogAction.customClick((view, audience) -> {
                    if (audience instanceof Player p) {
                        // 1. Chống spam click (Anti-Spam / Drop TPS)
                        if (!DialogRateLimiter.checkAndRecord(p.getUniqueId())) {
                            return; // Bỏ qua nếu click quá nhanh (<150ms)
                        }

                        // 2. Chạy logic và cập nhật ngay trên Entity Scheduler
                        SchedulerUtil.runForEntity(plugin, p, () -> {
                            onToggle.run();
                            // 3. KHÔNG GỌI player.closeDialog()! Gọi thẳng openSettingsDialog để làm mới
                            openSettingsDialog(p);
                        });
                    }
                }, ClickCallback.Options.builder()
                        .uses(1)
                        .lifetime(Duration.ofMinutes(10))
                        .build()))
                .build();
    }

    private void playToggleFeedback(Player player, boolean newState) {
        if (newState) {
            player.playSound(player.getLocation(), Sound.BLOCK_NOTE_BLOCK_PLING, 1.0f, 2.0f);
            player.playSound(player.getLocation(), Sound.UI_BUTTON_CLICK, 0.7f, 1.2f);
        } else {
            player.playSound(player.getLocation(), Sound.UI_BUTTON_CLICK, 0.7f, 0.8f);
        }
    }
}
```

---

## 10. Bảng Kiểm Thử Checklist (Pre-Flight Verification)

Trước khi đóng gói jar và đưa plugin lên server chạy chính thức, hãy rà soát kỹ bảng kiểm tra sau:

- [ ] **Mouse Stay Test**: Dùng chuột click liên tục 10 lần vào cùng 1 nút Bật/Tắt -> Con trỏ chuột có đứng nguyên vị trí không? (Nếu chuột bị nhảy về tâm màn hình dù chỉ 1 lần -> Kiểm tra xem có lệnh `closeDialog()` nào bị sót không, hoặc `afterAction` có đang để khác `NONE` không).
- [ ] **Zero-Flicker Test**: Khi click chuyển đổi, màn hình thế giới 3D phía sau có bị nháy hay chớp đen không?
- [ ] **Anti-Spam Stress Test**: Dùng auto-clicker với tốc độ 20 CPS nhấp vào menu trong 10 giây:
  - TPS server có giữ nguyên 20.0 không?
  - Băng thông mạng (tx/rx packets) có duy trì ổn định không bị tràn bộ đệm không?
- [ ] **White Aesthetic Test**: Rà soát toàn bộ chuỗi hiển thị, đảm bảo không có màu vàng, xanh lá, đỏ hoặc mã màu rainbow nào xuất hiện bừa bãi. Mọi chữ hiển thị chính đều phải là `NamedTextColor.WHITE` hoặc `GRAY` cho tooltip.
- [ ] **Mojibake UTF-8 Check**: Chạy thử trên môi trường Windows Command Prompt / Server Console, xác nhận không có ký tự lạ (`Ã¡`, `áº`, `?`) xuất hiện trong Dialog.
- [ ] **Folia Compatibility Test**: Đảm bảo toàn bộ các hàm `player.showDialog()` đều được bọc qua Entity Scheduler, không bao giờ gọi trực tiếp từ async network thread hoặc Bukkit global scheduler.
- [ ] **Player Quit Cleanup**: Đảm bảo sự kiện `PlayerQuitEvent` có dọn dẹp các cache trong `DialogRateLimiter` để không bị memory leak (rò rỉ RAM).
