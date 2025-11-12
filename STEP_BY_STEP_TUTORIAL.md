# Пошаговый туториал - от простого к сложному

Этот туториал проведет вас через создание бота от самого простого примера до сложного функционала.

## Урок 1: Самый простой бот - эхо-бот

**Цель:** Создать бота, который повторяет все, что вы написали.

### Шаг 1: Создайте файл обработчика

Создайте файл `bowling_bot/handlers/user/echo.py`:

```python
from aiogram import types
from aiogram.fsm.context import FSMContext

async def echo_handler(msg: types.Message, state: FSMContext) -> None:
    """Просто повторяет сообщение пользователя"""
    if msg.text:
        await msg.answer(f"Вы написали: {msg.text}")
    else:
        await msg.answer("Отправьте текстовое сообщение")
```

### Шаг 2: Зарегистрируйте обработчик

Откройте `bowling_bot/handlers/user/__init__.py` и добавьте:

```python
from aiogram import Router
from aiogram.filters import CommandStart
from bowling_bot.filters import ChatTypeFilter
from . import start
from . import echo  # Добавьте эту строку

def prepare_router() -> Router:
    user_router = Router()
    user_router.message.filter(ChatTypeFilter("private"))
    
    user_router.message.register(start.start, CommandStart())
    user_router.message.register(echo.echo_handler)  # Добавьте эту строку
    
    return user_router
```

### Шаг 3: Запустите бота

```bash
python -m bowling_bot.bot
```

### Шаг 4: Протестируйте

Напишите боту любое сообщение. Он должен повторить его.

**Что вы узнали:**
- Как создать простой обработчик
- Как зарегистрировать обработчик
- Как бот отвечает на сообщения

---

## Урок 2: Бот с командами

**Цель:** Создать бота с несколькими командами.

### Шаг 1: Создайте обработчики команд

Создайте файл `bowling_bot/handlers/user/commands.py`:

```python
from aiogram import types, html
from aiogram.filters import Command
from aiogram.fsm.context import FSMContext

async def help_command(msg: types.Message, state: FSMContext) -> None:
    """Обработчик команды /help"""
    help_text = """
    <b>Доступные команды:</b>
    
    /start - Начать работу
    /help - Показать эту справку
    /info - Информация о боте
    /time - Текущее время
    """
    await msg.answer(help_text, parse_mode="HTML")

async def info_command(msg: types.Message, state: FSMContext) -> None:
    """Обработчик команды /info"""
    if msg.from_user is None:
        return
    
    info_text = f"""
    <b>Информация о боте:</b>
    
    Ваш ID: <code>{msg.from_user.id}</code>
    Ваше имя: {html.quote(msg.from_user.full_name or 'Не указано')}
    Username: @{html.quote(msg.from_user.username or 'не указан')}
    """
    await msg.answer(info_text, parse_mode="HTML")

async def time_command(msg: types.Message, state: FSMContext) -> None:
    """Обработчик команды /time"""
    from datetime import datetime
    
    current_time = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    await msg.answer(f"Текущее время: {current_time}")
```

### Шаг 2: Зарегистрируйте команды

В `bowling_bot/handlers/user/__init__.py`:

```python
from aiogram import Router
from aiogram.filters import CommandStart, Command
from bowling_bot.filters import ChatTypeFilter
from . import start
from . import echo
from . import commands  # Добавьте

def prepare_router() -> Router:
    user_router = Router()
    user_router.message.filter(ChatTypeFilter("private"))
    
    user_router.message.register(start.start, CommandStart())
    user_router.message.register(echo.echo_handler)
    
    # Регистрируем команды
    user_router.message.register(commands.help_command, Command("help"))
    user_router.message.register(commands.info_command, Command("info"))
    user_router.message.register(commands.time_command, Command("time"))
    
    return user_router
```

### Шаг 3: Протестируйте

Попробуйте команды:
- `/help`
- `/info`
- `/time`

