# Driver-pos
from aiogram import Bot, Dispatcher, types
from aiogram.types import ParseMode
from aiogram.utils import executor
import sqlite3
import os
from datetime import datetime

# توکن ربات تلگرام
TOKEN = os.getenv("7616744306:AAE9cWJ8eszoaQvze3QMnokb8nyxyP0UGUI")

bot = Bot(token=TOKEN)
dp = Dispatcher(bot)

# اتصال به پایگاه داده SQLite
conn = sqlite3.connect('driver_payments.db')
c = conn.cursor()

# ایجاد جدول برای ذخیره اطلاعات پرداخت‌ها
c.execute('''
    CREATE TABLE IF NOT EXISTS payments (
        id INTEGER PRIMARY KEY,
        driver_name TEXT,
        amount REAL,
        shipment_number TEXT,
        transaction_id TEXT,
        financial_reference TEXT,
        payment_date TEXT
    )
''')
conn.commit()

# دریافت اطلاعات از کاربر
@dp.message_handler(commands=['start'])
async def start(message: types.Message):
    await message.reply("سلام! لطفاً اطلاعات زیر را وارد کنید:\n\n1️⃣ نام راننده\n2️⃣ مبلغ پرداخت\n3️⃣ شماره بارنامه\n4️⃣ شماره پیگیری تراکنش\n5️⃣ ارجاع به واحد مالی")

@dp.message_handler(commands=['add_payment'])
async def add_payment(message: types.Message):
    await message.reply("لطفاً نام راننده را وارد کنید.")
    await dp.message_handler(lambda msg: msg.text, state='driver_name')

@dp.message_handler(state='driver_name')
async def get_driver_name(message: types.Message):
    driver_name = message.text
    await message.reply(f"نام راننده: {driver_name}\n لطفاً مبلغ پرداخت را وارد کنید.")
    await dp.message_handler(lambda msg: msg.text, state='amount')

@dp.message_handler(state='amount')
async def get_amount(message: types.Message):
    amount = message.text
    await message.reply(f"مبلغ پرداخت: {amount}\n لطفاً شماره بارنامه را وارد کنید.")
    await dp.message_handler(lambda msg: msg.text, state='shipment_number')

@dp.message_handler(state='shipment_number')
async def get_shipment_number(message: types.Message):
    shipment_number = message.text
    await message.reply(f"شماره بارنامه: {shipment_number}\n لطفاً شماره پیگیری تراکنش را وارد کنید.")
    await dp.message_handler(lambda msg: msg.text, state='transaction_id')

@dp.message_handler(state='transaction_id')
async def get_transaction_id(message: types.Message):
    transaction_id = message.text
    await message.reply(f"شماره پیگیری تراکنش: {transaction_id}\n لطفاً ارجاع به واحد مالی را وارد کنید.")
    await dp.message_handler(lambda msg: msg.text, state='financial_reference')

@dp.message_handler(state='financial_reference')
async def get_financial_reference(message: types.Message):
    financial_reference = message.text
    payment_date = datetime.now().strftime("%Y-%m-%d")  # گرفتن تاریخ امروز
    c.execute('''
        INSERT INTO payments (driver_name, amount, shipment_number, transaction_id, financial_reference, payment_date)
        VALUES (?, ?, ?, ?, ?, ?)
    ''', (driver_name, float(amount), shipment_number, transaction_id, financial_reference, payment_date))
    conn.commit()
    await message.reply("✅ اطلاعات پرداخت ثبت شد.\nبرای مشاهده تاریخچه پرداخت‌ها، دستور /show_payments را وارد کنید.")

@dp.message_handler(commands=['show_payments'])
async def show_payments(message: types.Message):
    c.execute('SELECT * FROM payments')
    payments = c.fetchall()
    if payments:
        text = "تاریخچه پرداخت‌ها:\n\n"
        for row in payments:
            text += f"راننده: {row[1]}, مبلغ: {row[2]}, شماره بارنامه: {row[3]}, شماره پیگیری تراکنش: {row[4]}, ارجاع به واحد مالی: {row[5]}, تاریخ: {row[6]}\n\n"
        await message.reply(text, parse_mode=ParseMode.MARKDOWN)
    else:
        await message.reply("هیچ پرداختی ثبت نشده است.")

@dp.message_handler(commands=['daily_report'])
async def daily_report(message: types.Message):
    today = datetime.now().strftime("%Y-%m-%d")  # تاریخ امروز
    c.execute('SELECT SUM(amount) FROM payments WHERE payment_date = ?', (today,))
    total_amount = c.fetchone()[0]
    if total_amount:
        await message.reply(f"مجموع مبالغ پرداختی امروز ({today}): {total_amount} تومان.")
    else:
        await message.reply(f"هیچ پرداختی برای تاریخ امروز ({today}) ثبت نشده است.")

if __name__ == '__main__':
    executor.start_polling(dp, skip_updates=True)

conn.close()
