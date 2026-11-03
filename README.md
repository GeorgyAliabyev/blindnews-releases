# BlindNews — релизы

Здесь только собранные APK и списки изменений. Исходного кода в репозитории нет.

- **Последняя версия:** [Releases → latest](https://github.com/GeorgyAliabyev/blindnews-releases/releases/latest)
- **Машиночитаемый манифест обновлений:** [`update.json`](update.json)
- **История изменений:** [`CHANGELOG.md`](CHANGELOG.md)
- **Как приложение проверяет обновления:** [`docs/UPDATE_INTEGRATION_RU.md`](docs/UPDATE_INTEGRATION_RU.md)

## Что здесь появляется с каждым релизом

1. APK как ассет GitHub Release с тегом `v<versionName>` (например `v1.1-dev2`).
2. Запись в `CHANGELOG.md`.
3. Обновлённый `update.json` — по нему приложение узнаёт о новой версии.

## Прямые ссылки

Последний APK без указания версии:

```
https://github.com/GeorgyAliabyev/blindnews-releases/releases/latest/download/BlindNews.apk
```

Манифест, отдаваемый через CDN без лимитов API:

```
https://raw.githubusercontent.com/GeorgyAliabyev/blindnews-releases/main/update.json
```