**Что вы узнали:**
- Как обрабатывать команды
- Как использовать HTML форматирование
- Как получать информацию о пользователе

---

## Урок 3: Бот с кнопками (клавиатура)

**Цель:** Добавить кнопки для удобной навигации.

### Шаг 1: Создайте клавиатуру

Создайте файл `bowling_bot/keyboards/default/menu.py`:

```python
from aiogram.types import ReplyKeyboardMarkup
from .consts import DefaultConstructor

class MenuButtons(DefaultConstructor):
    @staticmethod
    def main_menu() -> ReplyKeyboardMarkup:
        """Главное меню"""
        schema = [2, 1]  # 2 кнопки в первой строке, 1 во второй
        btns = [
            "📊 Статистика",
            "⚙️ Настройки",
            "❓ Помощь",
        ]
        return MenuButtons._create_kb(btns, schema)
```

### Шаг 2: Обновите обработчик /start

В `bowling_bot/handlers/user/start.py`:

```python
from aiogram import html, types
from aiogram.fsm.context import FSMContext
from bowling_bot import states
from bowling_bot.keyboards.default.menu import MenuButtons  # Добавьте

async def start(msg: types.Message, state: FSMContext) -> None:
    if msg.from_user is None:
        return
    m = [
        f'Hello, <a href="tg://user?id={msg.from_user.id}">{html.quote(msg.from_user.full_name)}</a>',
    ]
    await msg.answer("\n".join(m), reply_markup=MenuButtons.main_menu())  # Добавьте клавиатуру
    await state.set_state(states.user.UserMainMenu.menu)
```

### Шаг 3: Создайте обработчики для кнопок

В `bowling_bot/handlers/user/menu.py`:

```python
from aiogram import types
from aiogram.fsm.context import FSMContext
from bowling_bot.filters import TextFilter
from bowling_bot import states

async def show_stats(msg: types.Message, state: FSMContext) -> None:
    """Показать статистику"""
    await msg.answer("Ваша статистика:\nСообщений отправлено: 0")

async def show_settings(msg: types.Message, state: FSMContext) -> None:
    """Показать настройки"""
    await msg.answer("Настройки:\nУведомления: Включены")

async def show_help(msg: types.Message, state: FSMContext) -> None:
    """Показать помощь"""
    await msg.answer("Помощь:\nИспользуйте кнопки для навигации")
```

### Шаг 4: Зарегистрируйте обработчики

В `bowling_bot/handlers/user/__init__.py`:

```python
from bowling_bot.filters import ChatTypeFilter, TextFilter
from bowling_bot import states
from . import menu  # Добавьте

def prepare_router() -> Router:
    user_router = Router()
    user_router.message.filter(ChatTypeFilter("private"))
    
    # ... существующие регистрации ...
    
    # Обработчики кнопок (только в состоянии menu)
    user_router.message.register(
        menu.show_stats,
        TextFilter("📊 Статистика"),
        StateFilter(states.user.UserMainMenu.menu),
    )
    user_router.message.register(
        menu.show_settings,
        TextFilter("⚙️ Настройки"),
        StateFilter(states.user.UserMainMenu.menu),
    )
    user_router.message.register(
        menu.show_help,
        TextFilter("❓ Помощь"),
        StateFilter(states.user.UserMainMenu.menu),
    )
    
    return user_router
```

### Шаг 5: Протестируйте

Нажмите кнопки в меню и посмотрите, как они работают.

**Что вы узнали:**
- Как создавать reply-клавиатуры
- Как обрабатывать нажатия кнопок
- Как использовать фильтры для кнопок

---

## Урок 4: Работа с базой данных

**Цель:** Сохранять и получать данные пользователей из базы данных.

### Шаг 1: Создайте таблицу в БД

Подключитесь к PostgreSQL и выполните:

```sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    telegram_id BIGINT UNIQUE NOT NULL,
    username VARCHAR(255),
    full_name VARCHAR(255),
    message_count INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_users_telegram_id ON users(telegram_id);
```

