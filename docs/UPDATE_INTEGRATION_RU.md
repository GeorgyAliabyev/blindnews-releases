# Обновление BlindNews из GitHub — спецификация интеграции

Документ для того, кто вносит код в `blindnews`. Всё, что нужно, чтобы приложение
само узнавало о новой версии и предлагало диалог обновления.

## Как это работает

Приложение раз в несколько часов читает статический JSON-манифест:

```
https://raw.githubusercontent.com/GeorgyAliabyev/blindnews-releases/main/update.json
```

`raw.githubusercontent.com` отдаёт файл через CDN: без авторизации, без лимита
GitHub API (60 запросов в час на IP — за NAT оператора это ломается за минуты).
Дальше приложение сравнивает `versionCode` из манифеста со своим
`BuildConfig.VERSION_CODE`. Если в манифесте больше — показывает диалог, качает
APK по `apkUrl`, проверяет SHA-256 и отдаёт системному установщику через
`FileProvider`.

Токены в приложение не встраиваются, репозиторий публичный.

## Что нужно добавить

Четыре файла в пакет `ru.blindnews.app.update`, правка манифеста, один ресурс и
один вызов. Внешних библиотек не требуется — только `org.json`, `androidx.core`
(в проекте уже есть) и стандартный `HttpURLConnection`.

### 1. `res/xml/file_paths.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<paths>
    <cache-path name="updates" path="updates/" />
</paths>
```

### 2. `AndroidManifest.xml`

Внутри `<manifest>`, рядом с остальными разрешениями:

```xml
<uses-permission android:name="android.permission.REQUEST_INSTALL_PACKAGES" />
```

Внутри `<application>`, рядом с уже существующим `LogFileProvider`:

```xml
<provider
    android:name="androidx.core.content.FileProvider"
    android:authorities="${applicationId}.fileprovider"
    android:exported="false"
    android:grantUriPermissions="true">
    <meta-data
        android:name="android.support.FILE_PROVIDER_PATHS"
        android:resource="@xml/file_paths" />
</provider>
```

Authority намеренно отличается от `LogFileProvider` — два провайдера с разными
authorities в одном приложении живут спокойно.

### 3. `update/UpdateInfo.java`

```java
package ru.blindnews.app.update;

import org.json.JSONArray;
import org.json.JSONObject;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

/** Разобранный update.json. */
public final class UpdateInfo {

    public final int versionCode;
    public final String versionName;
    public final String publishedAt;
    public final boolean mandatory;
    public final int minSupportedVersionCode;
    public final int minSdk;
    public final String apkUrl;
    public final long apkSize;
    public final String apkSha256;
    public final List<String> changelog;
    public final String changelogUrl;
    public final String releaseUrl;

    private UpdateInfo(JSONObject o) {
        versionCode = o.optInt("versionCode", 0);
        versionName = o.optString("versionName", "");
        publishedAt = o.optString("publishedAt", "");
        mandatory = o.optBoolean("mandatory", false);
        minSupportedVersionCode = o.optInt("minSupportedVersionCode", 0);
        minSdk = o.optInt("minSdk", 0);
        apkUrl = o.optString("apkUrl", "");
        apkSize = o.optLong("apkSize", 0L);
        apkSha256 = o.optString("apkSha256", "").toLowerCase();
        changelogUrl = o.optString("changelogUrl", "");
        releaseUrl = o.optString("releaseUrl", "");

        List<String> items = new ArrayList<>();
        JSONArray arr = o.optJSONArray("changelog");
        if (arr != null) {
            for (int i = 0; i < arr.length(); i++) {
                String s = arr.optString(i, "").trim();
                if (!s.isEmpty()) items.add(s);
            }
        }
        changelog = Collections.unmodifiableList(items);
    }

    public static UpdateInfo parse(String json) throws Exception {
        return new UpdateInfo(new JSONObject(json));
    }

    public boolean isNewerThan(int installedVersionCode) {
        return versionCode > installedVersionCode;
    }

    /** Обновление обязательно: ниже минимально поддерживаемой версии. */
    public boolean isMandatoryFor(int installedVersionCode) {
        return mandatory || installedVersionCode < minSupportedVersionCode;
    }
}
```

### 4. `update/GitHubUpdater.java`

```java
package ru.blindnews.app.update;

