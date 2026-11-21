# Заказики и MAX 📝

> Десктопное приложение для управления заметками, событиями-напоминаниями и заказами с глубокой интеграцией мессенджера MAX Messenger.

[![Python](https://img.shields.io/badge/Python-3.10%2B-blue.svg)](https://www.python.org/)
[![Windows](https://img.shields.io/badge/Platform-Windows%2010%2B-green.svg)](https://www.microsoft.com/windows)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

## 🌟 Обзор

**Заказики** — это нативное десктопное приложение для Windows, которое объединяет в себе три ключевых инструмента:

- **📝 Менеджер заметок** — создание, редактирование и категоризация текстовых заметок с поддержкой вложений
- **📅 Календарь событий** — система напоминаний с уведомлениями из системного трея
- **📥 Бот-интегратор MAX** — автоприём сообщений, фото, документов и файлов из мессенджера MAX Messenger

Приложение работает в фоновом режиме как иконка в системном трее, сворачиваясь вместо закрытия, и предоставляет удобный графический интерфейс для управления всеми данными.

## 🚀 Быстрый старт

### Требования

- **Python 3.10+** (или готовый `.exe`)
- **Windows 10/11** (для DPAPI и native API)
- **Аккаунт MAX Messenger** с доступом к бот-API

### Установка

```bash
# Клонируйте репозиторий
git clone https://github.com/username/my_note.git
cd my_note

# Установите зависимости
pip install -r requirements.txt

# Запустите приложение
python main.py
```

### Готовый `.exe`

Собранное приложение находится в папке `dist/Заказики.exe`. Для запуска не требуется Python.

## ✨ Возможности

### 📅 События и напоминания

| Возможность | Описание |
|---|---|
| Создание событий | Название, описание, точная дата и время напоминания |
| Авто-сортировка | События автоматически упорядочиваются по дате |
| Предупреждения | Системное уведомление за 60 секунд до события |
| Просроченные события | Визуальное выделение просроченных записей красным |
| Карточный UI | Hover-эффекты, кнопки редактирования и удаления |

### 📥 Интеграция с MAX Messenger

| Возможность | Описание |
|---|---|
| Автоприём сообщений | Long polling API MAX каждые 30 секунд |
| Текстовые заметки | Автосохранение текста в БД с карточками |
| Вложения | Фото, видео, документы, стикеры, GIF |
| Авто-типизация | Magic-bytes определение типа файла |
| Лимит 50 МБ | Защита от переполнения диска |
| Whitelist chat_id | Фильтрация сообщений от неавторизованных чатов |

### 🔒 Безопасность

| Компонент | Механизм |
|---|---|
| **Токен бота** | Windows DPAPI (`CryptProtectData`) — привязка к пользователю |
| **База данных** | SQLCipher AES-256 (опционально, fallback на SQLite) |
| **Вложения** | Path traversal защита через `os.path.realpath` + `commonpath` |
| **API запросы** | Rate-limit 30 req/s, CA-сертификат, timeout |
| **Singleton** | Global Mutex предотвращает двойной запуск |

## 🏗 Архитектура

```
Заказики/
├── main.py                   # Точка входа, mutex, шифрование, автозапуск
├── config.py                 # Константы, пути, цвета, шрифты
├── config.json               # Конфигурация API, бота, health-check
│
├── api_client.py             # Общий клиент: CA bundle, session, rate-limit
├── bot_receiver.py           # Бот: long polling, скачивание, запись в БД
├── test_direct.py            # Тест: проверка токена и подключения к API
│
├── database.py               # Абстракция _db_module (sqlite3 / sqlcipher)
├── crypto_key.py             # Управление ключом шифрования БД (DPAPI)
├── crypto_utils.py           # Шифрование/расшифровка токена (DPAPI)
│
├── ui/
│   ├── main_window.py        # Главное окно, вкладки, иконка в трее
│   ├── events_tab.py         # Вкладка "События" — CRUD + уведомления
│   ├── max_tab.py            # Вкладка "Из MAX" — карточки заметок
│   ├── orders_tab.py         # Вкладка "Заказы" — управление заказами
│   ├── order_dialog.py       # Диалог создания/редактирования заказа
│   ├── settings_dialog.py    # Настройки приложения
│   ├── bot_settings_dialog.py# Настройка токена бота (GUI)
│   └── file_watcher.py       # Мониторинг папки inbox (ReadDirectoryChangesW)
│
├── combined_ca.pem           # CA-сертификат MAX API
├── my_note.spec              # Конфигурация PyInstaller
└── requirements.txt          # Python-зависимости
```

## 📊 База данных

### Режимы

| Режим | Движок | Шифрование | Когда использовать |
|---|---|---|---|
| **Dev** | `sqlite3` | Нет | Разработка, тестирование |
| **Prod** | `sqlcipher` | AES-256 | Продуктивное использование |

### Автоматическая миграция

При первом запуске с `pysqlcipher3` приложение автоматически:

1. Создаёт резервную копию `notes.db.plaintext.backup`
2. Генерирует ключ шифрования (DPAPI)
3. Копирует данные из незашифрованной БД
4. Удалённо удаляет оригинал

### Схемы таблиц

```sql
-- Заметки и события
notes (
    id INTEGER PRIMARY KEY,
    title TEXT,
    description TEXT,
    created_at TEXT,
    remind_at TEXT,
    status TEXT DEFAULT 'active'
)

-- Вложения
attachments (
    id INTEGER PRIMARY KEY,
    note_id INTEGER REFERENCES notes(id) ON DELETE CASCADE,
    file_path TEXT,
    file_type TEXT,
    original_name TEXT
)

-- Заказы
orders (
    id INTEGER PRIMARY KEY,
    customer_name TEXT,
    deadline TEXT,
    file_path TEXT,
    weight_grams REAL,
    print_time_hours REAL,
    material_type TEXT,
    extra_services REAL,
    total_price REAL,
    note_id INTEGER,
    filament_id INTEGER,
    printer_id INTEGER,
    status TEXT DEFAULT 'Новый'
)

-- Принтеры (оборудование)
printers (
    id INTEGER PRIMARY KEY,
    name TEXT UNIQUE
)

-- Пластик
filaments (
    id INTEGER PRIMARY KEY,
    printer_id INTEGER REFERENCES printers(id),
    material_type TEXT,
    price_per_gram REAL
)
```

## ⚙️ Конфигурация

### config.json

```json
{
  "api": {
    "base_url": "https://platform-api2.max.ru",
    "rate_limit_rps": 30,
    "long_poll_timeout": 30,
    "max_message_length": 4000
  },
  "database": {
    "journal_mode": "WAL",
    "busy_timeout_ms": 10000
  },
  "bot": {
    "allowed_chat_ids": []
  },
  "health_check": {
    "enabled": false,
    "host": "127.0.0.1",
    "port": 8080
  }
}
```

**`allowed_chat_ids: []`** — пустой список = принимать сообщения от всех. Укажите `[123456789]` для whitelist.

### Переменные окружения (.env)

```env
MAX_BOT_TOKEN=your_bot_token_here
MAX_ROOT_CA_PATH=combined_ca.pem
LOG_LEVEL=INFO
```

## 🛠 Разработка

### Локальный запуск

```bash
# Запуск основного приложения
python main.py

# Запуск бота отдельно (терминал)
python bot_receiver.py

# Тест токена
python test_direct.py

# Настройка токена из терминала (fallback)
python bot_receiver.py --configure-token
```

### Health Check

```bash
# Включить в config.json:
"health_check": { "enabled": true, "port": 8080 }

# Проверить:
curl http://127.0.0.1:8080/health
```

### Сборка в .exe

```bash
pip install pyinstaller
pyinstaller my_note.spec
# Результат: dist/Заказики.exe
```

### Логирование

```
2026-11-10 12:00:00 [INFO] max-bot-receiver: Бот запущен.
2026-11-10 12:00:01 [INFO] database: Принтер: Elegoo Centauri Carbon
2026-11-10 12:00:02 [WARNING] bot_receiver: Превышен лимит запросов.
```

## 🎯 Ключевые паттерны

### Абстракция _db_module

```python
_db_module = _sql_cipher_module if _HAS_SQLCIPHER else sqlite3
# Все вызовы: _db_module.connect(...)
# Автоматическое переключение без изменений кода
```

### Безопасное скачивание файлов

```python
# 1. resolve .. -> realpath
full_path = os.path.realpath(os.path.join(APP_DIR, save_path))
# 2. Валидация: путь внутри INBOX_DIR
if commonpath((inbox_path, full_path)) != inbox_path:
    raise ValueError("Path traversal detected")
# 3. Magic-bytes определение расширения
ext = _detect_extension_by_content(file_path)
# 4. Лимит 50 МБ
if content_length > MAX_FILE_SIZE:
    return False
```

### Rate Limit (token bucket)

```python
_last_api_request = 0.0
MAX_API_MIN_INTERVAL = 1 / 30  # ≤30 req/s

def _wait_for_api_slot():
    delay = MAX_API_MIN_INTERVAL - (time.monotonic() - _last_api_request)
    if delay > 0:
        time.sleep(delay)
    _last_api_request = time.monotonic()
```

## 📦 Зависимости

| Пакет | Назначение |
|---|---|
| `customtkinter` | Современный GUI с темой |
| `requests` | HTTP-запросы к MAX API |
| `pillow` | Обработка изображений |
| `cryptography` | DPAPI-обёртки |
| `pystray` | Иконка в системном трее |
| `python-dotenv` | Загрузка .env переменных |
| `pysqlcipher3` | Шифрование БД AES-256 (опционально) |

## 📝 Структура проекта

```
Заказики/
├── main.py                  # Точка входа, mutex, инициализация
├── config.py                # Константы и пути
├── config.json              # runtime-конфигурация
├── api_client.py            # HTTP-клиент, CA bundle, rate-limit
├── bot_receiver.py          # Бот: long polling, вложения
├── test_direct.py           # Тест токена и API
├── database.py              # _db_module, миграции, CRUD
├── crypto_key.py            # Ключ шифрования БД
├── crypto_utils.py          # DPAPI токена
├── pricing.py               # Калькулятор стоимости
├── requirements.txt         # Python-зависимости
├── combined_ca.pem          # CA сертификат MAX
├── my_note.spec             # PyInstaller конфиг
└── ui/
    ├── main_window.py       # Главное окно
    ├── events_tab.py        # События и напоминания
    ├── max_tab.py           # Заметки из MAX
    ├── orders_tab.py        # Заказы
    ├── order_dialog.py      # Диалог заказа
    ├── settings_dialog.py   # Настройки
    ├── bot_settings_dialog.py # Токен бота
    └── file_watcher.py      # Мониторинг папки
```

## 🐛 Известные ограничения

| Ограничение | Причина | Обход |
|---|---|---|
| Foreign Key в миграциях | SQLite не поддерживает `ALTER TABLE ADD CONSTRAINT` | Связь работает только для `CREATE TABLE` |
| DPAPI только Windows | Криптографическая привязка к профилю | Для Linux — `cryptography.fernet` |
| pysqlcipher3 требует C++ Build Tools | C-расширения | Без шифрования — fallback на `sqlite3` |

## 📄 License

[MIT License](LICENSE) — Copyright (c) 2026 Ilya

## 🤝 Вклад

Проекты для портфолио приветствуются! Если нашли баг или есть идея — создавайте issue или PR.

---

*Разработано с ❤️ для интеграции с MAX Messenger*