### Шаг 2: Создайте репозиторий

Создайте файл `bowling_bot/db/repositories/user_repository.py`:

```python
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from bowling_bot.db.db_api.storages.postgres import PostgresConnection

class UserRepository:
    def __init__(self, db_conn: "PostgresConnection") -> None:
        self.db_conn = db_conn
    
    async def get_or_create(self, telegram_id: int, username: str | None, full_name: str) -> dict:
        """Получить пользователя или создать нового"""
        # Пытаемся найти
        result = await self.db_conn._fetchrow(
            "SELECT * FROM users WHERE telegram_id = $1",
            (telegram_id,),
        )
        
        if result.data:
            return result.data
        
        # Создаем нового
        await self.db_conn._execute(
            """
            INSERT INTO users (telegram_id, username, full_name)
            VALUES ($1, $2, $3)
            """,
            (telegram_id, username, full_name),
        )
        
        # Возвращаем созданного
        result = await self.db_conn._fetchrow(
            "SELECT * FROM users WHERE telegram_id = $1",
            (telegram_id,),
        )
        return result.data
    
    async def increment_message_count(self, telegram_id: int) -> None:
        """Увеличить счетчик сообщений"""
        await self.db_conn._execute(
            """
            UPDATE users 
            SET message_count = message_count + 1,
                updated_at = NOW()
            WHERE telegram_id = $1
            """,
            (telegram_id,),
        )
    
    async def get_stats(self, telegram_id: int) -> dict | None:
        """Получить статистику пользователя"""
        result = await self.db_conn._fetchrow(
            "SELECT message_count, created_at FROM users WHERE telegram_id = $1",
            (telegram_id,),
        )
        return result.data
```

### Шаг 3: Обновите обработчик /start

В `bowling_bot/handlers/user/start.py`:

```python
from aiogram import html, types
from aiogram.fsm.context import FSMContext
from typing import TYPE_CHECKING
from bowling_bot import states
from bowling_bot.keyboards.default.menu import MenuButtons

if TYPE_CHECKING:
    from bowling_bot.db.db_api.storages.postgres import PostgresConnection
    from bowling_bot.db.repositories.user_repository import UserRepository

async def start(
    msg: types.Message,
    state: FSMContext,
    db_conn: "PostgresConnection",
) -> None:
    if msg.from_user is None:
        return
    
    # Создаем репозиторий
    user_repo = UserRepository(db_conn)
    
    # Получаем или создаем пользователя
    user = await user_repo.get_or_create(
        telegram_id=msg.from_user.id,
        username=msg.from_user.username,
        full_name=msg.from_user.full_name or "Не указано",
    )
    
    m = [
        f'Hello, <a href="tg://user?id={msg.from_user.id}">{html.quote(msg.from_user.full_name)}</a>',
        f'Вы зарегистрированы с {user["created_at"]}',
    ]
    await msg.answer("\n".join(m), reply_markup=MenuButtons.main_menu())
    await state.set_state(states.user.UserMainMenu.menu)
```

### Шаг 4: Обновите обработчик статистики

В `bowling_bot/handlers/user/menu.py`:

```python
from aiogram import types
from aiogram.fsm.context import FSMContext
from typing import TYPE_CHECKING
from bowling_bot.filters import TextFilter
from bowling_bot import states

if TYPE_CHECKING:
    from bowling_bot.db.db_api.storages.postgres import PostgresConnection
    from bowling_bot.db.repositories.user_repository import UserRepository

async def show_stats(
    msg: types.Message,
    state: FSMContext,
    db_conn: "PostgresConnection",
) -> None:
    """Показать статистику"""
    if msg.from_user is None:
        return
    
    user_repo = UserRepository(db_conn)
    stats = await user_repo.get_stats(msg.from_user.id)
    
    if stats:
        await msg.answer(
            f"Ваша статистика:\n"
            f"Сообщений отправлено: {stats['message_count']}\n"
            f"Зарегистрирован: {stats['created_at']}"
        )
    else:
        await msg.answer("Статистика не найдена. Используйте /start")
```

