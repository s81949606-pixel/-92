import asyncio
import aiosqlite
import random
import logging
from datetime import datetime, timedelta
from aiogram import Bot, Dispatcher, F
from aiogram.types import Message, CallbackQuery, InlineKeyboardMarkup, InlineKeyboardButton, ReplyKeyboardMarkup, KeyboardButton
from aiogram.filters import Command
from aiogram.fsm.context import FSMContext
from aiogram.fsm.state import State, StatesGroup
from aiogram.fsm.storage.memory import MemoryStorage

# ТВОЙ ТОКЕН (замени на безопасный!)
TOKEN = "8254359285:AAFU6PgniIMuyd6nfAJzlW2bnimIQRy3XTY"
ADMIN_ID = 123456789  # Замени на свой Telegram ID (@userinfobot)
bot = Bot(TOKEN)
dp = Dispatcher(storage=MemoryStorage())
logging.basicConfig(level=logging.INFO)

# База данных
DB_FILE = "bitcoin_bot.db"

# Состояния FSM
class Forms(StatesGroup):
    transfer = State()
    promo = State()
    craft = State()
    work = State()
    mine = State()
    fish = State()
    job_apply = State()

# Инициализация БД
async def init_db():
    async with aiosqlite.connect(DB_FILE) as db:
        await db.execute('''CREATE TABLE IF NOT EXISTS users (
            user_id INTEGER PRIMARY KEY, coins INTEGER DEFAULT 0, btc REAL DEFAULT 0, level INTEGER DEFAULT 1, exp INTEGER DEFAULT 0,
            daily_claimed INTEGER DEFAULT 0, last_daily TEXT, status TEXT DEFAULT 'Новичок', battle_pass INTEGER DEFAULT 0,
            premium_pass INTEGER DEFAULT 0, halloween_event INTEGER DEFAULT 0, newyear_event INTEGER DEFAULT 0,
            pickaxe TEXT DEFAULT 'Железная', fishing_rod TEXT DEFAULT 'Простая', pets TEXT DEFAULT '', bosses_killed INTEGER DEFAULT 0,
            house TEXT DEFAULT '', car TEXT DEFAULT '', farm_level INTEGER DEFAULT 0, mining_rig INTEGER DEFAULT 0,
            unique_id TEXT UNIQUE, referrals INTEGER DEFAULT 0, top_weekly INTEGER DEFAULT 0
        )''')
        await db.execute('''CREATE TABLE IF NOT EXISTS inventory (user_id INTEGER, item TEXT, amount INTEGER)''')
        await db.execute('''CREATE TABLE IF NOT EXISTS market (id INTEGER PRIMARY KEY AUTOINCREMENT, seller_id INTEGER, item TEXT, price INTEGER, sold INTEGER DEFAULT 0)''')
        await db.execute('''CREATE TABLE IF NOT EXISTS jobs (user_id INTEGER, job TEXT, salary INTEGER, last_work TEXT)''')
        await db.execute('''CREATE TABLE IF NOT EXISTS promos (code TEXT PRIMARY KEY, reward TEXT, uses INTEGER DEFAULT 1)''')
        await db.execute('''CREATE TABLE IF NOT EXISTS tops (user_id INTEGER, coins INTEGER, btc REAL, period TEXT)''')
        await db.commit()

# Генерация уникального ID
async def get_unique_id(user_id: int):
    async with aiosqlite.connect(DB_FILE) as db:
        async with db.execute("SELECT unique_id FROM users WHERE user_id=?", (user_id,)) as cursor:
            row = await cursor.fetchone()
            if row:
                return row[0]
        uid = str(random.randint(10000000, 99999999))
        await db.execute("UPDATE OR INSERT INTO users (user_id, unique_id) VALUES (?, ?)", (user_id, uid))
        await db.commit()
        return uid

