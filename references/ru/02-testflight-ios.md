# Сборка iOS и TestFlight

Цикл, который мы повторяем на каждой сборке. Примерно 15–40 минут; большинство ловушек ниже стоят целого круга, если их пропустить.

## Проверь две вещи до сборки

Обе дёшевы в проверке и дороги в обнаружении постфактум.

1. **Этот номер сборки уже занят?** `GET /v1/builds?filter[app]={id}` — дубликат отклоняется при загрузке.
2. **Текущая строка версии уже опубликована?** Если версия в App Store в состоянии `READY_FOR_SALE`, её релизный поезд закрыт, и загрузка падает с **90186** / **ITMS-90062**. Нужно поднять **строку версии**, а не только номер сборки, и пересобрать: версия вкомпилирована в IPA.

На одном релизе мы потеряли на этом четыре сборки.

## Где на самом деле живёт версия

- **Managed workflow:** `app.json` → `expo.version` и `expo.ios.buildNumber`.
- **Bare workflow:** `ios/<App>/Info.plist` → `CFBundleShortVersionString` и `CFBundleVersion`, плюс `MARKETING_VERSION` в pbxproj. **`app.json` игнорируется.**

Если правишь `app.json` скриптом — распарси и сериализуй заново (`json.load` / `json.dump`). Чтение и запись через один и тот же дескриптор обрезает файл: это стоило нам сборки.

## Сборка

```bash
npx tsc --noEmit
npx expo export --platform ios --output-dir /tmp/exportcheck   # ошибки импорта до archive

LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8 \
  eas build --platform ios --profile production --local \
  --non-interactive --output ./app-buildNN.ipa
```

Префикс `LANG` не опционален. На Ruby 4.0 с CocoaPods 1.16 `pod install` умирает с `Unicode Normalization not appropriate for ASCII-8BIT`, особенно если в проекте есть не-ASCII символы.

Прямой `xcodebuild` тоже работает и служит запасным выходом, когда учётные данные EAS устарели:

```bash
LANG=en_US.UTF-8 xcodebuild -workspace App.xcworkspace -scheme App \
  -configuration Release -archivePath /tmp/AppNN.xcarchive \
  -allowProvisioningUpdates \
  -authenticationKeyPath ~/.appstoreconnect/private_keys/AuthKey_<KEY_ID>.p8 \
  -authenticationKeyID <KEY_ID> -authenticationKeyIssuerID <ISSUER_ID> archive
```

Ищи `ARCHIVE SUCCEEDED`, затем `-exportArchive` с `ExportOptions.plist`, где `method=app-store` и `signingStyle=automatic`.

### Пока идёт сборка

**Не редактируй исходники во время archive.** Metro вкомпилирует недописанный бандл, и приложение упадёт при запуске. Это не теория: стоило нам сборки и выглядело как загадочный краш.

## Загрузка

```bash
xcrun altool --upload-app -f ./app-buildNN.ipa -t ios \
  --apiKey <KEY_ID> --apiIssuer <ISSUER_ID>
```

Положи `.p8` в `~/.appstoreconnect/private_keys/AuthKey_<KEY_ID>.p8` — altool найдёт его там. Занимает около 15 секунд.

**Предпочитай это `eas submit`.** Мы видели, как `eas submit` висит 23 минуты без вывода и без загрузки, и падает с «Unable to upload archive. Failed to authenticate for session». altool показывает настоящую ошибку.

## После загрузки

**`UPLOAD SUCCEEDED` не значит «принято».** Опрашивай ASC API, пока сборка не станет `VALID`; Apple отклоняет и на обработке. Как только валидна — пометь предыдущую сборку истёкшей, чтобы тестировщики видели только новую:

```
PATCH /v1/builds/{id}   {"expired": true}
```

## Ошибки загрузки, которые стоит узнавать

| Код | Значение |
|---|---|
| **90062** / ITMS-90062 | Эта версия уже опубликована — подними строку версии |
| **90186** | Предрелизный поезд закрыт — та же причина |
| **90713** | У таргета нет `CFBundleIconName` — Watch и виджеты требуют собственную иконку |
| **ITMS-90863** | Предупреждение о символах Apple Silicon. **Нормально для Expo-приложений, не отказ.** Игнорируй. |

## Дополнительные таргеты

Таргеты Watch и Live Activity требуют собственных provisioning-профилей в `credentials.json`, все под одним distribution-сертификатом. Для архивации Watch-таргета на Mac нужна платформа watchOS **для устройств** — SDK симулятора не хватит:

```bash
xcodebuild -downloadPlatform watchOS    # ~4 ГБ
```

**Тестируй собственные сценарии дополнительного таргета.** Вход на нашем Watch-таргете не работал никогда — он отправлял `email`, а бэкенд читал `identifier` — и Apple нашла это раньше нас, отказом по 2.1.

## Когда Xcode обновляется посреди проекта

Автообновление оставляет сборки падать с `iOS <версия> Platform Not Installed`, даже если тем же утром всё собиралось:

```bash
xcodebuild -downloadPlatform iOS   # ~8,5 ГБ, sudo не нужен
xcodebuild -runFirstLaunch
```

## Уборка

Временные файлы EAS растут неограниченно — у нас `/var/folders/.../eas-cli-nodejs` дорос до 35 ГБ. Полный диск роняет сборку с `No space left`. Чисти между релизами. То, что номера сборок перескакивают после неудачных попыток, — нормально.

Дальше: `04-app-store-submission.md`.
