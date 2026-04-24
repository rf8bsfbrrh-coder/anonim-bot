# anonim-bot
import asyncio
import os
import logging
from datetime import datetime, timezone
from html import escape

from aiogram import Bot, Dispatcher, F
from aiogram.client.default import DefaultBotProperties
from aiogram.enums import ParseMode
from aiogram.filters import Command
from aiogram.fsm.context import FSMContext
from aiogram.fsm.state import State, StatesGroup
from aiogram.fsm.storage.memory import MemoryStorage
from aiogram.types import (
    KeyboardButton,
    Message,
    ReplyKeyboardMarkup,
    ReplyKeyboardRemove,
)

logging.basicConfig(level=logging.INFO)
log = logging.getLogger(__name__)

BOT_TOKEN = os.environ.get("TELEGRAM_BOT_TOKEN")
ADMIN_ID = int(os.environ.get("ADMIN_TELEGRAM_ID", "0"))

if not BOT_TOKEN:
    raise RuntimeError("TELEGRAM_BOT_TOKEN is not set.")
if not ADMIN_ID:
    raise RuntimeError("ADMIN_TELEGRAM_ID is not set.")

COOLDOWN_MINUTES = 10
log.info(f"Admin ID: {ADMIN_ID}")

bot = Bot(token=BOT_TOKEN, default=DefaultBotProperties(parse_mode=ParseMode.HTML))
dp = Dispatcher(storage=MemoryStorage())

cooldowns: dict[int, datetime] = {}
message_counts: dict[int, int] = {}
first_seen: dict[int, datetime] = {}
last_active: dict[int, datetime] = {}
reply_map: dict[int, int] = {}
total_messages: int = 0
bot_username: str = ""


class Form(StatesGroup):
    writing = State()


main_keyboard = ReplyKeyboardMarkup(
    keyboard=[
        [KeyboardButton(text="✉️ Написать анонимно")],
        [KeyboardButton(text="📊 Мой статус")],
    ],
    resize_keyboard=True,
    input_field_placeholder="Выбери действие...",
)

cancel_keyboard = ReplyKeyboardMarkup(
    keyboard=[[KeyboardButton(text="❌ Отмена")]],
    resize_keyboard=True,
    input_field_placeholder="Напиши сообщение и отправь...",
)


def cooldown_remaining(user_id: int) -> int | None:
    last = cooldowns.get(user_id)
    if last is None:
        return None
    elapsed = (datetime.now(timezone.utc) - last).total_seconds()
    remaining = COOLDOWN_MINUTES * 60 - elapsed
    return int(remaining) if remaining > 0 else None


def fmt_time(dt: datetime) -> str:
    return dt.strftime("%H:%M · %d.%m.%Y")


def msg_type_label(message: Message) -> str:
    if message.text:        return "💬 Текст"
    if message.photo:       return "🖼 Фото"
    if message.video:       return "🎬 Видео"
    if message.voice:       return "🎤 Голосовое"
    if message.audio:       return "🎵 Аудио"
    if message.video_note:  return "⭕ Видеосообщение"
    if message.document:    return "📎 Файл"
    if message.sticker:     return f"😄 Стикер {message.sticker.emoji or ''}"
    if message.animation:   return "🎞 GIF"
    if message.location:    return "📍 Геолокация"
    if message.contact:     return "👤 Контакт"
    return "📦 Другое"


def build_sender_card(user, message: Message, count: int, msg_number: int) -> str:
    name = escape(user.first_name or "")
    if user.last_name:
        name += f" {escape(user.last_name)}"

    now = datetime.now(timezone.utc)
    premium = "  ⭐ <b>Premium</b>" if getattr(user, "is_premium", False) else ""
    lang = getattr(user, "language_code", None)
    lang_line = f"🌍  {lang.upper()}" if lang else ""
    username_line = f"🔗  @{escape(user.username)}" if user.username else ""
    is_new = count == 1

    prev = last_active.get(user.id)
    if prev and not is_new:
        delta = now - prev
        h, rem = divmod(int(delta.total_seconds()), 3600)
        m = rem // 60
        since = f"{h} ч {m} мин назад" if h > 0 else f"{m} мин назад"
        activity = f"⏱  Был активен: {since}"
    else:
        activity = ""

    badge = "🆕  <i>Пишет впервые</i>" if is_new else f"💬  Написал уже <b>{count}</b> раз"

    lines = [
        f"┌{'─' * 24}┐",
        f"│       👤  <b>ОТПРАВИТЕЛЬ</b>        │",
        f"└{'─' * 24}┘",
        f"",
        f"🧑  <b>{name}</b>{premium}",
        f"🆔  <code>{user.id}</code>",
    ]
    if username_line:
        lines.append(username_line)
    if lang_line:
        lines.append(lang_line)
    lines += [f"", badge]
    if activity:
        lines.append(activity)
    lines += [
        f"🕐  {fmt_time(now)} UTC",
        f"📋  {msg_type_label(message)}",
    ]
    return "\n".join(lines)


