from telegram import Update, ReplyKeyboardMarkup, ReplyKeyboardRemove
from telegram.ext import Application, CommandHandler, MessageHandler, filters, ContextTypes, ConversationHandler
import logging
import datetime
import sqlite3
import re
from typing import Dict, Any

# Logging sozlamalari
logging.basicConfig(
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    level=logging.INFO
)
logger = logging.getLogger(__name__)

# Bot tokeni
BOT_TOKEN = "8492930144:AAFL_IOzDvd5JTpR4EFKr_y12BSaTZ9r6iM"

# Conversation holatlarini belgilash
NAME, SURNAME, PHONE, BIRTHDATE, EMAIL, CONFIRMATION = range(6)

# Adminlar ro'yxati (o'zingizning ID'ingizni qo'shing)
ADMIN_IDS = [123456789]  # O'z Telegram ID'ingizni qo'shing

# Ma'lumotlar bazasini yaratish
def init_database():
    conn = sqlite3.connect('green_card_applications.db')
    cursor = conn.cursor()
    cursor.execute('''
    CREATE TABLE IF NOT EXISTS applications (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        user_id INTEGER,
        username TEXT,
        first_name TEXT,
        last_name TEXT,
        phone TEXT,
        birthdate TEXT,
        email TEXT,
        status TEXT DEFAULT 'pending',
        application_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    )
    ''')
    conn.commit()
    conn.close()

# Ma'lumotlarni bazaga saqlash
def save_to_database(user_data: Dict[str, Any], user_id: int, username: str, first_name: str):
    conn = sqlite3.connect('green_card_applications.db')
    cursor = conn.cursor()
    cursor.execute('''
    INSERT INTO applications (user_id, username, first_name, last_name, phone, birthdate, email)
    VALUES (?, ?, ?, ?, ?, ?, ?)
    ''', (
        user_id, 
        username, 
        first_name, 
        user_data['surname'], 
        user_data['phone'], 
        user_data['birthdate'],
        user_data.get('email', '')
    ))
    conn.commit()
    conn.close()

# Telefon raqamini tekshirish
def validate_phone(phone: str) -> bool:
    pattern = r'^\+998\d{9}$'
    return re.match(pattern, phone) is not None

# Email manzilini tekshirish
def validate_email(email: str) -> bool:
    pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
    return re.match(pattern, email) is not None

# Sana formatini tekshirish
def validate_date(date_str: str) -> bool:
    try:
        day, month, year = map(int, date_str.split('.'))
        if not (1 <= day <= 31 and 1 <= month <= 12 and 1900 <= year <= datetime.datetime.now().year):
            return False
        datetime.date(year, month, day)
        return True
    except ValueError:
        return False

# Yoshni hisoblash
def calculate_age(birthdate_str: str) -> int:
    day, month, year = map(int, birthdate_str.split('.'))
    birthdate = datetime.date(year, month, day)
    today = datetime.date.today()
    age = today.year - birthdate.year - ((today.month, today.day) < (birthdate.month, birthdate.day))
    return age

# /start komandasi
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.message.from_user
    await update.message.reply_text(
        f"👋 Assalomu alaykum {user.first_name}!\n\n"
        "🎯 Green Card arizasi uchun ma'lumotlaringizni yig'ishga yordam beraman.\n\n"
        "📝 Iltimos, ismingizni kiriting:"
    )
    return NAME

