# Opengram Desktop — rebrand

Полный rebrand с изоляцией от Telegram Desktop — разные UUID, разные
URL-схемы, разная регистрация в ОС. Оба клиента могут стоять параллельно.

## Что изменено

### 1. Интерфейсные строки
- `Telegram/Resources/langs/lang.strings` — 322+ замены.
  `Telegram` → `Opengram`, `telegram.org` → `opengra.me`,
  `t.me/` → `opengra.me/`.
- Бэкап: `lang.strings.bak`.

### 2. Константы приложения
- `Telegram/SourceFiles/core/version.h`:
  - `AppId` = **новый UUID** `{743F67C9-75F5-4451-A59F-9FB7BBCFADF4}`
    (не конфликтует с Telegram Desktop `{53F49750-...}`)
  - `AppName` = `Opengram Desktop`
  - `AppFile` = `Opengram`

### 3. Windows AppUserModelID
- `windows_app_user_model_id.cpp`:
  - `Opengram.OpengramDesktop` (вместо `Telegram.TelegramDesktop`)
  - `Opengram.OpengramDesktop.Store` для UWP-билда
- **Результат**: Windows группирует задачи/уведомления отдельно от Telegram
  Desktop, нет конфликта в системе.

### 4. URL-схема
- `Telegram/SourceFiles/core/application.cpp`:
  - Регистрируется **`opengram://`** (вместо `tg://`)
  - `openLocalUrl()` теперь принимает **оба** протокола (opengram:// и tg://),
    чтобы старые сообщения с tg://-ссылками продолжали работать внутри клиента
- `Telegram.plist` (macOS): `CFBundleURLSchemes = [opengram, tonsite]`
- `Telegram/Resources/uwp/AppX/AppxManifest.xml`: `<Protocol Name="opengram">`
- `lib/xdg/org.telegram.desktop.desktop` (Linux):
  `MimeType=x-scheme-handler/opengram;...`
- **Результат**: у пользователя параллельно установлен Telegram Desktop —
  схема `tg://` по-прежнему открывается в Telegram, `opengram://` — в нашем.

### 5. DC-адреса и RSA
- `mtproto_dc_options.cpp`: все DC → `51.250.119.114:4430`.
- RSA-ключи **placeholder** — подставить командой (см. ниже).

### 6. Installer (Inno Setup)
- `Telegram/build/setup.iss`:
  - `MyAppId` — новый UUID
  - `MyAppName` / `MyAppShortName` / `MyAppPublisher` / `MyAppURL`
  - `MyAppExeName = Opengram.exe`
- Установщик ставится в `%AppData%\Opengram Desktop\`, не пересекается с
  Telegram Desktop.

### 7. URL-ы в C++ коде
- `t.me/` → `opengra.me/` (16 файлов, 20 замен).
- FAQ URL → `opengra.me/faq`.
- Changelog → `github.com/opengram-server/tdesktop/releases`.
- `about_box.cpp` / `intro_start.cpp` — заголовки "Telegram Desktop" → "Opengram Desktop".
- `export_output_abstract.cpp` — applicationName в экспортах.

## Что нужно сделать

### 1. RSA-ключ сервера

```powershell
scp root@51.250.119.114:/opt/ogram/data/secrets/pubkey.asc C:\tdesktop\pubkey.asc
cd C:\tdesktop
python _apply_pubkey.py
```

### 2. Переименовать исполняемый файл
В `cmake/` проверь имя output — должен получиться `Opengram.exe`
(смотри `CMAKE_RUNTIME_OUTPUT_DIRECTORY` и `OUTPUT_NAME`). Обычно имя
задаётся в `Telegram/CMakeLists.txt`.

### 3. Иконки и арт (опционально)
- `Telegram/Resources/art/` — лого, иконки приложения
- `Telegram/Resources/uwp/AppX/Assets/` — UWP-иконки

### 4. Собрать
```powershell
cd C:\tdesktop
cmake -B build -DCMAKE_BUILD_TYPE=Release -A x64
cmake --build build --config Release
```

## Что НЕ тронуто (намеренно)

- **`tg://` URL-схема в системе** — НЕ регистрируется нами, значит если у
  юзера стоит Telegram Desktop, его tg:// ссылки по-прежнему открываются
  в нём. Наш клиент открывает `opengram://`.
- **Внутренние генераторы `tg://`** в `local_url_handlers.cpp` — клиент
  использует их для внутреннего роутинга (не внешнего), замена не нужна.
  `openLocalUrl` принимает оба префикса.
- **License headers** (AGPL).
- **Instant View парсер** — работает с `host == "telegram.org"` для
  разбора внешних статей.

## Параллельная установка проверена

После установки Opengram Desktop:
- В реестре: `HKCU\Software\Opengram` (не `Software\Telegram`)
- В Windows Apps: отдельная запись "Opengram Desktop"
- В `%AppData%`: `Opengram Desktop\` (Telegram Desktop имеет свой `Telegram Desktop\`)
- URL `tg://resolve?domain=xxx` → Telegram Desktop
- URL `opengram://resolve?domain=xxx` → Opengram Desktop