### Шаг 5: Обновите echo_handler для подсчета сообщений

В `bowling_bot/handlers/user/echo.py`:

```python
from aiogram import types
from aiogram.fsm.context import FSMContext
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from bowling_bot.db.db_api.storages.postgres import PostgresConnection
    from bowling_bot.db.repositories.user_repository import UserRepository

async def echo_handler(
    msg: types.Message,
    state: FSMContext,
    db_conn: "PostgresConnection",
) -> None:
    """Просто повторяет сообщение пользователя"""
    if msg.from_user is None:
        return
    
    # Увеличиваем счетчик сообщений
    user_repo = UserRepository(db_conn)
    await user_repo.increment_message_count(msg.from_user.id)
    
    if msg.text:
        await msg.answer(f"Вы написали: {msg.text}")
    else:
        await msg.answer("Отправьте текстовое сообщение")
```

### Шаг 6: Протестируйте

1. Напишите `/start` - пользователь создастся в БД
2. Напишите несколько сообщений - счетчик увеличится
3. Нажмите "📊 Статистика" - увидите количество сообщений

**Что вы узнали:**
- Как работать с базой данных
- Как создавать репозитории
- Как сохранять и получать данные

---

## Урок 5: FSM - многошаговый диалог

**Цель:** Создать диалог регистрации с несколькими шагами.

### Шаг 1: Создайте состояния

В `bowling_bot/states/user.py` добавьте:

```python
from aiogram.fsm.state import State, StatesGroup

class UserMainMenu(StatesGroup):
    menu = State()

class RegistrationStates(StatesGroup):  # Добавьте
    waiting_name = State()
    waiting_age = State()
    waiting_confirmation = State()
```

### Шаг 2: Создайте обработчики регистрации

Создайте файл `bowling_bot/handlers/user/registration.py`:

```python
from aiogram import types
from aiogram.fsm.context import FSMContext
from bowling_bot import states
from bowling_bot.keyboards.default import BasicButtons

async def start_registration(msg: types.Message, state: FSMContext) -> None:
    """Начать регистрацию"""
    await state.set_state(states.user.RegistrationStates.waiting_name)
    await msg.answer(
        "Давайте зарегистрируем вас!\nВведите ваше имя:",
        reply_markup=BasicButtons.cancel(),
    )

async def process_name(msg: types.Message, state: FSMContext) -> None:
    """Обработать имя"""
    if msg.text is None:
        await msg.answer("Пожалуйста, отправьте текстовое сообщение.")
        return
    
    if len(msg.text) < 2:
        await msg.answer("Имя слишком короткое. Введите имя (минимум 2 символа):")
        return
    
    await state.update_data(name=msg.text)
    await state.set_state(states.user.RegistrationStates.waiting_age)
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
    
    await state.set_state(states.user.RegistrationStates.waiting_confirmation)
    await msg.answer(
        f"Проверьте данные:\n"
        f"Имя: {data['name']}\n"
        f"Возраст: {data['age']}",
        reply_markup=BasicButtons.confirmation(add_cancel=True),
    )

async def confirm_registration(msg: types.Message, state: FSMContext) -> None:
    """Подтвердить регистрацию"""
    data = await state.get_data()
    
    # Здесь можно сохранить в БД
    # user_repo = UserRepository(db_conn)
    # await user_repo.update_profile(...)
    
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

### Шаг 3: Зарегистрируйте обработчики

В `bowling_bot/handlers/user/__init__.py`:

```python
from aiogram.filters import StateFilter
from . import registration  # Добавьте

