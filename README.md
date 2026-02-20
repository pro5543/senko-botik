# senko-botik
import asyncio
import random
import sqlite3
import time
import os

from aiogram import Bot, Dispatcher
from aiogram.filters import Command, CommandStart
from aiogram.types import (
    Message,
    CallbackQuery,
    InlineKeyboardMarkup,
    InlineKeyboardButton,
    InputMediaPhoto,
)

# ================== НАСТРОЙКИ ==================

TOKEN = "BOT_TOKEN"
   # вставь НОВЫЙ токен (старый отозвать в BotFather)




ADMIN_IDS = {8133949767, 8337333717}
CHANNEL_USERNAME = "@SenkoFoxes"

CARD_COOLDOWN = 90  # сек
DB_PATH = "database.db"
INV_PAGE_SIZE = 20
# ===============================================

rarity_names = {
    "common": "⚪️ Обычная",
    "rare": "🟢 Редкая",
    "ultra": "🔵 Ультра редкая",
    "epic": "🟣 Эпическая",
    "legendary": "🟡 Легендарная",
    "chromatic": "🎴 ХРОМАТИЧЕСКАЯ",
}

# ---------- DB ----------
conn = sqlite3.connect(DB_PATH)
cursor = conn.cursor()

cursor.execute("""
CREATE TABLE IF NOT EXISTS users (
    user_id INTEGER PRIMARY KEY,
    balance INTEGER DEFAULT 0,
    last_card_time INTEGER DEFAULT 0
)
""")

cursor.execute("""
CREATE TABLE IF NOT EXISTS cards (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,
    rarity TEXT NOT NULL,
    description TEXT,
    image TEXT NOT NULL
)
""")

cursor.execute("""
CREATE TABLE IF NOT EXISTS inventory (
    user_id INTEGER,
    card_id INTEGER,
    amount INTEGER DEFAULT 1,
    PRIMARY KEY (user_id, card_id)
)
""")

cursor.execute("""
CREATE TABLE IF NOT EXISTS inv_pages (
    user_id INTEGER PRIMARY KEY,
    page INTEGER DEFAULT 0
)
""")

conn.commit()

# ---------- Bot ----------
bot = Bot(token=TOKEN)
dp = Dispatcher()


def roll_rarity() -> str:
    roll = random.uniform(0, 100)
    if roll <= 60:
        return "common"
    elif roll <= 83:
        return "rare"
    elif roll <= 93:
        return "ultra"
    elif roll <= 98:
        return "epic"
    elif roll <= 99.8:
        return "legendary"
    else:
        return "chromatic"


# ✅ /start с проверкой подписки
@dp.message(CommandStart())
async def start_handler(message: Message):
    user_id = message.from_user.id
    try:
        member = await bot.get_chat_member(CHANNEL_USERNAME, user_id)
        if member.status in ("member", "administrator", "creator"):
            cursor.execute("INSERT OR IGNORE INTO users (user_id) VALUES (?)", (user_id,))
            conn.commit()
            await message.answer("✅ Ты подписан и зарегистрирован!")
        else:
            await message.answer(
                "❌ Чтобы пользоваться ботом, подпишись на канал:\n"
                "https://t.me/SenkoFoxes"
            )
    except Exception:
        await message.answer(
            "❌ Бот не может проверить подписку.\n"
            "Убедись, что бот админ канала."
        )