import android.os.Handler;
import android.os.Looper;

import java.io.ByteArrayOutputStream;
import java.io.InputStream;
import java.net.HttpURLConnection;
import java.net.URL;
import java.nio.charset.StandardCharsets;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

/** Читает update.json, сравнивает versionCode. Никогда не бросает в главный поток. */
public final class GitHubUpdater {

    public static final String MANIFEST_URL =
            "https://raw.githubusercontent.com/GeorgyAliabyev/blindnews-releases/main/update.json";

    private static final int TIMEOUT_MS = 8000;
    private static final int MAX_MANIFEST_BYTES = 256 * 1024;

    public interface Callback {
        /** info == null — обновлений нет. */
        void onResult(UpdateInfo info);

        /** Сеть или разбор не удались — тихо игнорируем, это не ошибка для пользователя. */
        void onError(Exception e);
    }

    private static final ExecutorService IO = Executors.newSingleThreadExecutor();
    private static final Handler MAIN = new Handler(Looper.getMainLooper());

    private GitHubUpdater() {
    }

    public static void checkAsync(int installedVersionCode, String userAgent, Callback cb) {
        IO.execute(() -> {
            try {
                UpdateInfo info = fetch(userAgent);
                UpdateInfo newer = info.isNewerThan(installedVersionCode) ? info : null;
                MAIN.post(() -> cb.onResult(newer));
            } catch (Exception e) {
                MAIN.post(() -> cb.onError(e));
            }
        });
    }

    private static UpdateInfo fetch(String userAgent) throws Exception {
        HttpURLConnection c = (HttpURLConnection) new URL(MANIFEST_URL).openConnection();
        c.setConnectTimeout(TIMEOUT_MS);
        c.setReadTimeout(TIMEOUT_MS);
        c.setRequestProperty("User-Agent", userAgent);
        c.setRequestProperty("Accept", "application/json");
        c.setUseCaches(false);
        try {
            int code = c.getResponseCode();
            if (code != HttpURLConnection.HTTP_OK) {
                throw new IllegalStateException("HTTP " + code);
            }
            try (InputStream in = c.getInputStream()) {
                ByteArrayOutputStream buf = new ByteArrayOutputStream();
                byte[] chunk = new byte[4096];
                int n;
                while ((n = in.read(chunk)) > 0) {
                    if (buf.size() + n > MAX_MANIFEST_BYTES) {
                        throw new IllegalStateException("manifest too large");
                    }
                    buf.write(chunk, 0, n);
                }
                return UpdateInfo.parse(new String(buf.toByteArray(), StandardCharsets.UTF_8));
            }
        } finally {
            c.disconnect();
        }
    }
}
```

### 5. `update/UpdateInstaller.java`

```java
package ru.blindnews.app.update;

import android.app.Activity;
import android.content.ActivityNotFoundException;
import android.content.Context;
import android.content.Intent;
import android.net.Uri;
import android.os.Build;
import android.provider.Settings;

import androidx.core.content.FileProvider;

import java.io.File;
import java.io.FileOutputStream;
import java.io.IOException;
import java.io.InputStream;
import java.net.HttpURLConnection;
import java.net.URL;
import java.security.MessageDigest;

/** Скачивание APK, проверка SHA-256 и передача системному установщику. */
public final class UpdateInstaller {

    public static final String APK_MIME = "application/vnd.android.package-archive";

    private static final String DIR = "updates";
    private static final String NAME = "BlindNews.apk";
    private static final int TIMEOUT_MS = 15000;

    public interface Progress {
        /** 0..100, либо -1 если общий размер неизвестен. */
        void onProgress(int percent);
    }

    private UpdateInstaller() {
    }