def prepare_router() -> Router:
    user_router = Router()
    user_router.message.filter(ChatTypeFilter("private"))
    
    # ... существующие регистрации ...
    
    # Начало регистрации (из главного меню)
    user_router.message.register(
        registration.start_registration,
        TextFilter("📝 Регистрация"),
        StateFilter(states.user.UserMainMenu.menu),
    )
    
    # Шаги регистрации
    user_router.message.register(
        registration.process_name,
        StateFilter(states.user.RegistrationStates.waiting_name),
    )
    user_router.message.register(
        registration.process_age,
        StateFilter(states.user.RegistrationStates.waiting_age),
    )
    
    # Подтверждение/отмена
    user_router.message.register(
        registration.confirm_registration,
        TextFilter("✅Подтвердить"),
        StateFilter(states.user.RegistrationStates.waiting_confirmation),
    )
    user_router.message.register(
        registration.cancel_registration,
        TextFilter("🚫 Отмена"),
    )
    
    return user_router
```

### Шаг 4: Добавьте кнопку регистрации в меню

В `bowling_bot/keyboards/default/menu.py`:

```python
@staticmethod
def main_menu() -> ReplyKeyboardMarkup:
    schema = [2, 2, 1]  # Измените на 2, 2, 1
    btns = [
        "📊 Статистика",
        "⚙️ Настройки",
        "📝 Регистрация",  # Добавьте
        "👤 Профиль",
        "❓ Помощь",
    ]
    return MenuButtons._create_kb(btns, schema)
```

### Шаг 5: Протестируйте

1. Нажмите "📝 Регистрация"
2. Введите имя
3. Введите возраст
4. Подтвердите или отмените

**Что вы узнали:**
- Как использовать FSM для многошаговых диалогов
- Как валидировать ввод пользователя
- Как управлять состояниями

---

## Урок 6: Inline-клавиатуры

**Цель:** Создать inline-клавиатуру с кнопками под сообщением.

### Шаг 1: Создайте callback data

В `bowling_bot/keyboards/inline/callbacks.py` добавьте:

```python
from aiogram.filters.callback_data import CallbackData

class Action(CallbackData, prefix="act"):
    action: str

class MenuAction(CallbackData, prefix="menu"):  # Добавьте
    action: str
    item_id: int | None = None
```

### Шаг 2: Создайте inline-клавиатуру

Создайте файл `bowling_bot/keyboards/inline/user/menu.py`:

```python
from aiogram.types import InlineKeyboardMarkup
from ..consts import InlineConstructor
from ..callbacks import MenuAction

class UserMenu(InlineConstructor):
    @staticmethod
    def main_menu() -> InlineKeyboardMarkup:
        """Главное меню с inline кнопками"""
        schema = [1, 1, 1]
        btns = [
            {"text": "📊 Статистика", "callback_data": MenuAction(action="stats")},
            {"text": "⚙️ Настройки", "callback_data": MenuAction(action="settings")},
            {"text": "❓ Помощь", "callback_data": MenuAction(action="help")},
        ]
        return UserMenu._create_kb(btns, schema)
    
    @staticmethod
    def settings_menu() -> InlineKeyboardMarkup:
        """Меню настроек"""
        schema = [1, 1, 1]
        btns = [
            {"text": "🔔 Уведомления", "callback_data": MenuAction(action="notifications")},
            {"text": "🌐 Язык", "callback_data": MenuAction(action="language")},
            {"text": "◀️ Назад", "callback_data": MenuAction(action="back")},
        ]
        return UserMenu._create_kb(btns, schema)
```

### Шаг 3: Создайте обработчики callback

Создайте файл `bowling_bot/handlers/user/inline_menu.py`:

```python
from aiogram import types
from bowling_bot.keyboards.inline.callbacks import MenuAction
from bowling_bot.keyboards.inline.user.menu import UserMenu

