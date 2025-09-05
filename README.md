# Telegram-bot
A telegram bot for useful stuff
bot.py
from telegram import Update
from telegram.ext import Application, CommandHandler, MessageHandler, filters, ContextTypes
import os, json

TOKEN = os.getenv("BOT_TOKEN")  # safer: keep token in environment variable
DATA_FILE = "files.json"

if os.path.exists(DATA_FILE):
    with open(DATA_FILE, "r") as f:
        FILES = json.load(f)
else:
    FILES = {}

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("📂 Welcome! Use:\n/save folder_name → then send file\n/get folder_name → get files\n/folders → list folders")

async def save_file(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if len(context.args) == 0:
        await update.message.reply_text("❗Usage: /save folder_name (then send file)")
        return
    folder = context.args[0]
    context.user_data["save_folder"] = folder
    await update.message.reply_text(f"✅ Send file to save in **{folder}**")

async def handle_file(update: Update, context: ContextTypes.DEFAULT_TYPE):
    folder = context.user_data.get("save_folder")
    if not folder:
        await update.message.reply_text("❗Use /save folder_name before sending a file.")
        return
    file_id = None
    if update.message.document:
        file_id = update.message.document.file_id
    elif update.message.photo:
        file_id = update.message.photo[-1].file_id
    elif update.message.video:
        file_id = update.message.video.file_id
    if file_id:
        FILES.setdefault(folder, []).append(file_id)
        with open(DATA_FILE, "w") as f:
            json.dump(FILES, f)
        await update.message.reply_text(f"📌 File saved in: {folder}")

async def get_files(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if len(context.args) == 0:
        await update.message.reply_text("❗Usage: /get folder_name")
        return
    folder = context.args[0]
    if folder not in FILES or not FILES[folder]:
        await update.message.reply_text("⚠ No files found in this folder.")
        return
    for file_id in FILES[folder]:
        await update.message.reply_document(file_id)

async def list_folders(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not FILES:
        await update.message.reply_text("📂 No folders yet.")
        return
    folders = "\n".join([f"- {f} ({len(FILES[f])} files)" for f in FILES])
    await update.message.reply_text(f"📁 Your folders:\n\n{folders}")

def main():
    app = Application.builder().token(TOKEN).build()
    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("save", save_file))
    app.add_handler(CommandHandler("get", get_files))
    app.add_handler(CommandHandler("folders", list_folders))
    app.add_handler(MessageHandler(filters.Document.ALL | filters.PHOTO | filters.VIDEO, handle_file))
    app.run_polling()

if __name__ == "__main__":
    main()
