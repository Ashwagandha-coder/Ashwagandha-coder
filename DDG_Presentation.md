
## Слайд 1: Титульный

**Research-разбор: Поверхности для abuse в Android-приложениях**

**Цель:** DuckDuckGo Browser v5.288.0 (play-release)

**Артефакт:**
```
Package: com.duckduckgo.mobile.android
SHA256: BB 7B B3 1C 57 3C 46 A1 DA 7F C5 C5 28 A6 AC F4 
        32 10 84 56 FE EC 50 81 0C 7F 33 69 4E B3 D2 D4
APK: duckduckgo-5.288.0-play-release.apk
```

**Инструменты:** JADX, apkanalyzer, aapt, adb, unzip

---

## Слайд 2: Block 1 — Runtime: Launch Path

```
ТОЧКА ВХОДА:
Тап по иконке → activity-alias "Launcher" → LaunchBridgeActivity

ЦЕПОЧКА ЗАПУСКА (трассировано из бинарника):
LaunchBridgeActivity.onCreate()
  └→ SplashScreen.installSplashScreen()
  └→ LaunchViewModel.start(intent)
       ├→ seedTestScenario(intent.extras)    ← принимает внешние параметры
       ├→ waitForReferrerData(1500ms)        ← ждёт install referrer
       └→ showOnboardingOrHome()             ← ТОЧКА РАЗВЕТВЛЕНИЯ
            ├→ isNewUser()=true  → OnboardingActivity
            └→ isNewUser()=false → BrowserActivity

ПАРАМЕТРЫ АКТИВНОСТИ:
- launchMode: standard (по умолчанию) — новый экземпляр при каждом запуске
- exported: true (через activity-alias)
- permission: отсутствует — любое приложение может запустить
```

**Артефакт:** `adb shell am start -n com.duckduckgo.mobile.android/com.duckduckgo.app.launch.LaunchBridgeActivity`

---

## Слайд 3: Block 1 — Манифест: Риски

```xml
<!-- ТОЧКА ЗАПУСКА - ЭКСПОРТИРОВАНА БЕЗ PERMISSION -->
<activity-alias
    android:name="com.duckduckgo.app.launch.Launcher"
    android:targetActivity="com.duckduckgo.app.launch.LaunchBridgeActivity"
    android:exported="true">                    ← ЛЮБОЕ ПРИЛОЖЕНИЕ МОЖЕТ ЗАПУСТИТЬ
    <intent-filter>
        <action android:name="android.intent.action.MAIN"/>
        <category android:name="android.intent.category.LAUNCHER"/>
        <category android:name="android.intent.category.APP_BROWSER"/>
    </intent-filter>
</activity-alias>

<!-- САМА АКТИВНОСТЬ -->
<activity
    android:name="com.duckduckgo.app.launch.LaunchBridgeActivity"
    android:exported="true">                    ← ДУБЛИРУЕТ ЭКСПОРТ
    <intent-filter>
        <action android:name="android.intent.action.MAIN"/>
    </intent-filter>
</activity>

РИСКИ:
1. Activity-alias может быть переключён удалённо/локально
2. Нет android:permission — запуск без ограничений
3. launchMode=standard — множественные экземпляры в back stack
4. Intent extras попадают в seedTestScenario() без валидации
```

---

## Слайд 4: Block 2 — Состав APK

```
СТРУКТУРА APK:
├── AndroidManifest.xml (74 KB)
├── classes.dex ... classes10.dex  (11 DEX-файлов, ~85 MB всего)
│   ├── classes.dex  — 13.2 MB
│   ├── classes2.dex — 3.3 KB   ← ПОДОЗРИТЕЛЬНО МАЛ
│   ├── classes3.dex — 2.8 KB   ← ПОДОЗРИТЕЛЬНО МАЛ
│   └── classes4-10.dex — 7-13 MB каждый
├── lib/ (4 архитектуры: arm64, armeabi-v7, x86, x86_64)
│   ├── libhttps-bloom-lib.so   ← HTTPS-фильтрация
│   ├── libnetguard.so          ← Перехват трафика
│   ├── libddgcrypto.so         ← Криптография
│   ├── libsqlcipher.so         ← Шифрованная БД
│   └── libwg-go.so             ← WireGuard VPN
├── assets/
│   ├── autofill-debug.js (959 KB)  ← DEBUG В RELEASE-СБОРКЕ
│   ├── autofill.js (806 KB)
│   ├── brokers/ (80+ JSON — списки data-брокеров)
│   └── html/ (android.html, browser.html, iframe.html)
└── META-INF/ (сертификат: RSA 1024-bit, SHA1withRSA)
```