def build_footer() -> str:
    if bot_username:
        return (
            f"\n\n·  ·  ·  ·  ·  ·  ·  ·  ·  ·  ·  ·\n"
            f'✦  <a href="https://t.me/{bot_username}">напиши свой пост</a>  ✦'
        )
    return ""


async def forward_to_admin(user, message: Message, count: int, msg_number: int) -> list[int]:
    sender_card = build_sender_card(user, message, count, msg_number)
    footer = build_footer()
    title = (
        f"◈  ──────────────────  ◈\n"
        f"       📬  <b>АНОНИМНОЕ СООБЩЕНИЕ</b>\n"
        f"◈  ──────────────────  ◈\n\n"
    )
    sent_ids = []

    try:
        m1 = await bot.send_message(ADMIN_ID, sender_card)
        sent_ids.append(m1.message_id)

        if message.text:
            m2 = await bot.send_message(ADMIN_ID, title + escape(message.text) + footer)
            sent_ids.append(m2.message_id)
        elif message.photo:
            m2 = await bot.send_photo(ADMIN_ID, message.photo[-1].file_id, caption=title[:-2] + footer)
            sent_ids.append(m2.message_id)
        elif message.video:
            m2 = await bot.send_video(ADMIN_ID, message.video.file_id, caption=title[:-2] + footer)
            sent_ids.append(m2.message_id)
        elif message.voice:
            m2 = await bot.send_voice(ADMIN_ID, message.voice.file_id, caption=title[:-2] + footer)
            sent_ids.append(m2.message_id)
        elif message.audio:
            m2 = await bot.send_audio(ADMIN_ID, message.audio.file_id, caption=title[:-2] + footer)
            sent_ids.append(m2.message_id)
        elif message.document:
            m2 = await bot.send_document(ADMIN_ID, message.document.file_id, caption=title[:-2] + footer)
            sent_ids.append(m2.message_id)
        elif message.animation:
            m2 = await bot.send_animation(ADMIN_ID, message.animation.file_id, caption=title[:-2] + footer)
            sent_ids.append(m2.message_id)
        else:
            m2 = await bot.copy_message(ADMIN_ID, message.chat.id, message.message_id)
            m3 = await bot.send_message(ADMIN_ID, title[:-2] + footer)
            sent_ids.extend([m2.message_id, m3.message_id])

    except Exception as e:
        log.error(f"forward_to_admin error: {e}")
        raise

    return sent_ids


@dp.message(Command("start"))
async def cmd_start(message: Message, state: FSMContext):
    await state.clear()
    user = message.from_user

    if user.id == ADMIN_ID:
        total = sum(message_counts.values())
        await message.answer(
            f"👑 <b>Панель владельца</b>\n\n"
            f"Сообщений получено: <b>{total}</b>\n"
            f"Уникальных пользователей: <b>{len(message_counts)}</b>\n\n"
            f"Нажми <b>Reply</b> на любое сообщение, чтобы ответить анонимно.",
            reply_markup=ReplyKeyboardRemove(),
        )
        return

    mention = f"@{user.username}" if user.username else user.first_name
    await message.answer(
        f"👋 Привет, {mention}!\n\n"
        f"Здесь ты можешь отправить анонимное сообщение.\n"
        f"Просто напиши что хочешь — и я передам его.\n\n"
        f"🔒 Твоё имя останется скрытым для всех.\n"
        f"⏳ После отправки нужно подождать <b>{COOLDOWN_MINUTES} минут</b> "
        f"перед следующим сообщением.",
        reply_markup=main_keyboard,
    )


@dp.message(Command("cancel"))
@dp.message(F.text == "❌ Отмена")
async def cmd_cancel(message: Message, state: FSMContext):
    if await state.get_state() == Form.writing:
        await state.clear()
        await message.answer("↩️ Отменено.", reply_markup=main_keyboard)
    else:
        await message.answer("Нечего отменять.", reply_markup=main_keyboard)


