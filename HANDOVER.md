# VRE AC Stock — Project Handover

**Repo:** https://github.com/Ai-mali/Antigravity-Stock.git (private workflow — license tool lives OUTSIDE this repo, see §7)
**Product:** Windows desktop app for air-conditioner parts stock management (Daikin + other brands), Excel-backed, no server install.

---

## 1. What it does (business workflow)

```
Delivery-order photo → AI scan → Stock In → Available Stock
     → Dispatch/Stock Out (customer + dates) → Sell Record / Tracking
     → Return & RMA → Restock or Quarantine → Warranty tracking
```

Plus: customer history, brand/model catalog, activity audit log, auto backups, Excel/CSV exports, day/dark themes, custom wallpapers, all dates displayed `DD/MM/YYYY`.

## 2. Architecture — one process, three layers

```
desktop_app.py   PyWebView/WebView2 frameless window + native splash + window controls
      │          spawns backend in a thread (uvicorn, port 8000)
backend.py       FastAPI on 127.0.0.1:8000 — serves ac-stock-tracker.html at "/"
      │          every /api/* call serialized by a global asyncio.Lock
stock_store.py   StockStore wraps openpyxl → daikin_stock.xlsx (the entire database)
```

`ac-stock-tracker.html` (~17,400 lines) is the ENTIRE frontend — HTML + CSS + vanilla JS in one file. It talks to `127.0.0.1:8000/api/*` via `api()`/`apiGet`/`apiPost`/`apiDel` helpers.

## 3. File map

| File | Role |
|---|---|
| `ac-stock-tracker.html` | Whole UI: Stock In, Available Stock, Stock Out, Track List, Return tabs; modals; theme engine |
| `backend.py` | FastAPI routes (35+ endpoints), license middleware, config/ui-prefs, scan endpoint |
| `stock_store.py` | StockStore: workbook load/migrate/save, all business logic, backups, activity log |
| `desktop_app.py` | Desktop shell: splash screen, health check, frameless window, JS↔Python bridge (`DesktopApi`) |
| `license_check.py` | Embedded license verifier (Ed25519) — ships with the app |
| `scanner.py` | Vision-AI calls (Gemini etc.) via urllib; keys from config modal |
| `excel_exports.py` | Ready-export workbooks after each snapshot |
| `comet_layer.py`, `boot_debug_kit.py` | Boot splash animation + boot timing/debug |
| `test_fulfillment_groups.js` | Node test suite (73 tests) for tracking/fulfillment logic |
| `build-exe.bat` | PyInstaller onefile build → `dist\VRE AC Stock.exe` |

## 4. Data model — the Excel workbook IS the database

Sheets inside `daikin_stock.xlsx`:

| Sheet | Columns | Purpose |
|---|---|---|
| `MasterRecord` | Brand, Model, Serial, Date In, Status, Customer, Date Out, Batch | Every unit; Status ∈ `in_stock` / `sold` |
| `Brands` | Brand | Brand list → accordion grouping + colors |
| `ModelBrands` | Model, Brand | model→brand map; deleted when last unit of a model goes (unless sold history needs it) |
| `Returns` | Serial, Model, Customer, Reason, Condition, Notes, Action, Date, Date Out | RMA registry (restock/quarantine) |
| `ActivityLog` | … | audit trail |

**Serial uniqueness rule (important):** a duplicate = same **model + serial**. Same serial under a *different* model is legal. Unit lookups are model-context-aware (`_pair_key`, `_unit_find`).

**Non-serialized items** get synthetic auto-IDs (`_split_dupes(auto_ids=True)`, `_compact_synthetic_ids`).

**Date formats — three zones, never mix them:**
- Workbook canonical: `MM/DD/YYYY`
- Picker/API internal: ISO `YYYY-MM-DD`
- UI display: `DD/MM/YYYY` via `fmtDMY()` / `fmtDMYTime()`
- Editable fields (e.g. edit-unit Date In) parse day-first client-side, submit ISO, and `update_unit` re-canonicalizes to `MM/DD/YYYY`. Never `new Date(ambiguousString)` in JS — use `parseDateAny`/`parseDateDMY`.

**Warranty:** `compute_warranty(date_out, months=12)` → expiry shown in UI + tracked on sold units.

**Sidecar files/dirs next to the workbook:** `backups/` (auto snapshot on every save, pruned to 60 + `ready_exports/`), `delivery_orders/` (Recent DO scan photos, auto-purged >3 days), `permanent_do/` (photos copied here when a scan is committed; never auto-purged; names include `DO number`, `SO number`, document date, and scan date/time), `serial_provenance.json` (model/serial links to permanent DO/SO photos), and `ui_prefs.json` (theme, wallpaper, AI model config, API keys).

## 5. Backend specifics

- Global `_store_lock` serializes **all** `/api/` requests — the workbook isn't concurrency-safe; single-operator design so a lock is correct.
- `licenseGate` middleware (in `backend.py`) returns `403 {"ok":false,"error":"license_required","state":…,"machine":…,"detail":…}` for every `/api/*` EXCEPT `/api/license/status` and `/api/license/activate`. Frontend `api()` catches `license_required` → shows gate.
- `/api/scan` → one full-image vision AI request that extracts document metadata (DO/SO number/date/confidence) plus inventory rows; the uploaded image is archived to `delivery_orders/` before scanning and copied to `permanent_do/` when its stock-in batch is committed.
- `/api/config*` → AI engine router (Gemini keys etc.) stored in `ui_prefs.json`.
- Exports: `/api/export/available` (xlsx), `/api/export/sales`, CSV paths for returns/activity — all emit `DD/MM/YYYY`; filenames use `-` separators (`/` is illegal in filenames).

## 6. Desktop shell (`desktop_app.py`)