    /** Блокирующий вызов — только не в главном потоке. Возвращает проверенный APK. */
    public static File download(Context ctx, UpdateInfo info, Progress cb) throws Exception {
        if (info.apkUrl == null || info.apkUrl.isEmpty()) {
            throw new IOException("apkUrl пуст");
        }
        File dir = new File(ctx.getCacheDir(), DIR);
        if (!dir.exists() && !dir.mkdirs()) {
            throw new IOException("не создал cache/" + DIR);
        }
        File target = new File(dir, NAME);
        File part = new File(dir, NAME + ".part");

        HttpURLConnection c = (HttpURLConnection) new URL(info.apkUrl).openConnection();
        c.setConnectTimeout(TIMEOUT_MS);
        c.setReadTimeout(TIMEOUT_MS);
        c.setInstanceFollowRedirects(true);   // GitHub Releases отдаёт 302 на objects.githubusercontent.com
        c.setRequestProperty("User-Agent", "BlindNews-Android");
        try {
            int code = c.getResponseCode();
            if (code != HttpURLConnection.HTTP_OK) {
                throw new IOException("HTTP " + code);
            }
            long total = c.getContentLengthLong();
            MessageDigest sha = MessageDigest.getInstance("SHA-256");

            try (InputStream in = c.getInputStream();
                 FileOutputStream out = new FileOutputStream(part)) {
                byte[] chunk = new byte[32 * 1024];
                long done = 0L;
                int n;
                while ((n = in.read(chunk)) > 0) {
                    out.write(chunk, 0, n);
                    sha.update(chunk, 0, n);
                    done += n;
                    if (cb != null) {
                        cb.onProgress(total > 0 ? (int) (done * 100 / total) : -1);
                    }
                }
                out.getFD().sync();
            }

            String actual = toHex(sha.digest());
            if (info.apkSha256 != null && !info.apkSha256.isEmpty()
                    && !info.apkSha256.equalsIgnoreCase(actual)) {
                //noinspection ResultOfMethodCallIgnored
                part.delete();
                throw new IOException("SHA-256 не совпал: " + actual);
            }
            if (info.apkSize > 0 && part.length() != info.apkSize) {
                //noinspection ResultOfMethodCallIgnored
                part.delete();
                throw new IOException("размер не совпал: " + part.length());
            }

            if (target.exists() && !target.delete()) {
                throw new IOException("не перезаписал старый APK");
            }
            if (!part.renameTo(target)) {
                throw new IOException("не переименовал .part");
            }
            return target;
        } finally {
            c.disconnect();
        }
    }

    /**
     * Отдаёт APK установщику.
     *
     * @return true — установщик запущен; false — нужно разрешение «Установка
     * неизвестных приложений», пользователь отправлен в настройки. После возврата
     * из настроек вызвать ещё раз с тем же файлом.
     */
    public static boolean install(Activity activity, File apk) {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O
                && !activity.getPackageManager().canRequestPackageInstalls()) {
            Intent settings = new Intent(Settings.ACTION_MANAGE_UNKNOWN_APP_SOURCES,
                    Uri.parse("package:" + activity.getPackageName()));
            activity.startActivity(settings);
            return false;
        }
        Uri uri = FileProvider.getUriForFile(
                activity, activity.getPackageName() + ".fileprovider", apk);
        Intent intent = new Intent(Intent.ACTION_VIEW)
                .setDataAndType(uri, APK_MIME)
                .addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)
                .addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
        try {
            activity.startActivity(intent);
            return true;
        } catch (ActivityNotFoundException e) {
            return false;
        }
    }

    private static String toHex(byte[] bytes) {
        StringBuilder sb = new StringBuilder(bytes.length * 2);
        for (byte b : bytes) {
            sb.append(Character.forDigit((b >> 4) & 0xF, 16));
            sb.append(Character.forDigit(b & 0xF, 16));
        }
        return sb.toString();
    }
}
```

### 6. Диалог и вызов

```java
package ru.blindnews.app.update;

import android.app.Activity;
import android.content.Context;
import android.content.SharedPreferences;

import androidx.appcompat.app.AlertDialog;

/** Троттлинг проверок и сам диалог. Тексты — в strings.xml, не в коде. */
public final class UpdatePrompt {

    private static final String PREFS = "blindnews_update";
    private static final String KEY_LAST_CHECK = "last_check_ms";
    private static final String KEY_CHECKED_VERSION = "checked_version_code";
    private static final long CHECK_INTERVAL_MS = 12L * 60 * 60 * 1000;  // раз в 12 часов

    private UpdatePrompt() {
    }