# ✅ Добавление карточки: фото + подпись
# /addcard Название | common | Описание
@dp.message(Command("addcard"))
async def addcard_handler(message: Message):
    if message.from_user.id not in ADMIN_IDS:
        await message.answer("Нет доступа.")
        return

    if not message.photo:
        await message.answer("Пришли фото с подписью:\n/addcard Название | common | Описание")
        return

    caption = message.caption or ""
    args = caption.replace("/addcard", "", 1).strip()

    try:
        name, rarity, description = [p.strip() for p in args.split("|", 2)]
    except ValueError:
        await message.answer(
            "Формат: /addcard Название | common/rare/ultra/epic/legendary/chromatic | Описание"
        )
        return

    if rarity not in rarity_names:
        await message.answer("Неизвестная редкость. Используй: " + ", ".join(rarity_names.keys()))
        return

    file_id = message.photo[-1].file_id

    cursor.execute(
        "INSERT INTO cards (name, rarity, description, image) VALUES (?, ?, ?, ?)",
        (name, rarity, description, file_id),
    )
    conn.commit()

    await message.answer(f"✅ Добавлено: {name} — {rarity_names[rarity]}")


# ✅ Выдача карточки
@dp.message(Command("card"))
async def give_card(message: Message):
    user_id = message.from_user.id
    current_time = int(time.time())

    cursor.execute("SELECT last_card_time FROM users WHERE user_id = ?", (user_id,))
    user = cursor.fetchone()
    if not user:
        await message.answer("Сначала напиши /start")
        return

    last_time = user[0]
    if current_time - last_time < CARD_COOLDOWN:
        remaining = CARD_COOLDOWN - (current_time - last_time)
        h = remaining // 3600
        m = (remaining % 3600) // 60
        s = remaining % 60
        await message.answer(f"⏳ Новая карточка будет через:\n{h:02}:{m:02}:{s:02}")
        return

    rarity = roll_rarity()

    cursor.execute(
        "SELECT id, name, rarity, description, image FROM cards WHERE rarity = ? ORDER BY RANDOM() LIMIT 1",
        (rarity,),
    )
    card = cursor.fetchone()

    if not card:
        await message.answer("⚠️ Нет карточек этой редкости в базе.")
        return

    card_id, name, rarity, description, image = card

    cursor.execute(
        "UPDATE users SET last_card_time = ? WHERE user_id = ?",
        (current_time, user_id),
    )

    cursor.execute("""
        INSERT INTO inventory (user_id, card_id, amount)
        VALUES (?, ?, 1)
        ON CONFLICT(user_id, card_id) DO UPDATE SET amount = amount + 1
    """, (user_id, card_id))

    conn.commit()

    await message.answer_photo(
        photo=image,
        caption=(
            f"🎴 Ты получил карточку!\n\n"
            f"✨ {name}\n"
            f"Редкость: {rarity_names.get(rarity, rarity)}\n"
            f"📜 {description or ''}"
        ),
    )


# ---------- INVENTORY ----------
def inv_pages_kb(page: int) -> InlineKeyboardMarkup:
    return InlineKeyboardMarkup(inline_keyboard=[[
        InlineKeyboardButton(text="⬅️", callback_data=f"invpage:prev:{page}"),
        InlineKeyboardButton(text="🗑 Закрыть", callback_data="invpage:close"),
        InlineKeyboardButton(text="➡️", callback_data=f"invpage:next:{page}"),
    ]])


def get_inv_total_count(user_id: int) -> int:
    cursor.execute("SELECT COUNT(*) FROM inventory WHERE user_id = ?", (user_id,))
    return cursor.fetchone()[0]


def get_inv_page_rows(user_id: int, page: int):
    offset = page * INV_PAGE_SIZE
    cursor.execute("""
        SELECT c.id, c.name, c.rarity, i.amount
        FROM inventory i
        JOIN cards c ON c.id = i.card_id
        WHERE i.user_id = ?
        ORDER BY i.amount DESC, c.rarity ASC, c.name ASC
        LIMIT ? OFFSET ?
    """, (user_id, INV_PAGE_SIZE, offset))
    return cursor.fetchall()