- Splash drawn with GDI+ (`comet_layer` animated comet) while WebView2 + backend warm up.
- Health check polls **`/api/license/status`** — always 200 even unlicensed. Never point it at a gated endpoint (a 403 used to hang "Starting services…" forever — that bug is fixed; keep it that way).
- `webview.start(...)` guarded by `_webview2_present()` — missing runtime → MessageBox + official bootstrapper link.
- `DesktopApi` bridge (`window.pywebview.api`): `minimize/toggle_maximize/close`, `save_file`, `open_url` (scheme-whitelisted http/https/tg — used by Purchase Renewal → Telegram).
- Exit via `os._exit(0)` — kills the backend thread instantly, frees the port for instant relaunch.

## 7. Licensing — two repos, one keypair

**In-app (`license_check.py`):** `license.lic` + `license_state.json` beside the exe. Key format `VRE1.<b64url payload>.<b64url Ed25519 sig>`; payload `{app:"VRE-STOCK", customer, machine, expires, issued}`. Checks: signature → machine match → expiry → clock-rollback (last-seen date in `license_state.json`). `machine="*"` = master key (emergency only). `VRE_DEV_BYPASS=1` skips enforcement — **ignored when frozen**.

**Machine ID** = Windows `MachineGuid` → hex-16, grouped `XXXX-XXXX-…` (e.g. `C71A-4CBE-…`). `license_tool.py` reimplements the same algorithm — identical output required.

**Generator (NOT in this repo):** `D:\AI-Project\Devin\License key generator\` → `license_tool.py` (PyQt6 GUI, `license-tool.bat`, `menu`/`sign`/`init` CLI), `VRE-License-Generator.exe` + **`license_private.pem`** — the private key never enters any repo/chat; it must sit beside the generator exe. `License exe\` subfolder holds the pair.

**Customer flow:** app boots → status `missing` → gate shows Machine ID → customer clicks Purchase Renewal (copies ID + opens `https://t.me/chh_ck`) → you run the generator exe → send `VRE1.…` → they paste → `/api/license/activate` writes `license.lic` → unlocked. Expiry ≤30d → dismissible banner; expiry hit → gate (still shows reason + lets a new key activate).

**License admin modal** (topbar 🔑): `.licm-card` design — status banner variants `ok/warn/bad`, meta rows, machine-id input, renew textarea, footer buttons. If you edit copy/activate buttons, label text lives in `.copy-label`/`#licm-activate-txt` spans — writing `textContent` on the button destroys its SVG icon.

## 8. Packaging rules (frozen exe)

`sys.frozen` ⇒ all writable paths resolve to **`Path(sys.executable).parent`** — the exe's folder:
- `_resolve_db_path()` adopts the newest `*.xlsx` beside the exe if `daikin_stock.xlsx` is absent (ignores `~$*` lock files) — customers drop their real workbook with any name.
- `license.lic`, `license_state.json`, `ui_prefs.json`, `serial_provenance.json`, `backups/`, `delivery_orders/`, `permanent_do/` all beside the exe → **install folder must be writable** (never Program Files).
- Bundled read-only assets (`ac-stock-tracker.html`, icons) resolve via `sys._MEIPASS` (`BASE` in backend.py).
- `build-exe.bat` handles: `--collect-all cryptography` (bundled .pyd backends), `--collect-submodules webview`, `--hidden-import multipart` (FastAPI lazily imports it for UploadFile — photo upload breaks without it), uvicorn loop/protocol hidden imports.

## 9. Testing & verification

```bash
node test_fulfillment_groups.js        # 73 tests — tracking/fulfillment/date helpers
python -m py_compile backend.py stock_store.py desktop_app.py license_check.py
# embedded-JS syntax check (extracts every inline script and runs node --check):
node check_frontend_syntax.js
```

Test suite `eval`s extracted functions — when a render helper gains a dependency (e.g. `fmtDMY`), the harness must expose it too.

**License test matrix** (all verified): missing / malformed / tampered-sig / wrong-machine / expired / master / perpetual / live HTTP 403→activate→200.

## 10. Editing conventions & gotchas

- The HTML file stores some chars as **literal `\u2014`/`\u00b7` text** (6 chars, not the glyph) — exact-match edits and regexes must account for it; test the pattern before scripted replaces.
- Theme: `body.theme-light` = Day mode; CSS vars carry the palette (`--brand-ink` = darkened brand color for *text* in Day mode, `--brand-c` stays vivid for dots/borders). Inline JS-generated colors use `style="--brand-c:…"` custom props so theme toggle needs no re-render.
- Global `font-variant-numeric: slashed-zero` (0 vs 8 legibility).
- Custom pickers: `initDatePicker`, 24h `rs-time` time picker (00–23 × 00–59 min), popovers use explicit solid colors (wallpaper mode makes `--bg-app` translucent — never use it for floating panels).
- `localStorage` keys: `lic_warn_hide` (banner dismiss), theme/accent/wallpaper prefs (mirrored to `ui_prefs.json`).
- Port 8000 is single-instance — a stale backend serves new HTML with old API. Kill the process holding the port before relaunching.
- Modals: `.modal-overlay` + `.open` class; only hard gate (`#license-gate`) is a full-screen takeover.

## 11. Release checklist

