# План: Система плагинов для open-blockchain

## Архитектура

```
src/python/
├── main.py                        # CLI-админка (не меняется)
├── server.py                      # FastAPI сервер (добавить хуки)
├── libs/
│   └── clear.py
├── plugins/
│   ├── __init__.py
│   ├── base.py                    # BasePlugin — абстрактный класс
│   ├── loader.py                  # Загрузчик: сканирование, импорт, валидация
│   └── registry.py                # PluginRegistry — синглтон, реестр активных плагинов
└── plugins_available/            # Сюда класть плагины (по папкам)
    └── example/
        ├── __init__.py
        └── plugin.json
```

## 1. BasePlugin — базовый класс для всех плагинов

Файл: `src/python/plugins/base.py`

Методы (хуки), которые может переопределить плагин:

- `on_node_start(app: FastAPI)` — вызывается при старте сервера
- `on_node_stop()` — при остановке
- `register_routes(app: FastAPI)` — добавить свои эндпоинты
- `on_block_mined(block: dict)` — после майнинга блока
- `register_admin_commands() -> list[AdminCommand]` — команды для админ-панели
- `validate_block_data(data) -> (bool, str)` — валидация данных блока (расширяемый)
- `before_mine(data) -> data` — модификация данных перед майнингом
- `after_mine(block) -> block` — модификация блока после майнинга
- `custom_difficulty(current: int) -> int | None` — кастомная сложность

**Доки:**
- `abc.ABCMeta` — https://docs.python.org/3/library/abc.html
- `importlib` — https://docs.python.org/3/library/importlib.html

## 2. PluginRegistry — реестр плагинов

Файл: `src/python/plugins/registry.py`

Синглтон, хранит список загруженных плагинов. Методы:

- `register(plugin: BasePlugin)` — добавить плагин
- `get_plugins() -> list[BasePlugin]` — получить все
- `call_hook(hook_name: str, **kwargs)` — вызвать хук у всех плагинов
- `get_admin_commands() -> list[tuple[str, str, str]]` — (plugin_name, cmd_id, title)
- `exec_admin_command(plugin_name: str, cmd_id: str, args: dict)` — выполнить команду

**Доки:**
- Паттерн Singleton — https://refactoring.guru/ru/design-patterns/singleton/python/example

## 3. Plugin Loader — загрузчик

Файл: `src/python/plugins/loader.py`

- Сканирует `plugins_available/` через pathlib
- Читает `plugin.json` (имя, версия, описание, зависимости)
- Импортирует модуль через `importlib.import_module(path)`
- Ищет класс, наследующий `BasePlugin`
- Инстанцирует и регистрирует в PluginRegistry
- Обрабатывает ошибки (битый плагин — пропустить с логом)

**Доки:**
- pathlib — https://docs.python.org/3/library/pathlib.html
- importlib — https://docs.python.org/3/library/importlib.html#module-importlib
- inspect.getmembers — https://docs.python.org/3/library/inspect.html#inspect.getmembers
- json — https://docs.python.org/3/library/json.html

## 4. Хуки в server.py

Что добавить в `src/python/server.py`:

### 4.1 При старте сервера
```python
from plugins.loader import load_plugins
from plugins.registry import registry

load_plugins()  # загрузить все плагины
registry.call_hook("on_node_start", app=server_app)
```

### 4.2 В эндпоинт /mine
Перед майнингом:
- `validate_block_data(data)` — все плагины
- `before_mine(data)` — модификация данных

После майнинга:
- `after_mine(block)` — модификация блока
- `on_block_mined(block)` — уведомление плагинов

### 4.3 Новые эндпоинты (отдавать команды в админку)
- `GET /plugins/list` — список плагинов
- `GET /plugins/commands` — список команд для CLI
- `POST /plugins/exec` — выполнить команду плагина

**Доки:**
- FastAPI middleware/hooks — https://fastapi.tiangolo.com/tutorial/middleware/
- FastAPI routes — https://fastapi.tiangolo.com/tutorial/path-params/

## 5. Интеграция с админкой (main.py)

В `src/python/main.py` изменения минимальны:

```python
elif command == "5":  # новый пункт меню
    # GET /plugins/commands
    # отобразить как пункты меню
    # POST /plugins/exec с выбранной командой
```

Или динамически: запрашивать у сервера список команд и добавлять их в меню.

**Доки:**
- urllib.request — https://docs.python.org/3/library/urllib.request.html

## 6. Пример плагина

Файл: `src/python/plugins_available/example/__init__.py`

```python
from plugins.base import BasePlugin, AdminCommand
from fastapi import FastAPI

class ExamplePlugin(BasePlugin):
    name = "example"
    version = "1.0.0"

    def register_routes(self, app: FastAPI):
        @app.get("/plugin/example/hello")
        async def hello():
            return {"message": "Hello from plugin!"}

    def register_admin_commands(self):
        return [AdminCommand("hello", "Сказать привет")]

    def on_block_mined(self, block):
        print(f"[Example] Блок #{block['index']} добыт!")
```

## Этапы реализации

1. **Базовый скелет** — `plugins/__init__.py`, `base.py`, `loader.py`, `registry.py`
2. **Загрузчик** — сканирование папок, импорт, валидация
3. **Хуки в server.py** — интеграция хуков в `/mine` и `run_fastapi_server`
4. **API для админки** — эндпоинты `/plugins/list`, `/commands`, `/exec`
5. **Интеграция с main.py** — новый пункт меню для плагинов
6. **Пример плагина** — `plugins_available/example/`
7. **Тестирование** — на Windows, Linux, macOS

## Важно

- Весь код использует **только стандартную библиотеку Python** + FastAPI/Uvicorn
- Кроссплатформенность: pathlib, os.path.join, importlib — работают везде
- Плагины не могут сломать ядро — ошибки в `validate_block_data` не валидируют блок, ошибки в `on_block_mined` логируются
- Безопасность: плагины — это Python-код, он имеет полный доступ к серверу. В будущем можно добавить:
  - Разрешения в `plugin.json` (какие хуки/роуты разрешены)
  - Запуск плагинов в изолированном subprocess (но это ломает кроссплатформенность упрощением)