    public static void maybeCheck(Activity activity) {
        SharedPreferences p = activity.getSharedPreferences(PREFS, Context.MODE_PRIVATE);
        long last = p.getLong(KEY_LAST_CHECK, 0L);
        // После установки новой сборки проверяем сразу: отметка времени от
        // прошлой версии иначе глушит первую проверку на 12 часов, и человек
        // видит обновление только когда нажмёт проверку вручную.
        boolean firstRunOnThisVersion =
                p.getInt(KEY_CHECKED_VERSION, 0) != BuildConfig.VERSION_CODE;
        if (!firstRunOnThisVersion
                && System.currentTimeMillis() - last < CHECK_INTERVAL_MS) return;

        GitHubUpdater.checkAsync(BuildConfig.VERSION_CODE, "BlindNews-Android",
                new GitHubUpdater.Callback() {
                    @Override
                    public void onResult(UpdateInfo info) {
                        p.edit()
                                .putLong(KEY_LAST_CHECK, System.currentTimeMillis())
                                .putInt(KEY_CHECKED_VERSION, BuildConfig.VERSION_CODE)
                                .apply();
                        if (info != null && !activity.isFinishing()
                                && !activity.isDestroyed()) {
                            show(activity, info);
                        }
                    }

                    @Override
                    public void onError(Exception e) {
                        // Не двигаем отметку времени: сеть пропала — попробуем позже.
                    }
                });
    }

    public static void show(Activity activity, UpdateInfo info) {
        StringBuilder body = new StringBuilder();
        for (String line : info.changelog) {
            body.append("• ").append(line).append("\n");
        }
        if (body.length() == 0) {
            body.append("Вышла новая версия BlindNews.");
        }

        boolean forced = info.isMandatoryFor(BuildConfig.VERSION_CODE);

        new AlertDialog.Builder(activity)
                .setTitle("Что нового в BlindNews " + info.versionName)
                .setMessage(body.toString().trim())
                .setCancelable(!forced)
                .setPositiveButton("Скачать обновление",
                        (d, w) -> UpdateDownload.run(activity, info))
                .setNegativeButton(forced ? null : "Позже", null)
                .show();
    }
}
```

`UpdateDownload.run(activity, info)` — тонкая обёртка: прогресс в существующем
UI приложения (Notification или диалог), затем `UpdateInstaller.install`.
Минимальный вариант:

```java
static void run(Activity activity, UpdateInfo info) {
    android.app.ProgressDialog pd = new android.app.ProgressDialog(activity);
    pd.setMessage("Загрузка обновления");
    pd.setProgressStyle(android.app.ProgressDialog.STYLE_HORIZONTAL);
    pd.setCancelable(false);
    pd.show();
    new Thread(() -> {
        try {
            File apk = UpdateInstaller.download(activity, info,
                    percent -> activity.runOnUiThread(() -> {
                        if (percent >= 0) pd.setProgress(percent);
                    }));
            activity.runOnUiThread(() -> {
                pd.dismiss();
                confirmInstall(activity, apk);
            });
        } catch (Exception e) {
            activity.runOnUiThread(() -> {
                pd.dismiss();
                android.widget.Toast.makeText(activity,
                        "Не удалось скачать обновление: " + e.getMessage(),
                        android.widget.Toast.LENGTH_LONG).show();
            });
        }
    }, "update-download").start();
}

/** Говорит человеку, что делать дальше, и только по нажатию открывает установку. */
private static void confirmInstall(Activity activity, File apk) {
    new AlertDialog.Builder(activity)
            .setTitle("Обновление скачано")
            .setMessage("Сейчас откроется установка. Нажмите «Установить», "
                    + "потом «Обновить». BlindNews закроется и откроется уже "
                    + "новой версией. Данные и настройки сохранятся.")
            .setCancelable(true)
            .setPositiveButton("Продолжить",
                    (d, w) -> UpdateInstaller.install(activity, apk))
            .setNegativeButton("Позже", null)
            .show();
}
```

### 7. Точка вызова

В `FeedActivity.onCreate` (или там, где лента уже показана пользователю) —
один вызов, после отрисовки первого экрана, не блокируя поток:

```java
feedListView.post(() -> UpdatePrompt.maybeCheck(this));
```

Проверку не запускать во время воспроизведения в `PlayerActivity`: системный
диалог поверх плеера сбивает TalkBack-фокус. Правило простое — только на ленте
и только когда `activity.hasWindowFocus()`.

### 8. Возврат из настроек «Установка неизвестных приложений»

`UpdateInstaller.install` возвращает `false` и уводит пользователя в системные
настройки. Чтобы после возврата установка продолжилась сама, запомнить файл в
поле активити и повторить в `onResume`:

```java
private File pendingApk;