async def handle_menu_callback(
    callback: types.CallbackQuery,
    callback_data: MenuAction,
) -> None:
    """Обработка callback от inline кнопок"""
    if callback_data.action == "stats":
        await callback.answer("Загрузка статистики...")
        await callback.message.edit_text(
            "Ваша статистика:\nСообщений: 0",
            reply_markup=UserMenu.main_menu(),
        )
    
    elif callback_data.action == "settings":
        await callback.answer()
        await callback.message.edit_text(
            "Настройки:",
            reply_markup=UserMenu.settings_menu(),
        )
    
    elif callback_data.action == "back":
        await callback.answer()
        await callback.message.edit_text(
            "Главное меню:",
            reply_markup=UserMenu.main_menu(),
        )
    
    elif callback_data.action == "help":
        await callback.answer("Помощь", show_alert=True)
    
    else:
        await callback.answer("Неизвестное действие")
```

### Шаг 4: Зарегистрируйте обработчик

В `bowling_bot/handlers/user/__init__.py`:

```python
from . import inline_menu  # Добавьте

def prepare_router() -> Router:
    user_router = Router()
    # ... существующий код ...
    
    # Обработчик inline кнопок
    user_router.callback_query.register(
        inline_menu.handle_menu_callback,
        MenuAction.filter(),
    )
    
    return user_router
```

### Шаг 5: Добавьте команду для показа inline-меню

В `bowling_bot/handlers/user/commands.py`:

```python
from bowling_bot.keyboards.inline.user.menu import UserMenu

async def menu_command(msg: types.Message, state: FSMContext) -> None:
    """Показать inline меню"""
    await msg.answer(
        "Выберите действие:",
        reply_markup=UserMenu.main_menu(),
    )
```

И зарегистрируйте:

```python
user_router.message.register(commands.menu_command, Command("menu"))
```

### Шаг 6: Протестируйте

1. Напишите `/menu`
2. Нажмите на inline-кнопки
3. Посмотрите, как меняется сообщение

**Что вы узнали:**
- Как создавать inline-клавиатуры
- Как обрабатывать callback_query
- Как изменять сообщения с клавиатурами

---

## Урок 7: Комбинирование всего вместе

**Цель:** Создать полноценного бота с БД, FSM, клавиатурами.

### Финальная структура обработчиков:

```
handlers/user/
├── __init__.py          # Регистрация всех обработчиков
├── start.py            # /start команда
├── commands.py         # Другие команды
├── echo.py             # Эхо-обработчик
├── menu.py             # Обработчики reply-кнопок
├── registration.py      # Регистрация с FSM
└── inline_menu.py      # Обработчики inline-кнопок
```

### Что вы теперь умеете:

✅ Создавать обработчики сообщений  
✅ Работать с командами  
✅ Создавать reply-клавиатуры  
✅ Создавать inline-клавиатуры  
✅ Работать с базой данных  
✅ Использовать FSM для диалогов  
✅ Валидировать ввод пользователя  
✅ Комбинировать все вместе  

### Следующие шаги:

1. Изучите ARCHITECTURE.md для понимания архитектуры
2. Прочитайте USAGE_GUIDE.md для дополнительных примеров
3. Используйте CHEATSHEET.md как справочник
4. Экспериментируйте и создавайте свой функционал!

---

## Частые проблемы и решения

### Проблема 1: Обработчик не срабатывает

**Решение:**
- Проверьте, что обработчик зарегистрирован в `__init__.py`
- Проверьте фильтры (ChatTypeFilter, StateFilter)
- Проверьте логи на наличие ошибок

### Проблема 2: Ошибка подключения к БД

**Решение:**
- Проверьте настройки в `.env`
- Убедитесь, что PostgreSQL запущен
- Проверьте, что база данных создана

### Проблема 3: Состояние не сохраняется

**Решение:**
- Убедитесь, что Redis запущен
- Проверьте настройки FSM в `.env`
- Проверьте, что используете `await state.set_state()`

### Проблема 4: Клавиатура не отображается

**Решение:**
- Проверьте схему клавиатуры (количество кнопок должно совпадать)
- Убедитесь, что используете правильный тип клавиатуры
- Проверьте, что передаете `reply_markup` в `answer()`

---

Удачи в разработке! 🚀

