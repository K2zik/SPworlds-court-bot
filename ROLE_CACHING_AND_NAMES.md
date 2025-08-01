# Система кеширования ролей и старых имен игроков

## Обзор

Система реализует два основных компонента:
1. **Кеширование ролей игроков** - обновление раз в день с API `https://tph-api.del1t.me/get_roles`
2. **Сохранение старых имен игроков** - для поиска игроков по старым никнеймам

## 1. Кеширование ролей игроков

### Принцип работы

- **Обновление раз в день**: API ролей обновляется автоматически раз в 24 часа
- **База данных**: Роли сохраняются в таблице `player_roles`
- **Память**: Кеш в памяти с TTL 24 часа
- **Fallback**: При отсутствии в кеше - запрос к API

### Структура базы данных

```sql
CREATE TABLE player_roles (
    uuid TEXT PRIMARY KEY,
    role TEXT,
    last_updated TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### Функции

#### `get_player_role(uuid: str) -> Optional[str]`
Основная функция получения роли игрока:

1. Проверяет кеш в базе данных (не старше 1 дня)
2. Проверяет кеш в памяти
3. Если кеш пуст - обновляет все роли из API
4. Возвращает роль для конкретного UUID

#### `_update_all_roles_from_api()`
Обновляет все роли из API:

1. Загружает данные с `https://tph-api.del1t.me/get_roles`
2. Очищает старые роли из базы данных
3. Сохраняет новые роли в базу данных
4. Обновляет кеш в памяти

#### `_load_cached_roles()`
Загружает кешированные роли при запуске бота:

1. Читает роли из базы данных (не старше 1 дня)
2. Загружает их в кеш памяти
3. Логирует количество загруженных ролей

### Логика обновления

```python
# Проверка необходимости обновления
cached_count = await db.execute('''
    SELECT COUNT(*) FROM player_roles 
    WHERE last_updated > datetime('now', '-1 day')
''')

if cached_count == 0:
    # Обновляем все роли из API
    await self._update_all_roles_from_api()
else:
    # Пытаемся получить отдельную роль
    role = await self._fetch_single_role_from_api(uuid_nodash)
```

## 2. Система старых имен игроков

### Принцип работы

- **Сохранение всех имен**: Каждое имя игрока сохраняется в базе данных
- **Отметка текущего имени**: Поле `is_current` указывает на актуальное имя
- **Поиск по старым именам**: Если игрок не найден по текущему имени, ищется по старым

### Структура базы данных

```sql
CREATE TABLE player_names (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    uuid TEXT,
    username TEXT,
    is_current BOOLEAN DEFAULT 0,
    first_seen TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_seen TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(uuid, username)
);
```

### Функции

#### `_cache_player_name(uuid: str, username: str, is_current: bool = False)`
Кеширует имя игрока:

1. Если `is_current=True` - помечает все старые имена как неактуальные
2. Сохраняет новое имя в базу данных
3. Обновляет `last_seen` для существующих имен

#### `find_player_by_old_name(username: str) -> Optional[Dict]`
Ищет игрока по старому имени:

1. Ищет имя в таблице `player_names`
2. Если найдено - получает актуальную информацию об игроке
3. Обновляет кеш если имя изменилось
4. Возвращает информацию об игроке

#### Обновленная `get_player_info(username: str)`
Теперь поддерживает поиск по старым именам:

```python
# Пытаемся найти по текущим источникам
player_info = await self._get_player_info_impl(username)

if player_info:
    # Кешируем текущее имя
    await self._cache_player_name(player_info['uuid'], username, is_current=True)
else:
    # Пытаемся найти по старому имени
    player_info = await self.find_player_by_old_name(username)
```

### Логика поиска

```python
# Поиск в базе данных
SELECT uuid, username, is_current FROM player_names 
WHERE username = ?
ORDER BY last_seen DESC LIMIT 1

# Если найдено - получаем актуальную информацию
player_info = await self._get_player_info_impl(found_username)

# Обновляем кеш если имя изменилось
if player_info.get('username') != found_username:
    await self._cache_player_name(uuid, player_info['username'], is_current=True)
```

## 3. Индексы для производительности

```sql
-- Для ролей
CREATE INDEX idx_player_roles_last_updated ON player_roles(last_updated);

-- Для имен игроков
CREATE INDEX idx_player_names_uuid ON player_names(uuid);
CREATE INDEX idx_player_names_username ON player_names(username);
CREATE INDEX idx_player_names_is_current ON player_names(is_current);
```

## 4. Примеры использования

### Поиск игрока по старому имени

```python
# Пользователь ищет "OldPlayerName"
player_info = await bot.get_player_info("OldPlayerName")

# Если игрок сменил имя на "NewPlayerName"
# Бот найдет его по старому имени и покажет актуальную информацию
```

### Получение роли игрока

```python
# Получение роли с кешированием
role = await bot.get_player_role("uuid_without_dashes")

# Роль будет загружена из кеша или API
# и сохранена в базе данных на 24 часа
```

## 5. Преимущества системы

### Кеширование ролей
- **Снижение нагрузки на API**: Запросы раз в день вместо каждого обращения
- **Быстрый доступ**: Роли загружаются из памяти
- **Надежность**: Fallback на API при отсутствии в кеше
- **Автоматическое обновление**: Система сама следит за актуальностью

### Старые имена
- **Поиск по истории**: Находит игроков по любым их старым именам
- **Автоматическое обновление**: При получении нового имени обновляет кеш
- **Производительность**: Индексы ускоряют поиск
- **Точность**: Отслеживает какое имя является текущим

## 6. Мониторинг и логирование

### Логи системы
```
INFO: Loaded 1500 cached roles from database
INFO: Updated roles for 2000 players from API
INFO: Found player OldPlayerName by old name: uuid123
WARNING: Failed to fetch roles: 500
ERROR: Error loading cached roles: database error
```

### Метрики
- Количество загруженных ролей при запуске
- Количество обновленных ролей из API
- Успешность поиска по старым именам
- Время выполнения запросов к API

## 7. Конфигурация

### TTL кешей
```python
self.role_cache = TTLCache(maxsize=2000, ttl=86400)  # 24 часа
```

### Параметры API
```python
timeout = aiohttp.ClientTimeout(total=30, connect=10)
connector = aiohttp.TCPConnector(limit=10, limit_per_host=5)
```

### Лимиты базы данных
```sql
-- Максимальное количество имен на игрока
-- Ограничивается уникальностью (uuid, username)
```

Эта система обеспечивает эффективное кеширование ролей и надежный поиск игроков по любым их именам, что значительно улучшает пользовательский опыт. 