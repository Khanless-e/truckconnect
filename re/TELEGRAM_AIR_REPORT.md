# Telegram Air 2.12.3 (macOS): reverse-engineering report

Target: `Telegram Air.app` (from the attached zip; byte-identical to `/Applications/Telegram Air.app`)
Executable: `Contents/MacOS/telegram_air`, Mach-O x86_64 thin, 6,747,328 bytes
SHA-256: `3b03967e855489114634e6cee90515b103258606ef89dc88e3455e99ac876c21`
REA Evidence bundle: [telegram-air-evidence.json](telegram-air-evidence.json) (REA 6.9.0, 5 records)

Legend: **[O]** observed in the artifact · **[I]** inferred · **[?]** unknown / not verified

---

## 1. What the app is

**[O]** A **Tauri 2.12.2** desktop app written in Rust (wry 0.57 WebView + tao 0.37 windowing). It is not Electron and not native Swift/ObjC UI.

**[O]** The main window loads the **remote** URL `https://web.telegram.org/a/`, which is Telegram Web A. Only a single `/index.html` asset is embedded locally (`tauri://localhost`). No JS bundle ships in the binary.

**[I]** So the app is a thin native shell around Telegram Web A. All of the messaging UI, the MTProto client and the storage logic come from the web app at runtime, not from this binary.

**[O]** The binary contains the author string `Alexander Zinchuk`. The updater manifest is hosted on the GitHub gist of the user `Ajaxy`. **[I]** This is the same developer as Telegram Web A (open source: `github.com/Ajaxy/telegram-tt`, GPL-3.0).

**[O]** It was built on GitHub Actions: the cargo paths start with `/Users/runner/...`.

## 2. Identity and signing (Evidence `ev_63fe9847…`)

| Field | Value |
|---|---|
| Bundle ID | `org.telegram.TelegramAir` |
| Version | 2.12.3 (min macOS 10.13) |
| Signer | Developer ID Application: **Anyway Labs FZCO** (Team `Y54Z4K69Z9`) |
| Hardened runtime | enabled |
| Timestamp | Nov 11, 2026 10:10:10 PM |
| Entitlements | `cs.allow-jit`, `device.audio-input`, `device.camera`, `device.microphone`, `device.print`, `inherit`, `personal-information.location` |

**[O]** Info.plist does three relevant things:
- It registers the URL scheme `tg://`.
- Its ATS has an exception with an **empty domain key** that sets `NSExceptionAllowsInsecureHTTPLoads` and includes subdomains. **[?]** What effect this has is unclear.
- It has usage strings for the camera ("calls"), the microphone ("voice messages and calls") and the Downloads folder.

## 3. Hardening (Evidence `ev_47d48b25…`) and release exposure (`ev_58f2b43f…`)

- **[O]** PIE is on. Stack execution is off. `___stack_chk_fail` is imported. The image is not encrypted and has no `__RESTRICT` segment.
- **[O]** The release-exposure audit found no exposures:
  - no source maps
  - no debug companions
  - no credential markers
  - no extractable JS payload
- **[O]** There is no cargo-auditable `.dep-v0` metadata. The crate list below was recovered from embedded panic and source paths instead.

## 4. Native architecture

### 4.1 Crates and plugins (observed from embedded registry paths)

| Plugin | Version | Purpose in this app |
|---|---|---|
| tauri-plugin-deep-link | 2.6.1 | Handles `tg://` links and emits `deep-link://new-url` |
| tauri-plugin-single-instance | 2.5.2 | Keeps one running instance and forwards links to it |
| tauri-plugin-notification | 2.5.1 | Native notifications, plus a `window.Notification` polyfill (`mac-notification-sys`, `notify-rust`) |
| tauri-plugin-updater | 2.13.2 | Self-update with minisign verification |
| tauri-plugin-window-state | 2.5.0 | Remembers window size and position |
| tauri-plugin-shell | 2.4.1 | Opens external `http(s)`/`mailto:`/`tel:` links that use `target=_blank` in the system browser |
| tauri-plugin-os | 2.4.0 | Provides OS info to the web side |
| tauri-plugin-log | 2.10.0 | Writes logs under `~/Library/Logs` |

Other crates include reqwest 0.13, rustls 0.23 with the platform verifier, tokio 1.53, `window-vibrancy` 0.8.1 (`NSGlassEffectView`, the macOS 26 "Liquid Glass"), muda (menus), tray-icon, and image/png/jpeg/webp/tiff/exr decoders.

### 4.2 App-specific IPC commands (Rust ⇐ JS `invoke`)

**[O]** These names appear together with the author string in the app's own rodata:

| Command | Observed argument keys | [I] Role |
|---|---|---|
| `set_notifications_count` | `amount`, `isMuted` | Sets the Dock badge to the unread count |
| `set_menu_translations` | `translations` | Localizes the native menu bar from Web A's language pack |
| `open_new_window_cmd` | — | Opens another Telegram window or tab |
| `mark_title_bar_overlay` | `isOverlay` | Switches between the overlay title bar and the normal one |
| `set_window_title` | — | Sets the window title |
| `save_current_url` | — | Persists the last Web A route so it can be restored on relaunch |

There are also wrappers for `apply_liquid_glass` / `clear_liquid_glass` (macOS only) and a custom traffic-light delegate ("NS Window should exist to mount traffic light delegate").

**[O]** Events the native side sends to the web side:
- `will-enter-fullscreen`, `did-enter-fullscreen`, `will-exit-fullscreen`, `did-exit-fullscreen`
- the standard `tauri://focus|blur|resize|theme-changed|drag-drop…` events

### 4.3 How the web side detects the shell

**[O]** Initialization scripts are injected into the page. They define:
- `window.isTauri = true`
- `window.__TAURI_INTERNALS__` (with `invoke`, `transformCallback`, `convertFileSrc`, and per-request `Tauri-Invoke-Key`)
- `window.ipc.postMessage` → `webkit.messageHandlers.ipc`

**[I]** Web A checks `window.isTauri` to turn on its desktop-only behavior: the badge, the title-bar overlay, new windows and menu translations.

**[?]** The Tauri capability/ACL that lets the remote origin `web.telegram.org` call these commands could not be decoded with string analysis. Doing so needs a disassembler or decompiler (see §6).

### 4.4 Auto-update

**[O]** The update manifest URL is:
`https://gist.githubusercontent.com/Ajaxy/9bc7f25dec2f5bf4a9bd98960d7e7512/raw/cbf9ce3e98110fa8fcf631db55ea1ba3770e0cd0/telegram-air-updater.json`

**[O]** The minisign public key ID is `9F1F7090AD81299A`:
`RWSaKYGtkHAfn3HbUA8C37L6mA/AHvXDabnz8KkxSB2hA7qh0cCIm3kQ`

**[O]** The plugin exposes `check`, `download_and_install` and `restartAfterInstall`. The manifest structs are `ReleaseManifestPlatform` and `InnerRemoteRelease`.

## 5. Architecture diagram

```
┌──────────────── Telegram Air.app (Rust / Tauri 2) ────────────────┐
│ tao NSWindow (custom traffic lights, Liquid Glass / vibrancy)     │
│   └─ wry WKWebView ──► https://web.telegram.org/a/  (Telegram Web A)│
│          ▲  invoke(): set_notifications_count, set_menu_translations,│
│          │            open_new_window_cmd, mark_title_bar_overlay,   │
│          │            set_window_title, save_current_url, plugins…   │
│          ▼  events: *-fullscreen, deep-link://new-url, tauri://*     │
│ Plugins: deep-link(tg://) · single-instance · notification · updater│
│          window-state · shell(open links) · os · log                │
└───────────────────────────────────────────────────────────────────┘
        Web A ⇄ Telegram DCs over MTProto (WebSocket) — inside the WebView
```

## 6. Coverage, limits and open questions

- **Done with REA:** signature and entitlements, hardening, release-exposure audit, Rust metadata probe. Strings and linkage were read with `strings` and `otool`.
- **Not done:** decompiling the Rust command bodies. REA found no deep provider: Ghidra, Hopper, IDA and Kuna are all unconfigured. `inspect_macho` also hit its 60 s native-command deadline. Setting `GHIDRA_INSTALL_DIR` to Ghidra ≥ 12.1 would enable `analyze_function` and `procedure_pseudo_code`. That would confirm the command signatures and the remote-origin capability ACL.
- **Not done:** runtime observation, and extracting the contents of the embedded `index.html`.
- **The web client:** the real application logic is Telegram Web A. Its source is public (GPL-3.0), so reading the repository is better evidence than reverse-engineering minified bundles. If you reuse it in a clone, the clone must also be GPL-3.0.

## 7. Implications for `tgclone`

If the goal is to build a Telegram-like desktop client:
1. **Shell:** Tauri 2 plus the same set of plugins is enough. The native side is roughly 6 commands, plus badge, menu, title bar and deep-link wiring.
2. **Client:** a web UI that talks MTProto. Either fork Telegram Web A (accepting GPL) or build on a library such as GramJS or TDLib (WASM). Telegram API credentials (`api_id`/`api_hash`) come from my.telegram.org.
3. **Branding:** do not reuse Telegram's name, logo or icons in a clone.