1. `build-exe.bat` → `dist\VRE AC Stock.exe`
2. Test in a clean folder: exe + customer's `.xlsx` → boots to gate (no license)
3. Generate key with the generator exe (customer's Machine ID) → activate → data loads
4. Ship: exe + xlsx in a writable folder. Warn about SmartScreen (unsigned) — *More info → Run anyway*.

**Never commit/ship:** `license_private.pem`, `license_state.json`, `*.lic`, real customer workbooks.

## 12. Latest session update

- 2026-10-30 build: branch `ui-fit-laptop-v3`, source commit `c143378`.
- `dist\VRE AC Stock.exe` rebuilt successfully; smoke-launched for 12 seconds and closed cleanly.
- Executable SHA-256: `A34AB726A43BB64D7648783380A91075B05E4C08A2EFD5369F873D0679F1A18A` (86,609,141 bytes).
- On `ui-fit-laptop-v3`, moved the Stock Out second-hand `2nd` tag into `.so-card-actions`, immediately before the photo button, and left the bottom row with only Date In plus `Ready`/`In Cart ✓`. Added only a Stock Out-scoped alignment rule; Available Stock cards were unchanged. Live verification confirmed K002719 renders the tag in the top row, clicking the tag does not stage the card, clicking the card changes the bottom status to `In Cart ✓`, and the 73 tests/syntax/Python/diff checks pass. The shared browser canvas was narrower than the requested 1280x680, so exact target-size geometry was not measured. No executable rebuild was performed.
- On `ui-fit-laptop-v3`, appended the requested 28px/8px header-control CSS block verbatim after the active-date-chip rules in `ac-stock-tracker.html`; no application logic or other source code changed. Frontend syntax, all 73 Node tests, Python compilation, and `git diff --check` pass. The shared browser canvas loaded successfully but remained a narrow app preview rather than 1280x680 or 1140x680, so target-size real-click measurements for both themes could not be completed. No executable rebuild was performed.
- Created `ui-fit-laptop-v3` from `37c0c02` without changing `ui-fit-laptop-v2`. Appended the requested active-date-chip CSS block verbatim at the end of `ac-stock-tracker.html`'s main style block; no other source changes were made. Frontend syntax, all 73 Node tests, Python compilation, and `git diff --check` pass. The local server was unavailable and the shared browser canvas timed out, so the requested real-click checks at 1280x680 and 1500x680 in both themes could not be completed. No executable rebuild was performed.
- Shortened the desktop Available Stock and Stock Out search groups to a fixed 300px total with a 170px text area, and Sell Record to a 210px text area. Focused or non-empty fields expand smoothly to 260px without changing the existing right-aligned title-line toolbar; default inventory placeholders are now `Serial or model`. Validation passed; the shared browser canvas remained 339px wide, so requested 1280px/large-window geometry could not be measured in this session.
- Removed the dedicated Quick Date/brand-pill filter bars and their Available Stock/Stock Out filter-toggle buttons. Sidebar brand filtering, Date In picker filtering, search, Low Stock/2ND filters, exports, Stock Out actions, recent-customer toggle, and Sell Record breakdown toggle remain; shared `.filters-toggle-btn` and `.filter-pills-row` styles were retained where still used. Sticky brand/model offsets now derive directly from each page header because the removed bars no longer contribute height.
- Aligned the Available Stock and Stock Out desktop page-header toolbars with the title row instead of the full title/subtitle block. The title row and toolbar now share a 30px alignment band; Sell Record and Return were left unchanged. The available browser canvas was 339px wide, so the desktop (>1120px) geometry could not be measured there; wrapping remained expected in that narrow view.
- Fixed Day-theme unit-card hover contrast on Available Stock and Stock Out: serials now use `rgb(11,79,67)` without glow, dates use `rgb(63,90,87)`, and status tags use `rgb(15,118,110)` while dark-theme glow remains unchanged. CSS/browser checks measured at least 8.59:1 for highlighted serials, 6.76:1 for dates, and 4.96:1 for status tags; the available browser viewport was 339x956 rather than 1280x680.
- Fixed second-hand card styling on `ui-fit-laptop-v2`: amber left edges now win over Day-theme, hover, staged, and highlighted border rules for Available Stock and Stock Out; mouse clicks no longer leave the browser's black focus ring, while keyboard focus uses the app's teal `:focus-visible` ring. Validation and browser measurements are recorded with this change.
- Corrected the DO/SO strip markup so only the two date fields use centered alignment.
- Delivery Order and Sales Order number fields, tags, and boxes remain left-aligned.
- On the Stock In screen only, pressing Enter after loading/pasting an image triggers the Scan action; inputs, buttons, dialogs, and other tabs retain normal Enter behavior.
- Added the Option D frosted-pill Paste button inside the Stock In drop zone. It uses `navigator.clipboard.read()`, reuses `processLoadedPhoto(File)`, shows inline clipboard errors, and does not trigger the drop zone's file browser.
- Desktop Paste now uses a local `DesktopApi.read_clipboard_image()` Pillow bridge first, so the Windows WebView2 permission popup is avoided; browser/non-desktop mode retains the Clipboard API fallback.
- Added Pillow to `requirements.txt` for the local clipboard image bridge.
- Replaced the serial-linked photo viewer's DO/SO toggle with a styled Download button. The viewer now always shows the archived scan image, and desktop downloads use a native Save As dialog with the last-used folder remembered in `ui_prefs.json`; browser fallback uses `showSaveFilePicker()` or Downloads.
- The viewer download preserves the archived file bytes with `shutil.copyfile`, supports cancel without feedback, and shows inline save success/error status that clears when another serial is opened.
- Rebuilt `dist\VRE AC Stock.exe` with the native clipboard bridge on 2026-10-26. SHA-256: `51E976116642FFAFCEC3D02C4E80549F47BB834608A788FBA82490FA6CEF507C`.
- The rebuilt executable was smoke-launched successfully and remained running for 12 seconds before being closed.
- Rebuilt `dist\VRE AC Stock.exe` again on 2026-10-26 after the serial-photo Download/Save As viewer change. SHA-256: `EE124850865740847538D3D759695BD242FA0AA511C19C9F4231EDE7014CA714`.
- The latest executable was smoke-launched successfully and remained running for 12 seconds before being closed.
- Frontend syntax, `git diff --check`, Python compilation, and all 73 regression tests pass.
- The user explicitly requested that Force Scan not be implemented yet; the current version remains the one to use.
- This session pins the current source and built executable after the photo-viewer Download/Save As change.
- Published stable source commit: `2e3762f` (`Finalize photo download build`).
- Published annotated tag: `stable-20261026-photo-download`.
- Pushed to `origin/main` and `origin/stable-20261026-photo-download` at `https://github.com/Ai-mali/Github-Copilot`.
- Added DPI-aware desktop window sizing in `desktop_app.py`: the initial window now fits the Windows work area in CSS pixels, clamps its minimum size to that area, and centers itself. `VRE_PREVIEW_WORKAREA=1280x680` provides a laptop-sized development preview on a larger monitor.
- Added `run_desktop_laptop.bat`, which sets `VRE_PREVIEW_WORKAREA=1280x680` and launches the normal desktop runner.
- Verified the preview geometry returns `1280x680` with `min_size=(1050, 680)`; no executable rebuild was performed.
- The next step is intentionally paused: use the laptop-sized preview and report cut-off screens one at a time. Fix only those reports with small-window media queries or existing desktop-shell rules; never use CSS zoom/transform scaling, change larger-window behavior, alter business logic, or change the Excel schema.
- For the first laptop header report, added a `1121px–1320px` media rule: hides the Day/Dark word, shows only the active provider name (for example `Gemini`), narrows the model pill and tab spacing, and reduces header gaps/padding. The dynamic provider update also uses only the provider name. Larger windows keep their existing layout.
- Rebuilt `dist\VRE AC Stock.exe` on 2026-10-26 after the laptop header adjustment and DPI-aware window sizing. Artifact size: `86,594,937` bytes. SHA-256: `98206422A3FD5B909612BB26A0E487B657AB398204AE4DDE50E959AA27E2BC2B`. Smoke-launched for 12 seconds successfully, then closed.
- The build output contains the existing batch-wrapper footer warning `plus was unexpected at this time`, but PyInstaller completed successfully and produced the executable. No source commit, tag, or push was made in this build session. The current working tree still contains the laptop sizing/header changes and the existing untracked runtime/preview files.
- After laptop testing showed the app still had the taskbar and desktop margins visible, the desktop window was changed to start with pywebview `fullscreen=True`. This is intended to make the executable occupy the whole monitor rather than only the work area; the custom in-app minimize/maximize/close controls remain available.
- Rebuilt `dist\VRE AC Stock.exe` on 2026-10-26 with true fullscreen startup. Artifact size: `86,595,148` bytes. SHA-256: `F4D59B86EDCBDBE6A6BB4667B4DFC6F7EC8E3F4097CEE70B8187C2D74CAF8E63`. Smoke-launched successfully for 12 seconds, then closed. The build wrapper still prints `plus was unexpected at this time` after PyInstaller reports completion.
- The requested behavior is now clarified: keep the Windows taskbar visible, but fill the entire work area with no outer desktop margins. True pywebview fullscreen was removed; the desktop shell now uses the normal work-area geometry and removes its 5px body padding, rounded outer frame, border, and shadow.
- Rebuilt `dist\VRE AC Stock.exe` on 2026-10-26 for the taskbar-preserving edge-to-edge work-area layout. Artifact size: `86,595,454` bytes. SHA-256: `5FB68B771B95B2D6E9AF2C60121DFCBB5258E4C8B4ACAA64DD3C2D48AF168BC0`. Smoke-launched successfully for 12 seconds, then closed. The build wrapper still prints its known `plus was unexpected at this time` footer after PyInstaller completion.
- The resize cursor/gap issue was caused by the fixed resize handles being above the header and using oversized 34px corner / 14px edge hit zones. Resize handles are now behind the header controls (`z-index:35`), with smaller 18px corners and 8px edges, so the X button receives the pointer normally while edge resizing remains available.
- Rebuilt `dist\VRE AC Stock.exe` on 2026-10-26 with the resize-handle fix. Artifact size: `86,595,375` bytes. SHA-256: `01F06B86C8838DE6B95F16DF76878BEBFE8B7896B803DD744777A3DD60DAD0E7`. Smoke-launched successfully for 12 seconds, then closed.
- Reduced the desktop shell's main content padding from 16px/20px to 3px on all screens, matching the requested approximately 0.5mm gap between each screen's work area and the program frame. Browser and larger-window layouts are unchanged.
- Rebuilt `dist\VRE AC Stock.exe` on 2026-10-26 with the narrow screen-to-frame gap. Artifact size: `86,594,364` bytes. SHA-256: `61F45BCC33E74BD5049EF280471BCE33CA9B0EC9252A4494E31308D431F71256`. Smoke-launched successfully for 12 seconds, then closed.
- Compact desktop-shell layout history: removed the CSS outer frame and reduced main content padding to 2px; the browser header remains 56px while the desktop header is 40px with 30px logo/actions, a 32px tab pill, and 28px window controls. The desktop tab bar stays between the brand and right action group, and sticky offsets are measured dynamically.
- Desktop top resize hit zones are 4px for the edge and 10x10px for the top corners. The latest CSS raises only those three zones above the 40px header; left, right, and bottom resize zones remain unchanged.
- Runtime browser checks at the desktop URL found body, `.app-container`, and header all at top 0 with no top padding/margin/border; measured logo gaps are 4.5px above and 5.5px below. All five tabs and a configuration modal opened without clipping in the available 1126px preview.
- Added `VRE_NO_THICKFRAME=1` support in `desktop_app.py` and enabled it in `run_desktop_laptop.vbs` for native-frame diagnosis; default launch behavior is unchanged. `_desktop_window_geometry()` now converts the centered physical work-area origin to CSS-pixel x/y before passing it to pywebview. No executable rebuild was performed.
- Privacy cleanup: the two DO/SO preview HTML files are ignored and are not part of this branch. Validation for this session: 73 Node tests, Python compilation, embedded JavaScript syntax, and `git diff --check` passed.
- Added a defensive Win32 client-area fit for work-area-sized windows. After native frame setup and custom maximize, it measures the client rectangle in screen coordinates and adjusts only the outer rectangle until all sides are within 1px; smaller windows are unchanged, the taskbar remains protected, and optional `VRE_DEBUG_FRAME=1` diagnostics go to ignored `frame_debug.log`.
- Moved the existing frame setup/fit callback to run once before reveal while the window is hidden, with explicit debug markers around that pass; the original post-reveal pass remains as an idempotent compatibility fallback.
- Startup recording showed a residual visible resize before the pre-reveal pass, so the same idempotent frame setup now also runs immediately after the HWND is hidden and before the first `w.show()`; the later pre-reveal and post-reveal passes remain fail-open safeguards.
- **Stable laptop-fit point (2026-10-27):** on the real 1920x1080 laptop at 150% scaling, the usable work area is 1280x680 CSS px (approximately 294x156 mm excluding the taskbar). The latest EXE opens fully fitted from startup without the visible expansion; source and EXE use the same frameless client-area fit.

- **Stock In splitter fit (2026-10-27):** the Image Preview card now uses a flexing drop zone with an 80px minimum instead of a fixed 250px minimum, so the preview can shrink while the Load Image/SCAN/Clear row remains visible. Loaded images use contain sizing within the available drop-zone height. The splitter now measures the preview minimum from the live card DOM and uses the handle's actual 4px height plus the column gaps; Status Log can shrink to 80px. At short desktop heights (max 760px), the default drop-zone basis is 170px, the action-row margin is 8px, and the explanatory text/file tags are hidden. No scan/upload, window-edge resize, Excel, or EXE behavior changed.
- **Stock In splitter alignment follow-up (2026-10-27):** after commit `3eddbbe`, corrected the drag math to derive the preview maximum from `.stock-in-left-col.clientHeight` minus the 4px handle, both 16px gaps, and the 80px log minimum. Dragging now sets only the preview card flex basis; the Status Log remains flexible and its inline flex is cleared. The chosen preview height is retained and re-clamped on window resize so the left and right card bottoms stay aligned.
- **Stock In desktop splitters (2026-10-27):** desktop layouts above 1120px now use 2px grid/column gaps and a 2px horizontal splitter flow element with an approximately 8px invisible hit zone. Added a full-height vertical splitter with an approximately 8px hit zone, DOM-measured left-column minimum (404px in the 1280px preview), 520px Scan Results minimum, double-click reset to 410px, localStorage persistence, and resize clamping. Mobile/stacked layout rules remain unchanged. Live 1280px preview checks measured all requested gaps at 2px and kept Status Log/Scan Results bottoms aligned at 718px.
- Current layout testing should use `run_desktop_laptop.vbs`, which sets the 1280x680 preview and no-thickframe diagnostic mode. Continue making UI adjustments against that fixed target; do not rebuild the EXE unless explicitly requested. The latest shipped test EXE was built from commit `7059600`, size 86,600,009 bytes, SHA-256 `C2ED41F95F2E1552A2B07739E75A8AF3B33D6C8E0BC1D5460BCCEDEC0C772195`.
- Stable source branch is `ui-fit-laptop-v2`; `main` remains unchanged. The branch includes DPI-aware work-area sizing, corrected CSS-pixel placement, native-frame opt-out, runtime client-area correction, pre-show fitting to prevent startup flicker, compact 40px desktop header, safe resize zones, and privacy exclusions for real DO/SO preview/runtime data. No CSS/business-logic/Excel schema changes were made for the window-fit work.
- The attempted millimeter-based preview was removed because Windows monitor millimeter metadata produced an incorrect 944x501 CSS window. For reliable layout matching, use `run_desktop_laptop_screen.vbs` (or `.bat`), which explicitly previews the laptop's measured 1280x680 CSS-pixel work area. The existing `run_desktop_laptop.vbs` remains equivalent.
- Fixed the laptop launcher title-bar discrepancy: when `VRE_NO_THICKFRAME=1`, `desktop_app.py` now explicitly clears the native caption, border, dialog-frame, and thick-frame style bits. This keeps the VBS preview frameless like the intended laptop layout; no EXE rebuild was made.
- Confirmed on 2026-10-27 with the desktop screenshot: `run_desktop_laptop.vbs` now visually matches the laptop EXE startup—frameless, flush to the 1280x680 usable area, with the custom header and taskbar-preserving client fit. Continue using this VBS for layout tuning; rebuild the EXE only when explicitly requested.
- Reduced the fixed resize hit zones to minimize interference with the Stock In scan preview: top/bottom/left/right edges are now 2px (about 0.5mm CSS-equivalent), and all corners are 4x4px. Resize behavior remains available while the scan drop area keeps its full usable width.
- **Stock In compact persisted splitters (2026-10-27):** the desktop left-column splitter now has a hard 260px minimum; the prior DOM-measured width is used only as the compact-state threshold. At compact widths, preview actions become equal 38px icon-only buttons, Recent DOs becomes icon-only, and Status Log filters compress to one line while the Paste control remains visible. Both `stockInLeftW` and `stockInPreviewH` now persist through `/api/ui-prefs` on drag end/reset, with localStorage used only if that GET/POST path fails. Saved values are restored after the layout has real dimensions and are clamped for display without overwriting an out-of-range saved value. Verified at 1280x720 browser geometry for 260/300/350/410px widths: no button/header clipping and Status Log/Scan Results bottom difference 0px. Removing both runtime preference keys restored defaults (410px left width and 311px natural preview height). No scan/business/Excel/EXE changes.
- Validation for this change: all 73 Node tests passed, Python compilation passed for the four application modules, embedded JavaScript syntax passed, and `git diff --check` passed. Current work is uncommitted on `ui-fit-laptop-v2`; `serial_provenance.json` remains untracked and must not be committed. No executable rebuild was performed.
- Removed the redundant “Synced to DB” label and icon from the Stock In Status Log footer; the pending count and progress bar remain unchanged.
- **Available Stock desktop density (2026-10-27):** on `#view-available` only, collapsed brand accordions now measure 57px (34px header + 21px model peek) with a 6px inter-card gap; the colored accent and brand colors remain, metrics glow is removed, and text stays at or above 10px. The title/header toolbar is a single non-wrapping row on desktop: compact title/Units pill, 28px LOW STOCK/2ND/search/filter controls, and icon-only Export Excel below 1500px (label returns at wider sizes). At the live preview, collapsed cards visible within a 680px viewport increased from 4 to 6; the current dataset has 8 collapsed brands. Search input, Date In popover, and filter-bar toggle were exercised. Stock Out header/accordion styles were left unchanged. No logic, scan, Excel, mobile, CSS zoom/transform, or EXE changes.
- **Stock Out + Available desktop fit (2026-10-27, commit `3cdd7af`):** reused the 57px/6px compact brand accordion styling on Stock Out, forced Stock Out brand names to one-line ellipsis with full-name titles, and hid only the `INVENTORY` label at narrow middle-column widths. Desktop Stock Out and Available gaps now measure 2px. Added the 6px left-edge Dispatch & Checkout resizer with `--so-right-w`, 260–560px limits constrained by a 480px middle column, pointer capture, double-click reset, resize re-clamping, and `soRightW` persistence through `/api/ui-prefs`. At 1280x720, Stock Out measured 2px/2px gaps, 340px default panel, 57px collapsed cards with 6px margins, 6px sidebar/panel handles, and a non-wrapping header; forcing the panel to 260px produced no card overflow and a 772px middle column. Available measured a 2px sidebar/content gap after reload. The 73-test suite, Python compilation, embedded JavaScript syntax check, and `git diff --check` pass. No business, scan, Excel, mobile, zoom/transform, or EXE changes; `serial_provenance.json` remains untracked.
- **Startup initializer fix (2026-10-28, pending commit):** moved `initStockoutRightResizer()` out of the `initSideResizer()` mousemove handler to top level, preserving the existing sidebar drag handler and allowing the DOMContentLoaded initializer to continue into date-picker setup and the Stock Out panel resizer. Saved Stock Out width is retained while the initially hidden tab has zero layout width, then re-clamped when Stock Out is shown so reload restores the saved value instead of prematurely forcing 260px. Added `check_frontend_syntax.js`, which extracts each inline script and runs `node --check`; use `node check_frontend_syntax.js` to catch unbalanced braces/nested-function mistakes. Browser verification at 1280x720 showed no captured console errors, all picker states initialized (Return/Scan/Dispatch today; Available/Stock Out Date In placeholders), panel drag 322px→379px, and 379px restored after reload. No business, scan, Excel, mobile, zoom/transform, or EXE changes.
- **Sell Record + Return desktop fit (2026-10-28):** Sell Record now uses a 260px default sidebar override, 2px desktop layout/card gaps, a compact single-line title/toolbar treatment, hidden desktop subtitle, compact live badge, and icon-only Export Excel below 1500px. Return now uses a 320px CSS-variable left panel, 2px desktop gap, a 6px pointer-capture resizer with 280–520px limits and a 560px registry minimum, double-click reset, resize re-clamping, and `/api/ui-prefs` persistence under `returnLeftW`. Return registry columns use fixed widths, tighter cells, ellipsis/title behavior for Customer/Reason/Notes, and the Export CSV/Print RMA controls retain title/aria-label while becoming icon-only below 1500px. At the live 1280x720 preview, Sell Record overflow was 0px, its layout gap was 2px, and the persisted sidebar measured 243px; Return measured `320px 954px` with a 2px gap, registry table width 952px with no horizontal overflow, and all headers including Actions visible. Dragging Return left width to 400px restored 400px after reload; forcing 280px produced 0px panel overflow and left/right bottoms both at 718px. No business, scan, Excel, mobile, zoom/transform, or EXE changes. Validation: `node check_frontend_syntax.js`, all 73 Node tests, Python compilation, and `git diff --check` passed. Current changes are uncommitted on `ui-fit-laptop-v2`; `ui_prefs.json` and `serial_provenance.json` remain untracked/ignored runtime data.
- **Stock In compact labels (2026-10-28):** compact mode at the 260px left-column minimum now changes Recent DOs to visible `DOs` while retaining its icon and photo-count badge, and changes Status Log filter labels to visible `Iss` and `OK` while retaining their count badges. Full `Recent DOs`, `Issues`, and `Committed` labels remain unchanged at normal widths. At the live 1280x720 browser preview, dragging the column from 410px to 260px produced a compact state with measured widths of 69.4px (DOs + count), 50.4px (Iss + count), and 44.4px (OK + count); the photo badge measured 20px and all count badges remained visible. Restoring 410px showed the full labels and hid only the short-label spans. No business, backend, scan, Excel, mobile, zoom/transform, or EXE changes. Validation: `node check_frontend_syntax.js`, all 73 Node tests, Python compilation, and `git diff --check` passed. Current changes are uncommitted on `ui-fit-laptop-v2`; `serial_provenance.json` remains untracked.

## 13. Continuation prompt for another Copilot account

Copy the prompt below into the next Copilot chat after opening this repository:

> You are continuing work on the **VRE AC Stock** Windows desktop inventory application in this repository. Read `HANDOVER.md` completely before changing anything. Treat the repository files and this handover as the source of truth; do not assume the previous chat is available.
>
> Current stable source/build point: branch `ui-fit-laptop-v2`, commit `7059600` (`Fit window before first show`). The latest built executable is `dist\VRE AC Stock.exe`; SHA-256 `C2ED41F95F2E1552A2B07739E75A8AF3B33D6C8E0BC1D5460BCCEDEC0C772195`. Use `run_desktop_laptop.vbs` for ongoing 1280x680 CSS-pixel layout testing; do not rebuild unless explicitly requested.
>
> Important current behavior:
> - Stock In scans the full image once and extracts inventory rows plus DO/SO numbers, dates, and confidence.
> - Scanned photos first go to `delivery_orders` (Recent DOs, 3-day retention).
> - When scanned stock is committed, the photo is copied to `permanent_do`, which is never auto-purged.
> - Permanent filenames include DO number, SO number, document date, scan date, and scan time.
> - `serial_provenance.json` links committed model/serial pairs to the permanent photo and DO/SO metadata.
> - Existing legacy provenance filenames may remain in the JSON; preserve compatibility unless explicitly migrating them.
> - Visible dates use `DD/MM/YYYY`; API/picker values use ISO `YYYY-MM-DD`; workbook dates remain `MM/DD/YYYY`.
> - The DO/SO date picker is editable for correcting an OCR result. The date confidence label and date value are centered in the current UI.
>
> Current status and user decision:
> - The current build includes the background local clipboard bridge and the serial-photo viewer Download button with native Save As.
> - Do not implement Force Scan yet. The user's next planned feature is a low-confidence recovery flow: when scan confidence is below 60%, warn or offer Force Scan, then improve extraction with image enlargement, sharpening, contrast/deskew, and targeted DO/SO/model/serial crops or retries. This is only a future requirement, not current behavior.
> - The latest follow-up work covers Stock Out and Available desktop fit in commit `3cdd7af`. Use `run_desktop_laptop.vbs` (1280x680 preview) for any further geometry checks; do not rebuild unless explicitly requested. The changes add Stock Out panel resizing/persistence, shared compact brand cards, and 2px desktop gaps.
> - Inspect `git status` and the latest commit/tag before making changes.
>
> Required working rules:
> - Read relevant code before editing and preserve existing behavior.
> - Do not rebuild the executable unless explicitly requested.
> - Run targeted syntax/tests after changes; the normal regression suite is `node test_fulfillment_groups.js` (currently 73 tests).
> - Do not alter the Excel schema for provenance work.
> - Do not expose or commit credentials, private keys, real licenses, or real customer data.
> - If the user asks for a build, build with `build-exe.bat`, smoke-test the executable, report its path and SHA-256, and only then mark the task complete.
>
> Before ending every work session, update this `HANDOVER.md` with:
> 1. what changed,
> 2. what remains unfinished,
> 3. tests/builds run and their results,
> 4. the current commit/tag/build state,
> 5. any user decisions or constraints from that session.
>
> Do not claim completion while any requested change or verification remains unfinished. Use the existing project conventions and keep changes surgical.

### Latest session update — Return/Sell Record photo actions

- Removed the Return-tab Print RMA Tags header control, per-row print action, print-only sheet markup, print-only CSS, and the `printRmaTags()` / `reprintRmaTag()` functions. No other screen used those RMA-only pieces.
- Added read-only `document` provenance payloads to `track_list()` and `returns_list()`. Sell Record and Return unit rows now place a reusable DO/SO photo action immediately after the eye action; it is enabled only when `document.doPhoto` or `document.soPhoto` exists and otherwise shows `No DO/SO photo`.
- Browser verification at the available 1280px desktop preview found no right-edge overflow on Sell Record or Return, no Return Print RMA control, and correct disabled/enabled photo states using a temporary provenance fixture. The temporary fixture was restored and remains untracked; no customer data was committed.
- Validation passed: `node check_frontend_syntax.js`, all 73 Node tests, Python compilation for `backend.py stock_store.py desktop_app.py license_check.py`, and `git diff --check`. No executable rebuild was performed. Published commit is `b228608` on both `origin/main` and `origin/ui-fit-laptop-v2`.

### Latest session update — shared brand peek rows

- Available Stock and Stock Out now share the expanded brand-peek treatment: 28px rows, 18px dashed model chips at 10.5px text, 5px chip gaps, and no right-edge mask. Both renderers prepare up to 10 model chips and attach the total model count for measurement.
- Added a shared ResizeObserver/requestAnimationFrame fitting routine. It hides trailing chips when the row is narrow, keeps the row on one line, and recalculates `+N more` from the true hidden model count after rendering, window resize, sidebar resize, or Dispatch-panel resize.
- At the available 1280x720 browser preview, collapsed cards measured 64px high in both screens (previously approximately 57px); peek rows measured 28px. Available Stock showed DAIKIN-AIR CON 10 + 30, DAIKIN-REMOTE 7 with no more label, and NEW-IN 6 with no more label. Stock Out showed 7 + 33, 7, and 6 respectively because its middle column is narrower. A 485px narrow preview also re-fitted correctly without overflow. No console/page errors were observed.
- Validation passed: `node check_frontend_syntax.js`, all 73 Node tests, Python compilation, and `git diff --check`. No executable rebuild was performed. Changes are uncommitted on `ui-fit-laptop-v2`; runtime `serial_provenance.json` remains untracked.

### Latest session update — compact expanded model/unit cards

- On desktop widths only, Available Stock and Stock Out expanded model content now uses 30px model headers, 44px one-line serial/unit cards, compact 21px action icons, 6px grid gaps, reduced body/grid padding, and auto-fill grids (`170px` minimum for Available Stock and `150px` for Stock Out). Stock Out's repeated staging-cart hint was removed and retained as the model-header tooltip.
- Sticky model offsets now use the measured rendered brand-header height instead of a hard-coded 43px, keeping model headers directly below the brand header as the compact heights change.
- At the available 1280x720 desktop preview, the DAIKIN-ADAPTER five-unit model measured 44px cards and 30px model headers: Available Stock rendered 5 cards per row at about 188px each; Stock Out rendered 4 cards per row at about 166px each in its 700px middle column. DAIKIN-AIR CON expanded to all 40 models without overflow. Stock Out staging toggled on/off correctly, photo/action buttons remained present with 21px hitboxes, and no repeated hint line remained. No console/page errors or right-edge overflow were observed. The browser canvases available for this session were 1280x720; no separate large-window canvas was available.
- Validation passed: `node check_frontend_syntax.js`, all 73 Node tests, Python compilation, and `git diff --check`. No executable rebuild was performed. Changes are ready to commit on `ui-fit-laptop-v2`; runtime `serial_provenance.json` remains untracked.

### Latest session update — symmetric expanded grids

- Desktop-only expanded grids now use `scrollbar-width: thin`, `scrollbar-gutter: stable both-edges`, 6px padding, and 6px gaps. Available Stock uses a capped five-column formula; Stock Out uses a capped four-column formula. Mobile/stacked grid rules were not changed.
- At the available 1280x720 browser preview, Available Stock DAIKIN-AIR CON rendered five 186.39px cards per row with 16.00px left and 16.02px right gaps, both with no scrollbar and with a forced visible scrollbar. Its partial final row started at the same 16px left gap. Stock Out rendered four 165.25px cards per row with equal 16px gaps; its partial final row also started at 16px. Available cards remained 44px high, Stock Out cards remained 44px high, and no horizontal overflow was detected.
- The browser canvas reports 1280x720 rather than the requested 1280x680, and no separate large-window canvas was available. No console/page errors were observed during the grid checks. No executable rebuild was performed.
- Validation passed: `node check_frontend_syntax.js`, all 73 Node tests, Python compilation for `backend.py stock_store.py desktop_app.py license_check.py`, and `git diff --check`. Current source changes are uncommitted on `ui-fit-laptop-v2`; runtime `serial_provenance.json` remains untracked and must not be committed.

### Latest session update — compact unit serial cards

- Desktop Available Stock and Stock Out unit cards now render serials as head/tail spans: the head ellipsizes while the last six characters remain visible. Short serials stay whole, search marks still render against the full serial, and card click/keyboard activation continues to pass the full serial to the existing copy/staging handlers.
- Available Stock cards remain 44px high with a five-column maximum; the serial and staged/2nd-hand tags occupy row 1, while Date In and the existing photo/edit/delete actions occupy row 2. The normal `In Stock` label was removed from unstaged Available Stock cards; staged cards retain `In Cart ✓`. Second-hand cards retain the clickable `2nd` tag and amber left edge.
- Stock Out cards remain 44px high with four columns at about 166px in the current middle column. The photo button remains on row 1, the status words `Ready` / `In Cart ✓` remain on row 2, the status dot was removed, and the clickable `2nd` tag remains on the status row.
- Added one delegated, shared fixed tooltip for both grids. After 300ms hover or keyboard focus it shows full serial, model, Date In, available DO/SO numbers, and 2nd-hand status; it flips above near the bottom edge and hides on card change, click, scroll, or leave. Serial text no longer carries a native title.
- Browser verification used the available 1280x720 canvas (the requested 1280x680 and a separate large-window canvas were unavailable). `INTERFACE ADAPTER #1/#2` endings remained visible, card height was 44px, Available Stock measured five cards/row with 16.00/16.02px gutters, and Stock Out used four equal columns with no horizontal overflow. Tooltip visibility, search highlighting, staged `In Cart ✓`, photo action presence, and no console/page errors were observed. The live data contained no second-hand units, so the 2nd-hand state was verified structurally rather than with a real populated card.
- Validation passed: `node check_frontend_syntax.js`, all 73 Node tests, Python compilation for `backend.py stock_store.py desktop_app.py license_check.py`, and `git diff --check`. No executable rebuild was performed. Changes are pending commit on `ui-fit-laptop-v2`; runtime `serial_provenance.json` remains untracked.

### Latest session update — Available Stock action-button clearance

- Available Stock desktop unit cards now use 48px height/min-height with 5px vertical and 9px horizontal padding. Their photo/edit/delete buttons are 19px square with 11px SVG icons; Stock Out card sizing and the completed 5/4-column grid rules were left unchanged.
- At the available 1280x720 browser canvas, Available Stock DAIKIN-ADAPTER cards measured 48px high. Every visible action button measured 19px and ended 6px above the card bottom; right edges were at least 6px inside the card, and serial bounds stayed inside every card. The staged `K000387` and second-hand `K000590` cards also measured 48px and passed the same bounds checks. No console/page errors or horizontal overflow were observed.
- The browser canvas available for this session was 1280x720 rather than the requested 1280x680; no large-window canvas or Playwright package was available for an additional viewport run. No executable rebuild was performed.
- Validation passed: `node check_frontend_syntax.js`, all 73 Node tests, Python compilation for `backend.py stock_store.py desktop_app.py license_check.py`, and `git diff --check`. Changes are pending commit on `ui-fit-laptop-v2`; runtime `serial_provenance.json` remains untracked.

### Latest session update — native frameless window dragging

- Added `DesktopApi.start_native_drag()` using cached HWND, restore anchoring, `WM_NCLBUTTONDOWN/HTCAPTION`, and a daemon poller that preserves top-edge maximize behavior. `VRE_NATIVE_DRAG=0` disables the native path.
- Header pointerdown now releases capture and tries native drag first; the existing JS drag remains the fallback when the bridge is unavailable or returns false. Interactive controls, snap preview, restore behavior, and double-click maximize remain owned by the frontend.
- Added a full-work-area guard and retained move-only `DesktopApi.move_window()` for the JS fallback. Validation passed: `node check_frontend_syntax.js`, all 73 Node tests, Python compilation for all eight application modules, `git diff --check`, and stubbed no-HWND/disabled-native tests. Native drag and 1280x680 tab console checks could not be tested headless and need a real-PC check; no executable rebuild was performed. Changes are uncommitted on `ui-window-drag-native`; runtime `serial_provenance.json` remains untracked.
- Fixed DPI-aware work-area comparison and prevented top-edge maximize on a plain native click unless the cursor moved at least 8 px.

## 14. Handover maintenance rule

This file is the persistent cross-account context. **Update it at the end of every future implementation or build session**, including the current working-tree status, latest stable commit/tag, tests, build artifact/checksum when applicable, remaining work, and the next continuation prompt. Keep the continuation prompt aligned with the actual repository state; do not leave stale claims about a build or completed feature.
