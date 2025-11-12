# Шпаргалка по шаблону

## Быстрые ссылки

- [Обработчики](#обработчики)
- [Работа с БД](#работа-с-бд)
- [FSM](#fsm)
- [Клавиатуры](#клавиатуры)
- [Фильтры](#фильтры)
- [Логирование](#логирование)
- [Обработка ошибок](#обработка-ошибок)

## Обработчики

### Базовый обработчик сообщения

```python
from aiogram import types
from aiogram.fsm.context import FSMContext

async def handler(msg: types.Message, state: FSMContext) -> None:
    await msg.answer("Ответ")
```

### Обработчик команды

```python
from aiogram.filters import Command

# Регистрация
user_router.message.register(handler, Command("start"))
```

### Обработчик callback_query

```python
from aiogram import types
from bowling_bot.keyboards.inline.callbacks import Action

async def callback_handler(
    callback: types.CallbackQuery,
    callback_data: Action,
) -> None:
    await callback.answer("Обработано!")
    await callback.message.edit_text("Новый текст")
```

### Обработчик с зависимостями

```python
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from bowling_bot.db.db_api.storages.postgres import PostgresConnection

async def handler(
    msg: types.Message,
    db_conn: "PostgresConnection",
) -> None:
    result = await db_conn._fetchrow("SELECT * FROM users WHERE id = $1", (user_id,))
```

## Работа с БД

### Получение одного результата

```python
result = await db_conn._fetchrow(
    "SELECT * FROM users WHERE telegram_id = $1",
    (user_id,),
)
user = result.data  # dict | None
```

### Получение нескольких результатов

```python
results = await db_conn._fetch(
    "SELECT * FROM users WHERE active = $1",
    (True,),
)
users = results.data  # list[dict]
```

### Выполнение запроса без результата

```python
await db_conn._execute(
    "UPDATE users SET name = $1 WHERE id = $2",
    (name, user_id),
)
```

### Конвертация в Pydantic модель

```python
from bowling_bot.models import BaseModel

class User(BaseModel):
    id: int
    name: str

result = await db_conn._fetchrow("SELECT * FROM users WHERE id = $1", (user_id,))
user = result.convert(User)  # User | None
```

### Транзакция

```python
async with db_conn._pool.acquire() as conn:
    async with conn.transaction():
        await conn.execute("UPDATE ...")
        await conn.execute("INSERT ...")
```

### Использование репозитория

```python
# Создание репозитория
from bowling_bot.db.repositories.user_repository import UserRepository

user_repo = UserRepository(db_conn)

# Использование методов
user = await user_repo.get_by_telegram_id(telegram_id)
user = await user_repo.create(telegram_id, name)
await user_repo.update_name(telegram_id, new_name)
```

### Создание репозитория

```python
# db/repositories/user_repository.py
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from bowling_bot.db.db_api.storages.postgres import PostgresConnection

class UserRepository:
    def __init__(self, db_conn: "PostgresConnection") -> None:
        self.db_conn = db_conn
    
    async def get_by_telegram_id(self, telegram_id: int) -> dict | None:
        result = await self.db_conn._fetchrow(
            "SELECT * FROM users WHERE telegram_id = $1",
            (telegram_id,),
        )
        return result.data
```

## FSM

### Установка состояния

```python
await state.set_state(states.user.UserMainMenu.menu)
```

### Получение данных состояния

```python
data = await state.get_data()
name = data.get("name")
```

### Обновление данных состояния

```python
await state.update_data(name="Иван", age=25)
```

### Очистка состояния

```python
await state.clear()
```

### Проверка состояния в фильтре

```python
from aiogram.filters import StateFilter

user_router.message.register(
    handler,
    StateFilter(states.user.UserMainMenu.menu),
)
```

## Клавиатуры

### Reply-клавиатура (базовые кнопки)

```python
from bowling_bot.keyboards.default import BasicButtons

# Простые
keyboard = BasicButtons.back()
keyboard = BasicButtons.cancel()
keyboard = BasicButtons.back_n_cancel()

# Подтверждение
keyboard = BasicButtons.confirmation(add_back=True, add_cancel=True)

# Да/Нет
keyboard = BasicButtons.yes_n_no(add_back=True)

# Пропустить
keyboard = BasicButtons.skip(add_back=True)
```

### Reply-клавиатура (кастомная)

```python
from bowling_bot.keyboards.default.consts import DefaultConstructor

class MyButtons(DefaultConstructor):
    @staticmethod
    def menu() -> ReplyKeyboardMarkup:
        schema = [2, 1]  # 2 кнопки в первой строке, 1 во второй
        btns = ["Кнопка1", "Кнопка2", "Кнопка3"]
        return MyButtons._create_kb(btns, schema)
```

### Inline-клавиатура

```python
from bowling_bot.keyboards.inline.consts import InlineConstructor
from bowling_bot.keyboards.inline.callbacks import Action

class MyInline(InlineConstructor):
    @staticmethod
    def menu() -> InlineKeyboardMarkup:
        schema = [1, 1]
        btns = [
            {"text": "Кнопка1", "callback_data": Action(action="action1")},
            {"text": "Кнопка2", "callback_data": Action(action="action2")},
        ]
        return MyInline._create_kb(btns, schema)
```

### Создание callback data

```python
# keyboards/inline/callbacks.py
from aiogram.filters.callback_data import CallbackData

class MyCallback(CallbackData, prefix="my"):
    action: str
    item_id: int | None = None

# Использование
callback = MyCallback(action="delete", item_id=123)
```

## Фильтры

### ChatTypeFilter

```python
from bowling_bot.filters import ChatTypeFilter

# Только приватные чаты
user_router.message.filter(ChatTypeFilter("private"))

# Приватные или группы
user_router.message.filter(ChatTypeFilter(["private", "group"]))
```

### TextFilter

```python
from bowling_bot.filters import TextFilter

# Точное совпадение
user_router.message.register(handler, TextFilter("Текст"))

# Одно из значений
user_router.message.register(handler, TextFilter(["Текст1", "Текст2"]))
```

### Комбинация фильтров

```python
from aiogram.filters import StateFilter

user_router.message.register(
    handler,
    TextFilter("Текст"),
    StateFilter(states.user.UserMainMenu.menu),
)
```

## Логирование

### Получение логгера

```python
# В обработчике
business_logger = dp["business_logger"]
db_logger = dp["db_logger"]
```

### Логирование с контекстом

```python
logger = business_logger.bind(
    user_id=msg.from_user.id,
    action="something",
)
logger.info("Событие произошло")
```

### Уровни логирования

```python
logger.debug("Отладочное сообщение")
logger.info("Информационное сообщение")
logger.warning("Предупреждение")
logger.error("Ошибка")
logger.exception("Исключение с traceback")
```

## Обработка ошибок

### Try/except в обработчике

```python
async def handler(msg: types.Message) -> None:
    try:
        # Код
        pass
    except Exception as e:
        logger.exception("Ошибка в обработчике", error=str(e))
        await msg.answer("Произошла ошибка")
```

### Глобальный обработчик ошибок

```python
# handlers/errors.py
async def error_handler(update: types.Update, exception: Exception) -> None:
    logger.exception("Необработанное исключение", error=str(exception))
    if update.message:
        await update.message.answer("Произошла ошибка")

# Регистрация в bot.py
dp.errors.register(error_handler)
```

## Регистрация обработчиков

### Базовый роутер

```python
# handlers/user/__init__.py
from aiogram import Router
from bowling_bot.filters import ChatTypeFilter

def prepare_router() -> Router:
    user_router = Router()
    user_router.message.filter(ChatTypeFilter("private"))
    
    user_router.message.register(start.start, Command("start"))
    
    return user_router
```

### Регистрация в bot.py

```python
# bot.py
def setup_handlers(dp: Dispatcher) -> None:
    dp.include_router(handlers.user.prepare_router())
```

## Работа с медиа

### Получение файла

```python
if msg.photo:
    photo = msg.photo[-1]  # Наибольший размер
    file = await msg.bot.get_file(photo.file_id)
```

### Отправка фото

```python
await msg.answer_photo(photo=photo.file_id, caption="Подпись")
```

## Кэширование (если USE_CACHE=true)

### Сохранение

```python
cache_pool: Redis = dp["cache_pool"]
await cache_pool.set("key", "value", ex=3600)  # TTL 1 час
```

### Получение

```python
value = await cache_pool.get("key")
```

### Удаление

```python
await cache_pool.delete("key")
```

## Полезные паттерны

### Многошаговый диалог

```python
# Шаг 1
await state.set_state(states.user.Registration.waiting_name)
await msg.answer("Введите имя:", reply_markup=BasicButtons.cancel())

# Шаг 2 (в другом обработчике)
data = await state.get_data()
name = data.get("name")
await state.update_data(age=age)
await state.set_state(states.user.Registration.waiting_confirmation)

# Завершение
await state.clear()
```

### Пагинация

```python
from bowling_bot.utils.chunks import chunks

items = list(range(100))
page = 1
per_page = 10

start = (page - 1) * per_page
end = start + per_page
page_items = items[start:end]
```

### Валидация ввода

```python
async def process_age(msg: types.Message, state: FSMContext) -> None:
    try:
        age = int(msg.text)
        if age < 0 or age > 150:
            raise ValueError
    except ValueError:
        await msg.answer("Введите корректный возраст (0-150)")
        return
    
    await state.update_data(age=age)
```

### Ответ на callback с задержкой

```python
await callback.answer("Обрабатывается...", show_alert=False)
# Долгая операция
await callback.message.edit_text("Готово!")
```

## Переменные окружения

```env
# Обязательные
BOT_TOKEN=
PG_HOST=
PG_PORT=
PG_USER=
PG_PASSWORD=
PG_DATABASE=
FSM_HOST=
FSM_PORT=
FSM_PASSWORD=

# Опциональные
USE_CACHE=false
USE_WEBHOOK=false
LOGGING_LEVEL=10
DROP_PREVIOUS_UPDATES=false
USE_CUSTOM_API_SERVER=false
```

## Структура файлов

```
handlers/
  user/
    __init__.py    # prepare_router()
    start.py       # Обработчики
    profile.py
    
keyboards/
  default/
    basic.py       # Базовые кнопки
    consts.py      # DefaultConstructor
  inline/
    callbacks.py   # CallbackData классы
    consts.py      # InlineConstructor

states/
  user.py          # StatesGroup классы

db/
  repositories/    # Репозитории для работы с БД
```

## Частые ошибки

### ❌ Неправильно

```python
# Прямое создание соединения
conn = await asyncpg.connect(...)

# Логирование без контекста
print("Something happened")

# Синхронные операции в async функции
time.sleep(1)
```

### ✅ Правильно

```python
# Использование connection pool
db_conn: PostgresConnection = dp["db_conn"]
result = await db_conn._fetchrow(...)

# Структурированное логирование
logger.info("Something happened", context="value")

# Асинхронные операции
await asyncio.sleep(1)
```

## Команды

```bash
# Запуск
python -m bowling_bot.bot

# Линтинг
make lint

# Инициализация проекта
make init_project
```