@Override
protected void onResume() {
    super.onResume();
    if (pendingApk != null) {
        File apk = pendingApk;
        pendingApk = null;
        if (!UpdateInstaller.install(this, apk)) {
            pendingApk = apk;   // пользователь ещё не выдал разрешение
        }
    }
}
```

И в `UpdateDownload.run` вместо прямого `install`:

```java
if (!UpdateInstaller.install(activity, apk)) {
    if (activity instanceof FeedActivity) {
        ((FeedActivity) activity).pendingApk = apk;
        Toast.makeText(activity,
                "Разрешите установку из этого источника и вернитесь в приложение",
                Toast.LENGTH_LONG).show();
    }
}
```

## Жёсткие требования, без которых обновление не встанет

1. **Та же подпись.** APK обязан быть подписан тем же `blindnews-release.jks`
   (`D:\BlindNews\blindnews-release.jks`). Другая подпись — Android откажет с
   `INSTALL_FAILED_UPDATE_INCOMPATIBLE`, и пользователю придётся удалять
   приложение с потерей данных.
2. **`versionCode` строго больше.** Равный или меньший — отказ
   `INSTALL_FAILED_VERSION_DOWNGRADE`. `versionName` можно любой, Android его
   не сравнивает.
3. **`applicationId` не меняется** — `ru.blindnews.app`. Смена пакета превращает
   APK в другое приложение.
4. **`minSdk` не поднимать выше 26** без предупреждения: на старых устройствах
   установка упадёт.
5. **Имя ассета `BlindNews.apk`** одинаковое на каждом релизе — от него зависит
   стабильная ссылка `releases/latest/download/BlindNews.apk`.

## Формат `update.json`

| Ключ | Что это |
|---|---|
| `schema` | Версия формата. Сейчас `1`. |
| `versionCode` | Целое. Сравнивается с `BuildConfig.VERSION_CODE`. Главное поле. |
| `versionName` | Строка для заголовка диалога. |
| `publishedAt` | Дата `YYYY-MM-DD`, для человека. |
| `mandatory` | `true` — диалог без кнопки «Позже». |
| `minSupportedVersionCode` | Ниже этого значения обновление обязательно. `0` — не используется. |
| `minSdk` | Информационно: минимальный Android для этого APK. |
| `apkUrl` | Прямая ссылка на APK конкретной версии. |
| `apkSize` | Размер в байтах — проверяется после скачивания. |
| `apkSha256` | Хеш в hex, нижний регистр — проверяется после скачивания. |
| `changelog` | Массив строк, по одному пункту. Идёт в диалог. |
| `changelogUrl` | Ссылка на полную историю. |
| `releaseUrl` | Ссылка на список релизов. |

Ключ `mandatory` и `minSupportedVersionCode` — единственный способ заставить
обновиться. Использовать редко: блокирующий диалог для незрячего пользователя
означает, что приложение недоступно, пока он не скачает APK.

## Как выпускается новый релиз

1. Подписанный APK переименовать в `BlindNews.apk`.
2. `sha256sum BlindNews.apk` → значение в `apkSha256`, размер в `apkSize`.
3. Создать GitHub Release с тегом `v<versionName>` (например `v1.1-dev2`),
   приложить `BlindNews.apk`, в описание — список изменений.
4. Обновить `update.json`: `versionCode`, `versionName`, `publishedAt`,
   `apkUrl` (указывает на тег этой версии), `changelog`.
5. Дописать раздел в `CHANGELOG.md`, закоммитить и запушить.

`apkUrl` всегда указывает на **конкретный тег**, а не на `latest`: манифест и
файл тогда не могут разъехаться — если `update.json` ещё старый, приложение
скачает старую версию, а не новую под старым номером.

## Проверка перед выдачей пользователям

- Установить на устройство предыдущую версию, дать ей увидеть манифест,
  убедиться, что диалог появляется и TalkBack читает заголовок и список
  изменений целиком.
- Пройти установку до конца: файл скачивается, SHA-256 сходится, системный
  установщик открывается, приложение обновляется поверх с сохранением данных.
- Отдельно проверить путь, где «Установка неизвестных приложений» выключена:
  пользователь должен попасть в нужный экран настроек и после возврата —
  в установку.