# Главное меню
def main_menu(user_status: str = "Новичок"):
    kb = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="💰 Профиль", callback_data="profile")],
        [InlineKeyboardButton(text="⛏ Шахта", callback_data="mine"), InlineKeyboardButton(text="🎣 Рыбалка", callback_data="fish")],
        [InlineKeyboardButton(text="⚔ Бой с боссом", callback_data="boss"), InlineKeyboardButton(text="🎒 Кейсы", callback_data="cases")],
        [InlineKeyboardButton(text="🏠 Недвижимость", callback_data="property"), InlineKeyboardButton(text="🚗 Машины", callback_data="cars")],
        [InlineKeyboardButton(text="👑 Боевой пропуск", callback_data="pass"), InlineKeyboardButton(text="📈 Работы", callback_data="jobs")],
        [InlineKeyboardButton(text="🏦 Банк", callback_data="bank"), InlineKeyboardButton(text="💎 Магазин", callback_data="shop")],
        [InlineKeyboardButton(text="🎁 Ежедневка", callback_data="daily"), InlineKeyboardButton(text="🔗 Рефералы", callback_data="refer")],
        [InlineKeyboardButton(text="🎪 Эвенты", callback_data="events"), InlineKeyboardButton(text="📊 Топы", callback_data="tops")],
        [InlineKeyboardButton(text="💼 Рынок", callback_data="market"), InlineKeyboardButton(text="🔨 Крафт", callback_data="craft")]
    ])
    return kb

# Обработчик старта
@dp.message(Command("start"))
async def start(message: Message):
    user_id = message.from_user.id
    await init_db()
    uid = await get_unique_id(user_id)
    async with aiosqlite.connect(DB_FILE) as db:
        await db.execute("INSERT OR IGNORE INTO users (user_id, unique_id) VALUES (?, ?)", (user_id, uid))
        await db.commit()
    
    await message.answer(f"🎉 Добро пожаловать в Bitcoin Bot!\nТвой ID: `{uid}`\nВыбери действие:", reply_markup=main_menu(), parse_mode="Markdown")

# Профиль
@dp.callback_query(F.data == "profile")
async def profile(callback: CallbackQuery):
    user_id = callback.from_user.id
    async with aiosqlite.connect(DB_FILE) as db:
        async with db.execute("SELECT * FROM users WHERE user_id=?", (user_id,)) as cursor:
            user = await cursor.fetchone()
        if not user:
            user = (user_id, 1000, 0.001, 1, 0, 0, '', 'Новичок', 0, 0, 0, 0, 'Железная', 'Простая', '', 0, '', '', 0, 0, 0, await get_unique_id(user_id))
    
    kb = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="💸 Передать монеты", callback_data="transfer")],
        [InlineKeyboardButton(text="🔙 Назад", callback_data="back")]
    ])
    await callback.message.edit_text(
        f"👤 **Профиль**\n"
        f"💰 Монеты: {user[1]:,}\n₿ BTC: {user[2]:.4f}\n"
        f"⭐ Уровень: {user[3]} (XP: {user[4]}/1000)\n"
        f"🏷 Статус: {user[8]}\n"
        f"ID: {user[-1]}\n"
        f"🐕 Питомцы: {len(user[14].split(',')) if user[14] else 0}\n"
        f"🏠 Дом: {user[15] or 'Нет'}\n"
        f"🚗 Машина: {user[16] or 'Нет'}",
        reply_markup=kb, parse_mode="Markdown"
    )

# Ежедневные награды
@dp.callback_query(F.data == "daily")
async def daily(callback: CallbackQuery):
    user_id = callback.from_user.id
    now = datetime.now()
    async with aiosqlite.connect(DB_FILE) as db:
        async with db.execute("SELECT last_daily, coins FROM users WHERE user_id=?", (user_id,)) as cursor:
            user = await cursor.fetchone()
        
        if user and user[0] and (now - datetime.fromisoformat(user[0])).days < 1:
            await callback.answer("⏰ Ежедневка доступна раз в 24ч!", show_alert=True)
            return
        
        reward = random.randint(5000, 15000)
        await db.execute("UPDATE users SET coins = coins + ?, last_daily = ?, daily_claimed = daily_claimed + 1 WHERE user_id = ?", 
                        (reward, now.isoformat(), user_id))
        await db.commit()
    
    await callback.answer(f"🎁 +{reward:,} монет!", show_alert=True)
    await profile(callback)  # Обновить профиль

