import logging
import re
import os
from flask import Flask
from threading import Thread
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import Application, CommandHandler, ContextTypes, CallbackQueryHandler

# Logging configuration
logging.basicConfig(format='%(asctime)s - %(name)s - %(levelname)s - %(message)s', level=logging.INFO)

# 🔴 PUT YOUR TELEGRAM BOT TOKEN HERE BETWEEN THE QUOTES
TOKEN = "8767024318:AAFWSf4hh0986o4tUzSR5agHaaQX9K5kvd4"

# In-memory database
db = {}

# Flask web server for Render keep-alive
app = Flask('')

@app.route('/')
def home():
    return "Bot is running 24/7!"

def run_flask():
    app.run(host='0.0.0.0', port=int(os.environ.get('PORT', 8080)))

def keep_alive():
    t = Thread(target=run_flask)
    t.start()

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    help_text = (
        "✈️ **Welcome to the Standby & Flight Hours Bot**\n\n"
        "**How to register your shift:**\n"
        "Type the command like this:\n"
        "`/set from day 4 06:00 to 18:00 hours 65`\n\n"
        "**To hide your hours from others, add 'hidden' at the end:**\n"
        "`/set from day 4 06:00 to 18:00 hours 65 hidden`\n\n"
        "**Other Commands:**\n"
        "/current - Display the standby crew list sorted by hours and grouped by days.\n"
        "/clear - Delete specific day or all data (Admins only)."
    )
    try:
        await update.message.reply_text(help_text, parse_mode="Markdown")
    except Exception:
        pass

async def set_standby(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user
    user_id = user.id
    username = user.username or user.first_name
    original_msg = update.message
    
    text = " ".join(context.args).lower().strip()

    if not text:
        try:
            await original_msg.delete()
            await context.bot.send_message(chat_id=user_id, text="❌ Please provide details with the `/set` command.")
        except Exception:
            pass
        return

    is_hidden = "hidden" in text

    day_match = re.search(r'day\s*(\d+)', text)
    times = re.findall(r'(\d{1,2}:\d{2})', text)
    hours_match = re.search(r'(?:hours|hour)\s*(\d+)', text)

    if not hours_match:
        hours_match = re.search(r'(?:hours|hour)(\d+)', text)
    if not day_match:
        day_match = re.search(r'day(\d+)', text)

    if day_match and len(times) >= 2 and hours_match:
        day_num = int(day_match.group(1))
        start_time = times[0]
        end_time = times[1]
        hours = int(hours_match.group(1))

        if day_num not in db:
            db[day_num] = {}
        
        db[day_num][user_id] = {
            "username": username,
            "shift": f"from {start_time} to {end_time}",
            "hours": hours,
            "hidden": is_hidden
        }

        privacy_msg = " (Hours are Hidden 🔒)" if is_hidden else " (Hours are Visible 👁)"
        await original_msg.reply_text(f"✅ Captain @{username}, your shift for **Day {day_num}** has been registered successfully.{privacy_msg}", parse_mode="Markdown")
    
    else:
        try:
            await original_msg.delete()
            error_instruction = (
                f"❌ **Hello Captain @{username}, your previous command in the group had a typo.**\n\n"
                f"To protect your privacy, the bot deleted your message from the group.\n"
                f"Please copy, correct, and re-send the command below directly here or in the group:\n\n"
                f"`/set from day {day_match.group(1) if day_match else '4'} 06:00 to 18:00 hours {hours_match.group(1) if hours_match else '65'}{' hidden' if is_hidden else ''}`"
            )
            await context.bot.send_message(chat_id=user_id, text=error_instruction, parse_mode="Markdown")
        except Exception:
            await original_msg.reply_text(f"⚠️ @{username} Command format error. Message deleted for privacy. Please check your command format.", parse_mode="Markdown")

async def current_list(update: Update, context: ContextTypes.DEFAULT_TYPE):
    has_data = False
    if db:
        for d in db:
            if db[d]:
                has_data = True
                break

    if not has_data:
        await update.message.reply_text("😴 The standby list is currently empty.")
        return

    response = "📋 **Standby Crew & Monthly Flight Hours List:**\n"
    response += "⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯\n\n"

    for day in sorted(db.keys()):
        if not db[day]:
            continue
        
        response += f"📅 **DAY {day}**\n"
        sorted_members = sorted(db[day].values(), key=lambda x: x.get('hours', 0), reverse=True)
        
        for member in sorted_members:
            hours_disp = "Private 🔒" if member.get('hidden', False) else f"{member.get('hours', 0)} hours"
            response += (
                f"👨‍✈️ **@{member.get('username', 'Unknown')}**\n"
                f"⏱ Shift: {member.get('shift', 'N/A')}\n"
                f"📊 Current Hours: {hours_disp}\n"
                f"⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯\n"
            )
        response += "\n"

    await update.message.reply_text(response, parse_mode="Markdown")

async def clear_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    chat_member = await context.bot.get_chat_member(update.effective_chat.id, update.effective_user.id)
    if chat_member.status not in ['creator', 'administrator']:
        await update.message.reply_text("❌ This command is restricted to Group Administrators only.")
        return

    has_data = False
    if db:
        for d in db:
            if db[d]:
                has_data = True
                break

    if not has_data:
        await update.message.reply_text("🧹 The database is already empty.")
        return

    keyboard = []
    for day in sorted(db.keys()):
        if db[day]:
            keyboard.append([InlineKeyboardButton(f"Clear Day {day}", callback_query_data=f"clear_day_{day}")])
    
    keyboard.append([InlineKeyboardButton("Clear All Days ⚠️", callback_query_data="clear_all")])
    reply_markup = InlineKeyboardMarkup(keyboard)
    
    await update.message.reply_text("Select which day you want to clear from the standby list:", reply_markup=reply_markup)

async def button_click(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    
    chat_member = await context.bot.get_chat_member(update.effective_chat.id, query.from_user.id)
    if chat_member.status not in ['creator', 'administrator']:
        return

    data = query.data

    if data == "clear_all":
        db.clear()
        await query.edit_message_text("🧹 Entire standby database has been completely wiped.")
    elif data.startswith("clear_day_"):
        target_day = int(data.replace("clear_day_", ""))
        if target_day in db:
            db[target_day].clear()
            await query.edit_message_text(f"✅ All entries for **Day {target_day}** have been cleared.", parse_mode="Markdown")

def main():
    # Start Web Server for Render
    keep_alive()
    
    app_bot = Application.builder().token(TOKEN).build()
    
    app_bot.add_handler(CommandHandler("start", start))
    app_bot.add_handler(CommandHandler("set", set_standby))
    app_bot.add_handler(CommandHandler("current", current_list))
    app_bot.add_handler(CommandHandler("clear", clear_command))
    app_bot.add_handler(CallbackQueryHandler(button_click))
    
    print("🚀 Bot is running flawlessly on Render...")
    app_bot.run_polling()

if __name__ == '__main__':
    main()
