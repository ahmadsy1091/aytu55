import telebot
from telebot.types import InlineKeyboardMarkup, InlineKeyboardButton
import os

# استبدال هذا بالتوكن الخاص بك
TOKEN = "YOUR_TELEGRAM_BOT_TOKEN"
ADMIN_IDS = [123456789]  # أرقام المسؤولين

bot = telebot.TeleBot(TOKEN)

# بيانات البوت (يمكن تخزينها في قاعدة بيانات)
user_data = {}
admin_data = {
    'sirital_cash_number': "123456789",
    'prices': {
        'free_fire': 100,
        'pubg': 150
    }
}

# لوحة المفاتيح الرئيسية
def main_menu():
    markup = InlineKeyboardMarkup()
    markup.row_width = 2
    markup.add(
        InlineKeyboardButton("شحن الألعاب", callback_data="charge_games"),
        InlineKeyboardButton("حسابي", callback_data="my_account"),
        InlineKeyboardButton("الدعم والمساعدة", callback_data="support")
    )
    return markup

# قائمة الألعاب
def games_menu():
    markup = InlineKeyboardMarkup()
    markup.row_width = 1
    markup.add(
        InlineKeyboardButton("فري فاير", callback_data="free_fire"),
        InlineKeyboardButton("ببجي", callback_data="pubg"),
        InlineKeyboardButton("🔙 رجوع", callback_data="main_menu")
    )
    return markup

# قائمة الإدارة
def admin_menu():
    markup = InlineKeyboardMarkup()
    markup.row_width = 1
    markup.add(
        InlineKeyboardButton("تغير رقم محفظة سيرياتيل كاش", callback_data="change_sirital"),
        InlineKeyboardButton("تغير سعر المنتج", callback_data="change_price"),
        InlineKeyboardButton("معرفات المنتجات", callback_data="product_ids"),
        InlineKeyboardButton("🔙 رجوع", callback_data="main_menu")
    )
    return markup

# بدء البوت
@bot.message_handler(commands=['start', 'help'])
def send_welcome(message):
    bot.send_message(message.chat.id, "مرحباً بك في بوت الشحن!", reply_markup=main_menu())

    # إظهار قائمة الإدارة للمسؤولين
    if message.from_user.id in ADMIN_IDS:
        bot.send_message(message.chat.id, "قائمة الإدارة:", reply_markup=admin_menu())

# معالجة الأزرار
@bot.callback_query_handler(func=lambda call: True)
def callback_query(call):
    if call.data == "main_menu":
        bot.edit_message_text(chat_id=call.message.chat.id, message_id=call.message.message_id, 
                             text="مرحباً بك في بوت الشحن!", reply_markup=main_menu())

    elif call.data == "charge_games":
        bot.edit_message_text(chat_id=call.message.chat.id, message_id=call.message.message_id, 
                             text="اختر اللعبة:", reply_markup=games_menu())

    elif call.data == "my_account":
        user_id = call.from_user.id
        credit = user_data.get(user_id, {}).get('credit', 0)
        bot.edit_message_text(chat_id=call.message.chat.id, message_id=call.message.message_id, 
                             text=f"رصيدك الحالي: {credit} كرديت", 
                             reply_markup=InlineKeyboardMarkup().add(
                                 InlineKeyboardButton("شحن رصيدي", callback_data="charge_balance")
                             ))

    elif call.data == "charge_balance":
        msg = f"💳 رقم محفظتنا سيرياتيل كاش:\n{admin_data['sirital_cash_number']}\n\n"
        msg += "📌 أرسل رقم العملية بعد التحويل\n"
        msg += "💰 كم المبلغ الذي أرسلته؟"
        bot.edit_message_text(chat_id=call.message.chat.id, message_id=call.message.message_id, 
                             text=msg)
        bot.register_next_step_handler(call.message, process_payment)

    elif call.data in ["free_fire", "pubg"]:
        game = call.data
        price = admin_data['prices'][game]
        bot.edit_message_text(chat_id=call.message.chat.id, message_id=call.message.message_id, 
                             text=f"سعر شحن {game}: {price} كرديت")

    elif call.data == "support":
        bot.edit_message_text(chat_id=call.message.chat.id, message_id=call.message.message_id, 
                             text="للتواصل مع الدعم الفني:\n@username")

    # أوامر الإدارة
    elif call.data == "change_sirital" and call.from_user.id in ADMIN_IDS:
        bot.edit_message_text(chat_id=call.message.chat.id, message_id=call.message.message_id, 
                             text="أرسل الرقم الجديد لمحفظة سيرياتيل كاش:")
        bot.register_next_step_handler(call.message, update_sirital_number)

    elif call.data == "change_price" and call.from_user.id in ADMIN_IDS:
        bot.edit_message_text(chat_id=call.message.chat.id, message_id=call.message.message_id, 
                             text="أرسل معرف المنتج والسعر الجديد (مثال: free_fire 150):")
        bot.register_next_step_handler(call.message, update_product_price)

    elif call.data == "product_ids" and call.from_user.id in ADMIN_IDS:
        products = "\n".join([f"{k}: {v}" for k, v in admin_data['prices'].items()])
        bot.edit_message_text(chat_id=call.message.chat.id, message_id=call.message.message_id, 
                             text=f"معرفات المنتجات:\n{products}")

# معالجة الدفع
def process_payment(message):
    try:
        amount = int(message.text)
        user_id = message.from_user.id
        if user_id not in user_data:
            user_data[user_id] = {'credit': 0}
        
        user_data[user_id]['credit'] += amount
        bot.send_message(message.chat.id, f"تم شحن {amount} كرديت بنجاح! رصيدك الآن: {user_data[user_id]['credit']} كرديت",
                         reply_markup=main_menu())
    except ValueError:
        bot.send_message(message.chat.id, "المبلغ يجب أن يكون رقماً! أعد المحاولة.")

# وظائف الإدارة
def update_sirital_number(message):
    if message.from_user.id in ADMIN_IDS:
        admin_data['sirital_cash_number'] = message.text
        bot.send_message(message.chat.id, "تم تحديث رقم المحفظة بنجاح!", reply_markup=admin_menu())

def update_product_price(message):
    if message.from_user.id in ADMIN_IDS:
        try:
            product_id, new_price = message.text.split()
            admin_data['prices'][product_id] = int(new_price)
            bot.send_message(message.chat.id, f"تم تحديث سعر {product_id} إلى {new_price} كرديت",
                           reply_markup=admin_menu())
        except:
            bot.send_message(message.chat.id, "صيغة غير صحيحة! مثال: free_fire 150")

# تشغيل البوت
bot.infinity_polling()
