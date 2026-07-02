# АварМи для Android — PLAN.md

Полная инструкция: как сделать Android-версию словаря (аварско-русский и
русско-аварский) и опубликовать в Google Play. Зеркало того, что уже сделано
для iOS, с учётом специфики Android/Google Play.

> iOS-репозитории (`avar/ios`, `avarme-ios`) **не трогаем**. Всё новое — только
> в `../android/` (этот репозиторий) и `../avarme-android/`.

---

# 1. Схема

```
sources.avar.me (av-ru.jsonl + ru-av.jsonl)   ← те же исходники, что у iOS
      ↓
GitHub Actions (в репо android)
      ↓
dictionary.sqlite (тот же формат) → GitHub Pages
      ↓
https://android.avar.me   (лендинг + latest.json + releases/*.zip)
      ↓
Android-приложение скачивает обновления словаря
```

## Два репозитория

- **`avar-me/android`** (этот, **публичный**) — сайт-промо (ссылка на Google Play,
  QR), privacy policy, data-pipeline, публикуемые архивы. Хостинг: **GitHub Pages**,
  домен `android.avar.me` (CNAME; DNS `avar.me` на Cloudflare — как для iOS).
- **`avar-me/avarme-android`** (**приватный, без GitHub Pages**) — код приложения
  (Kotlin/Compose), Gradle-проект, скрипты, keystore-инструкции.

## Данные — те же самые
Формат словаря (`dictionary.sqlite` с таблицами `entries`, `search`, `meta`)
**идентичен** iOS — SQLite кроссплатформенный. Пайплайн — копия iOS-скриптов
(Python 3.12, без внешних зависимостей). Нормализация (палочка) — та же логика.

Возможны два варианта данных (выбрать один):
- **A (рекомендуется, самодостаточно):** в репо `android` лежит копия пайплайна
  iOS (`scripts/`), публикует на `android.avar.me`. Полная независимость.
- **B (проще, но связность):** Android-приложение ходит за данными на
  существующий `ios.avar.me`. Меньше кода, но платформы связаны. По ТЗ выбираем **A**.

---

# 2. Data pipeline (репозиторий `android`)

Практически копия iOS. Структура:

```
android/
├── README.md
├── PLAN.md                      # этот файл
├── scripts/                     # копия из avar/ios/scripts (Python 3.12)
│   ├── config.py                # SITE_BASE_URL = https://android.avar.me
│   ├── normalize.py             # без изменений (палочка)
│   ├── download_sources.py
│   ├── analyze_sources.py
│   ├── build_dictionary.py      # тот же формат SQLite
│   ├── validate_dictionary.py
│   ├── package_release.py
│   └── generate_latest.py
├── tests/test_normalization.py
├── public/                      # GitHub Pages → android.avar.me
│   ├── CNAME                    # android.avar.me
│   ├── index.html               # лендинг: Google Play badge + QR
│   ├── privacy/index.html       # privacy policy (можно тот же текст, что iOS)
│   ├── assets/                  # icon.png, qr.png
│   ├── latest.json
│   ├── releases/dictionary-vN.zip
│   └── checksums/dictionary-vN.sha256
└── .github/workflows/build-and-publish.yml   # с авто-инкрементом версии и встроенным деплоем
```

Взять готовые скрипты из `avar/ios/scripts` (скопировать), поменять в `config.py`
`SITE_BASE_URL` на `https://android.avar.me`. Workflow — копия финальной
iOS-версии (авто-инкремент версии + встроенный `deploy-pages`, устойчивый push).

**Формат `latest.json` и архива — без изменений** (см. iOS). Android-приложение
читает те же поля: `dictionary_version`, `package_url`, `package_sha256`,
`package_size_bytes`, `min_app_version`, `sources[]`.

## GitHub Pages
Как для iOS: `build_type=workflow`, публикует `public/` через `deploy-pages`.
Домен `android.avar.me` — добавить CNAME-запись в Cloudflare на `avar-me.github.io`
и файл `public/CNAME` с `android.avar.me`.

