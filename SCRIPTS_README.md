# Скрипты для парсинга старых имен игроков

## Обзор

Набор скриптов для анализа файлов судебных дел и заполнения базы данных старыми именами игроков.

## 📁 Файлы

### `analyze_lawsuits_structure.py`
Скрипт для анализа структуры файлов судебных дел.

**Функции:**
- Анализирует структуру `lawsuits.json` и `lawsuitssp.json`
- Находит поля с UUID и именами игроков
- Показывает примеры данных
- Помогает понять формат файлов

**Запуск:**
```bash
python analyze_lawsuits_structure.py
```

**Вывод:**
```
=== НАЙДЕННЫЕ ПОЛЯ ===
Поля с UUID (5):
  plaintiff_uuid
  defendant_uuid
  player_uuid
  uuid

Поля с именами (3):
  plaintiff_name
  defendant_name
  username

=== ПРИМЕРЫ ЗНАЧЕНИЙ ===
Примеры UUID (3):
  2c104d7ab88c48f1a9991f9b6a8495ce
  9a8b7c6d5e4f3a2b1c0d9e8f7a6b5c4d

Примеры имен (10):
  PlayerName1
  PlayerName2
  OldPlayerName
```

### `parse_old_names.py`
Основной скрипт для парсинга и заполнения базы данных.

**Функции:**
- Загружает данные из файлов судебных дел
- Извлекает UUID и имена игроков
- Получает актуальные имена через API
- Сохраняет в базу данных `player_names`

**Запуск:**
```bash
python parse_old_names.py
```

**Процесс:**
1. Инициализация базы данных
2. Загрузка данных из файлов
3. Извлечение UUID и имен
4. Получение актуальных имен через API
5. Сохранение в базу данных
6. Вывод статистики

## 🔧 Использование

### 1. Анализ структуры файлов

Сначала запустите анализ, чтобы понять формат данных:

```bash
python analyze_lawsuits_structure.py
```

Это поможет:
- Понять структуру файлов
- Найти правильные поля для извлечения
- Увидеть примеры данных

### 2. Парсинг и заполнение базы данных

После анализа запустите основной скрипт:

```bash
python parse_old_names.py
```

Скрипт выполнит:
- Создание таблицы `player_names` если её нет
- Загрузку данных из `lawsuits.json` и `lawsuitssp.json`
- Извлечение всех UUID и имен игроков
- Получение актуальных имен через Crafty.gg и Mojang API
- Сохранение всех имен в базу данных
- Вывод подробной статистики

## 📊 Структура данных

### Таблица `player_names`

```sql
CREATE TABLE player_names (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    uuid TEXT,                    -- UUID игрока (без дефисов)
    username TEXT,                -- Имя игрока
    is_current BOOLEAN DEFAULT 0, -- Текущее ли это имя
    first_seen TIMESTAMP,         -- Когда имя впервые встретилось
    last_seen TIMESTAMP,          -- Когда имя последний раз встречалось
    UNIQUE(uuid, username)        -- Уникальность пары UUID-имя
);
```

### Индексы

```sql
CREATE INDEX idx_player_names_uuid ON player_names(uuid);
CREATE INDEX idx_player_names_username ON player_names(username);
CREATE INDEX idx_player_names_is_current ON player_names(is_current);
```

## 🔍 Логика извлечения данных

### Извлечение UUID и имен

Скрипт ищет данные в следующих полях:

**UUID поля:**
- `uuid`
- `player_uuid`
- `plaintiff_uuid`
- `defendant_uuid`

**Поля с именами:**
- `username`
- `player_name`
- `plaintiff_name`
- `defendant_name`

### Обработка данных

1. **Извлечение из файлов:**
   - Читает JSON файлы
   - Ищет поля с UUID и именами
   - Создает пары UUID-имя

2. **Получение актуальных имен и истории:**
   - Пробует Laby API для получения текущего имени и истории
   - Fallback на Mojang API
   - Форматирует UUID для API

3. **Сохранение в БД:**
   - Сохраняет все найденные имена (из файлов + история)
   - Помечает актуальное имя как `is_current = 1`
   - Обновляет `last_seen` для существующих имен

## 📈 Статистика

После выполнения скрипт покажет:

```
=== СТАТИСТИКА ===
Всего записей: 1500
Уникальных UUID: 800
Уникальных имен: 1200
Текущих имен: 800

Топ игроков по количеству имен:
  UUID 2c104d7ab88c48f1a9991f9b6a8495ce: 5 имен
  UUID 9a8b7c6d5e4f3a2b1c0d9e8f7a6b5c4d: 3 имени
```

## ⚙️ Настройка

### Константы в `parse_old_names.py`

```python
DB_PATH = 'court_bot.db'                    # Путь к базе данных
LAWSUITS_FILES = ['lawsuits.json', 'lawsuitssp.json']  # Файлы для обработки
```

### Параметры API

```python
timeout = aiohttp.ClientTimeout(total=30, connect=10)  # Таймаут для API
connector = aiohttp.TCPConnector(limit=10, limit_per_host=5)  # Лимиты соединений
```

### Задержка между запросами

```python
await asyncio.sleep(0.1)  # Задержка 100ms между запросами к API
```

## 🐛 Обработка ошибок

### Логирование

Скрипты создают логи:
- `parse_old_names.log` - основной скрипт
- Консольный вывод для анализа структуры

### Типы ошибок

1. **Файл не найден:**
   ```
   WARNING: Файл lawsuits.json не найден, пропускаем
   ```

2. **Ошибка API:**
   ```
   DEBUG: Ошибка Crafty.gg для UUID xxx: timeout
   ```

3. **Ошибка БД:**
   ```
   ERROR: Ошибка сохранения имен для UUID xxx: UNIQUE constraint failed
   ```

## 🚀 Примеры использования

### Полный процесс

```bash
# 1. Анализируем структуру
python analyze_lawsuits_structure.py

# 2. Парсим и заполняем БД
python parse_old_names.py

# 3. Проверяем результат
sqlite3 court_bot.db "SELECT COUNT(*) FROM player_names;"
```

### Проверка результатов

```sql
-- Общая статистика
SELECT COUNT(*) as total_records,
       COUNT(DISTINCT uuid) as unique_uuids,
       COUNT(DISTINCT username) as unique_names,
       COUNT(*) FILTER (WHERE is_current = 1) as current_names
FROM player_names;

-- Топ игроков по количеству имен
SELECT uuid, COUNT(*) as name_count
FROM player_names
GROUP BY uuid
ORDER BY name_count DESC
LIMIT 10;

-- Поиск конкретного игрока
SELECT * FROM player_names WHERE username LIKE '%PlayerName%';
```

## 📝 Примечания

1. **Производительность:** Скрипт делает запросы к API с задержкой, чтобы не перегружать серверы
2. **Дублирование:** Используется `INSERT OR IGNORE` для избежания дублирования
3. **Текущие имена:** Только одно имя на UUID помечается как `is_current = 1`
4. **Логирование:** Подробные логи помогают отладить проблемы
5. **Fallback:** Если API недоступен, используется первое найденное имя

Эти скрипты обеспечивают полное заполнение базы данных историческими именами игроков для эффективного поиска по старым никнеймам. 