async def render_inv_page(chat_id: int, user_id: int, page: int, edit_message: Message | None = None):
    total = get_inv_total_count(user_id)
    if total == 0:
        if edit_message:
            await edit_message.edit_text("Инвентарь пуст. Напиши /card", reply_markup=None)
        else:
            await bot.send_message(chat_id, "Инвентарь пуст. Напиши /card")
        return

    max_page = (total - 1) // INV_PAGE_SIZE
    rows = get_inv_page_rows(user_id, page)

    cursor.execute("""
        INSERT INTO inv_pages (user_id, page)
        VALUES (?, ?)
        ON CONFLICT(user_id) DO UPDATE SET page = excluded.page
    """, (user_id, page))
    conn.commit()

    text = f"🎒 Твой инвентарь — страница {page+1}/{max_page+1}\n\n"
    # номера убраны
    for cid, name, rarity, amount in rows:
        text += f"🆔{cid} • {rarity_names.get(rarity, rarity)} ×{amount} — {name}\n"

    if edit_message:
        await edit_message.edit_text(text, reply_markup=inv_pages_kb(page))
    else:
        await bot.send_message(chat_id, text, reply_markup=inv_pages_kb(page))


@dp.message(Command("inv"))
async def inv_handler(message: Message):
    user_id = message.from_user.id

    cursor.execute("SELECT 1 FROM users WHERE user_id = ?", (user_id,))
    if not cursor.fetchone():
        await message.answer("Сначала напиши /start")
        return

    cursor.execute("SELECT page FROM inv_pages WHERE user_id = ?", (user_id,))
    row = cursor.fetchone()
    page = row[0] if row else 0

    await render_inv_page(message.chat.id, user_id, page, edit_message=None)


@dp.callback_query(lambda c: c.data and c.data.startswith("invpage:"))
async def invpage_callbacks(call: CallbackQuery):
    user_id = call.from_user.id

    if call.data == "invpage:close":
        await call.message.delete()
        await call.answer()
        return

    _, action, page_str = call.data.split(":")
    page = int(page_str)
    new_page = page + 1 if action == "next" else page - 1

    total = get_inv_total_count(user_id)
    if total == 0:
        await call.message.edit_text("Инвентарь пуст. Напиши /card", reply_markup=None)
        await call.answer()
        return

    max_page = (total - 1) // INV_PAGE_SIZE
    if new_page < 0 or new_page > max_page:
        await call.answer("Больше нет карточек.")
        return

    await render_inv_page(call.message.chat.id, user_id, new_page, edit_message=call.message)
    await call.answer()


# ---------- LOOK ----------
def look_kb(card_id: int) -> InlineKeyboardMarkup:
    return InlineKeyboardMarkup(inline_keyboard=[[
        InlineKeyboardButton(text="⬅️", callback_data=f"look:prev:{card_id}"),
        InlineKeyboardButton(text="🗑 Закрыть", callback_data="look:close"),
        InlineKeyboardButton(text="➡️", callback_data=f"look:next:{card_id}"),
    ]])


def get_card_by_id(card_id: int):
    cursor.execute("""
        SELECT id, name, rarity, description, image
        FROM cards
        WHERE id = ?
    """, (card_id,))
    return cursor.fetchone()


def get_prev_card_id(card_id: int):
    cursor.execute("SELECT id FROM cards WHERE id < ? ORDER BY id DESC LIMIT 1", (card_id,))
    row = cursor.fetchone()
    if row:
        return row[0]
    cursor.execute("SELECT id FROM cards ORDER BY id DESC LIMIT 1")
    row = cursor.fetchone()
    return row[0] if row else None


def get_next_card_id(card_id: int):
    cursor.execute("SELECT id FROM cards WHERE id > ? ORDER BY id ASC LIMIT 1", (card_id,))
    row = cursor.fetchone()
    if row:
        return row[0]
    cursor.execute("SELECT id FROM cards ORDER BY id ASC LIMIT 1")
    row = cursor.fetchone()
    return row[0] if row else None


