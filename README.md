from telegram import Update
from telegram.ext import Updater, CommandHandler, MessageHandler, Filters, CallbackContext
import datetime
import pytz

# بيانات الحضور: {user_id: {"name": "الاسم", "days": [تاريخ1, تاريخ2,...]}}
attendance = {}
TOKEN = "7763244132:AAHsa6HFWQR32FZ3EOeK4Ab0swwhR06eLt4"
GROUP_ID = -4604511408  # آيدي مجموعتك

def start(update: Update, context: CallbackContext):
    update.message.reply_text("""
    🌙 بوت 40 يوم مع الفجر:
    - اكتب 'توثيق' لتسجيل حضورك اليوم.
    - /stats : إحصائياتك الشخصية.
    - /group_stats : إحصائيات المجموعة (للمشرفين).
    """)

def remind_daily(context: CallbackContext):
    context.bot.send_message(
        chat_id=GROUP_ID,
        text="⏰ تذكير جماعي: صلاة الفجر بعد 15 دقيقة! اكتب 'توثيق' عند الوصول للمسجد."
    )

def record_attendance(update: Update, context: CallbackContext):
    if update.message.chat.id != GROUP_ID:
        return  # يقتصر التوثيق على الجروب فقط

    user_id = update.message.from_user.id
    user_name = update.message.from_user.first_name
    today = datetime.datetime.now(pytz.timezone('Asia/Riyadh')).date().isoformat()

    if user_id not in attendance:
        attendance[user_id] = {"name": user_name, "days": []}
    
    if today in attendance[user_id]["days"]:
        update.message.reply_text(f"⚠️ {user_name}، لقد سجلت حضورك اليوم مسبقًا!")
    else:
        attendance[user_id]["days"].append(today)
        update.message.reply_text(f"✅ تم تسجيل حضور {user_name} اليوم! (إجمالي الأيام: {len(attendance[user_id]['days']})")

def get_stats(update: Update, context: CallbackContext):
    user_id = update.message.from_user.id
    if user_id in attendance:
        update.message.reply_text(
            f"📊 {attendance[user_id]['name']}، حضورك: {len(attendance[user_id]['days'])} يوم من 40 يوم!"
        )
    else:
        update.message.reply_text("⚠️ لم تسجل حضورك بعد.")

def get_group_stats(update: Update, context: CallbackContext):
    if update.message.chat.id == GROUP_ID:
        stats_text = "🏆 إحصائيات المجموعة:\n"
        sorted_users = sorted(attendance.items(), key=lambda x: len(x[1]["days"]), reverse=True)
        
        for user_id, data in sorted_users:
            stats_text += f"- {data['name']}: {len(data['days'])} يوم\n"
        
        update.message.reply_text(stats_text)
    else:
        update.message.reply_text("⛔ هذا الأمر متاح فقط في الجروب الرئيسي.")

def handle_message(update: Update, context: CallbackContext):
    if update.message.chat.id == GROUP_ID and update.message.text.lower() == "توثيق":
        record_attendance(update, context)

def error_handler(update: Update, context: CallbackContext):
    print(f"⚠️ خطأ: {context.error}")

def main():
    updater = Updater(TOKEN)
    dp = updater.dispatcher

    # handlers
    dp.add_handler(CommandHandler("start", start))
    dp.add_handler(CommandHandler("stats", get_stats))
    dp.add_handler(CommandHandler("group_stats", get_group_stats))
    dp.add_handler(MessageHandler(Filters.text & ~Filters.command, handle_message))

    # جدولة التذكير
    job_queue = updater.job_queue
    job_queue.run_daily(
        remind_daily,
        time=datetime.time(3, 45, 0, tzinfo=pytz.timezone('Asia/Riyadh')),
        days=(0, 1, 2, 3, 4, 5, 6)  # كل يوم
    )

    # معالجة الأخطاء
    dp.add_error_handler(error_handler)

    updater.start_polling()
    print("✅ البوت يعمل...")
    updater.idle()

if __name__ == '__main__':
    main()
    