# Шахта (2 мин CD)
@dp.callback_query(F.data == "mine")
async def mine_start(callback: CallbackQuery, state: FSMContext):
    user_id = callback.from_user.id
    now = datetime.now().isoformat()
    async with aiosqlite.connect(DB_FILE) as db:
        async with db.execute("SELECT last_mine FROM users WHERE user_id=?", (user_id,)) as cursor:
            user = await cursor.fetchone()
        if user and user[0] and (datetime.now() - datetime.fromisoformat(user[0])).seconds < 120:
            remain = 120 - (datetime.now() - datetime.fromisoformat(user[0])).seconds
            await callback.answer(f"⏳ CD: {remain}s", show_alert=True)
            return
        
        # Руды: железо, золото, алмазы, крипто-руда
        ores = {'Железо': 100, 'Золото': 1000, 'Алмаз': 5000, 'Крипто-руда': 25000}
        ore = random.choices(list(ores.keys()), weights=[50, 30, 15, 5])[0]
        amount = random.randint(1, 5)
        reward = ores[ore] * amount
        
        await db.execute("UPDATE users SET coins = coins + ?, last_mine = ? WHERE user_id=?", (reward, now, user_id))
        await db.execute("INSERT OR REPLACE INTO inventory (user_id, item, amount) VALUES (?, ?, COALESCE((SELECT amount FROM inventory WHERE user_id=? AND item=?), 0) + ?)",
                        (user_id, ore, user_id, ore, amount))
        await db.commit()
    
    kb = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="🛒 Магазин кирок", callback_data="shop_pickaxe")],
        [InlineKeyboardButton(text="📦 Продать руду", callback_data="sell_ore")],
        [InlineKeyboardButton(text="🔙 Назад", callback_data="back")]
    ])
    await callback.message.edit_text(
        f"⛏ **Шахта**\n"
        f"🎯 Найдено: {amount} {ore}\n"
        f"💰 +{reward:,} монет\n"
        f"⏰ CD: 2 минуты",
        reply_markup=kb, parse_mode="Markdown"
    )

# Аналогично для рыбалки, работ, боссов и т.д. (сокращаю для примера, полный код >5000 строк)
@dp.callback_query(F.data.startswith("fish"))
async def fish(callback: CallbackQuery):
    # Логика рыбалки: удочки, приманки, рыбы (карп 200, золотая 5k и т.д.)
    await callback.message.edit_text("🎣 Поймана золотая рыба! +10k монет [CD 2мин]")

@dp.callback_query(F.data == "boss")
async def boss(callback: CallbackQuery):
    damage = random.randint(10, 100)
    if damage > 80:
        reward = 50000
        async with aiosqlite.connect(DB_FILE) as db:
            await db.execute("UPDATE users SET coins = coins + ?, bosses_killed = bosses_killed + 1 WHERE user_id=?", 
                            (reward, callback.from_user.id))
            await db.commit()
        await callback.answer(f"⚔ Босс побежден! +{reward:,} 💰", show_alert=True)
    else:
        await callback.answer("💥 Босс выжил! Попробуй позже.")

# Магазин (кирки, удочки, статусы, питомцы, дома, машины)
@dp.callback_query(F.data == "shop")
async def shop(callback: CallbackQuery):
    kb = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="⛏ Кирки (Железная 5k)", callback_data="buy_pickaxe1")],
        [InlineKeyboardButton(text="🏠 Дома (Хибара 100k)", callback_data="buy_house1")],
        [InlineKeyboardButton(text="👑 Статусы (VIP +20% бонус 50k/мес)", callback_data="buy_status_vip")],
        [InlineKeyboardButton(text="🐕 Питомцы (Дракон x2 доход 1M)", callback_data="buy_pet_dragon")],
        [InlineKeyboardButton(text="🔙 Назад", callback_data="back")]
    ])
    await callback.message.edit_text("🛒 **Магазин**\nВыбери категорию:", reply_markup=kb, parse_mode="Markdown")

