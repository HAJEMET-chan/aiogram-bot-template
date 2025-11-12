# Практическое руководство по использованию шаблона

## Содержание

1. [Быстрый старт](#быстрый-старт)
2. [Примеры обработчиков](#примеры-обработчиков)
3. [Работа с БД](#работа-с-бд)
4. [FSM и диалоги](#fsm-и-диалоги)
5. [Клавиатуры](#клавиатуры)
6. [Обработка ошибок](#обработка-ошибок)
7. [Тестирование](#тестирование)
8. [Деплой](#деплой)

## Быстрый старт

### 1. Установка зависимостей

```bash
# Установка Poetry (если еще не установлен)
curl -sSL https://install.python-poetry.org | python3 -

# Установка зависимостей проекта
poetry install
```

### 2. Настройка окружения

```bash
# Копирование примера конфигурации
cp example.env .env

# Редактирование .env
nano .env  # или ваш любимый редактор
```

Минимальная конфигурация для начала:

```env
BOT_TOKEN=123456789:ABCdefGHIjklMNOpqrsTUVwxyz
PG_HOST=localhost
PG_PORT=5432
PG_USER=postgres
PG_PASSWORD=your_password
PG_DATABASE=bot_db
FSM_HOST=127.0.0.1
FSM_PORT=6379
FSM_PASSWORD=
USE_WEBHOOK=false
LOGGING_LEVEL=10
```

### 3. Настройка БД

```sql
-- Создание базы данных
CREATE DATABASE bot_db;

-- Пример таблицы пользователей
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    telegram_id BIGINT UNIQUE NOT NULL,
    username VARCHAR(255),
    full_name VARCHAR(255),
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Индекс для быстрого поиска
CREATE INDEX idx_users_telegram_id ON users(telegram_id);
```

### 4. Запуск бота

```bash
# Активация виртуального окружения
poetry shell

# Запуск бота
python -m bowling_bot.bot
```

## Примеры обработчиков

### Простой обработчик команды

```python
# handlers/user/help.py
from aiogram import html, types
from aiogram.fsm.context import FSMContext

async def help_command(msg: types.Message, state: FSMContext) -> None:
    """Обработчик команды /help"""
    help_text = """
    <b>Доступные команды:</b>
    
    /start - Начать работу с ботом
    /help - Показать эту справку
    /profile - Показать профиль
    """
    await msg.answer(help_text, parse_mode="HTML")
```

Регистрация:

```python
# handlers/user/__init__.py
from aiogram import Router
from aiogram.filters import Command
from bowling_bot.filters import ChatTypeFilter
from . import help as help_module

def prepare_router() -> Router:
    user_router = Router()
    user_router.message.filter(ChatTypeFilter("private"))
    
    user_router.message.register(help_module.help_command, Command("help"))
    
    return user_router
```

### Обработчик с работой БД

```python
# handlers/user/profile.py
from aiogram import html, types
from aiogram.fsm.context import FSMContext
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from bowling_bot.db.db_api.storages.postgres import PostgresConnection

async def show_profile(
    msg: types.Message,
    state: FSMContext,
    db_conn: "PostgresConnection",н
) -> None:
    """Показать профиль пользователя"""
    if msg.from_user is None:
        return
    
    # Получение данных из БД
    result = await db_conn._fetchrow(
        """
        SELECT username, full_name, created_at 
        FROM users 
        WHERE telegram_id = $1
        """,
        (msg.from_user.id,),
    )
    
    if result.data is None:
        await msg.answer("Профиль не найден. Используйте /start для регистрации.")
        return
    
    user_data = result.data
    profile_text = f"""
    <b>Ваш профиль:</b>
    
    Имя: {html.quote(user_data['full_name'] or 'Не указано')}
    Username: @{html.quote(user_data['username'] or 'не указан')}
    Дата регистрации: {user_data['created_at']}
    """
    
    await msg.answer(profile_text, parse_mode="HTML")
```

### Обработчик callback_query

```python
# handlers/user/callbacks.py
from aiogram import types
from bowling_bot.keyboards.inline.callbacks import Action

async def handle_action(
    callback: types.CallbackQuery,
    callback_data: Action,
) -> None:
    """Обработка inline кнопок"""
    if callback_data.action == "show_info":
        await callback.answer("Информация загружается...")
        await callback.message.edit_text(
            "Это информация, которую вы запросили!",
            reply_markup=None  # Убрать клавиатуру
        )
    elif callback_data.action == "delete":
        await callback.answer("Удалено!")
        await callback.message.delete()
```

Регистрация:

```python
# handlers/user/__init__.py
from aiogram import Router
from aiogram.filters.callback_data import CallbackQuery
from bowling_bot.keyboards.inline.callbacks import Action
from . import callbacks

def prepare_router() -> Router:
    user_router = Router()
    
    user_router.callback_query.register(
        callbacks.handle_action,
        Action.filter()
    )
    
    return user_router
```

### Обработчик медиа

```python
# handlers/user/media.py
from aiogram import types
from aiogram.fsm.context import FSMContext

async def handle_photo(msg: types.Message, state: FSMContext) -> None:
    """Обработка фотографий"""
    if msg.photo is None:
        return
    
    # Получение фото наибольшего размера
    photo = msg.photo[-1]
    
    # Скачивание файла
    file = await msg.bot.get_file(photo.file_id)
    
    await msg.answer(
        f"Получено фото!\n"
        f"Размер: {photo.width}x{photo.height}\n"
        f"File ID: {photo.file_id}"
    )
```

## Работа с БД

### Создание репозитория

```python
# db/repositories/user_repository.py
from typing import TYPE_CHECKING
from datetime import datetime

if TYPE_CHECKING:
    from bowling_bot.db.db_api.storages.postgres import PostgresConnection

class UserRepository:
    def __init__(self, db_conn: "PostgresConnection") -> None:
        self.db_conn = db_conn
    
    async def get_by_telegram_id(self, telegram_id: int) -> dict | None:
        """Получить пользователя по Telegram ID"""
        result = await self.db_conn._fetchrow(
            "SELECT * FROM users WHERE telegram_id = $1",
            (telegram_id,),
        )
        return result.data
    
    async def create(
        self,
        telegram_id: int,
        username: str | None,
        full_name: str,
    ) -> dict:
        """Создать нового пользователя"""
        await self.db_conn._execute(
            """
            INSERT INTO users (telegram_id, username, full_name, created_at)
            VALUES ($1, $2, $3, NOW())
            ON CONFLICT (telegram_id) DO UPDATE
            SET username = EXCLUDED.username,
                full_name = EXCLUDED.full_name,
                updated_at = NOW()
            RETURNING *
            """,
            (telegram_id, username, full_name),
        )
        return await self.get_by_telegram_id(telegram_id)
    
    async def update_last_activity(self, telegram_id: int) -> None:
        """Обновить время последней активности"""
        await self.db_conn._execute(
            "UPDATE users SET updated_at = NOW() WHERE telegram_id = $1",
            (telegram_id,),
        )
```

### Использование репозитория

```python
# handlers/user/start.py
from aiogram import types
from aiogram.fsm.context import FSMContext
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from bowling_bot.db.db_api.storages.postgres import PostgresConnection
    from bowling_bot.db.repositories.user_repository import UserRepository

async def start(
    msg: types.Message,
    state: FSMContext,
    db_conn: "PostgresConnection",
) -> None:
    """Обработчик команды /start"""
    if msg.from_user is None:
        return
    
    # Создание репозитория
    user_repo = UserRepository(db_conn)
    
    # Создание или обновление пользователя
    user = await user_repo.create(
        telegram_id=msg.from_user.id,
        username=msg.from_user.username,
        full_name=msg.from_user.full_name or "Не указано",
    )
    
    await msg.answer(f"Добро пожаловать, {msg.from_user.full_name}!")
```

### Транзакции

```python
# Пример использования транзакций
async def transfer_points(
    from_user_id: int,
    to_user_id: int,
    points: int,
    db_conn: "PostgresConnection",
) -> None:
    """Перевод очков между пользователями"""
    async with db_conn._pool.acquire() as conn:
        async with conn.transaction():
            # Проверка баланса
            from_balance = await conn.fetchval(
                "SELECT balance FROM users WHERE telegram_id = $1",
                from_user_id,
            )
            
            if from_balance < points:
                raise ValueError("Недостаточно очков")
            
            # Списание
            await conn.execute(
                "UPDATE users SET balance = balance - $1 WHERE telegram_id = $2",
                points,
                from_user_id,
            )
            
            # Начисление
            await conn.execute(
                "UPDATE users SET balance = balance + $1 WHERE telegram_id = $2",
                points,
                to_user_id,
            )
```

## FSM и диалоги

### Многошаговый диалог

```python
# states/user.py
from aiogram.fsm.state import State, StatesGroup

class RegistrationStates(StatesGroup):
    waiting_for_name = State()
    waiting_for_age = State()
    waiting_for_confirmation = State()
```

```python
# handlers/user/registration.py
from aiogram import types
from aiogram.fsm.context import FSMContext
from bowling_bot import states
from bowling_bot.keyboards.default import BasicButtons

async def start_registration(msg: types.Message, state: FSMContext) -> None:
    """Начать регистрацию"""
    await state.set_state(states.user.RegistrationStates.waiting_for_name)
    await msg.answer(
        "Давайте зарегистрируем вас!\nВведите ваше имя:",
        reply_markup=BasicButtons.cancel(),
    )

async def process_name(msg: types.Message, state: FSMContext) -> None:
    """Обработать имя"""
    if msg.text is None:
        await msg.answer("Пожалуйста, отправьте текстовое сообщение.")
        return
    
    await state.update_data(name=msg.text)
    await state.set_state(states.user.RegistrationStates.waiting_for_age)
    await msg.answer("Отлично! Теперь введите ваш возраст:")

async def process_age(msg: types.Message, state: FSMContext) -> None:
    """Обработать возраст"""
    if msg.text is None:
        await msg.answer("Пожалуйста, отправьте текстовое сообщение.")
        return
    
    try:
        age = int(msg.text)
        if age < 0 or age > 150:
            raise ValueError
    except ValueError:
        await msg.answer("Пожалуйста, введите корректный возраст (число от 0 до 150).")
        return
    
    await state.update_data(age=age)
    data = await state.get_data()
    
    await state.set_state(states.user.RegistrationStates.waiting_for_confirmation)
    await msg.answer(
        f"Проверьте данные:\n"
        f"Имя: {data['name']}\n"
        f"Возраст: {data['age']}",
        reply_markup=BasicButtons.confirmation(add_cancel=True),
    )

async def confirm_registration(msg: types.Message, state: FSMContext) -> None:
    """Подтвердить регистрацию"""
    data = await state.get_data()
    
    # Сохранение в БД
    # ... ваш код ...
    
    await state.clear()
    await msg.answer(
        "Регистрация завершена!",
        reply_markup=types.ReplyKeyboardRemove(),
    )

async def cancel_registration(msg: types.Message, state: FSMContext) -> None:
    """Отменить регистрацию"""
    await state.clear()
    await msg.answer(
        "Регистрация отменена.",
        reply_markup=types.ReplyKeyboardRemove(),
    )
```

Регистрация:

```python
# handlers/user/__init__.py
from aiogram import Router
from aiogram.filters import StateFilter
from bowling_bot import states
from bowling_bot.filters import ChatTypeFilter, TextFilter
from . import registration

def prepare_router() -> Router:
    user_router = Router()
    user_router.message.filter(ChatTypeFilter("private"))
    
    # Начало регистрации
    user_router.message.register(
        registration.start_registration,
        TextFilter("📝 Регистрация"),
    )
    
    # Шаги регистрации
    user_router.message.register(
        registration.process_name,
        StateFilter(states.user.RegistrationStates.waiting_for_name),
    )
    user_router.message.register(
        registration.process_age,
        StateFilter(states.user.RegistrationStates.waiting_for_age),
    )
    user_router.message.register(
        registration.confirm_registration,
        TextFilter("✅Подтвердить"),
        StateFilter(states.user.RegistrationStates.waiting_for_confirmation),
    )
    user_router.message.register(
        registration.cancel_registration,
        TextFilter("🚫 Отмена"),
    )
    
    return user_router
```

## Клавиатуры

### Создание reply-клавиатуры

```python
# keyboards/default/menu.py
from aiogram.types import ReplyKeyboardMarkup
from .consts import DefaultConstructor

class MenuButtons(DefaultConstructor):
    @staticmethod
    def main_menu() -> ReplyKeyboardMarkup:
        schema = [2, 2, 1]
        btns = [
            "📊 Статистика",
            "⚙️ Настройки",
            "📝 Регистрация",
            "👤 Профиль",
            "❓ Помощь",
        ]
        return MenuButtons._create_kb(btns, schema)
    
    @staticmethod
    def settings_menu() -> ReplyKeyboardMarkup:
        schema = [1, 1, 1]
        btns = [
            "🔔 Уведомления",
            "🌐 Язык",
            "◀️Назад",
        ]
        return MenuButtons._create_kb(btns, schema)
```

Использование:

```python
from bowling_bot.keyboards.default.menu import MenuButtons

await msg.answer(
    "Главное меню:",
    reply_markup=MenuButtons.main_menu(),
)
```

### Создание inline-клавиатуры

```python
# keyboards/inline/callbacks.py
from aiogram.filters.callback_data import CallbackData

class MenuAction(CallbackData, prefix="menu"):
    action: str
    item_id: int | None = None

class SettingsAction(CallbackData, prefix="settings"):
    action: str
    setting: str | None = None
```

```python
# keyboards/inline/user/menu.py
from aiogram.types import InlineKeyboardMarkup
from ..consts import InlineConstructor
from ..callbacks import MenuAction

class UserMenu(InlineConstructor):
    @staticmethod
    def main_menu() -> InlineKeyboardMarkup:
        schema = [1, 1, 1]
        btns = [
            {"text": "📊 Статистика", "callback_data": MenuAction(action="stats")},
            {"text": "⚙️ Настройки", "callback_data": MenuAction(action="settings")},
            {"text": "❓ Помощь", "callback_data": MenuAction(action="help")},
        ]
        return UserMenu._create_kb(btns, schema)
    
    @staticmethod
    def paginated_list(items: list[dict], page: int, per_page: int = 5) -> InlineKeyboardMarkup:
        """Клавиатура с пагинацией"""
        start = (page - 1) * per_page
        end = start + per_page
        page_items = items[start:end]
        
        schema = []
        btns = []
        
        # Кнопки элементов
        for item in page_items:
            schema.append(1)
            btns.append({
                "text": item["name"],
                "callback_data": MenuAction(action="select", item_id=item["id"]),
            })
        
        # Кнопки навигации
        nav_btns = []
        if page > 1:
            nav_btns.append({
                "text": "◀️ Назад",
                "callback_data": MenuAction(action="page", item_id=page - 1),
            })
        if end < len(items):
            nav_btns.append({
                "text": "Вперед ▶️",
                "callback_data": MenuAction(action="page", item_id=page + 1),
            })
        
        if nav_btns:
            schema.append(len(nav_btns))
            btns.extend(nav_btns)
        
        return UserMenu._create_kb(btns, schema)
```

Использование:

```python
from bowling_bot.keyboards.inline.user.menu import UserMenu

await msg.answer(
    "Выберите действие:",
    reply_markup=UserMenu.main_menu(),
)
```

## Обработка ошибок

### Базовый обработчик ошибок

```python
# handlers/errors.py
from aiogram import Dispatcher, types
from aiogram.exceptions import TelegramBadRequest, TelegramAPIError
import structlog

logger = structlog.get_logger()

async def error_handler(
    update: types.Update,
    exception: Exception,
) -> None:
    """Глобальный обработчик ошибок"""
    if isinstance(exception, TelegramBadRequest):
        logger.warning(
            "Telegram Bad Request",
            error=str(exception),
            update_id=update.update_id if update else None,
        )
    elif isinstance(exception, TelegramAPIError):
        logger.error(
            "Telegram API Error",
            error=str(exception),
            update_id=update.update_id if update else None,
        )
    else:
        logger.exception(
            "Unhandled exception",
            error=str(exception),
            update_id=update.update_id if update else None,
        )
    
    # Отправка сообщения пользователю (опционально)
    if update and update.message:
        try:
            await update.message.answer(
                "Произошла ошибка. Пожалуйста, попробуйте позже."
            )
        except Exception:
            pass  # Игнорируем ошибки при отправке сообщения об ошибке
```

Регистрация:

```python
# bot.py
from bowling_bot.handlers import errors

def setup_error_handlers(dp: Dispatcher) -> None:
    dp.errors.register(errors.error_handler)
```

### Обработка ошибок БД

```python
# utils/db_errors.py
from asyncpg import PostgresError, UniqueViolationError
import structlog

logger = structlog.get_logger()

async def handle_db_error(error: Exception) -> str:
    """Обработка ошибок БД с возвратом понятного сообщения"""
    if isinstance(error, UniqueViolationError):
        return "Эта запись уже существует."
    elif isinstance(error, PostgresError):
        logger.error("Database error", error=str(error))
        return "Ошибка базы данных. Попробуйте позже."
    else:
        logger.exception("Unexpected database error", error=str(error))
        return "Произошла непредвиденная ошибка."
```

Использование:

```python
try:
    await user_repo.create(telegram_id, username, full_name)
except Exception as e:
    error_msg = await handle_db_error(e)
    await msg.answer(error_msg)
```

## Тестирование

### Пример теста обработчика

```python
# tests/test_handlers.py
import pytest
from aiogram import types
from aiogram.fsm.context import FSMContext
from aiogram.fsm.storage.memory import MemoryStorage
from unittest.mock import AsyncMock, MagicMock

from bowling_bot.handlers.user import start

@pytest.mark.asyncio
async def test_start_handler():
    # Создание моков
    msg = MagicMock(spec=types.Message)
    msg.from_user = MagicMock()
    msg.from_user.id = 123456
    msg.from_user.full_name = "Test User"
    msg.answer = AsyncMock()
    
    storage = MemoryStorage()
    state = FSMContext(storage=storage, key=storage.resolve_key(123456, 123456))
    
    # Вызов обработчика
    await start.start(msg, state)
    
    # Проверки
    assert msg.answer.called
    assert await state.get_state() is not None
```

## Деплой

### Docker

```dockerfile
# Dockerfile
FROM python:3.13-slim

WORKDIR /app

# Установка Poetry
RUN pip install poetry

# Копирование файлов зависимостей
COPY pyproject.toml poetry.lock ./

# Установка зависимостей
RUN poetry config virtualenvs.create false && \
    poetry install --no-dev

# Копирование кода
COPY . .

# Запуск
CMD ["python", "-m", "bowling_bot.bot"]
```

```yaml
# docker-compose.yml
version: '3.8'

services:
  bot:
    build: .
    env_file: .env
    depends_on:
      - postgres
      - redis
    restart: unless-stopped

  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: bot_db
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
    volumes:
      - postgres_data:/var/lib/postgresql/data
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    command: redis-server --requirepass ${FSM_PASSWORD}
    restart: unless-stopped

volumes:
  postgres_data:
```

### Systemd

```ini
# /etc/systemd/system/bot.service
[Unit]
Description=Telegram Bot
After=network.target postgresql.service redis.service

[Service]
Type=simple
User=bot
WorkingDirectory=/opt/bot
Environment="PATH=/opt/bot/.venv/bin"
ExecStart=/opt/bot/.venv/bin/python -m bowling_bot.bot
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

### Nginx для webhook

```nginx
# /etc/nginx/sites-available/bot
server {
    listen 443 ssl http2;
    server_name yourdomain.com;

    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;

    location /tg/webhooks/ {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

## Полезные советы

1. **Используйте типизацию** - помогает избежать ошибок
2. **Логируйте важные действия** - упрощает отладку
3. **Обрабатывайте все исключения** - улучшает UX
4. **Используйте FSM для сложных диалогов** - упрощает код
5. **Кэшируйте часто запрашиваемые данные** - улучшает производительность
6. **Используйте connection pooling** - не создавайте новые соединения для каждого запроса
7. **Тестируйте локально с polling** - проще для разработки
8. **Используйте webhook в продакшене** - более эффективно

## Дополнительные ресурсы

- [Документация aiogram](https://docs.aiogram.dev/)
- [Документация asyncpg](https://magicstack.github.io/asyncpg/)
- [Документация structlog](https://www.structlog.org/)
- [Telegram Bot API](https://core.telegram.org/bots/api)

