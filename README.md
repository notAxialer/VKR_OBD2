# AutoDiag: Мобильное приложение для автомобильной диагностики OBD-II
## Аннотация
AutoDiag представляет собой программное решение класса Mobile Diagnostic Tool, разработанное на платформе Flutter для проведения комплексной диагностики автомобилей через интерфейс OBD-II (On-Board Diagnostics). Приложение обеспечивает взаимодействие с транспортными средствами посредством Bluetooth-адаптеров стандарта ELM327, реализуя полный цикл диагностических процедур: от чтения кодов неисправностей (Diagnostic Trouble Codes, DTC) до мониторинга параметров работы двигателя в режиме реального времени с последующим сохранением результатов в локальную реляционную базу данных SQLite.
Настоящая документация содержит исчерпывающее описание архитектуры приложения, используемых библиотек, структуры базы данных, алгоритмов обработки данных и взаимосвязей между компонентами системы.
---
## Оглавление
1. [Общая характеристика системы](#общая-характеристика-системы)
2. [Архитектурная организация](#архитектурная-организация)
3. [Технологический стек и зависимости](#технологический-стек-и-зависимости)
4. [Структура проекта](#структура-проекта)
5. [Слой представления данных (Presentation Layer)](#слой-представления-данных-presentation-layer)
6. [Слой хранения данных (Data Layer)](#слой-хранения-данных-data-layer)
7. [Ядро системы (Core Layer)](#ядро-системы-core-layer)
8. [База данных: схема и миграции](#база-данных-схема-и-миграции)
9. [Протокол OBD-II и парсинг данных](#протокол-obd-ii-и-парсинг-данных)
10. [Алгоритмы функционирования](#алгоритмы-функционирования)
11. [Модели данных](#модели-данных)
12. [Управление состоянием приложения](#управление-состоянием-приложения)
13. [Экспорт данных и генерация отчётов](#экспорт-данных-и-генерация-отчётов)
14. [Система рекомендаций](#система-рекомендаций)
15. [Модуль технического обслуживания](#модуль-технического-обслуживания)
16. [VIN-декодирование и автоопределение автомобиля](#vin-декодирование-и-автоопределение-автомобиля)
17. [Расширение функциональности](#расширение-функциональности)
---
## Общая характеристика системы
### Назначение системы
AutoDiag предназначено для решения следующих задач:
1. **Диагностика систем автомобиля**: чтение, интерпретация и очистка кодов ошибок двигателя и других систем транспортного средства.
2. **Мониторинг параметров в реальном времени**: отображение текущих значений датчиков двигателя (обороты, температура, нагрузка, давление и др.).
3. **Ведение истории диагностических сеансов**: сохранение результатов диагностики с привязкой к конкретному автомобилю и пробегу.
4. **Управление парком автомобилей**: регистрация и хранение информации о нескольких транспортных средствах.
5. **Планирование технического обслуживания**: настройка регламентов ТО с уведомлением о приближающихся или просроченных работах.
6. **Генерация отчётов**: экспорт результатов диагностики в форматах PDF и CSV для дальнейшего анализа или передачи специалистам.
### Поддерживаемые стандарты
- **OBD-II SAE J1979**: стандарт обмена диагностическими данными между автомобилем и внешним оборудованием.
- **ISO 14230 (KWP2000)**: протокол диагностики по K-линии.
- **ISO 15765-4 (CAN)**: контроллерная сеть автомобиля.
- **ELM327**: популярный чип-интерпретатор для OBD-II адаптеров.
### Требования к оборудованию
- Android-устройство с поддержкой Bluetooth Classic (BR/EDR).
- OBD-II адаптер на базе чипа ELM327 версии 1.5 или выше.
- Автомобиль с поддержкой стандарта OBD-II (выпуска после 1996 года для США, после 2000 года для Европы).
---
## Архитектурная организация
Приложение реализует многоуровневую архитектуру с чётким разделением ответственности между слоями:
```
┌─────────────────────────────────────────────────────────────────┐
│                    Presentation Layer                           │
│  ┌──────────────┐  ┌───────────────┐  ┌─────────────────────┐   │
│  │   Screens    │  │   Providers   │  │   Reusable Widgets  │   │
│  │   (UI/UX)    │◄─┤   (State Mgmt)│  │   (Components)      │   │
│  └──────────────┘  └───────────────┘  └─────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              ▲
                              │ (Dependency Injection via Provider)
                              │
┌─────────────────────────────────────────────────────────────────┐
│                      Data Layer                                 │
│  ┌──────────────────┐  ┌──────────────┐  ┌──────────────────┐   │
│  │   Repository     │  │   Models     │  │   Database       │   │
│  │   (CRUD Ops)     │  │   (Entities) │  │   (SQLite)       │   │
│  └──────────────────┘  └──────────────┘  └──────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              ▲
                              │ (Business Logic Interface)
                              │
┌─────────────────────────────────────────────────────────────────┐
│                       Core Layer                                │
│  ┌──────────────────┐  ┌──────────────┐  ┌──────────────────┐   │
│  │   OBD Parser     │  │   Services   │  │   Metadata       │   │
│  │   (J1979 Decoding)│ │   (BT/VIN)   │  │   (PID/DTC)      │   │
│  └──────────────────┘  └──────────────┘  └──────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```
### Принципы архитектуры
1. **MVVM (Model-View-ViewModel)**: разделение логики отображения (ViewModel/Provider), пользовательского интерфейса (View/Screen) и данных (Model).
2. **Repository Pattern**: единая точка доступа к данным, абстрагирующая источник данных (локальная БД, удалённый API).
3. **Dependency Injection**: внедрение зависимостей через пакет `provider`, обеспечивающее тестируемость и гибкость системы.
4. **Single Source of Truth**: база данных выступает единственным достоверным источником состояния для долгосрочных данных.
---
## Технологический стек и зависимости
### Основные зависимости (pubspec.yaml)
| Пакет                      | Версия  | Назначение                                          |
| -------------------------- | ------- | --------------------------------------------------- |
| `flutter`                  | SDK     | Фреймворк для кроссплатформенной разработки         |
| `provider`                 | ^6.0.0  | Управление состоянием, реализация паттерна Observer |
| `sqflite`                  | ^2.3.0  | Обёртка над SQLite для локального хранения данных   |
| `path`                     | ^1.8.0  | Утилиты для работы с файловыми путями               |
| `flutter_bluetooth_serial` | —       | Работа с Bluetooth Classic (RFCOMM)                 |
| `permission_handler`       | —       | Запрос и проверка системных разрешений Android/iOS  |
| `pdf`                      | ^3.10.0 | Генерация PDF-документов на стороне клиента         |
| `share_plus`               | —       | Экспорт файлов через системный диалог sharing       |
| `intl`                     | —       | Интернационализация и форматирование дат/чисел      |
| `path_provider`            | —       | Доступ к системным директориям устройства           |
### Системные требования
- **Минимальная версия Android**: API level 21 (Android 5.0 Lollipop).
- **Разрешения**:
  - `BLUETOOTH`, `BLUETOOTH_ADMIN`: подключение к Bluetooth-устройствам.
  - `BLUETOOTH_CONNECT`, `BLUETOOTH_SCAN`: разрешения для Android 12+.
  - `ACCESS_FINE_LOCATION`: требуется для сканирования Bluetooth на Android < 12.
  - `POST_NOTIFICATIONS`: отображение уведомлений о ТО (Android 13+).
---
## Структура проекта
```
/workspace/
├── main.dart                          # Точка входа, инициализация зависимостей
├── autodiag_app.dart                  # Корневой виджет MaterialApp, темизация
│
├── core/                              # Ядро бизнес-логики, независимое от UI
│   ├── diagnostics/
│   │   └── recommendation_engine.dart # Движок формирования рекомендаций
│   ├── obd/
│   │   ├── dtc_catalog.dart           # Загрузчик каталога DTC
│   │   ├── dtc_hints.dart             # Расширенные описания DTC (~1000+ записей)
│   │   ├── obd_parser.dart            # Парсер ответов ELM327 (SAE J1979)
│   │   └── pid_metadata.dart          # Метаданные PID-параметров
│   └── services/
│       ├── bluetooth_obd_service.dart # Сервис взаимодействия с ELM327
│       ├── export_service.dart        # Генерация отчётов (PDF/CSV)
│       ├── notification_service.dart  # Локальные уведомления
│       └── vin_decoder.dart           # Декодирование VIN-номера
│
├── data/                              # Слой доступа к данным
│   ├── db/
│   │   └── app_database.dart          # Инициализация SQLite, миграции схемы
│   ├── models/
│   │   └── autodiag_models.dart       # Классы-модели данных
│   ├── autodiag_repository.dart       # Репозиторий для CRUD-операций
│   └── car_metadata.dart              # Каталог марок/моделей/поколений
│
└── presentation/                      # Слой представления (UI)
    ├── providers/                     # State Management (ChangeNotifier)
    │   ├── cars_provider.dart         # Управление списком автомобилей
    │   ├── diagnostics_provider.dart  # Логика диагностики, опрос PID
    │   ├── history_provider.dart      # История диагностических сеансов
    │   ├── maintenance_provider.dart  # Управление регламентами ТО
    │   └── settings_provider.dart     # Настройки приложения
    └── screens/autodiag/
        ├── main_shell.dart            # Главный экран с навигацией (BottomNavigationBar)
        ├── home_tab.dart              # Главная страница (сводка)
        ├── diagnostics_tab.dart       # Вкладка диагностики (OBD)
        ├── history_tab.dart           # История сеансов
        ├── maintenance_tab.dart       # Техническое обслуживание
        ├── cars_tab.dart              # Управление автомобилями
        ├── add_car_wizard.dart        # Мастер добавления автомобиля
        ├── edit_car_screen.dart       # Редактирование данных автомобиля
        ├── session_detail_screen.dart # Детальный просмотр сеанса
        ├── device_picker_sheet.dart   # Выбор Bluetooth-устройства
        └── vin_detection_screen.dart  # Экран определения по VIN
```
---
## Слой представления данных (Presentation Layer)
### Провайдеры состояния (State Providers)
#### DiagnosticsProvider
Класс `DiagnosticsProvider` расширяет `ChangeNotifier` и управляет всем циклом диагностики:
**Основные свойства:**
- `BluetoothObdService _obd` — сервис для работы с OBD-адаптером
- `AutodiagRepository _repo` — репозиторий для сохранения данных
- `SettingsProvider _settings` — настройки приложения
- `List<LiveDtc> liveDtcs` — текущие коды ошибок
- `Map<String, double> livePidValues` — текущие значения PID-параметров
- `Map<String, List<double>> chartHistory` — история для графиков
- `Set<String> supportedPids` — поддерживаемые автомобилем PID
- `Timer? _pollTimer` — таймер периодического опроса
- `VinInfo? _detectedVinInfo` — информация о распознанном автомобиле
**Ключевые методы:**
```dart
// Подключение к OBD-адаптеру
Future<void> connectTo(String address, {bool save = true})
// Начало периодического опроса PID-параметров
void startPolling()
// Чтение кодов ошибок (режимы 03 и 07)
Future<void> refreshDtc()
// Сохранение текущего сеанса диагностики
Future<int?> saveSession({required Car car, String notes = ''})
// Автоматическое определение автомобиля по VIN
Future<void> _tryAutoDetectCar()
// Определение поддерживаемых PID через битовые маски
Future<void> _refreshSupportedPids()
```
**Алгоритм опроса PID:**
Метод `_pollOnce()` выполняется с интервалом, заданным в настройках (`pidPollMs`):
1. Проверяется наличие подключения к адаптеру
2. Для каждого PID из множества `supportedPids` отправляется запрос `01<pid>`
3. Полученный ответ передаётся в метод `_decodePidValue()` для расчёта физического значения
4. Значение сохраняется в `livePidValues` и добавляется в `chartHistory` для построения графика
5. Вызывается `notifyListeners()` для обновления UI
#### CarsProvider
Управляет списком автомобилей пользователя:
```dart
class CarsProvider extends ChangeNotifier {
  List<Car> cars = [];
  Car? get active => cars.where((c) => c.isActive).firstOrNull;
  
  Future<void> refresh() async {
    cars = await _repo.getAllCars();
    notifyListeners();
  }
  
  Future<void> setActiveCar(int id) async {
    await _repo.setActiveCar(id);
    await refresh();
  }
}
```
#### HistoryProvider
Отвечает за загрузку и хранение истории диагностических сеансов:
```dart
class HistoryProvider extends ChangeNotifier {
  List<DiagnosticSessionRow> sessions = [];
  
  Future<void> refresh() async {
    sessions = await _repo.listSessions();
    notifyListeners();
  }
  
  Future<void> loadSessionDetails(int sessionId) async {
    dtcs = await _repo.sessionDtcs(sessionId);
    params = await _repo.sessionParams(sessionId);
    notifyListeners();
  }
}
```
#### MaintenanceProvider
Управление регламентами технического обслуживания:
```dart
class MaintenanceProvider extends ChangeNotifier {
  List<MaintenanceRow> operations = [];
  
  Future<void> loadForCar(int? carId, Car? car, bool enableNotifications) async {
    if (carId == null) return;
    operations = await _repo.listMaintenance(carId);
    // Расчёт статуса (ok/soon/overdue) для каждой операции
    notifyListeners();
  }
  
  Future<void> markDone(int opId) async {
    final car = carsProvider.active;
    if (car == null) return;
    await _repo.markMaintenanceDone(opId, car.currentMileage, DateTime.now());
    await loadForCar(car.id, car, true);
  }
}
```
#### SettingsProvider
Хранение настроек приложения в SharedPreferences:
```dart
class SettingsProvider extends ChangeNotifier {
  ThemeMode themeMode = ThemeMode.system;
  bool obdSimulation = false;
  bool autoConnectBt = false;
  String? lastBtAddress;
  int pidPollMs = 500;
  bool autoSaveSessionOnLeave = true;
  
  Future<void> load() async {
    final prefs = await SharedPreferences.getInstance();
    themeMode = ThemeMode.values.byName(prefs.getString('theme') ?? 'system');
    obdSimulation = prefs.getBool('simulation') ?? false;
    // ... загрузка остальных настроек
  }
}
```
---
## Слой хранения данных (Data Layer)
### AutodiagRepository
Репозиторий предоставляет единый интерфейс для всех операций с данными:
**Операции с автомобилями:**
- `getActiveCar()` — получение активного автомобиля
- `getAllCars()` — список всех автомобилей
- `insertCar()` — добавление нового автомобиля
- `updateCar()` — обновление данных автомобиля
- `setActiveCar()` — установка автомобиля активным
- `deleteCar()` — удаление автомобиля (каскадно удаляет сеансы и ТО)
**Операции с диагностическими сеансами:**
- `saveDiagnosticSession()` — сохранение полного сеанса диагностики
- `listSessions()` — получение списка сеансов с агрегированными данными
- `sessionDtcs()` — коды ошибок конкретного сеанса
- `sessionParams()` — значения PID-параметров сеанса
- `sessionParamsHistory()` — полная история параметров для анализа
- `sessionRecommendations()` — рекомендации, сгенерированные для сеанса
**Операции с ТО:**
- `insertMaintenance()` — создание новой операции ТО
- `markMaintenanceDone()` — отметка о выполнении ТО
- `listMaintenance()` — список операций для автомобиля
### Алгоритм сохранения сеанса диагностики
Метод `saveDiagnosticSession()` выполняет следующие действия в рамках транзакции:
1. Создаёт запись в таблице `diagnostic_session` с текущим timestamp
2. Для каждого DTC:
   - Проверяет наличие в `dtc_dictionary`, при отсутствии добавляет
   - Вставляет запись в `session_dtc` с типом (current/pending)
   - Извлекает рекомендацию из `dtc_reference` или `dtc_hints.dart`
   - Сохраняет рекомендацию в таблицу `recommendation`
3. Для каждого PID-параметра:
   - Проверяет наличие в `pid_parameter`, при отсутствии создаёт
   - Сохраняет значение в `session_parameter`
4. Вызывает `RecommendationEngine.build()` для генерации дополнительных рекомендаций на основе комбинации DTC и PID
5. Возвращает ID созданного сеанса
---
## Ядро системы (Core Layer)
### ObdParser
Класс для парсинга ответов от ELM327 согласно стандарту SAE J1979.
**Метод нормализации HEX-строки:**
```dart
static String normalizeHex(String raw) {
  return raw
      .replaceAll(RegExp(r'\s'), '')
      .replaceAll('SEARCHING', '')
      .replaceAll(RegExp(r'[^0-9A-Fa-f]'), '')
      .toUpperCase();
}
```
**Декодирование DTC из двух байт:**
Каждый код ошибки кодируется двумя байтами согласно спецификации:
```dart
static String dtcFromTwoBytes(int a, int b) {
  const types = ['P', 'C', 'B', 'U'];  // Powertrain, Chassis, Body, Network
  final t = types[(a >> 6) & 0x03];     // Биты 7-6 определяют тип системы
  final d2 = (a >> 4) & 0x03;           // Биты 5-4 второй цифры
  final d3 = a & 0x0F;                  // Биты 3-0 третьей цифры
  final d4 = (b >> 4) & 0x0F;           // Биты 7-4 четвёртой цифры
  final d5 = b & 0x0F;                  // Биты 3-0 пятой цифры
  
  String h(int n) => n.toRadixString(16).toUpperCase();
  return '$t$d2${h(d3)}${h(d4)}${h(d5)}';  // Пример: P0301
}
```
**Парсинг ответа режима 03 (сохранённые DTC):**
```dart
static List<String> parseDtcResponse(String raw, {required int mode}) {
  final bytes = hexToBytes(raw);
  if (bytes.length < 3) return [];
  
  final expect = 0x40 + mode;  // Для mode=3 ожидаем 0x43
  if (bytes[0] != expect) {
    final ni = bytes.indexOf(expect);
    if (ni < 0) return [];
    return _parseDtcBytes(bytes.sublist(ni));
  }
  return _parseDtcBytes(bytes);
}
static List<String> _parseDtcBytes(List<int> bytes) {
  final list = <String>[];
  var i = 1;  // Пропускаем байт режима (0x43)
  while (i + 1 < bytes.length) {
    final a = bytes[i];
    final b = bytes[i + 1];
    if (a == 0 && b == 0) break;  // Нулевые байты = конец списка
    list.add(dtcFromTwoBytes(a, b));
    i += 2;  // Каждый DTC занимает 2 байта
  }
  return list;
}
```
**Парсинг значений PID (режим 01):**
```dart
static double? parseMode01Value(String pidHex, String raw) {
  final bytes = hexToBytes(raw);
  if (bytes.length < 4) return null;
  
  var off = 0;
  while (off + 3 < bytes.length) {
    if (bytes[off] == 0x41) {  // Ответ на режим 01
      final p = bytes[off + 1];
      final want = int.parse(pidHex, radix: 16);
      if (p == want && off + 2 < bytes.length) {
        final a = bytes[off + 2];
        final b = off + 3 < bytes.length ? bytes[off + 3] : 0;
        return _formula(pidHex, a, b, bytes, off + 2);
      }
    }
    off++;
  }
  return null;
}
static double? _formula(String pid, int a, int b, List<int> all, int startIndex) {
  switch (pid.toUpperCase()) {
    case '04':  // Нагрузка двигателя (%)
      return a / 2.55;
    case '05':  // Температура ОЖ (°C)
      return a - 40.0;
    case '0C':  // Обороты двигателя (об/мин)
      return ((a * 256) + b) / 4.0;
    case '0D':  // Скорость (км/ч)
      return a.toDouble();
    case '0E':  // Угол опережения зажигания (°)
      return a / 2.0 - 64.0;
    case '0F':  // Температура воздуха (°C)
      return a - 40.0;
    case '10':  // MAF (г/с)
      return ((a * 256) + b) / 100.0;
    case '11':  // Дроссель (%)
      return a / 2.55;
    case '42':  // Напряжение бортовой сети (В)
      return ((a * 256) + b) / 1000.0;
    default:
      return a.toDouble();
  }
}
```
### BluetoothObdService
Сервис низкого уровня для взаимодействия с ELM327 через Bluetooth RFCOMM.
**Подключение и инициализация:**
```dart
Future<void> connect(String address) async {
  await disconnect();
  final conn = await BluetoothConnection.toAddress(address);
  _connection = conn;
  connectedAddress = address;
  _rx.clear();
  _sub = conn.input.listen((data) {
    _rx.write(utf8.decode(data, allowMalformed: true));
  });
}
Future<String> initElm({Duration atzTimeout = const Duration(seconds: 3)}) async {
  await sendRaw('ATZ\r', timeout: atzTimeout);  // Сброс адаптера
  await sendRaw('ATE0\r');  // Отключить эхо
  await sendRaw('ATL0\r');  // Отключить длинные строки
  await sendRaw('ATSP0\r'); // Автовыбор протокола
  final id = await sendRaw('ATI\r');  // Идентификация
  return id;
}
```
**Чтение DTC:**
```dart
Future<List<String>> readStoredDtc() async {
  final r = await sendObd('03', timeout: const Duration(seconds: 2));
  return ObdParser.parseDtcResponse(r, mode: 3);
}
Future<List<String>> readPendingDtc() async {
  final r = await sendObd('07', timeout: const Duration(seconds: 2));
  return ObdParser.parseDtcResponse(r, mode: 7);
}
```
**Чтение VIN:**
```dart
Future<String?> readVin() async {
  try {
    final response = await sendObd('0902', timeout: const Duration(seconds: 3));
    return _parseVinResponse(response);
  } catch (e) {
    return null;
  }
}
String? _parseVinResponse(String response) {
  final cleaned = response.replaceAll('0902:', '').trim();
  final hexParts = cleaned.split(' ').where((part) => part.isNotEmpty).toList();
  
  final vinChars = hexParts.map((hex) {
    if (hex.length == 2) {
      final code = int.parse(hex, radix: 16);
      return String.fromCharCode(code);
    }
    return '';
  }).toList();
  
  final vin = vinChars.join('').trim();
  
  // Валидация VIN (17 символов, без I, O, Q)
  if (vin.length == 17 && !vin.contains(RegExp(r'[IOQ]'))) {
    return vin;
  }
  return null;
}
```
### PidMetaExtended
Класс метаданных для PID-параметров:
```dart
class PidMetaExtended {
  final String pid;
  final String name;
  final String? unit;
  final double? min;
  final double? max;
  final double? normalMin;
  final double? normalMax;
  final bool isExtended;
}
```
Все поддерживаемые PID хранятся в глобальной карте `allPidMeta`:
| PID | Название | Ед. изм. | Мин | Макс | Норма мин | Норма макс |
|-----|----------|----------|-----|------|-----------|------------|
| 04 | Расчётная нагрузка двигателя | % | 0 | 100 | 0 | 100 |
| 05 | Температура охлаждающей жидкости | °C | -40 | 215 | 70 | 120 |
| 0C | Обороты двигателя | об/мин | 0 | 16383 | 700 | 3000 |
| 0D | Скорость автомобиля | км/ч | 0 | 255 | 0 | 200 |
| 0F | Температура всасываемого воздуха | °C | -40 | 215 | -10 | 50 |
| 10 | Массовый расход воздуха (MAF) | г/с | 0 | 655.35 | 2 | 300 |
| 11 | Положение дроссельной заслонки | % | 0 | 100 | 0 | 30 |
| 34 | Напряжение блока управления | В | 0 | 65.535 | 11.5 | 15.0 |
| 4E | Температура масла двигателя | °C | -40 | 215 | 70 | 120 |
### DtcHints
Файл `dtc_hints.dart` содержит расширенную базу знаний о кодах ошибок:
```dart
const Map<String, String> kDtcDescriptionsRu = {
  'P0100': 'Неисправность цепи датчика массового расхода воздуха (MAF)',
  'P0101': 'Неисправность цепи ДМРВ (MAF/MAP) - выход за пределы диапазона',
  'P0171': 'Слишком бедная смесь (Bank 1)',
  'P0300': 'Случайные/множественные пропуски зажигания',
  'P0301': 'Пропуски зажигания — цилиндр 1',
  'P0420': 'Эффективность катализатора ниже порога (Bank 1)',
  // ... более 500 кодов
};
const Map<String, String> kDtcRecommendationsRu = {
  'P0100': 'Проверить цепь ДМРВ, заменить датчик, проверить подсос воздуха',
  'P0171': 'Проверить подсос воздуха, ДМРВ/ДАД, давление топлива, лямбда-зонд',
  'P0301': 'Проверить свечу зажигания цилиндра 1, катушку, компрессию',
  'P0420': 'Проверить герметичность выпуска, лямбда-зонды, состояние катализатора',
  // ... более 170 рекомендаций
};
```
### VinDecoder
Декодирование VIN-номера для автоопределения автомобиля:
```dart
class VinDecoder {
  static const Map<String, String> _wmiCodes = {
    'JT': 'Toyota',
    'JH': 'Honda',
    'JN': 'Nissan',
    'WBA': 'BMW',
    'WDB': 'Mercedes-Benz',
    'WAU': 'Audi',
    'WVW': 'Volkswagen',
    'XTA': 'Lada',
    // ... другие коды
  };
  static VinInfo decodeVin(String vin) {
    if (vin.length != 17) {
      return VinInfo(vin: vin, manufacturer: 'Unknown', isValid: false);
    }
    final normalizedVin = vin.toUpperCase();
    final wmi = normalizedVin.substring(0, 3);  // World Manufacturer Identifier
    final yearCode = normalizedVin[9];           // Код года
    final manufacturer = _wmiCodes[wmi] ?? 'Unknown';
    final brand = _extractBrand(manufacturer);
    final year = _decodeYear(yearCode);
    return VinInfo(
      vin: normalizedVin,
      manufacturer: manufacturer,
      brand: brand,
      year: year,
      isValid: true,
    );
  }
  static int? _decodeYear(String yearCode) {
    const Map<String, int> yearCodes = {
      'A': 2010, 'B': 2011, 'C': 2012, 'D': 2013, 'E': 2014, 'F': 2015,
      'G': 2016, 'H': 2017, 'J': 2018, 'K': 2019, 'L': 2020, 'M': 2021,
      'N': 2022, 'P': 2023, 'R': 2024, 'S': 2025, 'T': 2026, 'V': 2027,
    };
    return yearCodes[yearCode];
  }
}
```
---
## База данных: схема и миграции
### Таблицы базы данных
#### 1. Таблица `car` — Автомобили
```sql
CREATE TABLE car (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  brand TEXT NOT NULL,
  model TEXT NOT NULL,
  generation TEXT,
  year INTEGER,
  vin TEXT,
  current_mileage INTEGER NOT NULL DEFAULT 0,
  is_active INTEGER NOT NULL DEFAULT 0
)
```
#### 2. Таблица `diagnostic_session` — Сеансы диагностики
```sql
CREATE TABLE diagnostic_session (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  car_id INTEGER NOT NULL,
  date_time INTEGER NOT NULL,
  mileage_at_session INTEGER,
  obd_distance_with_mil_km INTEGER,
  notes TEXT,
  FOREIGN KEY (car_id) REFERENCES car(id) ON DELETE CASCADE
)
```
Поле `mileage_at_session` добавлено в версии 6 для фиксации пробега на момент диагностики.
Поле `obd_distance_with_mil_km` хранит расстояние, пройденное с момента загорания лампы MIL (Check Engine).
#### 3. Таблица `dtc_dictionary` — Словарь кодов ошибок
```sql
CREATE TABLE dtc_dictionary (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  code TEXT NOT NULL UNIQUE,
  description TEXT NOT NULL
)
```
#### 4. Таблица `session_dtc` — Связь сеансов и ошибок
```sql
CREATE TABLE session_dtc (
  session_id INTEGER NOT NULL,
  dtc_id INTEGER NOT NULL,
  dtc_type TEXT NOT NULL,  -- 'current' или 'pending'
  PRIMARY KEY (session_id, dtc_id, dtc_type),
  FOREIGN KEY (session_id) REFERENCES diagnostic_session(id) ON DELETE CASCADE,
  FOREIGN KEY (dtc_id) REFERENCES dtc_dictionary(id) ON DELETE CASCADE
)
```
#### 5. Таблица `pid_parameter` — Параметры PID
```sql
CREATE TABLE pid_parameter (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  pid_code TEXT NOT NULL UNIQUE,
  name TEXT NOT NULL,
  unit TEXT,
  normal_min REAL,
  normal_max REAL
)
```
#### 6. Таблица `session_parameter` — Значения параметров в сеансе
```sql
CREATE TABLE session_parameter (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  session_id INTEGER NOT NULL,
  parameter_id INTEGER NOT NULL,
  value REAL NOT NULL,
  timestamp INTEGER NOT NULL,
  FOREIGN KEY (session_id) REFERENCES diagnostic_session(id) ON DELETE CASCADE,
  FOREIGN KEY (parameter_id) REFERENCES pid_parameter(id)
)
```
#### 7. Таблица `recommendation` — Рекомендации
```sql
CREATE TABLE recommendation (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  text TEXT NOT NULL,
  severity INTEGER NOT NULL,  -- 1: низкая, 2: средняя, 3: высокая
  session_id INTEGER,
  dtc_id INTEGER,
  FOREIGN KEY (session_id) REFERENCES diagnostic_session(id) ON DELETE CASCADE,
  FOREIGN KEY (dtc_id) REFERENCES dtc_dictionary(id) ON DELETE CASCADE
)
```
#### 8. Таблица `maintenance_operation` — Операции ТО
```sql
CREATE TABLE maintenance_operation (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  car_id INTEGER NOT NULL,
  title TEXT NOT NULL,
  interval_type TEXT NOT NULL CHECK(interval_type IN ('mileage', 'date')),
  interval_value INTEGER NOT NULL,
  last_done_mileage INTEGER,
  last_done_date INTEGER,
  next_due_mileage INTEGER,
  next_due_date INTEGER,
  last_notified_stage TEXT,
  last_notified_at INTEGER,
  is_completed INTEGER NOT NULL DEFAULT 0,
  FOREIGN KEY (car_id) REFERENCES car(id) ON DELETE CASCADE
)
```
Поля `last_notified_stage` и `last_notified_at` добавлены в версии 5 для отслеживания отправленных уведомлений.
#### 9. Таблица `dtc_reference` — Расширенный справочник DTC
```sql
CREATE TABLE dtc_reference (
  code TEXT PRIMARY KEY,
  description TEXT NOT NULL,
  recommendation TEXT,
  severity INTEGER NOT NULL DEFAULT 2,
  category TEXT,
  source TEXT NOT NULL DEFAULT 'fallback',
  updated_at INTEGER NOT NULL
)
```
Добавлена в версии 4 для хранения обновляемых описаний ошибок.
#### 10. Справочники автомобилей
```sql
CREATE TABLE car_brands (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  name TEXT NOT NULL UNIQUE,
  logo TEXT
)
CREATE TABLE car_models (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  brand_id INTEGER NOT NULL,
  name TEXT NOT NULL,
  FOREIGN KEY (brand_id) REFERENCES car_brands (id) ON DELETE CASCADE,
  UNIQUE(brand_id, name)
)
CREATE TABLE car_generations (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  model_id INTEGER NOT NULL,
  name TEXT NOT NULL,
  years_start INTEGER,
  years_end INTEGER,
  FOREIGN KEY (model_id) REFERENCES car_models (id) ON DELETE CASCADE,
  UNIQUE(model_id, name)
)
```
### Индексы
Для оптимизации производительности созданы следующие индексы:
```sql
CREATE INDEX idx_session_car ON diagnostic_session(car_id);
CREATE INDEX idx_session_date ON diagnostic_session(date_time);
CREATE INDEX idx_maintenance_car ON maintenance_operation(car_id);
CREATE INDEX idx_session_param_session ON session_parameter(session_id);
CREATE INDEX idx_dtc_ref_category ON dtc_reference(category);
```
### Миграции схемы
Текущая версия схемы: **6**
| Версия | Изменения |
|--------|-----------|
| 1 | Базовая схема таблиц |
| 2 | Добавлена таблица `car_generations`, поле `generation` в `car` |
| 3 | Обновление метаданных PID-параметров |
| 4 | Добавлена таблица `dtc_reference` для расширяемого справочника DTC |
| 5 | Добавлены поля `last_notified_stage` и `last_notified_at` в `maintenance_operation` |
| 6 | Добавлены поля `mileage_at_session` и `obd_distance_with_mil_km` в `diagnostic_session` |
---
## Протокол OBD-II и парсинг данных
### Режимы OBD-II
Приложение использует следующие режимы стандарта SAE J1979:
| Режим | Описание | Команда |
|-------|----------|---------|
| 01 | Показ текущих данных | `01<pid>` |
| 03 | Показать сохранённые коды ошибок | `03` |
| 04 | Очистить коды ошибок | `04` |
| 07 | Показать отложенные коды ошибок | `07` |
| 09 | Запрос информации о транспортном средстве | `0902` (VIN) |
### Формат запросов и ответов
**Запрос PID:**
```
TX: 010C  (запрос оборотов двигателя)
RX: 41 0C 1A F8  (ответ: 0x41 = режим 01+0x40, 0x0C = PID, 0x1A 0xF8 = данные)
```
**Расчёт оборотов:**
```
RPM = ((A * 256) + B) / 4
RPM = ((0x1A * 256) + 0xF8) / 4 = (26 * 256 + 248) / 4 = 6904 об/мин
```
**Определение поддерживаемых PID:**
Запрос `0100` возвращает битовую маску поддерживаемых PID 01-20:
```
TX: 0100
RX: 41 00 BE 1F A8 13
```
Байты `BE 1F A8 13` представляют битовую маску:
- `BE` = 10111110 → PID 01, 02, 03, 04, 05, 06, 07 поддерживаются
- `1F` = 00011111 → PID 09, 10, 11, 12, 13 поддерживаются
- и т.д.
---
## Алгоритмы функционирования
### Алгоритм подключения к автомобилю
1. Пользователь выбирает Bluetooth-устройство из списка сопряжённых
2. `DiagnosticsProvider.connectTo()`:
   - Проверяет режим симуляции (`obdSimulation`)
   - Запрашивает разрешения Bluetooth через `PermissionHandler`
   - Вызывает `BluetoothObdService.connect(address)`
   - Выполняет инициализацию ELM327 (`initElm()`):
     - `ATZ` — сброс адаптера
     - `ATE0` — отключение эха
     - `ATL0` — короткие строки ответов
     - `ATSP0` — автоматический выбор протокола
     - `ATI` — идентификация адаптера
   - Сохраняет адрес в настройках (если `save=true`)
   - Вызывает `refreshDtc()` для чтения ошибок
   - Вызывает `_refreshSupportedPids()` для определения доступных PID
   - Вызывает `_tryAutoDetectCar()` для автоопределения по VIN
### Алгоритм периодического опроса PID
1. `startPolling()` запускает `Timer.periodic` с интервалом `pidPollMs`
2. Каждый тик таймера вызывает `_pollOnce()`:
   - Проверяет подключение
   - Для каждого PID из `supportedPids`:
     - Отправляет команду `01<pid>`
     - Парсит ответ через `ObdParser.parseMode01Value()`
     - Сохраняет значение в `livePidValues`
     - Добавляет в `chartHistory[pid]` для графика
     - Ограничивает историю `maxChartPoints` (120 точек)
   - Вызывает `notifyListeners()` для обновления UI
### Алгоритм сохранения сеанса
1. Пользователь завершает диагностику или переходит на другую вкладку
2. Если включено `autoSaveSessionOnLeave`:
   - `DiagnosticsProvider.saveSession()`:
     - Создаёт снимок `livePidValues`
     - Читает PID 21 (дистанция с горящей MIL) отдельно
     - Вызывает `AutodiagRepository.saveDiagnosticSession()`
3. Репозиторий в транзакции:
   - Создаёт запись сеанса
   - Для каждого DTC сохраняет связь и рекомендацию
   - Для каждого PID сохраняет значение
   - Вызывает `RecommendationEngine.build()` для расширенных рекомендаций
   - Возвращает ID сеанса
---
## Модели данных
### Car
```dart
class Car {
  final int id;
  final String brand;
  final String model;
  final String? generation;
  final int? year;
  final String? vin;
  final int currentMileage;
  final bool isActive;
  
  String get displayName => generation != null
      ? '$brand $model ($generation)'
      : '$brand $model';
}
```
### LiveDtc
```dart
class LiveDtc {
  final String code;
  final String description;
  final String type;  // 'current' или 'pending'
}
```
### DiagnosticSessionRow
```dart
class DiagnosticSessionRow {
  final int id;
  final int carId;
  final DateTime dateTime;
  final String? notes;
  final String carLabel;
  final int dtcCount;
  final int? mileageAtSession;
  final int? obdDistanceWithMilKm;
}
```
### SessionParamRow
```dart
class SessionParamRow {
  final String pidCode;
  final String name;
  final String? unit;
  final double value;
  final DateTime at;
}
```
### MaintenanceRow
```dart
class MaintenanceRow {
  final int id;
  final int carId;
  final String title;
  final String intervalType;  // 'mileage' или 'date'
  final int intervalValue;
  final int? lastDoneMileage;
  final DateTime? lastDoneDate;
  final int? nextDueMileage;
  final DateTime? nextDueDate;
  final String? lastNotifiedStage;
  final DateTime? lastNotifiedAt;
  final int isCompleted;
}
```
---
## Управление состоянием приложения
### Инициализация в main.dart
```dart
Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  
  // 1. Инициализация базы данных
  final db = AppDatabase.instance;
  await db.database;
  
  // 2. Загрузка каталога DTC
  await DtcCatalog.instance.load();
  
  // 3. Инициализация уведомлений
  await NotificationService.instance.init();
  await NotificationService.instance.requestAndroidPermission();
  
  // 4. Загрузка настроек
  final settings = SettingsProvider();
  await settings.load();
  
  // 5. Создание репозитория и провайдеров
  final repo = AutodiagRepository(db);
  final export = ExportService(repo);
  final cars = CarsProvider(repo);
  await cars.refresh();
  
  // 6. Регистрация провайдеров
  runApp(
    MultiProvider(
      providers: [
        Provider.value(value: db),
        Provider.value(value: repo),
        Provider.value(value: export),
        ChangeNotifierProvider.value(value: settings),
        ChangeNotifierProvider.value(value: cars),
        ChangeNotifierProvider(
          create: (_) => DiagnosticsProvider(repo, settings),
        ),
        ChangeNotifierProvider(
          create: (_) => HistoryProvider(repo),
        ),
        ChangeNotifierProvider(
          create: (_) => MaintenanceProvider(repo),
        ),
      ],
      child: const AutoDiagApp(),
    ),
  );
}
```
### Темизация приложения
```dart
class AutoDiagApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Consumer<SettingsProvider>(
      builder: (context, settings, _) {
        final lightScheme = ColorScheme.fromSeed(
          seedColor: Colors.teal,
          brightness: Brightness.light,
        );
        final darkScheme = ColorScheme.fromSeed(
          seedColor: Colors.teal,
          brightness: Brightness.dark,
        );
        return MaterialApp(
          title: 'AutoDiag',
          themeMode: settings.themeMode,
          theme: ThemeData(colorScheme: lightScheme, useMaterial3: true),
          darkTheme: ThemeData(colorScheme: darkScheme, useMaterial3: true),
          home: const MainShell(),
        );
      },
    );
  }
}
```
---
## Экспорт данных и генерация отчётов
### ExportService
Класс для генерации PDF-отчётов о диагностике:
```dart
class ExportService {
  ExportService(this._repo);
  final AutodiagRepository _repo;
  
  Future<File> sessionToPdf({
    required int sessionId,
    required DateTime sessionTime,
    required List<SessionDtcRow> dtcs,
    required List<SessionParamRow> params,
    required String notes,
    required String carLabel,
  }) async {
    final doc = pw.Document();
    final font = await _getFont();  // Шрифт с поддержкой кириллицы
    
    // Загрузка рекомендаций и статистики
    final recs = await _repo.sessionRecommendations(sessionId);
    final history = await _repo.sessionParamsHistory(sessionId);
    final stats = await _computeStats(history);
    
    doc.addPage(pw.MultiPage(
      build: (ctx) => [
        pw.Text('AutoDiag — отчёт о диагностике', style: headerStyle),
        pw.Text('Автомобиль: $carLabel'),
        pw.Text('Дата: ${df.format(sessionTime)}'),
        
        // Таблица DTC
        if (dtcs.isNotEmpty) _buildDtcTable(dtcs, baseStyle, boldStyle, headerStyle),
        
        // Таблица параметров
        if (params.isNotEmpty) _buildParamsTable(params, baseStyle, boldStyle, headerStyle),
        
        // Статистика
        if (stats.isNotEmpty) _buildStatsTable(stats, params, baseStyle, boldStyle, headerStyle),
        
        // Рекомендации
        if (recs.isNotEmpty) _buildRecommendationsSection(recs, baseStyle, boldStyle, headerStyle),
        
        // Заметки
        if (notes.isNotEmpty) _buildNotesSection(notes, baseStyle, boldStyle, headerStyle),
      ],
    ));
    
    final dir = await getTemporaryDirectory();
    final file = File(p.join(dir.path, 'autodiag_session_$sessionId.pdf'));
    await file.writeAsBytes(await doc.save());
    return file;
  }
}
```
---
## Система рекомендаций
### RecommendationEngine
Движок генерирует рекомендации на основе:
1. **Кодов DTC** — из справочника `dtc_reference` или `dtc_hints.dart`
2. **Комбинаций DTC+PID** — специальные правила для частых случаев
3. **Истории PID** — анализ трендов параметров
4. **Повторяющихся DTC** — повышенный приоритет для хронических проблем
```dart
class RecommendationEngine {
  List<Recommendation> build({
    required Map<String, double> pidSnapshot,
    required List<LiveDtc> dtcs,
    required Map<String, List<double>> pidHistory,
    required Set<String> recurringDtcs,
  }) {
    final recommendations = <Recommendation>[];
    
    // Рекомендации по повторяющимся DTC
    for (final code in recurringDtcs) {
      recommendations.add(Recommendation(
        text: 'Код $code появляется повторно. Рекомендуется углублённая диагностика.',
        severity: 3,
      ));
    }
    
    // Анализ аномалий PID
    for (final entry in pidSnapshot.entries) {
      final meta = allPidMeta[entry.key];
      if (meta == null) continue;
      
      if (entry.value > (meta.max ?? double.infinity)) {
        recommendations.add(Recommendation(
          text: 'Параметр "${meta.name}" превышает максимальное значение.',
          severity: 2,
        ));
      }
    }
    
    return recommendations;
  }
}
```
---
## Модуль технического обслуживания
### Расчёт статуса ТО
```dart
enum MaintUiStatus { ok, soon, overdue }
MaintUiStatus maintenanceStatus(
  MaintenanceRow row,
  int currentMileage, {
  int warnMileageRemaining = 500,
}) {
  final now = DateTime.now();
  
  if (row.intervalType == 'mileage') {
    final next = row.nextDueMileage;
    if (next == null) return MaintUiStatus.ok;
    if (currentMileage >= next) return MaintUiStatus.overdue;
    if (next - currentMileage <= warnMileageRemaining) return MaintUiStatus.soon;
    return MaintUiStatus.ok;
  } else {
    final next = row.nextDueDate;
    if (next == null) return MaintUiStatus.ok;
    if (now.isAfter(next)) return MaintUiStatus.overdue;
    if (next.difference(now).inDays <= 7) return MaintUiStatus.soon;
    return MaintUiStatus.ok;
  }
}
```
### Уведомления о ТО
Система уведомлений отслеживает стадии приближения ТО:
- `early` — за 30 дней / 1000 км
- `medium` — за 7 дней / 500 км
- `urgent` — просрочено
---
## VIN-декодирование и автоопределение автомобиля
### Процесс автоопределения
1. После подключения к автомобилю вызывается `_tryAutoDetectCar()`
2. `BluetoothObdService.readVin()` отправляет команду `0902`
3. Полученный VIN передаётся в `VinDecoder.decodeVin()`
4. Декодируются:
   - WMI (World Manufacturer Identifier) — первые 3 символа
   - Год выпуска — 9-й символ
5. Результат отображается пользователю
6. Предлагается создать автомобиль с заполненными данными
### Таблица кодов годов
| Код | Год | Код | Год |
|-----|-----|-----|-----|
| A | 2010 | N | 2022 |
| B | 2011 | P | 2023 |
| C | 2012 | R | 2024 |
| D | 2013 | S | 2025 |
| E | 2014 | T | 2026 |
| F | 2015 | V | 2027 |
| G | 2016 | W | 2028 |
| H | 2017 | X | 2029 |
| J | 2018 | Y | 2030 |
| K | 2019 | 1 | 2031 |
| L | 2020 | 2 | 2032 |
| M | 2021 | 3 | 2033 |
---
## Расширение функциональности
### Добавление нового PID-параметра
1. Открыть файл `/workspace/core/obd/pid_metadata.dart`
2. Добавить запись в `allPidMeta`:
```dart
'60': PidMetaExtended(
  pid: '60',
  name: 'Новый параметр',
  unit: 'ед.',
  min: 0,
  max: 100,
  normalMin: 10,
  normalMax: 90,
),
```
3. Добавить формулу декодирования в `ObdParser._formula()`:
```dart
case '60':
  return ((a * 256) + b) / 10.0;
```
### Добавление нового кода DTC
1. Открыть файл `/workspace/core/obd/dtc_hints.dart`
2. Добавить описание в `kDtcDescriptionsRu`:
```dart
'P9999': 'Описание новой ошибки',
```
3. Добавить рекомендацию в `kDtcRecommendationsRu`:
```dart
'P9999': 'Рекомендация по устранению',
```
### Добавление нового автомобиля в каталог
1. Открыть файл `/workspace/data/car_metadata.dart`
2. Найти нужную марку или добавить новую:
```dart
CarBrandMeta(
  name: 'NewBrand',
  models: [
    CarModelMeta(
      name: 'ModelName',
      generations: [
        CarGenerationMeta(name: 'Gen1', yearStart: 2020, yearEnd: null),
      ],
    ),
  ],
),
```
---
## Заключение
AutoDiag представляет собой полнофункциональное решение для мобильной автомобильной диагностики, реализующее современные подходы к архитектуре мобильных приложений. Использование паттерна MVVM, Dependency Injection и Repository Pattern обеспечивает высокую поддерживаемость кода и возможность расширения функциональности.
Приложение поддерживает полный цикл диагностических процедур согласно стандарту SAE J1979, включая чтение и расшифровку кодов ошибок, мониторинг параметров в реальном времени, ведение истории сеансов и планирование технического обслуживания.
Локальная база данных SQLite гарантирует сохранность данных даже без подключения к интернету, а модуль экспорта позволяет формировать профессиональные отчёты в формате PDF.