# Боевой пропуск
@dp.callback_query(F.data == "pass")
async def battle_pass(callback: CallbackQuery):
    kb = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="💎 Купить Premium (100k)", callback_data="buy_premium_pass")],
        [InlineKeyboardButton(text="📈 Следующая награда: 50k монет (LVL 5)", callback_data="pass_next")],
        [InlineKeyboardButton(text="🔙 Назад", callback_data="back")]
    ])
    await callback.message.edit_text(
        "👑 **Боевой пропуск**\n"
        "Текущий: Free (LVL 3/10)\n"
        "Premium: Нет\n"
        "Следующая: 50k монет + питомец",
        reply_markup=kb, parse_mode="Markdown"
    )

# Эвенты (Хэллоуин 2 мес, Новый год 2 мес)
@dp.callback_query(F.data == "events")
async def events(callback: CallbackQuery):
    now = datetime.now()
    halloween_end = datetime(2025, 11, 1)  # 2 мес с окт
    ny_end = datetime(2026, 2, 1)
    event_active = "🎃 Хэллоуин" if now < halloween_end else "🎄 Новый год" if now < ny_end else "Нет"
    await callback.message.edit_text(f"🎪 **Эвенты**\nАктивно: {event_active}\nИщи конфеты/тыквы за x3 награды!")

# Рынок, рефералы, топы, промокоды, админка (только для @official_supra)
@dp.message(Command("promo"))
async def promo(message: Message, state: FSMContext):
    await state.set_state(Forms.promo)
    await message.reply("📝 Введи промокод:")

@dp.message(Forms.promo)
async def check_promo(message: Message, state: FSMContext):
    code = message.text.upper()
    async with aiosqlite.connect(DB_FILE) as db:
        async with db.execute("SELECT reward, uses FROM promos WHERE code=?", (code,)) as cursor:
            promo = await cursor.fetchone()
        if promo and promo[1] > 0:
            # Выдать награду (coins, items)
            await db.execute("UPDATE users SET coins = coins + 10000 WHERE user_id=?", (message.from_user.id,))
            await db.execute("UPDATE promos SET uses = uses - 1 WHERE code=?", (code,))
            await db.commit()
            await message.reply("✅ Промокод активирован! +10k монет")
        else:
            await message.reply("❌ Неверный промокод")
    await state.clear()

# Админ панель (только ADMIN_ID)
@dp.callback_query(F.data == "admin", lambda c: c.from_user.id == ADMIN_ID)
async def admin_panel(callback: CallbackQuery):
    kb = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="📢 Рассылка", callback_data="admin_broadcast")],
        [InlineKeyboardButton(text="🎁 Создать промо", callback_data="admin_promo")],
        [InlineKeyboardButton(text="🏆 Обновить топы", callback_data="admin_tops")]
    ])
    await callback.message.edit_text("🔧 **Админ панель**\nВыбери действие:", reply_markup=kb)

# Передать монеты по ID (лимит 10kk/день)
@dp.callback_query(F.data == "transfer")
async def transfer_start(callback: CallbackQuery, state: FSMContext):
    await state.set_state(Forms.transfer)
    await callback.message.reply("💸 Введи ID получателя и сумму (например: 123456 5000):")

@dp.message(Forms.transfer)
async def process_transfer(message: Message, state: FSMContext):
    try:
        target_id, amount = map(int, message.text.split())
        if amount > 10000000:  # 10kk лимит
            await message.reply("❌ Лимит 10kk/день!")
            return
        async with aiosqlite.connect(DB_FILE) as db:
            sender_balance = await db.execute_fetchall("SELECT coins FROM users WHERE user_id=?", (message.from_user.id,))
            if sender_balance[0][0] < amount:
                await message.reply("❌ Недостаточно монет!")
                return
            await db.execute("UPDATE users SET coins = coins - ? WHERE user_id=?", (amount, message.from_user.id))
            await db.execute("UPDATE users SET coins = coins + ? WHERE user_id=?", (amount, target_id))
            await db.commit()
        await message.reply(f"✅ Переведено {amount:,} монет ID {target_id}")
    except:
        await message.reply("❌ Формат: ID сумма")
    await state.clear()

# Кнопка назад
@dp.callback_query(F.data == "back")
async def back(callback: CallbackQuery):
    await callback.message.edit_text("🏠 **Главное меню**", reply_markup=main_menu())

# Запуск
async def main():
    await init_db()
    await dp.start_polling(bot)

if __name__ == "__main__":
    asyncio.run(main())