---

## Слайд 5: Block 2 — Выводы по сборке

```
НОРМАЛЬНО:
- Multi-ABI поддержка (arm64, armeabi-v7, x86, x86_64)
- Baseline profile (baseline.prof) для оптимизации запуска
- ProGuard применён (имена методов минифицированы)
- Структурированные assets по назначению

ПОДОЗРИТЕЛЬНО:
- autofill-debug.js (959 KB) в play-release сборке
- Сертификат: RSA 1024-bit (депрекация с 2010 г., NIST с 2014 г.)
- Сертификат: SHA1withRSA (депрекация с 2017 г.)
- Срок действия: 2011–3010 (999 лет — нет ротации ключей)
- classes2.dex и classes3.dex подозрительно малы (3 KB)

ЧТО ИСПРАВИТЬ В СБОРКЕ:
1. Удалить autofill-debug.js из release-конфигурации
2. Обновить сертификат подписи (RSA 2048+ с SHA256)
3. Проверить содержимое classes2.dex и classes3.dex
4. Добавить проверку имён файлов на debug/staging в CI/CD
```

---

## Слайд 6: Block 3 — Уязвимость 1: Intent Spoofing

```
ЭКСПОРТИРОВАННАЯ АКТИВНОСТЬ БЕЗ PERMISSION

КОМПОНЕНТ: LaunchBridgeActivity
ЭКСПОРТ: android:exported="true"
PERMISSION: отсутствует
LAUNCH MODE: standard

ЦЕПОЧКА ДАННЫХ:
Intent extras → LaunchViewModel.start()
              → toStringMap(extras)
              → seedTestScenario(params)
              → TestScenarioSeeder.seedIfNeeded()

ПРОВЕРКА (ПОДТВЕРЖДЕНО):
$ adb shell am start -n com.duckduckgo.mobile.android/\
    com.duckduckgo.app.launch.LaunchBridgeActivity \
    -e test_key "test_value"
→ Starting: Intent { (has extras) }   ← extras приняты

ТЕКУЩИЙ УРОВЕНЬ РИСКА: LOW (NoOpTestScenarioSeeder в релизе)
ПОТЕНЦИАЛЬНЫЙ РИСК: HIGH (если реализация seeder'а будет заменена)
```

---

## Слайд 7: Block 3 — Уязвимость 2: JS Interface + Секрет

```
JAVASCRIPT-МОСТ С ЖЁСТКО ЗАШИТЫМ СЕКРЕТОМ

КОМПОНЕНТ: DuckPlayerScriptsJsMessaging
ИМЯ МОСТА: "specialPages" (доступен из любого JS в WebView)

КОД (из JADX):
webView.addJavascriptInterface(this, "specialPages");

АУТЕНТИФИКАЦИЯ:
this.secret = "duckduckgo-android-messaging-secret";  ← ОТКРЫТЫЙ ТЕКСТ

ДОМЕННАЯ ПРОВЕРКА (ОБХОДИТСЯ):
this.allowedDomains = CollectionsKt.emptyList();  ← ПУСТОЙ СПИСОК
isUrlAllowed() → true для ЛЮБОГО URL (empty list = разрешить всё)

МЕТОДЫ, ДОСТУПНЫЕ ЧЕРЕЗ МОСТ:
"setUserValues"         ← ИЗМЕНЕНИЕ НАСТРОЕК ПОЛЬЗОВАТЕЛЯ
"initialSetup"          ← ПЕРЕИНИЦИАЛИЗАЦИЯ
"openSettings"          ← НАВИГАЦИЯ В ПРИЛОЖЕНИИ
"reportPageException"   ← ОТПРАВКА ДАННЫХ ОБ ОШИБКАХ
"reportInitException"   ← ОТПРАВКА ДАННЫХ ОБ ОШИБКАХ
"reportYouTubeError"    ← ОТПРАВКА ДАННЫХ

РИСК: XSS в WebView → вызов specialPages.process() → контроль над WebView
```