---

# 3. Приложение (репозиторий `avarme-android`)

## 3.1. Стек
- **Язык:** Kotlin
- **UI:** Jetpack Compose + Material 3 (аналог SwiftUI)
- **Архитектура:** MVVM (ViewModel + StateFlow), корутины
- **БД:** системный SQLite через `android.database.sqlite` (без сторонних зависимостей;
  Room не берём, т.к. БД собрана вне Room и имеет свою схему `search`)
- **Сеть:** `HttpURLConnection` или `OkHttp` (можно stdlib, чтобы без зависимостей)
- **ZIP:** `java.util.zip.ZipInputStream` (встроен — в отличие от iOS, свой распаковщик НЕ нужен)
- **Хэш:** `java.security.MessageDigest` (SHA-256)
- **Сборка:** Gradle (Kotlin DSL), Android Studio

## 3.2. Версии SDK
- `minSdk = 24` (Android 7; покрывает ~99% устройств)
- `compileSdk` / `targetSdk` = **последний требуемый Google Play** (на 2025 — API 35 /
  Android 15; проверить актуальное требование перед релизом — Play требует таргетить
  свежий API в течение ~года после релиза версии Android).

## 3.3. Структура проекта
```
avarme-android/
├── settings.gradle.kts
├── build.gradle.kts
├── gradle/…  gradlew  gradlew.bat
├── scripts/sync_dictionary.sh       # копирует dictionary.sqlite из ../android/build
├── keystore/README.md               # как создать upload keystore (сам keystore НЕ коммитить)
├── store/                           # метаданные Google Play, ответы на вопросы
│   ├── listing.md
│   ├── data-safety.md
│   └── screenshots.md
└── app/
    ├── build.gradle.kts
    ├── proguard-rules.pro
    └── src/main/
        ├── AndroidManifest.xml
        ├── assets/dictionary/
        │   ├── dictionary.sqlite    # bundled (НЕ в git — синкается скриптом)
        │   └── metadata.json
        ├── res/                     # adaptive icon (mipmap), themes, strings
        └── java/me/avar/dictionary/
            ├── App.kt
            ├── data/
            │   ├── DictionaryEntry.kt      # модели (data classes) под JSON entries.data
            │   ├── DictionaryStore.kt      # открытие SQLite, поиск
            │   ├── DictionaryProvider.kt   # выбор bundled/downloaded
            │   ├── DictionaryUpdater.kt    # проверка/скачивание/установка
            │   ├── ChecksumUtil.kt         # SHA-256
            │   └── TextNormalizer.kt       # нормализация + палочка (зеркало normalize.py)
            ├── ui/
            │   ├── search/SearchScreen.kt + SearchViewModel.kt
            │   ├── detail/EntryDetailScreen.kt
            │   └── settings/SettingsScreen.kt
            └── MainActivity.kt
```

## 3.4. Чтение bundled-словаря
SQLite не может открыть файл прямо из `assets/` (это не обычный путь). Поэтому:
1. При первом запуске **копируем** `assets/dictionary/dictionary.sqlite` в
   `filesDir/dictionary/bundled/` (если ещё не скопирован / версия изменилась).
2. Открываем `SQLiteDatabase.openDatabase(path, null, OPEN_READONLY)`.
3. Скачанный словарь (обновление) кладём в `filesDir/dictionary/current/` и
   предпочитаем его bundled.

## 3.5. Поиск (тот же принцип, что iOS)
Таблица `search(lang, term, entry_id, weight)`. Запрос:
- нормализуем ввод (для аварского — складываем палочку: `ӏ Ӏ 1 l I |` → `ӏ`);
- точное совпадение + префикс: `WHERE lang=? AND term>=? AND term<?` (диапазон с
  верхней границей `term + ჿFF`), сорт: exact desc, weight asc;
