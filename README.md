# AutoDiag — Мобильное приложение для автомобильной диагностики OBD-II

**AutoDiag** — это Flutter-приложение для диагностики автомобилей через Bluetooth OBD-II адаптер (ELM327). Приложение позволяет читать и расшифровывать коды ошибок (DTC), отслеживать параметры двигателя в реальном времени, вести историю диагностических сеансов и управлять регламентом технического обслуживания (ТО).

---

## 📋 Содержание

1. [Архитектура приложения](#архитектура-приложения)
2. [Структура проекта](#структура-проекта)
3. [База данных](#база-данных)
4. [Ядро системы (Core)](#ядро-системы-core)
5. [Слой данных (Data)](#слой-данных-data)
6. [Слой представления (Presentation)](#слой-представления-presentation)
7. [Взаимосвязи между компонентами](#взаимосвязи-между-компонентами)
8. [Алгоритмы работы](#алгоритмы-работы)
9. [Примеры использования](#примеры-использования)

---

## 🏗 Архитектура приложения

Приложение построено по архитектуре **MVVM (Model-View-ViewModel)** с использованием паттерна **Provider** для управления состоянием.

```
┌─────────────────────────────────────────────────────────┐
│                    Presentation Layer                    │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────┐  │
│  │   Screens   │  │   Providers  │  │   Widgets     │  │
│  │   (UI)      │◄─┤   (State)    │  │   (Reusable)  │  │
│  └─────────────┘  └──────────────┘  └───────────────┘  │
└─────────────────────────────────────────────────────────┘
                          ▲
                          │
┌─────────────────────────────────────────────────────────┐
│                      Data Layer                          │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────┐  │
│  │ Repository  │  │   Models     │  │   Database    │  │
│  └─────────────┘  └──────────────┘  └───────────────┘  │
└─────────────────────────────────────────────────────────┘
                          ▲
                          │
┌─────────────────────────────────────────────────────────┐
│                       Core Layer                         │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────┐  │
│  │ OBD Parser  │  │   Services   │  │   Metadata    │  │
│  └─────────────┘  └──────────────┘  └───────────────┘  │
└─────────────────────────────────────────────────────────┘
```

---

## 📁 Структура проекта

```
/workspace/
├── main.dart                          # Точка входа, инициализация
├── autodiag_app.dart                  # Корневой виджет MaterialApp
│
├── core/                              # Ядро бизнес-логики
│   ├── obd/
│   │   ├── dtc_catalog.dart           # Полный каталог DTC кодов
│   │   ├── dtc_hints.dart             # Описания и рекомендации по DTC (~1077 строк)
│   │   ├── obd_parser.dart            # Парсинг ответов ELM327
│   │   └── pid_metadata.dart          # Метаданные PID параметров
│   └── services/
│       ├── bluetooth_obd_service.dart # Работа с Bluetooth/ELM327
│       ├── export_service.dart        # Экспорт в PDF/CSV
│       ├── notification_service.dart  # Уведомления
│       └── vin_decoder.dart           # Декодирование VIN
│
├── data/                              # Слой данных
│   ├── db/
│   │   └── app_database.dart          # SQLite база данных
│   ├── models/
│   │   └── autodiag_models.dart       # Модели данных
│   ├── autodiag_repository.dart       # Репозиторий для работы с БД
│   └── car_metadata.dart              # Каталог марок/моделей авто
│
└── presentation/                      # UI слой
    ├── providers/                     # State Management (Provider)
    │   ├── cars_provider.dart         # Управление автомобилями
    │   ├── diagnostics_provider.dart  # Диагностика, опрос PID
    │   ├── history_provider.dart      # История сеансов
    │   ├── maintenance_provider.dart  # ТО и регламенты
    │   └── settings_provider.dart     # Настройки приложения
    └── screens/autodiag/              # Экраны приложения
        ├── main_shell.dart            # Главный экран с навигацией
        ├── home_tab.dart              # Главная страница
        ├── diagnostics_tab.dart       # Вкладка диагностики
        ├── history_tab.dart           # История сеансов
        ├── maintenance_tab.dart       # Техническое обслуживание
        ├── cars_tab.dart              # Управление автомобилями
        ├── add_car_wizard.dart        # Мастер добавления авто
        ├── edit_car_screen.dart       # Редактирование авто
        ├── session_detail_screen.dart # Детали сеанса
        ├── device_picker_sheet.dart   # Выбор Bluetooth устройства
        └── vin_detection_screen.dart  # Определение по VIN
```

---

## 🗄 База данных

### Схема БД (SQLite)

База данных хранится в файле `autodiag.db` и содержит следующие таблицы:

#### 1. Таблица `car` — Автомобили
```sql
CREATE TABLE car (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  brand TEXT NOT NULL,           -- Марка (например, "Toyota")
  model TEXT NOT NULL,           -- Модель (например, "Camry")
  generation TEXT,               -- Поколение (например, "XV70")
  year INTEGER,                  -- Год выпуска
  vin TEXT,                      -- VIN-номер
  current_mileage INTEGER NOT NULL DEFAULT 0,
  is_active INTEGER NOT NULL DEFAULT 0  -- Флаг активного авто
)
```

#### 2. Таблица `diagnostic_session` — Сеансы диагностики
```sql
CREATE TABLE diagnostic_session (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  car_id INTEGER NOT NULL,       -- Ссылка на car.id
  date_time INTEGER NOT NULL,    -- Timestamp начала сеанса
  notes TEXT,                    -- Заметки пользователя
  FOREIGN KEY (car_id) REFERENCES car(id) ON DELETE CASCADE
)
```

#### 3. Таблица `dtc_dictionary` — Справочник кодов ошибок
```sql
CREATE TABLE dtc_dictionary (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  code TEXT NOT NULL UNIQUE,     -- Код ошибки (например, "P0301")
  description TEXT NOT NULL      -- Описание ошибки
)
```

#### 4. Таблица `session_dtc` — Ошибки в сеансе
```sql
CREATE TABLE session_dtc (
  session_id INTEGER NOT NULL,
  dtc_id INTEGER NOT NULL,
  dtc_type TEXT NOT NULL,        -- "current" или "pending"
  PRIMARY KEY (session_id, dtc_id, dtc_type),
  FOREIGN KEY (session_id) REFERENCES diagnostic_session(id) ON DELETE CASCADE,
  FOREIGN KEY (dtc_id) REFERENCES dtc_dictionary(id) ON DELETE CASCADE
)
```

#### 5. Таблица `pid_parameter` — Параметры PID
```sql
CREATE TABLE pid_parameter (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  pid_code TEXT NOT NULL UNIQUE, -- Код PID (например, "0C")
  name TEXT NOT NULL,            -- Название (например, "Обороты двигателя")
  unit TEXT,                     -- Единица измерения (например, "об/мин")
  normal_min REAL,               -- Минимальное нормальное значение
  normal_max REAL                -- Максимальное нормальное значение
)
```

#### 6. Таблица `session_parameter` — Значения параметров в сеансе
```sql
CREATE TABLE session_parameter (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  session_id INTEGER NOT NULL,
  parameter_id INTEGER NOT NULL,
  value REAL NOT NULL,           -- Измеренное значение
  timestamp INTEGER NOT NULL,    -- Время замера
  FOREIGN KEY (session_id) REFERENCES diagnostic_session(id) ON DELETE CASCADE,
  FOREIGN KEY (parameter_id) REFERENCES pid_parameter(id)
)
```

#### 7. Таблица `recommendation` — Рекомендации
```sql
CREATE TABLE recommendation (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  text TEXT NOT NULL,            -- Текст рекомендации
  severity INTEGER NOT NULL,     -- Важность (1-3)
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
  title TEXT NOT NULL,           -- Название (например, "Замена масла")
  interval_type TEXT NOT NULL CHECK(interval_type IN ('mileage', 'date')),
  interval_value INTEGER NOT NULL, -- Интервал (км или дни)
  last_done_mileage INTEGER,     -- Пробег последнего ТО
  last_done_date INTEGER,        -- Дата последнего ТО
  next_due_mileage INTEGER,      -- Следующее ТО по пробегу
  next_due_date INTEGER,         -- Следующее ТО по дате
  is_completed INTEGER NOT NULL DEFAULT 0,
  FOREIGN KEY (car_id) REFERENCES car(id) ON DELETE CASCADE
)
```

#### 9. Таблицы справочников автомобилей
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

### Инициализация БД

Файл: `/workspace/data/db/app_database.dart`

```dart
class AppDatabase {
  AppDatabase._();
  static final AppDatabase instance = AppDatabase._();
  Database? _db;
  
  static const int _version = 3;  // Версия схемы БД
  
  Future<Database> get database async {
    if (_db != null) return _db!;
    _db = await _open();
    return _db!;
  }
  
  Future<Database> _open() async {
    final dir = await getDatabasesPath();
    final path = p.join(dir, 'autodiag.db');
    return openDatabase(
      path,
      version: _version,
      onConfigure: (db) async {
        await db.execute('PRAGMA foreign_keys = ON');  // Включаем внешние ключи
      },
      onCreate: (db, version) async {
        await _createTables(db);
        await _seedPidParameters(db);    // Заполняем PID параметры
        await _seedCarData(db);          // Заполняем марки/модели
      },
      onUpgrade: (db, oldVersion, newVersion) async {
        // Миграции между версиями
        if (oldVersion < 2) {
          await _createNewTablesV2(db);
          await _seedCarData(db);
        }
        if (oldVersion < 3) {
          await _updatePidParameters(db);
        }
      },
    );
  }
}
```

---

## ⚙️ Ядро системы (Core)

### 1. OBD Parser (`/workspace/core/obd/obd_parser.dart`)

Парсер отвечает за декодирование сырых ответов от ELM327 адаптера согласно стандарту SAE J1979.

#### Ключевые методы:

**Нормализация HEX-строки:**
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
```dart
static String dtcFromTwoBytes(int a, int b) {
  const types = ['P', 'C', 'B', 'U'];  // Powertrain, Chassis, Body, Network
  final t = types[(a >> 6) & 0x03];     // Первый символ (тип системы)
  final d2 = (a >> 4) & 0x03;           // Вторая цифра
  final d3 = a & 0x0F;                  // Третья цифра
  final d4 = (b >> 4) & 0x0F;           // Четвёртая цифра
  final d5 = b & 0x0F;                  // Пятая цифра
  
  String h(int n) => n.toRadixString(16).toUpperCase();
  return '$t$d2${h(d3)}${h(d4)}${h(d5)}';  // Например: P0301
}
```

**Парсинг ответа на режим 03 (сохранённые DTC):**
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

### 2. DTC Hints (`/workspace/core/obd/dtc_hints.dart`)

Расширенная база описаний и рекомендаций по кодам ошибок (1077 строк).

#### Структура данных:

```dart
/// Расширенные описания DTC
const Map<String, String> kDtcDescriptionsRu = {
  'P0100': 'Неисправность цепи датчика массового расхода воздуха (MAF)',
  'P0101': 'Неисправность цепи ДМРВ (MAF/MAP) - выход за пределы диапазона',
  'P0171': 'Слишком бедная смесь (Bank 1)',
  'P0300': 'Случайные/множественные пропуски зажигания',
  'P0301': 'Пропуски зажигания — цилиндр 1',
  'P0420': 'Эффективность катализатора ниже порога (Bank 1)',
  // ... ещё ~500 кодов
};

/// Рекомендации по устранению
const Map<String, String> kDtcRecommendationsRu = {
  'P0100': 'Проверить цепь ДМРВ, заменить датчик, проверить подсос воздуха',
  'P0171': 'Проверить подсос воздуха, ДМРВ/ДАД, давление топлива, лямбда-зонд',
  'P0301': 'Проверить свечу зажигания цилиндра 1, катушку, компрессию',
  'P0420': 'Проверить герметичность выпуска, лямбда-зонды, состояние катализатора',
  // ... ещё ~170 рекомендаций
};
```

#### Функции доступа:

```dart
String? dtcDescriptionRu(String code) {
  return kDtcDescriptionsRu[code.toUpperCase()];
}

String? dtcRecommendationRu(String code) {
  return kDtcRecommendationsRu[code.toUpperCase()];
}
```

### 3. PID Metadata (`/workspace/core/obd/pid_metadata.dart`)

Метаданные для всех поддерживаемых PID параметров.

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

final Map<String, PidMetaExtended> allPidMeta = {
  '04': PidMetaExtended(
    pid: '04',
    name: 'Расчётная нагрузка двигателя',
    unit: '%',
    min: 0,
    max: 100,
    normalMin: 0,
    normalMax: 100,
  ),
  '05': PidMetaExtended(
    pid: '05',
    name: 'Температура охлаждающей жидкости',
    unit: '°C',
    min: -40,
    max: 215,
    normalMin: 70,
    normalMax: 120,
  ),
  '0C': PidMetaExtended(
    pid: '0C',
    name: 'Обороты двигателя',
    unit: 'об/мин',
    min: 0,
    max: 16383,
    normalMin: 700,
    normalMax: 3000,
  ),
  // ... ещё параметры
};
```

### 4. Bluetooth OBD Service (`/workspace/core/services/bluetooth_obd_service.dart`)

Сервис для работы с Bluetooth OBD-II адаптером.

#### Подключение и инициализация:

```dart
class BluetoothObdService {
  BluetoothConnection? _connection;
  StreamSubscription<List<int>>? _sub;
  final StringBuffer _rx = StringBuffer();
  
  Future<void> connect(String address) async {
    await disconnect();
    final conn = await BluetoothConnection.toAddress(address);
    _connection = conn;
    connectedAddress = address;
    
    // Слушаем входящие данные
    _sub = conn.input.listen((data) {
      _rx.write(utf8.decode(data, allowMalformed: true));
    });
  }
  
  /// Инициализация ELM327
  Future<String> initElm({Duration atzTimeout = const Duration(seconds: 3)}) async {
    await sendRaw('ATZ\r', timeout: atzTimeout);  // Сброс адаптера
    await sendRaw('ATE0\r');  // Отключить эхо
    await sendRaw('ATL0\r');  // Отключить длинные строки
    await sendRaw('ATSP0\r'); // Автовыбор протокола
    final id = await sendRaw('ATI\r');  // Идентификация
    return id;
  }
  
  Future<String> sendRaw(String line, {Duration? timeout}) async {
    _rx.clear();
    _connection!.output.add(utf8.encode(line));
    await _connection!.output.allSent;
    return _readUntilPrompt(timeout ?? const Duration(milliseconds: 1200));
  }
  
  Future<String> _readUntilPrompt(Duration timeout) async {
    final deadline = DateTime.now().add(timeout);
    while (DateTime.now().isBefore(deadline)) {
      final s = _rx.toString();
      if (s.contains('>')) {  // '>' = prompt ELM327
        final i = s.indexOf('>');
        return s.substring(0, i).trim();
      }
      await Future.delayed(const Duration(milliseconds: 30));
    }
    return _rx.toString().trim();
  }
}
```

#### Чтение DTC:

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

#### Чтение VIN:

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

### 5. VIN Decoder (`/workspace/core/services/vin_decoder.dart`)

Декодирование VIN-номера для автоопределения автомобиля.

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
    // ... ещё коды
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
      // ...
    };
    return yearCodes[yearCode];
  }
}
```

### 6. Export Service (`/workspace/core/services/export_service.dart`)

Экспорт отчётов в PDF и CSV.

#### Генерация PDF отчёта:

```dart
Future<File> sessionToPdf({
  required int sessionId,
  required DateTime sessionTime,
  required List<SessionDtcRow> dtcs,
  required List<SessionParamRow> params,
  required String notes,
  required String carLabel,
}) async {
  final doc = pw.Document();
  final font = await _getFont();  // Загрузка шрифта с поддержкой кириллицы
  
  // Получаем рекомендации и статистику
  final recs = await _repo.sessionRecommendations(sessionId);
  final history = await _repo.sessionParamsHistory(sessionId);
  final stats = await _computeStats(history);
  
  doc.addPage(pw.MultiPage(
    build: (ctx) => [
      // Заголовок
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
```

---

## 💾 Слой данных (Data)

### AutodiagRepository (`/workspace/data/autodiag_repository.dart`)

Репозиторий предоставляет единый интерфейс для работы с базой данных.

#### Сохранение диагностического сеанса:

```dart
Future<int> saveDiagnosticSession({
  required int carId,
  required List<LiveDtc> dtcs,
  required Map<String, double> pidSnapshot,
  String notes = '',
}) async {
  final db = await _d;
  final now = DateTime.now().millisecondsSinceEpoch;
  
  return db.transaction<int>((txn) async {
    // 1. Создаём сеанс
    final sid = await txn.insert('diagnostic_session', {
      'car_id': carId,
      'date_time': now,
      'notes': notes,
    });
    
    // 2. Сохраняем DTC
    for (final d in dtcs) {
      final did = await _ensureDtcTxn(txn, d.code, d.description);
      await txn.insert('session_dtc', {
        'session_id': sid,
        'dtc_id': did,
        'dtc_type': d.type,
      }, conflictAlgorithm: ConflictAlgorithm.replace);
      
      // 3. Добавляем рекомендацию из базы
      final rec = dtcRecommendationRu(d.code);
      if (rec != null) {
        await txn.insert('recommendation', {
          'text': rec,
          'severity': 2,
          'session_id': sid,
          'dtc_id': did,
        });
      }
    }
    
    // 4. Сохраняем значения PID
    for (final e in pidSnapshot.entries) {
      final pidUpper = e.key.toUpperCase().padLeft(2, '0');
      final pRow = await txn.query(
        'pid_parameter',
        where: 'pid_code = ?',
        whereArgs: [pidUpper],
        limit: 1,
      );
      
      int paramId;
      if (pRow.isEmpty) {
        paramId = await txn.insert('pid_parameter', {
          'pid_code': pidUpper,
          'name': 'PID $pidUpper',
          'unit': '',
        });
      } else {
        paramId = pRow.first['id'] as int;
      }
      
      await txn.insert('session_parameter', {
        'session_id': sid,
        'parameter_id': paramId,
        'value': e.value,
        'timestamp': now,
      });
    }
    
    return sid;
  });
}
```

#### Получение истории сеансов:

```dart
Future<List<DiagnosticSessionRow>> listSessions({int? carIdFilter}) async {
  final db = await _d;
  final where = carIdFilter != null ? 'WHERE s.car_id = ?' : '';
  final args = carIdFilter != null ? [carIdFilter] : <Object?>[];
  
  final rows = await db.rawQuery('''
    SELECT s.id, s.car_id, s.date_time, s.notes,
      (SELECT COUNT(*) FROM session_dtc sd WHERE sd.session_id = s.id) AS dtc_count,
      printf('%s %s', c.brand, c.model) AS car_label
    FROM diagnostic_session s
    JOIN car c ON c.id = s.car_id
    $where
    ORDER BY s.date_time DESC
  ''', args);
  
  return rows.map((m) => DiagnosticSessionRow(
    id: (m['id'] as num).toInt(),
    carId: (m['car_id'] as num).toInt(),
    dateTime: DateTime.fromMillisecondsSinceEpoch((m['date_time'] as num).toInt()),
    notes: m['notes'] as String?,
    carLabel: m['car_label'] as String? ?? '',
    dtcCount: (m['dtc_count'] as num?)?.toInt() ?? 0,
  )).toList();
}
```

---

## 🎨 Слой представления (Presentation)

### Providers (State Management)

#### DiagnosticsProvider (`/workspace/presentation/providers/diagnostics_provider.dart`)

Управляет состоянием диагностики: подключение, опрос PID, чтение DTC.

```dart
class DiagnosticsProvider extends ChangeNotifier {
  final AutodiagRepository _repo;
  final SettingsProvider _settings;
  final BluetoothObdService _obd;
  
  // Состояние
  bool scanning = false;
  bool connecting = false;
  String status = 'Не подключено';
  List<LiveDtc> liveDtcs = [];
  final Map<String, double> livePidValues = {};
  final Map<String, List<double>> chartHistory = {};
  Set<String> supportedPids = {};
  
  Timer? _pollTimer;
  bool polling = false;
  
  // Подключение
  Future<void> connectTo(String address, {bool save = true}) async {
    if (_settings.obdSimulation) {
      await startSimulation();
      return;
    }
    
    connecting = true;
    status = 'Подключение…';
    notifyListeners();
    
    try {
      await _obd.connect(address);
      await _obd.initElm();  // Инициализация ELM327
      
      if (save) await _settings.setLastBt(address);
      
      status = 'Подключено';
      reconnectAttempts = 0;
      
      await refreshDtc();  // Чтение ошибок
      await _refreshSupportedPids();  // Определение поддерживаемых PID
      await _tryAutoDetectCar();  // Автоопределение по VIN
      
      connecting = false;
      notifyListeners();
    } catch (e) {
      connecting = false;
      status = 'Ошибка: $e';
      await _obd.disconnect();
      notifyListeners();
    }
  }
  
  // Опрос PID
  void startPolling() {
    if (!isConnected || polling) return;
    polling = true;
    chartHistory.clear();
    
    _pollTimer = Timer.periodic(
      Duration(milliseconds: _settings.pidPollMs),
      (_) => _pollOnce()
    );
    notifyListeners();
    _pollOnce();
  }
  
  Future<void> _pollOnce() async {
    final pids = supportedPids.toList();
    if (pids.isEmpty) return;
    
    for (final pid in pids) {
      try {
        final v = await readPidValue(pid);
        if (v != null) {
          livePidValues[pid.toUpperCase()] = v;
          
          // Добавляем в историю для графика
          final key = pid.toUpperCase();
          chartHistory.putIfAbsent(key, () => []);
          chartHistory[key]!.add(v);
          
          while (chartHistory[key]!.length > maxChartPoints) {
            chartHistory[key]!.removeAt(0);
          }
        }
      } catch (_) {}
    }
    notifyListeners();
  }
  
  // Чтение DTC
  Future<void> refreshDtc() async {
    if (!isConnected) return;
    
    liveDtcs.clear();
    
    // Читаем сохранённые ошибки (режим 03)
    final stored = await _obd.readStoredDtc();
    for (final code in stored) {
      liveDtcs.add(LiveDtc(
        code: code,
        description: dtcDescriptionRu(code) ?? 'Неизвестная ошибка',
        type: 'current',
      ));
    }
    
    // Читаем отложенные ошибки (режим 07)
    final pending = await _obd.readPendingDtc();
    for (final code in pending) {
      if (!liveDtcs.any((d) => d.code == code)) {
        liveDtcs.add(LiveDtc(
          code: code,
          description: dtcDescriptionRu(code) ?? 'Неизвестная ошибка',
          type: 'pending',
        ));
      }
    }
    
    notifyListeners();
  }
}
```

### Main Shell (`/workspace/presentation/screens/autodiag/main_shell.dart`)

Главный экран с нижней навигационной панелью.

```dart
class MainShell extends StatefulWidget {
  @override
  State<MainShell> createState() => _MainShellState();
}

class _MainShellState extends State<MainShell> {
  int index = 0;
  static const _titles = ['Главная', 'Диагностика', 'История', 'ТО', 'Авто'];
  
  @override
  void initState() {
    super.initState();
    WidgetsBinding.instance.addPostFrameCallback((_) async {
      final cars = context.read<CarsProvider>();
      final hist = context.read<HistoryProvider>();
      final maint = context.read<MaintenanceProvider>();
      final diag = context.read<DiagnosticsProvider>();
      
      await hist.refresh();
      final active = cars.active;
      await maint.loadForCar(active?.id, active, settings.maintenanceNotify);
      await diag.tryAutoConnect();  // Автоподключение к последнему устройству
    });
  }
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: IndexedStack(
        index: index,
        children: [
          HomeTab(...),
          const DiagnosticsTab(),
          const HistoryTab(),
          const MaintenanceTab(),
          const CarsTab(),
        ],
      ),
      bottomNavigationBar: NavigationBar(
        selectedIndex: index,
        onDestinationSelected: (i) async {
          final prev = index;
          setState(() => index = i);
          
          // Автосохранение сеанса при уходе с вкладки диагностики
          if (prev == 1 && i != 1) {
            if (st.autoSaveSessionOnLeave) {
              final diag = context.read<DiagnosticsProvider>();
              final cars = context.read<CarsProvider>();
              final active = cars.active;
              
              if (diag.isConnected && active != null) {
                await diag.saveSession(car: active);
                await context.read<HistoryProvider>().refresh();
              }
            }
          }
        },
        destinations: const [
          NavigationDestination(icon: Icon(Icons.home_outlined), label: 'Главная'),
          NavigationDestination(icon: Icon(Icons.sensors_outlined), label: 'OBD'),
          NavigationDestination(icon: Icon(Icons.history_outlined), label: 'История'),
          NavigationDestination(icon: Icon(Icons.build_circle_outlined), label: 'ТО'),
          NavigationDestination(icon: Icon(Icons.directions_car_outlined), label: 'Авто'),
        ],
      ),
    );
  }
}
```

---

## 🔗 Взаимосвязи между компонентами

### Диаграмма зависимостей

```
main.dart
  │
  ├─► AppDatabase.instance (инициализация БД)
  ├─► DtcCatalog.instance.load() (загрузка каталога DTC)
  ├─► NotificationService.instance.init()
  ├─► SettingsProvider().load()
  │
  └─► MultiProvider (регистрация провайдеров)
       │
       ├─► Provider<AppDatabase>
       ├─► Provider<AutodiagRepository>
       ├─► Provider<ExportService>
       ├─► ChangeNotifierProvider<SettingsProvider>
       ├─► ChangeNotifierProvider<CarsProvider>
       ├─► ChangeNotifierProvider<DiagnosticsProvider>
       ├─► ChangeNotifierProvider<HistoryProvider>
       └─► ChangeNotifierProvider<MaintenanceProvider>
            │
            └─► AutoDiagApp
                 │
                 └─► MainShell
                      │
                      ├─► HomeTab
                      ├─► DiagnosticsTab ◄─► DiagnosticsProvider
                      ├─► HistoryTab ◄─► HistoryProvider
                      ├─► MaintenanceTab ◄─► MaintenanceProvider
                      └─► CarsTab ◄─► CarsProvider
```

### Поток данных при диагностике

```
1. Пользователь нажимает "Подключиться"
   │
   ▼
2. DiagnosticsProvider.connectTo(address)
   │
   ├─► BluetoothObdService.connect(address)
   ├─► BluetoothObdService.initElm()
   │     ├─► ATZ (сброс)
   │     ├─► ATE0 (отключить эхо)
   │     ├─► ATL0 (короткие строки)
   │     ├─► ATSP0 (автопротокол)
   │     └─► ATI (идентификация)
   │
   ├─► DiagnosticsProvider.refreshDtc()
   │     ├─► BluetoothObdService.readStoredDtc() → Режим 03
   │     ├─► ObdParser.parseDtcResponse()
   │     └─► dtcDescriptionRu() (описание из базы)
   │
   ├─► DiagnosticsProvider._refreshSupportedPids()
   │     ├─► 0100 → PID 01-20
   │     ├─► 0120 → PID 21-40
   │     ├─► 0140 → PID 41-60
   │     └─► 0160 → PID 61-80
   │
   └─► DiagnosticsProvider._tryAutoDetectCar()
         ├─► BluetoothObdService.readVin() → Режим 0902
         ├─► VinDecoder.decodeVin()
         └─► Создание Car из VIN

3. Пользователь запускает опрос
   │
   ▼
4. DiagnosticsProvider.startPolling()
   │
   └─► Timer.periodic(...)
        │
        └─► _pollOnce()
             │
             ├─► Для каждого PID из supportedPids:
             │     ├─► readPidValue(pid)
             │     │     ├─► sendObd('01{PID}')
             │     │     └─► ObdParser.parseMode01Value()
             │     │
             │     ├─► livePidValues[pid] = value
             │     └─► chartHistory[pid].add(value)
             │
             └─► notifyListeners() (обновление UI)

5. Пользователь сохраняет сеанс
   │
   ▼
6. DiagnosticsProvider.saveSession(car: active)
   │
   └─► AutodiagRepository.saveDiagnosticSession()
        │
        ├─► INSERT INTO diagnostic_session
        ├─► Для каждого DTC:
        │     ├─► INSERT/SELECT dtc_dictionary
        │     ├─► INSERT session_dtc
        │     └─► INSERT recommendation (из dtcRecommendationRu)
        │
        └─► Для каждого PID:
              └─► INSERT session_parameter
```

---

## 🧠 Алгоритмы работы

### 1. Алгоритм чтения DTC

```dart
// Шаг 1: Отправка команды режима 03
sendObd('03') → "43 10 03 00 00 00 00"

// Шаг 2: Парсинг ответа
bytes = [0x43, 0x10, 0x03, 0x00, 0x00, 0x00, 0x00]
       ↑    ↑    ↑
       │    │    └─ DTC 1: 0x10, 0x03 → P0003
       │    └─ DTC 0: 0x43 (режим)
       └─ 0x43 = ответ на режим 03

// Шаг 3: Декодирование пары байт в DTC
a = 0x10 = 00010000
b = 0x03 = 00000011

typeIndex = (a >> 6) & 0x03 = 0 → 'P' (Powertrain)
digit2 = (a >> 4) & 0x03 = 0 → 0
digit3 = a & 0x0F = 0 → 0
digit4 = (b >> 4) & 0x0F = 0 → 0
digit5 = b & 0x0F = 3 → 3

Result: P0003
```

### 2. Алгоритм определения поддерживаемых PID

```dart
// Запрос: 0100
// Ответ: "41 00 BE 1F A8 13"
//        ↑  ↑  └─ Байты битовой маски
//        │  └─ PID 00
//        └─ Режим 01

// Парсим битовую маску:
BE = 10111110
1F = 00011111
A8 = 10101000
13 = 00010011

// Бит 1 = 1 → PID 01 поддерживается
// Бит 2 = 1 → PID 02 поддерживается
// ...
// Бит 5 = 1 → PID 05 (температура ОЖ) поддерживается
// Бит 12 = 1 → PID 0C (обороты) поддерживается
// Бит 13 = 1 → PID 0D (скорость) поддерживается
```

### 3. Алгоритм расчёта статистики параметров

```dart
Future<Map<String, _ParamStats>> _computeStats(List<SessionParamRow> history) async {
  final stats = <String, _ParamStats>{};
  
  // Группируем по PID
  final grouped = <String, List<SessionParamRow>>{};
  for (final p in history) {
    grouped.putIfAbsent(p.pidCode, () => []).add(p);
  }
  
  // Для каждой группы считаем статистику
  for (final entry in grouped.entries) {
    final values = entry.value.map((p) => p.value).toList();
    
    final min = values.reduce((a, b) => a < b ? a : b);
    final max = values.reduce((a, b) => a > b ? a : b);
    final avg = values.reduce((a, b) => a + b) / values.length;
    final last = values.last;
    
    stats[entry.key] = _ParamStats(min: min, max: max, avg: avg, last: last);
  }
  
  return stats;
}
```

### 4. Алгоритм автоподключения

```dart
Future<void> tryAutoConnect() async {
  // 1. Проверяем режим симуляции
  if (_settings.obdSimulation) {
    await startSimulation();
    return;
  }
  
  // 2. Получаем последний адрес из настроек
  final addr = _settings.lastBtAddress;
  
  // 3. Проверяем флаг автоподключения
  if (!_settings.autoConnectBt || addr == null || addr.isEmpty) return;
  
  // 4. Подключаемся
  await connectTo(addr, save: false);
}
```

---

## 📝 Примеры использования

### Добавление нового автомобиля

```dart
// Через репозиторий
final carId = await repo.insertCar(
  brand: 'Toyota',
  model: 'Camry',
  generation: 'XV70',
  year: 2020,
  vin: '4T1B11HK8LU123456',
  mileage: 50000,
  setActive: true,
);

// Через VIN (автоопределение)
await diag.obd.readVin();  // Чтение VIN из ЭБУ
final vinInfo = VinDecoder.decodeVin(vin);  // Декодирование

await repo.insertCar(
  brand: vinInfo.brand,
  model: vinInfo.model,
  year: vinInfo.year,
  vin: vinInfo.vin,
);
```

### Чтение параметров в реальном времени

```dart
// Подписка на изменения
Consumer<DiagnosticsProvider>(
  builder: (context, diag, _) {
    final rpm = diag.livePidValues['0C'];  // Обороты
    final temp = diag.livePidValues['05']; // Температура ОЖ
    final speed = diag.livePidValues['0D']; // Скорость
    
    return Column(
      children: [
        Text('Обороты: ${rpm?.toStringAsFixed(0)} об/мин'),
        Text('Температура: ${temp?.toStringAsFixed(1)} °C'),
        Text('Скорость: ${speed?.toStringAsFixed(0)} км/ч'),
      ],
    );
  },
)
```

### Сохранение сеанса диагностики

```dart
// Сохранение с заметками
final sessionId = await diag.saveSession(
  car: activeCar,
  notes: 'Проверка перед покупкой',
);

// Сеанс автоматически включает:
// 1. Все текущие DTC с описаниями
// 2. Моментальные значения всех PID
// 3. Рекомендации по каждой ошибке
```

### Экспорт отчёта в PDF

```dart
final file = await export.sessionToPdf(
  sessionId: sessionId,
  sessionTime: sessionTime,
  dtcs: dtcs,
  params: params,
  notes: notes,
  carLabel: carLabel,
);

await Share.shareXFiles([XFile(file.path)]);
```

---

## 📊 Производительность и оптимизация

### Индексы БД

```sql
CREATE INDEX idx_session_car ON diagnostic_session(car_id);
CREATE INDEX idx_session_date ON diagnostic_session(date_time);
CREATE INDEX idx_maintenance_car ON maintenance_operation(car_id);
CREATE INDEX idx_session_param_session ON session_parameter(session_id);
```

### Транзакции

Все операции записи используют транзакции для обеспечения целостности:

```dart
await db.transaction((txn) async {
  final sid = await txn.insert('diagnostic_session', {...});
  for (final dtc in dtcs) {
    await txn.insert('session_dtc', {...});
  }
  for (final param in params) {
    await txn.insert('session_parameter', {...});
  }
});
```

### Кэширование

- `DtcCatalog` загружается один раз при старте приложения
- `PidMetaExtended` хранится в памяти как `Map`
- Шрифт для PDF кэшируется после первой загрузки

---

## 🔐 Безопасность

- Внешние ключи включены: `PRAGMA foreign_keys = ON`
- При удалении автомобиля каскадно удаляются все связанные сеансы и ТО
- VIN не содержит символов I, O, Q (валидация при чтении)

---

## 📦 Зависимости

Основные пакеты (из `pubspec.yaml`):

```yaml
dependencies:
  flutter:
    sdk: flutter
  provider: ^6.0.0          # State Management
  sqflite: ^2.3.0           # SQLite база данных
  path: ^1.8.0              # Работа с путями
  flutter_bluetooth_serial: # Bluetooth связь
  permission_handler:       # Разрешения Android/iOS
  pdf: ^3.10.0              # Генерация PDF
  share_plus:               # Шаринг файлов
  intl:                     # Форматирование дат
  path_provider:            # Директории устройства
```

---

## 🛠 Расширение функциональности

### Добавление нового PID

1. Откройте `/workspace/core/obd/pid_metadata.dart`
2. Добавьте запись в `allPidMeta`:

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

3. Добавьте формулу декодирования в `ObdParser._formula()`:

```dart
case '60':
  return ((a * 256) + b) / 10.0;
```

### Добавление нового DTC

1. Откройте `/workspace/core/obd/dtc_hints.dart`
2. Добавьте описание в `kDtcDescriptionsRu`:

```dart
'P9999': 'Описание новой ошибки',
```

3. Добавьте рекомендацию в `kDtcRecommendationsRu`:

```dart
'P9999': 'Рекомендация по устранению',
```

### Добавление нового автомобиля в каталог

1. Откройте `/workspace/data/car_metadata.dart`
2. Найдите нужную марку или добавьте новую:

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

## 📞 Контакты и поддержка

Документация создана для разработчиков и поддерживает актуальность на момент версии 1.0.

Для вопросов и предложений обращайтесь к исходному коду в директории `/workspace`.
