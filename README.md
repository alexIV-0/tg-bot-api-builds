# tg-bot-api-builds

CI-репозиторий для автосборки **[telegram-bot-api](https://github.com/tdlib/telegram-bot-api)**
(локальный Bot API server) под нужные ОС. Приложение **fs.manager** использует этот сервер,
чтобы скачивать/постить файлы больше лимитов облачного Bot API (20 МБ скачивание / 50 МБ
загрузка → до **2 ГБ** в обе стороны).

Готовые бинари публикуются в **Release с тегом `latest`** — оттуда приложение их скачивает
(или скачиваешь вручную и указываешь путь в настройках).

---

## Что внутри

```
.github/workflows/build.yml   ← CI: собирает бинари и кладёт в релиз
README.md                     ← этот файл
```

Workflow собирает **3 бинаря**:
| Файл в релизе | ОС / арх |
|---|---|
| `telegram-bot-api-macos-x86_64` | macOS Intel (под твои iMac) |
| `telegram-bot-api-macos-arm64` | macOS Apple Silicon |
| `telegram-bot-api-windows-x86_64.exe` | Windows x64 |

---

## Как поднять (один раз)

1. Создай на GitHub **новый ПУБЛИЧНЫЙ репозиторий**, напр. `tg-bot-api-builds`.
   > Публичный — намеренно. В бинарях **нет секретов** (это сборка открытого ПО; `api_id`/`api_hash`
   > в них НЕ зашиты — они вводятся в настройках приложения на каждой машине). Публичный = скачивание
   > по прямой ссылке без токена. Приватный потребовал бы зашить GitHub-токен в приложение — лишняя
   > возня без пользы.
2. Скопируй в него **этот** `.github/workflows/build.yml` (и при желании README.md).
3. Запушь в ветку **`main`**.
4. Сборка запустится автоматически. **Только `main`** триггерит сборку — другие ветки её НЕ запускают
   (см. `on.push.branches: [main]`). Ещё можно запустить вручную: вкладка **Actions → build-telegram-bot-api → Run workflow**.
5. Сборка идёт ~20–40 мин (компилируется tdlib). После успеха появится **Release `latest`** с тремя бинарями.

---

## Ссылки на скачивание (стабильные, тег `latest`)

```
https://github.com/<owner>/<repo>/releases/download/latest/telegram-bot-api-macos-x86_64
https://github.com/<owner>/<repo>/releases/download/latest/telegram-bot-api-macos-arm64
https://github.com/<owner>/<repo>/releases/download/latest/telegram-bot-api-windows-x86_64.exe
```
(подставь свой `<owner>/<repo>`). Эти URL и пропишем в авто-загрузку приложения (как ffmpeg),
либо качаешь вручную и указываешь путь к бинарю в настройках fs.manager.

---

## Обновление версии telegram-bot-api

Workflow клонирует **последний** `main` из `tdlib/telegram-bot-api` при каждом запуске. Чтобы
пересобрать на свежую версию — просто запусти workflow заново (push в main или Run workflow).
Если нужна **воспроизводимость** — запинь конкретный тег: в `build.yml` в шаге
`Clone telegram-bot-api` добавь `--branch <tag>` к `git clone`.

---

## Заметки / возможные правки

- **macOS**: сборка обычно проходит без правок (`brew install gperf cmake openssl`, OPENSSL через
  `brew --prefix openssl`). `macos-13` = Intel-раннер (x86_64), `macos-14` = ARM.
- **Windows**: самая капризная часть (vcpkg: `gperf`, `openssl`, `zlib` + toolchain-file). Если упадёт —
  смотри лог шага `Build (Windows)`; обычно дело в vcpkg-портах или путях. Это ожидаемо и правится точечно.
- **Запуск бинаря** требует `api_id` + `api_hash` (с https://my.telegram.org) — они задаются в
  настройках приложения, НЕ в репозитории и НЕ в бинаре.
- **macOS Gatekeeper**: скачанный неподписанный бинарь может требовать снятия карантина —
  `xattr -dr com.apple.quarantine <путь>` (приложение может делать это автоматически при установке).

---

## Как приложение это запускает (для справки)

```
telegram-bot-api --local --api-id=<API_ID> --api-hash=<API_HASH> --http-port=8081 --dir=<рабочая_папка>
```
`--local` снимает лимиты и заставляет `getFile` отдавать локальный путь (скачивания нет — move).
Приложение поднимает сервер при старте обработки и гасит на Stop.