- `avarToRussian` → `lang='av'`, `russianToAvar` → `lang='ru'`.

`TextNormalizer.kt` обязан давать **тот же** результат, что `normalize.py` и
iOS-`TextNormalizer.swift` (проверить на тех же кейсах: `маг1на`→`магӏна`, `ё→е`).

## 3.6. Карточка слова
Рендерим как на iOS: заголовок (+ `exclamation`, `precomment` статьи), значения
(precomment над переводом, перевод, comment под, labels-чипы), примеры (сворачиваемые,
превью первого; внутри — av/ru, comment курсивом, labels), «см. также» и грамм-связи
как кликабельные переходы по `target_id`.

## 3.7. Обновление словаря
Endpoint: `https://android.avar.me/latest.json`. Алгоритм (как iOS, но проще с ZIP):
1. Скачать `latest.json`; если `dictionary_version <= локальной` — ничего.
2. Проверить `min_app_version <= versionName`.
3. Скачать `package_url` во временный файл.
4. Проверить **SHA-256** = `package_sha256`.
5. Распаковать (`ZipInputStream`) во временную папку.
6. Проверить `metadata.json` (schema_version, версия) и что SQLite открывается.
7. Переместить `current`→`previous`, новый→`current` (атомарно через rename).
8. Сохранить дату проверки в `SharedPreferences`.
При любой ошибке — остаться на текущем словаре. Только `https://android.avar.me/`.
Проверка при запуске, если прошло >24ч; ручная кнопка в настройках.
(Фоновое обновление — `WorkManager`, опционально, позже.)

---

# 4. Иконка
- **В приложении:** adaptive icon (`res/mipmap-*/ic_launcher` + `ic_launcher_foreground`/
  `_background` в `mipmap-anydpi-v26`). Взять арт `avar/avar.me.ios.png` (или новый),
  сделать foreground/background слои. Google накладывает маску (круг/сквиркл).
- **Для Play (store listing):** иконка **512×512**, 32-bit PNG (альфа разрешена, в
  отличие от App Store). Кладём в `store/`.

---

# 5. Подпись
- **Upload keystore:** сгенерировать `keytool -genkeypair -v -keystore upload.jks
  -keyalg RSA -keysize 2048 -validity 10000 -alias upload`. **Хранить надёжно, НЕ
  коммитить.** Пароли — в `~/.gradle/gradle.properties` или переменных окружения.
- **Play App Signing:** включить в консоли (Google хранит app signing key; ты грузишь
  подписанный upload-ключом AAB). Это стандарт и безопаснее (потеря upload-ключа
  восстановима).
- В `app/build.gradle.kts` — `signingConfigs { create("release") { … } }`, читать
  пароли из `gradle.properties` (в `.gitignore`).

---

# 6. Сборка релиза
- Формат: **AAB** (`.aab`, Android App Bundle) — обязателен для Play (APK только для
  сайдлоада/тестов). `./gradlew bundleRelease`.
- **versionCode** — целое, **растёт с каждой загрузкой** (1, 2, 3…).
- **versionName** — «1.0.0» (человекочитаемая).
- Перед сборкой: `scripts/sync_dictionary.sh` — вшить свежий `dictionary.sqlite`.
- `applicationId = me.avar.dictionary` (можно тот же, что iOS bundle id; в Play он
  неизменен после публикации).

---

# 7. Google Play Console
1. **Аккаунт разработчика:** регистрация, разовый взнос **$25** (не ежегодный, в
   отличие от Apple). Верификация личности/адреса (Google требует, аналог DSA).
2. **Создать приложение:** имя «АварМи», язык, тип «Приложение», бесплатное.
3. **Package name** = `me.avar.dictionary` (задаётся первым загруженным AAB, неизменно).
4. **Store listing:**
   - Название (≤30), краткое описание (**≤80**), полное описание (**≤4000**).
   - **Иконка 512×512**, **feature graphic 1024×500** (обязателен!),
     скриншоты телефона (мин. 2; напр. 1080×1920), планшета (желательно).
   - Категория: Education (или Books & Reference).
