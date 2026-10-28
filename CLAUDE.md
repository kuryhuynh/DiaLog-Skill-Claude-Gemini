# CLAUDE.md - PaperMC Native Dialogs Development Guide

This repository provides architectural guidelines, code patterns, and constraints for developing **Minecraft Native Dialog Plugins (PaperMC 1.21+)** using the Paper Native Dialogs API (`io.papermc.paper.dialog.*`).

Follow these guidelines whenever writing, refactoring, or reviewing Minecraft Dialog plugin code.

---

## ⚡ Core Invariants (Non-Negotiable Rules)

1. **Zero-Mouse-Reset Architecture**:
   - **NEVER** call `player.closeDialog()` when updating states, paginating, or toggling actions.
   - **ALWAYS** configure `DialogBase.builder(...).afterAction(DialogBase.DialogAfterAction.NONE)` on all interactive buttons.
   - In Minecraft client, calling `closeDialog()` or using `DialogAfterAction.CLOSE` resets the cursor to the center `(width / 2, height / 2)` and causes visual flicker.
   - To update a dialog, directly call `player.showDialog(newDialog)` in-place on the player's Entity Scheduler.

2. **Zero-Flicker & Zero-Jitter Transitions**:
   - Do **NOT** introduce artificial tick delays (`runTaskLater(1)`).
   - Keep the grid structure and button positions identical across re-renders so the mouse pointer remains hovered over the clicked button.
   - Use subtle audio cues: `Sound.BLOCK_NOTE_BLOCK_PLING` (pitch 2.0f) for ON, `Sound.UI_BUTTON_CLICK` (pitch 0.8f) for OFF.

3. **Anti-TPS Drop & Folia Threading Compliance**:
   - **Zero I/O in Callbacks**: Never query SQLite, MySQL, or read YAML files inside `DialogAction.customClick`. Read and write to in-memory RAM cache (`ConcurrentHashMap` / `PlayerData`). Dispatch persistence asynchronously.
   - **Folia Regional Scheduler**: Always dispatch player actions via `SchedulerUtil.runForEntity(plugin, player, ...)` or `player.getScheduler().run(...)`. Never use `Bukkit.getScheduler()` synchronously for player UI.
   - **Debounce / Rate Limiter**: Enforce a minimum 150ms interval per player UUID to neutralize auto-clickers and macro button spam.

4. **Anti-Network Eater & Packet Hygiene**:
   - Always bind `ClickCallback.Options.builder().uses(1).lifetime(Duration.ofMinutes(10)).build()`.
   - **Dirty Checking**: Do not resend a dialog packet if the toggle state did not change.
   - Never run live countdowns or clocks at tick frequency (50ms) inside dialogs.

5. **Clean White Typography (Monochrome Vanilla Aesthetic)**:
   - **No rainbow/garish colors**: Avoid gold, yellow, pink, and bright green text.
   - **Title**: `NamedTextColor.WHITE` + Bold.
   - **Body**: `NamedTextColor.WHITE` (Primary), `NamedTextColor.GRAY` (Secondary hints).
   - **States**: `[ BẬT ]` (`NamedTextColor.WHITE`, Bold) / `[ TẮT ]` (`NamedTextColor.GRAY`).
   - **Mojibake Protection**: Encode all accented Vietnamese characters with Unicode escape sequences (`\uXXXX`).

6. **Icon Sprites & Player Head Integration**:
   - Use `ObjectContents.sprite(atlas, spriteKey)` wrapped in `Component.object(...)` for 2D item textures.
   - Map 3D block skulls (`SKELETON_SKULL`) to `minecraft:skull_banner_pattern` for flat 2D rendering.
   - For Player Heads:
     - 2D in labels: `ObjectContents.playerHead().hat(true).id(uuid).build()`.
     - 3D in body: `DialogBody.item(headItemStack)`.
   - **CRITICAL**: Never trigger synchronous Mojang SessionServer skin requests for offline players on the Main Thread.

---

## 🛠️ Build and Testing Commands

- **Build project**: `./gradlew build`
- **Clean build**: `./gradlew clean build`
- **Run test server**: `./gradlew runServer` (if paperweight-userdev is configured)

---

## 📂 Project Structure Reference

- `.agents/skills/paper-dialog-plugin/SKILL.md`: Google Antigravity & Gemini Agent skill definition.
- `.claude/skills/paper-dialog-plugin/SKILL.md`: Anthropic Claude Code & Claude Desktop skill definition.
- `skills/paper-dialog-plugin/SKILL.md`: Universal skill file for multi-agent workflows.

---

## 🧩 Essential Code Snippets

### 1. Folia-Safe Scheduler (`SchedulerUtil.java`)
```java
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
            if (Bukkit.isPrimaryThread()) runnable.run();
            else Bukkit.getScheduler().runTask(plugin, runnable);
        }
    }
}
```

### 2. Smooth Toggle Dialog Construction
```java
Dialog dialog = Dialog.create(b -> b.empty()
    .base(DialogBase.builder(Component.text("C\u00e0i \u0110\u1eb7t", NamedTextColor.WHITE, TextDecoration.BOLD))
        .canCloseWithEscape(true)
        .pause(false)
        .afterAction(DialogBase.DialogAfterAction.NONE) // MUST BE NONE!
        .body(List.of(DialogBody.plainMessage(Component.text("M\u00f4 t\u1ea3", NamedTextColor.WHITE))))
        .build()
    )
    .type(DialogType.multiAction(buttons, exitButton, 2))
);
player.showDialog(dialog);
```
