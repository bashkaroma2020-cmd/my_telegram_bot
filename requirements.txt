import os
import logging
import asyncio
import asyncpg
from aiogram import Bot, Dispatcher, types, F
from aiogram.filters import Command

logging.basicConfig(level=logging.INFO)

BOT_TOKEN = os.getenv("BOT_TOKEN")
DATABASE_URL = os.getenv("DATABASE_URL")
PROVIDER_TOKEN = os.getenv("PROVIDER_TOKEN")

bot = Bot(token=BOT_TOKEN)
dp = Dispatcher()
db_pool = None

async def init_db():
    global db_pool
    try:
        db_pool = await asyncpg.create_pool(DATABASE_URL)
        async with db_pool.acquire() as conn:
            await conn.execute('''
                CREATE TABLE IF NOT EXISTS users (
                    user_id BIGINT PRIMARY KEY,
                    username VARCHAR(100),
                    is_premium BOOLEAN DEFAULT FALSE,
                    joined_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
                );
            ''')
            await conn.execute('''
                CREATE TABLE IF NOT EXISTS payments (
                    id SERIAL PRIMARY KEY,
                    user_id BIGINT,
                    amount INT,
                    status VARCHAR(50),
                    charge_id VARCHAR(100),
                    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
                );
            ''')
        logging.info("Database initialized successfully.")
    except Exception as e:
        logging.error(f"Database connection error: {e}")

@dp.message(Command("start"))
async def cmd_start(message: types.Message):
    user_id = message.from_user.id
    username = message.from_user.username

    async with db_pool.acquire() as conn:
        await conn.execute('''
            INSERT INTO users (user_id, username)
            VALUES ($1, $2)
            ON CONFLICT (user_id) DO NOTHING;
        ''', user_id, username)

    await message.answer(f"Привіт, {message.from_user.first_name}! Бот готовий до роботи. Натисніть /buy для підписки.")

@dp.message(Command("buy"))
async def cmd_buy(message: types.Message):
    if not PROVIDER_TOKEN:
        await message.answer("Помилка налаштування платежів.")
        return

    await bot.send_invoice(
        chat_id=message.chat.id,
        title="Преміум підписка",
        description="Доступ до функцій монетизації",
        payload="premium_pack",
        provider_token=PROVIDER_TOKEN,
        currency="UAH",
        prices=[types.LabeledPrice(label="Преміум", amount=10000)],
        start_parameter="premium-pay"
    )

@dp.pre_checkout_query()
async def process_pre_checkout_query(pre_checkout_query: types.PreCheckoutQuery):
    await bot.answer_pre_checkout_query(pre_checkout_query.id, ok=True)

@dp.message(F.successful_payment)
async def process_successful_payment(message: types.Message):
    payment_info = message.successful_payment
    user_id = message.from_user.id
    amount = payment_info.total_amount // 100

    async with db_pool.acquire() as conn:
        await conn.execute('UPDATE users SET is_premium = TRUE WHERE user_id = $1;', user_id)
        await conn.execute('''
            INSERT INTO payments (user_id, amount, status, charge_id)
            VALUES ($1, $2, $3, $4);
        ''', user_id, amount, "SUCCESS", payment_info.telegram_payment_charge_id)

    await message.answer(f"Дякуємо! Активовано Преміум статус на {amount} грн.")

async def main():
    await init_db()
    await dp.start_polling(bot)

if __name__ == "__main__":
    asyncio.run(main())