5. **Privacy policy URL:** `https://android.avar.me/privacy` (текст — как у iOS).
6. **Data safety** (аналог App Privacy): «No data collected / not shared». Заполнить форму.
7. **Content rating:** пройти анкету IARC → получить рейтинг (будет «Everyone / 3+»).
8. **Target audience & content:** не для детей специально; аудитория 13+/18+ на выбор.

---

# 8. ⚠️ Тестирование перед продакшеном (важно для новых аккаунтов!)
Для **персональных** (individual) аккаунтов, созданных после 13 ноября 2023, Google
требует **закрытое тестирование**: минимум **12 тестировщиков**, которые оставались
подписанными **≥14 дней**, прежде чем откроется доступ к production. Это дольше и
муторнее, чем ревью Apple — заложить время и людей.

Треки релиза: **Internal testing** (быстро, до 100 тестеров, без 14-дневного правила
для самого билда) → **Closed testing** (те самые 12 на 14 дней) → **Open testing** →
**Production**. Организационный (company) аккаунт от этого требования освобождён —
если планируешь много приложений, возможно, стоит org-аккаунт. **Проверить актуальное
правило в консоли перед стартом.**

---

# 9. Отличия от iOS (сводка)
| | iOS | Android |
|---|---|---|
| Взнос | $99/год | $25 разово |
| Формат сборки | Archive → .ipa | AAB (.aab) |
| Распаковка ZIP | свой распаковщик | `ZipInputStream` (встроен) ✅ |
| Подпись | Xcode automatic + Team ID | upload keystore + Play App Signing |
| Проект | XcodeGen (генерится) | Gradle (коммитим целиком) |
| Публикация данных | ios.avar.me | android.avar.me |
| Приватность | App Privacy | Data Safety |
| Проверка | App Review ~1–2 дня | Closed testing 12×14 дней (новые акк.) + ревью |
| Иконка store | 1024×1024 без альфы | 512×512 (альфа ок) + feature graphic 1024×500 |
| Мин. версия | iOS 18 | minSdk 24, target = свежий API |

---

# 10. Roadmap (по фазам, как делали iOS)
- **Phase 1:** репо `android` — скопировать пайплайн, `config.py` на android.avar.me,
  собрать `dictionary.sqlite`, поднять GitHub Pages + DNS `android.avar.me`, лендинг, privacy.
- **Phase 2:** `avarme-android` — Gradle+Compose проект, bundled SQLite, поиск (палочка),
  карточка слова, переключатель направления, случайные слова. Запуск на устройстве/эмуляторе.
- **Phase 3:** обновление словаря (latest.json → SHA-256 → unzip → атомарная установка),
  экран настроек с версией и кнопкой.
- **Phase 4:** избранное, история, `WorkManager` фоновое обновление.
- **Phase 5:** подпись, AAB, Play Console, listing, data safety, content rating,
  closed testing (12×14) → production.

---

# 11. Что нужно от тебя
1. **GitHub:** создать репозитории `avar-me/android` (public) и `avar-me/avarme-android`
   (**private**).
2. **DNS:** в Cloudflare добавить `android.avar.me` → GitHub Pages (CNAME на `avar-me.github.io`).
3. **Google Play:** аккаунт разработчика ($25), решить individual vs organization
   (влияет на требование 12×14 тестеров).
4. **Инструменты:** Android Studio (последняя), JDK 17+.
5. **Ассеты:** иконку (можно тот же арт), при желании — feature graphic 1024×500.
6. **Решения:** applicationId (`me.avar.dictionary`?), домен (`android.avar.me`?),
   вариант данных A/B (по умолчанию A).

---

# 12. Первый шаг
Рекомендую начать с **Phase 1** (данные + сайт) — как на iOS: это стабильный контракт,
на который потом сядет приложение. Дальше — Gradle/Compose приложение.