async def show_card(message_or_call, card_id: int, edit: bool = False):
    card = get_card_by_id(card_id)
    if not card:
        if hasattr(message_or_call, "answer"):
            await message_or_call.answer("Карточка с таким номером не найдена.")
        else:
            await bot.send_message(message_or_call.chat.id, "Карточка с таким номером не найдена.")
        return

    cid, name, rarity, description, image = card
    caption = (
        f"🆔 №{cid}\n"
        f"✨ {name}\n"
        f"Редкость: {rarity_names.get(rarity, rarity)}\n\n"
        f"{description or ''}"
    )

    if edit:
        try:
            media = InputMediaPhoto(media=image, caption=caption)
            await message_or_call.message.edit_media(media=media, reply_markup=look_kb(cid))
        except Exception:
            await bot.send_photo(message_or_call.message.chat.id, photo=image, caption=caption, reply_markup=look_kb(cid))
    else:
        await bot.send_photo(message_or_call.chat.id, photo=image, caption=caption, reply_markup=look_kb(cid))


@dp.message(Command("look"))
async def look_cmd(message: Message):
    args = (message.text or "").split(maxsplit=1)
    if len(args) < 2 or not args[1].isdigit():
        await message.answer("Формат: /look НОМЕР_КАРТОЧКИ\nПример: /look 12")
        return
    card_id = int(args[1])
    await show_card(message, card_id, edit=False)


@dp.callback_query(lambda c: c.data == "look:close" or (c.data and c.data.startswith("look:")))
async def look_callbacks(call: CallbackQuery):
    if call.data == "look:close":
        await call.message.delete()
        await call.answer()
        return

    parts = call.data.split(":")
    if len(parts) != 3:
        await call.answer()
        return

    action = parts[1]
    current_id = int(parts[2])

    if action == "prev":
        new_id = get_prev_card_id(current_id)
    else:
        new_id = get_next_card_id(current_id)

    if new_id is None:
        await call.answer("Карточек нет.")
        return

    await show_card(call, new_id, edit=True)
    await call.answer()











@dp.message(Command("lastcards"))
async def lastcards_handler(message: Message):
    if message.from_user.id not in ADMIN_IDS:
        await message.answer("Нет доступа.")
        return

    args = (message.text or "").split(maxsplit=1)
    n = int(args[1]) if len(args) > 1 and args[1].isdigit() else 10

    cursor.execute("""
        SELECT id, name, rarity
        FROM cards
        ORDER BY id DESC
        LIMIT ?
    """, (n,))
    rows = cursor.fetchall()

    if not rows:
        await message.answer("Карточек нет.")
        return

    text = "Последние карточки:\n\n"
    for cid, name, rarity in rows:
        text += f"🆔{cid} — {name} ({rarity_names.get(rarity, rarity)})\n"

    await message.answer(text)


@dp.message(Command("delcard"))
async def delcard_handler(message: Message):
    if message.from_user.id not in ADMIN_IDS:
        await message.answer("Нет доступа.")
        return

    args = (message.text or "").split(maxsplit=1)
    if len(args) < 2 or not args[1].isdigit():
        await message.answer("Формат: /delcard ID\nПример: /delcard 12")
        return

    card_id = int(args[1])

    # Проверка что карта существует
    cursor.execute("SELECT name, rarity FROM cards WHERE id = ?", (card_id,))
    row = cursor.fetchone()
    if not row:
        await message.answer("Карточка с таким ID не найдена.")
        return

    name, rarity = row

    # Чистим у пользователей, если уже выпадала
    cursor.execute("DELETE FROM inventory WHERE card_id = ?", (card_id,))
    # Удаляем саму карточку
    cursor.execute("DELETE FROM cards WHERE id = ?", (card_id,))
    conn.commit()

    await message.answer(f"🗑 Удалено: №{card_id} — {name} ({rarity_names.get(rarity, rarity)})")




# ---------- RUN ----------
async def main():
    await dp.start_polling(bot)


if __name__ == "__main__":
    asyncio.run(main())
