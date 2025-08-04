# Y2Kdreamkg-
import os
import asyncio
from aiogram import Bot, Dispatcher, types
from aiogram.utils.exceptions import TelegramAPIError
from aiogram.types import InlineKeyboardMarkup, InlineKeyboardButton
from apscheduler.schedulers.asyncio import AsyncIOScheduler
from dotenv import load_dotenv
from utils import parse_products_from_category, calculate_markup, usd_to_kgs

load_dotenv()

BOT_TOKEN = os.getenv("BOT_TOKEN")
CHANNEL_USERNAME = os.getenv("CHANNEL_USERNAME")
OWNER_USERNAME = os.getenv("OWNER_USERNAME")
USD_TO_KGS_RATE = float(os.getenv("USD_TO_KGS", "92.5"))
POSTS_PER_HOUR = int(os.getenv("POSTS_PER_HOUR", "2"))

bot = Bot(token=BOT_TOKEN)
dp = Dispatcher(bot)

# Ссылки на категории
CATEGORIES = {
    "women": "https://y2kdream.com/collections/women",
    "men": "https://y2kdream.com/collections/men",
    "shoes": "https://y2kdream.com/collections/shoes"
}

posted_products = set()  # для хранения уже опубликованных товаров


async def post_product(product):
    title = product["title"]
    price_usd = calculate_markup(product["price_usd"])
    price_kgs = usd_to_kgs(price_usd, USD_TO_KGS_RATE)
    image_url = product["image_url"]

    text = (
        f"🔥 {title}\n"
        f"💸 Цена: ${price_usd} / ≈ {price_kgs} KGS\n\n"
        f"👇 Заказать:\n"
        f"🔗 {OWNER_USERNAME}\n\n"
        f"#Y2K #мода #одежда"
    )

    keyboard = InlineKeyboardMarkup()
    keyboard.add(InlineKeyboardButton("🛒 Купить", url=f"https://t.me/{OWNER_USERNAME.strip('@')}"))

    try:
        await bot.send_photo(
            chat_id=CHANNEL_USERNAME,
            photo=image_url,
            caption=text,
            reply_markup=keyboard,
            parse_mode=types.ParseMode.HTML
        )
        posted_products.add(title)
        print(f"Опубликовано: {title}")
    except TelegramAPIError as e:
        print(f"Ошибка публикации: {e}")


async def scheduled_posting():
    while True:
        try:
            for category, url in CATEGORIES.items():
                products = parse_products_from_category(url, max_items=10)
                for product in products:
                    if product["title"] not in posted_products:
                        await post_product(product)
                        # Публикуем 2 товара в час → интервал 30 мин между постами
                        await asyncio.sleep(1800 // POSTS_PER_HOUR)
        except Exception as e:
            print(f"Ошибка в планировщике: {e}")
        await asyncio.sleep(60)  # Ждём минуту перед следующей итерацией


if __name__ == "__main__":
    scheduler = AsyncIOScheduler()
    scheduler.add_job(lambda: asyncio.create_task(scheduled_posting()), "interval", minutes=30)
    scheduler.start()

    print("Бот запущен...")
    asyncio.run(dp.start_polling())