@dp.message(Command("status"))
@dp.message(F.text == "📊 Мой статус")
async def cmd_status(message: Message):
    user_id = message.from_user.id
    if user_id == ADMIN_ID:
        total = sum(message_counts.values())
        await message.answer(
            f"📊 <b>Статистика бота</b>\n\n"
            f"Уникальных пользователей: <b>{len(message_counts)}</b>\n"
            f"Всего сообщений: <b>{total}</b>"
        )
        return

    remaining = cooldown_remaining(user_id)
    count = message_counts.get(user_id, 0)
    if remaining:
        mins, secs = divmod(remaining, 60)
        await message.answer(
            f"⏳ <b>Следующее сообщение через {mins}м {secs}с</b>\n"
            f"Всего отправлено тобой: <b>{count}</b>",
            reply_markup=main_keyboard,
        )
    else:
        await message.answer(
            f"✅ <b>Можешь отправить сообщение прямо сейчас!</b>\n"
            f"Всего отправлено тобой: <b>{count}</b>",
            reply_markup=main_keyboard,
        )


@dp.message(Command("whoami"))
async def cmd_whoami(message: Message):
    u = message.from_user
    await message.answer(
        f"🪪 <b>Твои данные</b>\n\n"
        f"ID: <code>{u.id}</code>\n"
        f"Имя: {u.first_name}\n"
        f"Username: @{u.username or '—'}"
    )


@dp.message(F.text == "✉️ Написать анонимно")
async def btn_write(message: Message, state: FSMContext):
    if message.from_user.id == ADMIN_ID:
        return

    remaining = cooldown_remaining(message.from_user.id)
    if remaining is not None:
        mins, secs = divmod(remaining, 60)
        await message.answer(
            f"⏳ Подожди ещё <b>{mins}м {secs}с</b> перед следующим сообщением.",
            reply_markup=main_keyboard,
        )
        return

    await state.set_state(Form.writing)
    await message.answer(
        "✏️ <b>Напиши своё сообщение</b>\n\n"
        "Поддерживается текст, фото, видео, голосовые, стикеры и файлы.\n"
        "Нажми ❌ Отмена, чтобы передумать.",
        reply_markup=cancel_keyboard,
    )


@dp.message(Form.writing)
async def receive_message(message: Message, state: FSMContext):
    global total_messages
    user = message.from_user
    await state.clear()

    remaining = cooldown_remaining(user.id)
    if remaining is not None:
        mins, secs = divmod(remaining, 60)
        await message.answer(
            f"⏳ Подожди ещё <b>{mins}м {secs}с</b>.",
            reply_markup=main_keyboard,
        )
        return

    total_messages += 1
    count = message_counts.get(user.id, 0) + 1
    now = datetime.now(timezone.utc)

    if user.id not in first_seen:
        first_seen[user.id] = now
    last_active[user.id] = cooldowns.get(user.id)

    message_counts[user.id] = count
    cooldowns[user.id] = now

    try:
        sent_ids = await forward_to_admin(user, message, count, total_messages)
        for mid in sent_ids:
            reply_map[mid] = user.id

        await message.answer(
            f"✅ <b>Сообщение #{total_messages} отправлено анонимно!</b>\n\n"
            f"⏳ Следующее через <b>{COOLDOWN_MINUTES} минут</b>.",
            reply_markup=main_keyboard,
        )
    except Exception as e:
        cooldowns.pop(user.id, None)
        message_counts[user.id] = count - 1
        total_messages -= 1
        await message.answer("❌ Не удалось отправить. Попробуй позже.", reply_markup=main_keyboard)
        log.error(f"Failed to forward message: {e}")


@dp.message(F.chat.func(lambda c: c.id == ADMIN_ID) & F.reply_to_message)
async def admin_reply(message: Message):
    replied_id = message.reply_to_message.message_id
    target_user_id = reply_map.get(replied_id)

    if not target_user_id:
        await message.answer("⚠️ Не могу найти отправителя этого сообщения.")
        return

    try:
        await bot.send_message(target_user_id, "💬 <b>Вам ответили анонимно:</b>")
        await bot.copy_message(
            chat_id=target_user_id,
            from_chat_id=message.chat.id,
            message_id=message.message_id,
        )
        await message.answer("✅ Ответ отправлен.")
    except Exception as e:
        await message.answer(f"❌ Не удалось: {e}")
        log.error(f"Admin reply failed: {e}")


@dp.message()
async def fallback(message: Message):
    if message.from_user.id == ADMIN_ID:
        return
    await message.answer(
        "Нажми кнопку ниже, чтобы отправить анонимное сообщение.",
        reply_markup=main_keyboard,
    )


async def main():
    global bot_username
    me = await bot.get_me()
    bot_username = me.username
    log.info(f"Bot started as @{bot_username}")
    await dp.start_polling(bot)


if __name__ == "__main__":
    asyncio.run(main())
