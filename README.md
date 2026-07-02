# avar-me/android

Публичный репозиторий: сайт-промо + data-pipeline + публикуемые архивы словаря
для Android-приложения **АварМи** (аварско-русский / русско-аварский).

- Хостинг: **GitHub Pages**, домен `android.avar.me` (CNAME), DNS в Cloudflare.
- `android.avar.me` отдаёт лендинг (ссылка на Google Play + QR), privacy policy
  и endpoint обновлений словаря (`latest.json` + `releases/*.zip`).
- Приложение (Kotlin/Compose) — в приватном репозитории `avar-me/avarme-android`.

Полная инструкция (обе части, весь путь до Google Play) — в [`PLAN.md`](PLAN.md).

Данные словаря те же, что у iOS (формат `dictionary.sqlite` идентичен). Пайплайн —
копия iOS-скриптов; в `scripts/config.py` меняется только `SITE_BASE_URL` на
`https://android.avar.me`.

> iOS-репозитории (`avar/ios`, `avarme-ios`) в этой работе не изменяются.