# Ismni qayta ishlash
async def get_name(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.message.from_user
    name = update.message.text.strip()
    
    if len(name) < 2:
        await update.message.reply_text("❌ Iltimos, to'g'ri ism kiriting (kamida 2 belgi):")
        return NAME
    
    context.user_data['name'] = name
    logger.info("%s ismi: %s", user.first_name, name)
    await update.message.reply_text("👥 Familyangizni kiriting:")
    return SURNAME

# Familyani qayta ishlash
async def get_surname(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.message.from_user
    surname = update.message.text.strip()
    
    if len(surname) < 2:
        await update.message.reply_text("❌ Iltimos, to'g'ri familya kiriting (kamida 2 belgi):")
        return SURNAME
    
    context.user_data['surname'] = surname
    logger.info("%s familyasi: %s", user.first_name, surname)
    await update.message.reply_text(
        "📞 Telefon raqamingizni kiriting:\n\n"
        "📋 Format: +998XXXXXXXXX\n"
        "Masalan: +998901234567"
    )
    return PHONE

# Telefon raqamini qayta ishlash
async def get_phone(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.message.from_user
    phone = update.message.text.strip()
    
    # Telefon raqamini tekshirish
    if not validate_phone(phone):
        await update.message.reply_text(
            "❌ Iltimos, telefon raqamingizni to'g'ri formatda kiriting:\n\n"
            "📋 Format: +998XXXXXXXXX\n"
            "Masalan: +998901234567"
        )
        return PHONE
    
    context.user_data['phone'] = phone
    logger.info("%s telefon raqami: %s", user.first_name, phone)
    await update.message.reply_text(
        "🎂 Tug'ilgan sanangizni kiriting:\n\n"
        "📋 Format: KK.OO.YYYY\n"
        "Masalan: 15.08.1990"
    )
    return BIRTHDATE

# Tug'ilgan sanani qayta ishlash
async def get_birthdate(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.message.from_user
    birthdate_str = update.message.text.strip()
    
    # Sana formatini tekshirish
    if not validate_date(birthdate_str):
        await update.message.reply_text(
            "❌ Iltimos, to'g'ri sana formatida kiriting:\n\n"
            "📋 Format: KK.OO.YYYY\n"
            "Masalan: 15.08.1990"
        )
        return BIRTHDATE
    
    # Yoshni tekshirish (kamida 18 yosh)
    age = calculate_age(birthdate_str)
    if age < 18:
        await update.message.reply_text(
            "❌ Siz Green Card arizasi uchun juda yoshsiz.\n\n"
            "📋 Kamida 18 yosh bo'lishingiz kerak."
        )
        return ConversationHandler.END
    
    context.user_data['birthdate'] = birthdate_str
    logger.info("%s tug'ilgan sanasi: %s", user.first_name, birthdate_str)
    
    await update.message.reply_text(
        "📧 Email manzilingizni kiriting (ixtiyoriy):\n\n"
        "Agar email manzilingiz bo'lmasa, 'O'tkazib yuborish' tugmasini bosing.",
        reply_markup=ReplyKeyboardMarkup(
            [['Oʻtkazib yuborish']], 
            one_time_keyboard=True,
            resize_keyboard=True
        )
    )
    return EMAIL

# Email manzilini qayta ishlash
async def get_email(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.message.from_user
    email_input = update.message.text.strip()
    
    if email_input == "Oʻtkazib yuborish":
        context.user_data['email'] = ""
    else:
        if not validate_email(email_input):
            await update.message.reply_text(
                "❌ Iltimos, to'g'ri email manzil kiriting:\n\n"
                "Masalan: example@mail.com\n\n"
                "Yoki 'O'tkazib yuborish' tugmasini bosing.",
                reply_markup=ReplyKeyboardMarkup(
                    [['Oʻtkazib yuborish']], 
                    one_time_keyboard=True,
                    resize_keyboard=True
                )
            )
            return EMAIL
        context.user_data['email'] = email_input
    
    logger.info("%s email manzili: %s", user.first_name, context.user_data.get('email', 'Yoʻq'))
    
    # Ma'lumotlarni tasdiqlash
    confirmation_message = (
        "🔍 Iltimos, kiritgan ma'lumotlaringizni tekshiring:\n\n"
        f"👤 Ism: {context.user_data['name']}\n"
        f"👥 Familya: {context.user_data['surname']}\n"
        f"📞 Telefon: {context.user_data['phone']}\n"
        f"🎂 Tug'ilgan sana: {context.user_data['birthdate']}\n"
        f"📧 Email: {context.user_data.get('email', 'Koʻrsatilmagan')}\n\n"
        "Ma'lumotlar to'g'ri bo'lsa '✅ Tasdiqlash' ni bosing, aks holda '❌ Qayta kiritish' ni bosing."
    )
    
    reply_keyboard = [['✅ Tasdiqlash', '❌ Qayta kiritish']]
    await update.message.reply_text(
        confirmation_message,
        reply_markup=ReplyKeyboardMarkup(
            reply_keyboard, 
            one_time_keyboard=True,
            resize_keyboard=True,
            input_field_placeholder='Maʼlumotlar toʻgʻrimi?'
        )
    )
    
    return CONFIRMATION

# Ma'lumotlarni tasdiqlash
async def confirmation(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.message.from_user
    choice = update.message.text
    
    if choice == '✅ Tasdiqlash':
        # Ma'lumotlarni bazaga saqlash
        save_to_database(
            context.user_data, 
            user.id, 
            user.username, 
            user.first_name
        )
        
        # Adminlarga xabar yuborish
        await notify_admins(context, user, context.user_data)
        
        await update.message.reply_text(
            "✅ Ma'lumotlaringiz muvaffaqiyatli qabul qilindi!\n\n"
            "📋 Green Card arizangiz ko'rib chiqiladi.\n"
            "⏳ Aloqada bo'ling, tez orada siz bilan bog'lanamiz.\n\n"
            "🔄 Yana ariza topshirmoqchi bo'lsangiz /start buyrug'ini yuboring.",
            reply_markup=ReplyKeyboardRemove()
        )
        logger.info("%s ma'lumotlari bazaga saqlandi", user.first_name)
    else:
        await update.message.reply_text(
            "❌ Ma'lumotlaringiz bekor qilindi.\n\n"
            "🔄 Qayta ariza topshirmoqchi bo'lsangiz /start buyrug'ini yuboring.",
            reply_markup=ReplyKeyboardRemove()
        )
        logger.info("%s ma'lumotlari bekor qilindi", user.first_name)
    
    return ConversationHandler.END

# Adminlarga bildirishnoma yuborish
async def notify_admins(context: ContextTypes.DEFAULT_TYPE, user: Any, user_data: Dict[str, Any]):
    notification_text = (
        "🆕 Yangi Green Card arizasi!\n\n"
        f"👤 Foydalanuvchi: {user.first_name} {user_data['surname']}\n"
        f"📞 Telefon: {user_data['phone']}\n"
        f"🎂 Tug'ilgan sana: {user_data['birthdate']}\n"
        f"📧 Email: {user_data.get('email', 'Koʻrsatilmagan')}\n"
        f"🆔 User ID: {user.id}\n"
        f"👤 Username: @{user.username if user.username else 'Yoʻq'}"
    )
    
    for admin_id in ADMIN_IDS:
        try:
            await context.bot.send_message(chat_id=admin_id, text=notification_text)
        except Exception as e:
            logger.error("Adminga xabar yuborishda xatolik: %s", e)

# Jarayonni bekor qilish
async def cancel(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.message.from_user
    logger.info("%s foydalanuvchi jarayonni bekor qildi.", user.first_name)
    await update.message.reply_text(
        "❌ Jarayon bekor qilindi.\n\n"
        "🔄 Ariza topshirmoqchi bo'lsangiz /start buyrug'ini yuboring.",
        reply_markup=ReplyKeyboardRemove()
    )
    return ConversationHandler.END

# Admin uchun ma'lumotlarni ko'rish
async def show_applications(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.message.from_user
    
    # Faqat adminlar uchun
    if user.id not in ADMIN_IDS:
        await update.message.reply_text("❌ Sizda bu buyruqni ishlatish uchun ruxsat yo'q.")
        return
    
    try:
        conn = sqlite3.connect('green_card_applications.db')
        cursor = conn.cursor()
        cursor.execute('SELECT * FROM applications ORDER BY application_date DESC LIMIT 10')
        applications = cursor.fetchall()
        conn.close()
        
        if not applications:
            await update.message.reply_text("📭 Hozircha hech qanday ariza topilmadi.")
            return
        
        message = "📋 So'ngi 10 ta ariza:\n\n"
        for app in applications:
            status_emoji = "✅" if app[8] == 'approved' else "⏳" if app[8] == 'pending' else "❌"
            message += (
                f"{status_emoji} ID: {app[0]}\n"
                f"👤 Ism: {app[3]} {app[4]}\n"
                f"📞 Tel: {app[5]}\n"
                f"🎂 Sana: {app[6]}\n"
                f"📅 Qo'shilgan: {app[9]}\n"
                f"────────────────────\n\n"
            )
        
        # Telegram xabar chegarasi (4096 belgi)
        if len(message) > 4096:
            for x in range(0, len(message), 4096):
                await update.message.reply_text(message[x:x+4096])
        else:
            await update.message.reply_text(message)
            
    except Exception as e:
        await update.message.reply_text(f"❌ Xatolik yuz berdi: {str(e)}")

# Yordam buyrug'i
async def help_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    help_text = (
        "🤖 Green Card Bot Yordam\n\n"
        "📋 Buyruqlar ro'yxati:\n"
        "/start - Yangi ariza topshirish\n"
        "/help - Yordak ko'rsatish\n"
        "/applications - Ariza ro'yxati (faqat adminlar uchun)\n\n"
        "📞 Aloqa: Agar qandaydir muammo bo'lsa, admin bilan bog'laning."
    )
    await update.message.reply_text(help_text)

# Asosiy funksiya
def main():
    # Ma'lumotlar bazasini ishga tushirish
    init_database()
    
    # Botni yaratish
    application = Application.builder().token(BOT_TOKEN).build()
    
    # ConversationHandler yaratish
    conv_handler = ConversationHandler(
        entry_points=[CommandHandler('start', start)],
        states={
            NAME: [MessageHandler(filters.TEXT & ~filters.COMMAND, get_name)],
            SURNAME: [MessageHandler(filters.TEXT & ~filters.COMMAND, get_surname)],
            PHONE: [MessageHandler(filters.TEXT & ~filters.COMMAND, get_phone)],
            BIRTHDATE: [MessageHandler(filters.TEXT & ~filters.COMMAND, get_birthdate)],
            EMAIL: [MessageHandler(filters.TEXT & ~filters.COMMAND, get_email)],
            CONFIRMATION: [MessageHandler(filters.Regex('^(✅ Tasdiqlash|❌ Qayta kiritish)$'), confirmation)],
        },
        fallbacks=[CommandHandler('cancel', cancel)],
    )
    
    # Handlerlarni qo'shish
    application.add_handler(conv_handler)
    application.add_handler(CommandHandler('applications', show_applications))
    application.add_handler(CommandHandler('help', help_command))
    application.add_handler(CommandHandler('start', start))
    
    # Botni ishga tushirish
    print("🤖 Bot ishga tushdi...")
    application.run_polling()

if __name__ == '__main__':
    main()