---

## Слайд 8: Block 3 — Уязвимость 3: Debug-файлы

```
DEBUG-JAVASCRIPT В PRODUCTION-СБОРКЕ

ФАЙЛ: autofill-debug.js (959 KB)
СБОРКА: play-release (не должна содержать отладочных артефактов)

СОСЕДНИЕ ФАЙЛЫ:
autofill.js             — 806 KB (production-версия)
autofill-debug.js       — 959 KB (debug-версия В РЕЛИЗЕ)
autofill.css            — 96 KB
autofill-design-tokens.css — 87 KB
autofillImport.js       — 413 KB

РИСКИ:
- Отладочное логирование (console.log) в production
- Возможное раскрытие внутренних endpoint'ов
- Сниженные проверки безопасности в debug-коде
- Увеличение поверхности атаки при загрузке в WebView

ВЕРИФИКАЦИЯ:
$ unzip -l duckduckgo-5.288.0-play-release.apk | grep debug
→ 959935  autofill-debug.js
```

---

## Слайд 9: Block 4 — Split-View: Differential Test

```
A/B-TEST: НОВЫЙ ПОЛЬЗОВАТЕЛЬ vs ВОЗВРАЩАЮЩИЙСЯ

┌──────────────────────┬──────────────────┬──────────────────┐
│ СИГНАЛ               │ ПРОФИЛЬ A        │ ПРОФИЛЬ B        │
│                      │ (Новый)          │ (Возвращающийся) │
├──────────────────────┼──────────────────┼──────────────────┤
│ Activity входа       │ LaunchBridge     │ LaunchBridge     │
│                      │ Activity         │ Activity         │
├──────────────────────┼──────────────────┼──────────────────┤
│ Результирующая       │ Onboarding       │ Browser          │
│ Activity             │ Activity         │ Activity         │
├──────────────────────┼──────────────────┼──────────────────┤
│ Точка принятия       │ isNewUser()      │ isNewUser()      │
│ решения              │ = true           │ = false          │
├──────────────────────┼──────────────────┼──────────────────┤
│ Install Referrer     │ Ожидание 1500ms  │ Не вызывается    │
├──────────────────────┼──────────────────┼──────────────────┤
│ Feature Toggle       │ BrandDesign      │ BrandDesign      │
│                      │ Update (remote)  │ Update (remote)  │
├──────────────────────┼──────────────────┼──────────────────┤
│ Firebase RemoteConf  │ НЕ НАЙДЕНО       │ НЕ НАЙДЕНО       │
└──────────────────────┴──────────────────┴──────────────────┘

КОМАНДЫ:
Профиль A: adb shell pm clear com.duckduckgo.mobile.android
           adb shell am start -n ...LaunchBridgeActivity
Профиль B: adb shell am start -n ...LaunchBridgeActivity
           (без очистки данных)
```

---

## Слайд 10: Block 4 — Split-View: Механизмы

```
ГДЕ ЖИВЁТ РАЗНОЕ ПОВЕДЕНИЕ:

1. UserStageStore (локальное хранилище)
   - isNewUser() определяет показ onboarding vs browser
   - Манипуляция: pm clear сбрасывает состояние

2. BrandDesignUpdateToggles (удалённые флаги)
   - Влияет на выбор старого/нового onboarding
   - Точка входа: showOnboardingOrHome()

3. Install Referrer (внешний сигнал)
   - waitForReferrerData() ждёт 1500ms
   - Может изменить поведение при первом запуске

4. TestScenarioSeeder (заглушка в релизе)
   - seedTestScenario() принимает intent extras
   - Сейчас NoOp, но архитектура готова к внедрению

ВЫВОД:
- Клиент определяет поведение по локальному состоянию
- Удалённые флаги существуют, но Firebase RemoteConfig не используется
- Механизм раздвоения легитимный, но манипулируемый
```

---

## Слайд 11: Block 5 — Detection Signals

