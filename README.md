<div align="center">

# رقيب · Raqeeb

**Control your Windows PC from your phone — end-to-end encrypted, no open ports.**
**تحكّم بجهاز Windows من جوالك — تشفير طرف-لطرف، بلا منافذ مفتوحة.**

![Platform](https://img.shields.io/badge/platform-Windows%2010%2F11-0078D6)
![.NET](https://img.shields.io/badge/.NET-8-512BD4)
![License](https://img.shields.io/badge/license-MIT-green)
![Encryption](https://img.shields.io/badge/E2E-AES--256--GCM-blueviolet)

[English](#english) · [العربية](#العربية)

</div>

---

## English

Raqeeb is a small Windows tray app that lets you see and control your PC from a phone web app (PWA) through a **blind relay server**. The relay only forwards encrypted bytes — it cannot read or forge commands. This repository contains everything: the Windows agent, the relay, and the phone app.

### Features

| | |
|---|---|
| 🖥 **Live screen** | Adaptive-quality preview, works on the lock screen with the helper |
| 🖱 **Remote input** | Touch-pad mouse, keyboard, shortcuts, text, scroll |
| 🔓 **Unlock** | Unlock Windows from the phone (optional helper service) |
| ☕ **Keep awake** | Prevent sleep / screen-off with periodic reminders and a max duration |
| 🕶 **Privacy cover** | Black out the laptop screen while you keep seeing the real one |
| 📁 **Files** | Browse, search, download, upload, open, fetch a URL to the PC, send to Telegram |
| ⚡ **Actions** | Lock, sleep, hibernate, shutdown/restart (cancellable), kill process, notify |
| 📊 **Status** | CPU, RAM, disks, battery, processes, suspicious connections |
| 🔐 **Phone lock** | Fingerprint / face app-lock (WebAuthn) |
| 🖧 **PowerShell** | Optional, off by default |

### Architecture

```
 Phone (PWA) ──HTTPS──▶  Relay server  ◀──outbound WebSocket── Raqeeb.exe (your PC)
      └───────── AES-256-GCM commands; the relay never has the key ─────────┘
```

| Path | What it is |
|---|---|
| `src/Raqeeb` | Windows tray agent (.NET 8, WinForms) |
| `src/Raqeeb.Helper` | Optional LocalSystem service for the lock screen |
| `relay/` | Relay server (FastAPI, single file) + Docker |
| `pwa/` | Phone web app |
| `tests/` | End-to-end test (real relay + real exe + simulated phone) |

### Quick start

**1. PC** — download `Raqeeb.exe` from [Releases](../../releases) and run it once. It creates `%APPDATA%\Raqeeb\.env` with a random master secret. From the tray icon choose **Pair phone** and copy `RAQEEB_RELAY_AUTH_SHA256` (or run `Raqeeb.exe --pair`).

**2. Relay** — on any server behind HTTPS:

```bash
pip install -r relay/requirements.txt
RAQEEB_RELAY_AUTH_SHA256=<value> uvicorn raqeeb_relay:app --app-dir relay --host 0.0.0.0 --port 8080 --no-access-log
# or
RAQEEB_RELAY_AUTH_SHA256=<value> docker compose up -d --build
```

Optional Telegram notifications/files: `RAQEEB_TG_BOT_TOKEN`, `RAQEEB_TG_CHAT_ID`. Put it behind Caddy, nginx or a Cloudflare Tunnel — never expose it without HTTPS.

**3. Connect** — in `.env` set `RAQEEB_RELAY_URL=https://your-relay` and `RAQEEB_PIN=`, then **Reload settings** from the tray. Choose **Pair phone** and scan the QR code with your phone camera. Install the page to your home screen and enable the fingerprint lock.

The pairing link carries the secret after the `#`, so it is never sent to any server; it is stored only in the phone's browser.

### Configuration (`.env`)

See [`.env.example`](.env.example). Highlights:

| Key | Default | Meaning |
|---|---|---|
| `RAQEEB_RELAY_URL` | – | Your relay's public URL |
| `RAQEEB_SECRET` | generated | 32-byte master secret (hex) |
| `RAQEEB_PIN` | – | PIN required for every action (min 4, 6+ recommended) |
| `RAQEEB_ALLOW_ACTIONS` / `_SCREENSHOT` / `_INPUT` / `_FILES` | `true` | Capability switches |
| `RAQEEB_ALLOW_UNLOCK` / `_SHELL` | `false` | High-risk capabilities |
| `RAQEEB_FILE_ROOTS` | defaults | `;`-separated folders allowed for file access |
| `RAQEEB_NOTIFY_TELEGRAM` | `true` | Telegram alert on sensitive commands |

### Security model

- Everything is off by default except read-only status; every action needs the **PIN** (PBKDF2); 5 wrong PINs lock actions for 15 minutes.
- Each command is AES-256-GCM sealed and bound to its id (AAD), with a 120 s freshness window and replay protection.
- The PC only makes **outbound** connections: no open ports, no public IP.
- File access is limited to allowed folders; keys, cookies, vaults and executables are blocked.
- The relay stores only `sha256(token)` and sees only ciphertext.
- The phone fingerprint lock is a local app lock; the cryptographic protection is the master secret plus the PIN.
- Keep `.env` private. If a device is lost, change `RAQEEB_SECRET` and re-pair.

> ⚠ Remote input and the unlock helper are equivalent to sitting at your computer. Enable them only on machines you own.

### Protocol

- Master secret `M` (32 bytes). Agent key = `HKDF-SHA256(M, info="control-panel/agent/v1")`; auth token = `hex(HKDF-SHA256(M, info="control-panel/web-auth/v1"))`.
- Agent: `GET {relay}/agent/ws?t=<token>&d=<deviceId>&n=<name>` (outbound WebSocket).
- Phone: `POST {relay}/api/agent/cmd` with `{device, id, ct}` → `{ct}`; `GET /api/agent/devices`.
- `ct = nonce(12) ‖ AES-GCM(key, JSON{op,args,pin,ts}) ‖ tag(16)`, AAD = `"cmd:"+id` / `"res:"+id`.
- Telegram: `POST /api/ql/notify`, `POST /api/ql/sendfile` (Bearer token).

### Build & test

```powershell
powershell -ExecutionPolicy Bypass -File publish.ps1      # dist\Raqeeb.exe + RaqeebHelper.exe
pip install fastapi "uvicorn[standard]" cryptography
python tests/e2e_relay.py dist\Raqeeb.exe                  # end-to-end test
```

Requirements: .NET 8 SDK, Windows 10 (2004+) or 11.

### Status

Early preview (0.2). Verified by the end-to-end test: connection, status, PIN/replay/key checks, screenshots, file roots, keep-awake, capability gating. Still to be validated on real hardware: injected input, privacy cover, clipboard-to-Telegram, helper install and real lock-screen unlock.

### License

[MIT](LICENSE)

---

## العربية

**رقيب** برنامج صغير لويندوز (في صينية النظام) يتيح لك رؤية جهازك والتحكم به من جوالك عبر تطبيق ويب (PWA) وخادم وسيط **أعمى**: يمرّر بايتات مشفّرة فقط ولا يستطيع قراءتها أو تزويرها. المستودع يحوي كل شيء: وكيل ويندوز، والخادم الوسيط، وتطبيق الجوال.

### الميزات

- **معاينة حيّة للشاشة** (حتى شاشة القفل مع المساعد)
- **فأرة وكيبورد** من الجوال (تراك باد باللمس)
- **فتح قفل ويندوز** من الجوال (اختياري)
- **منع السكون وإطفاء الشاشة** مع تذكير وحد أقصى للمدة
- **غطاء خصوصية** للشاشة (أنت ترى الحقيقية والجالس بجانبك يرى الأسود؛ إزالة محلية `Ctrl+Alt+Shift+P`)
- **ملفات:** تصفح، بحث، تنزيل، رفع، فتح، تنزيل رابط إلى الجهاز، إرسال إلى تلغرام
- **إجراءات:** قفل، سكون، سبات، إيقاف/إعادة تشغيل (قابلة للإلغاء)، إنهاء برنامج
- **حالة الجهاز:** المعالج، الذاكرة، الأقراص، البطارية، العمليات، الاتصالات المشبوهة
- **قفل التطبيق بالبصمة/الوجه** على الجوال
- **PowerShell** (اختياري، معطّل افتراضيًا)

### التشغيل السريع

1. **اللابتوب:** حمّل `Raqeeb.exe` من [Releases](../../releases) وشغّله مرة؛ ينشئ `%APPDATA%\Raqeeb\.env` بسرّ رئيسي عشوائي. من أيقونة الصينية ← **اقتران الجوال** ← انسخ `RAQEEB_RELAY_AUTH_SHA256` (أو `Raqeeb.exe --pair`).
2. **الخادم الوسيط** (أي خادم خلف HTTPS): `pip install -r relay/requirements.txt` ثم
   `RAQEEB_RELAY_AUTH_SHA256=<القيمة> uvicorn raqeeb_relay:app --app-dir relay --host 0.0.0.0 --port 8080 --no-access-log`
   أو `docker compose up -d --build`.
3. **الربط:** في `.env` اكتب `RAQEEB_RELAY_URL=https://عنوان-خادمك` و`RAQEEB_PIN=` ثم **إعادة تحميل الإعدادات**، وامسح رمز QR من **اقتران الجوال** بكاميرا الجوال. ثبّت الصفحة على الشاشة الرئيسية وفعّل القفل بالبصمة.

رابط الاقتران يحمل السرّ بعد `#` فلا يصل إلى أي خادم، ويُحفظ في متصفح الجوال فقط.

### الأمان

- كل شيء مغلق افتراضيًا عدا القراءة؛ كل إجراء يتطلب **PIN**؛ 5 أخطاء تقفل الإجراءات 15 دقيقة.
- كل أمر مشفّر AES-256-GCM ومربوط بمعرّفه، مع نافذة زمنية 120 ثانية ومنع إعادة التشغيل.
- الجهاز يتصل «للخارج» فقط: لا منافذ مفتوحة ولا IP عام.
- الملفات محصورة بمجلدات مسموحة، والمفاتيح والكوكيز والتنفيذيات محجوبة.
- الخادم لا يحفظ إلا `sha256(token)` ولا يرى إلا نصًا مشفّرًا.
- احفظ `.env` سرّيًا. عند ضياع جهاز غيّر `RAQEEB_SECRET` وأعد الاقتران.

> ⚠ التحكم بالفأرة ومساعد الفتح يعادلان الجلوس أمام جهازك: فعّلهما على أجهزتك أنت فقط.

### الحالة

نسخة تجريبية مبكرة (0.2). اختبار طرف-لطرف يغطي: الاتصال، الحالة، فحوص PIN والتكرار والمفتاح، لقطة الشاشة، الملفات، منع السكون، وبوابة الصلاحيات. لم تُختبر بعد على عتاد حقيقي: حقن الإدخال، غطاء الخصوصية، الحافظة إلى تلغرام، تثبيت المساعد وفتح قفل حقيقي.

### الترخيص

[MIT](LICENSE)