```
СИГНАЛЫ ДЛЯ ANTI-ABUSE:

1. ЭКСПОРТИРОВАННЫЕ КОМПОНЕНТЫ БЕЗ PERMISSION
   Проверка: aapt dump xmltree APK AndroidManifest.xml | grep 'exported="true"'
   Флаг: exported=true + нет android:permission
   Ложные срабатывания: системные приложения, launcher-активности
   Отсев: требовать обоснование для каждого экспортированного компонента

2. ХАРДКОД-СЕКРЕТЫ В JS-МОСТАХ
   Проверка: grep -r "SECRET\|API_KEY\|token" decompiled/ --include="*.java"
   Флаг: строковые константы "secret"/"key" + addJavascriptInterface
   Ложные срабатывания: не-секретные ключи конфигурации
   Отсев: проверять контекст использования (JS-мост или нет)

3. DEBUG-ФАЙЛЫ В RELEASE-СБОРКЕ
   Проверка: unzip -l APK | grep -iE "debug|test|staging"
   Флаг: *debug*.js, *test*.json в папках assets/lib
   Ложные срабатывания: сторонние SDK с "debug" в легитимных именах
   Отсев: whitelist известных SDK-файлов

4. SPLIT-VIEW ПОВЕДЕНИЕ
   Проверка: запуск с разным состоянием → сравнение ActivityStack
   Флаг: один entry-point → разные Activity для разных профилей
   Ложные срабатывания: легитимный onboarding
   Отсев: флаг только при переходе на скрытые/дебажные Activity

5. JS-МОСТЫ С ПУСТЫМ DOMAIN WHITELIST
   Проверка: grep -r "allowedDomains\|emptyList" decompiled/ | grep -A5 "addJavascriptInterface"
   Флаг: addJavascriptInterface + allowedDomains = []
   Ложные срабатывания: мосты к документированным SDK
   Отсев: проверять непустой список доменов и нетривиальный секрет
```

---

## Слайд 12: Block 6 — Выводы

```
МАТРИЦА РИСКОВ:

┌──────────────────────────────────────┬────────┬──────────────┐
│ НАХОДКА                              │ РИСК   │ УВЕРЕННОСТЬ  │
├──────────────────────────────────────┼────────┼──────────────┤
│ Экспортированная LaunchBridge        │ MEDIUM │ ВЫСОКАЯ      │
│ Activity без permission              │        │ adb + JADX   │
├──────────────────────────────────────┼────────┼──────────────┤
│ JS-мост + жёсткий секрет +           │ HIGH   │ ВЫСОКАЯ      │
│ пустой domain whitelist              │        │ код + строки │
├──────────────────────────────────────┼────────┼──────────────┤
│ Debug-JS в production (959 KB)       │ LOW    │ ВЫСОКАЯ      │
│                                      │        │ листинг APK  │
├──────────────────────────────────────┼────────┼──────────────┤
│ Split-view: разное поведение от      │ LOW    │ СРЕДНЯЯ      │
│ состояния пользователя               │        │ легитимно    │
├──────────────────────────────────────┼────────┼──────────────┤
│ Test Seeding path (NoOp в релизе)    │ LOW    │ СРЕДНЯЯ      │
│                                      │        │ код + adb   │
└──────────────────────────────────────┴────────┴──────────────┘

ЧТО ДЕЛАТЬ ДАЛЬШЕ:
1. Заменить хардкод-секрет на runtime-сгенерированный токен
2. Убрать autofill-debug.js из release-конфигурации сборки
3. Добавить permission check в LaunchBridgeActivity
4. Заполнить allowedDomains для JS-моста (вместо пустого списка)
5. Удалить вызов seedTestScenario() из production-кода

ЧТО ПРОВЕРИЛИ И НЕ ПОДТВЕРДИЛОСЬ:
- Firebase RemoteConfig: НЕ НАЙДЕНО в APK
- Энумерация duck:// deep links: НЕ ПОЛНОСТЬЮ
- Эксплуатация JS-моста: ПУТЬ ПОДТВЕРЖДЁН, ЭКСПЛОИТ НЕ ТЕСТИРОВАЛСЯ
- Network security config (certificate pinning): НЕ ПРОВЕРЕН
```

