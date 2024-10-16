import logging
from datetime import datetime
import pytz
import re
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import ApplicationBuilder, CommandHandler, CallbackQueryHandler, ContextTypes, MessageHandler, filters

# Настройка логирования
logging.basicConfig(format='%(asctime)s - %(name)s - %(levelname)s - %(message)s', level=logging.INFO)
logger = logging.getLogger(__name__)

# Настройка логирования для httpx и telegram.ext.Application на уровне WARNING
logging.getLogger('httpx').setLevel(logging.WARNING)
logging.getLogger('telegram.ext.Application').setLevel(logging.WARNING)

users = {}
market = []
admins = ['ArtemKirss']
next_user_id = 0
page_size = 40

#Получаем время по Киеву
def get_current_time_kiev():
    kiev_tz = pytz.timezone('Europe/Kiev')
    now = datetime.now(kiev_tz)
    return now.strftime("%H:%M; %d/%m/%Y")

# Команда /start
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    global next_user_id
    username = update.effective_user.username
    telegram_user_id = update.effective_user.id

    if any(info['telegram_user_id'] == telegram_user_id for info in users.values()):
        await update.message.reply_text(f"Пользователь {username} уже зарегистрирован.")
        return

    user_id = next_user_id
    next_user_id += 1

    users[user_id] = {
        "username": username,
        "join_date": get_current_time_kiev(),
        "coins": 10,
        "items": ["😀"],
        "user_bot_id": user_id,
        "telegram_user_id": telegram_user_id
    }

    await update.message.reply_text(f"Добро пожаловать, {username}! Ты добавлен в систему. Твой ID: {user_id}")

#Форматируем инвентарь
def format_inventory(items):
    item_counts = {}
    for item in items:
        match = re.match(r'(\d+)?(.+)', item)
        if match:
            count = int(match.group(1)) if match.group(1) else 1
            emoji = match.group(2).strip()
            if emoji in item_counts:
                item_counts[emoji] += count
            else:
                item_counts[emoji] = count
        else:
            if item in item_counts:
                item_counts[item] += 1
            else:
                item_counts[item] = 1
    return "; ".join([f"({count}){emoji}" if count > 1 else emoji for emoji, count in item_counts.items()])

# Функция для генерации кнопок инвентаря с учетом страниц
def generate_inventory_buttons(items, page):
    start_index = page * page_size
    end_index = start_index + page_size
    items_on_page = items[start_index:end_index]

    keyboard = []
    for i in range(0, len(items_on_page), 5):
        keyboard.append([InlineKeyboardButton(item, callback_data=f'emoji_{item}') for item in items_on_page[i:i + 5]])

    total_pages = (len(items) - 1) // page_size + 1

    prev_button_text = '❌' if page == 0 else '⏪'
    next_button_text = '❌' if page >= total_pages - 1 else '⏩'

    nav_buttons = [
        InlineKeyboardButton(prev_button_text, callback_data='prev_page'),
        InlineKeyboardButton(f'Страница {page + 1}/{total_pages}', callback_data='current_page'),
        InlineKeyboardButton(next_button_text, callback_data='next_page')
    ]

    keyboard.append(nav_buttons)

    return InlineKeyboardMarkup(keyboard)

# Обработка нажатий кнопок навигации и выбора эмодзи
async def handle_inventory_button_click(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    query = update.callback_query
    await query.answer()

    user_id = context.user_data.get('user_id')
    page = context.user_data.get('page', 0)

    if user_id is not None and user_id in users:
        user_info = users[user_id]
        items = user_info['items']

    if query.data.startswith('emoji_'):
        selected_emoji = query.data.split('_')[1]
        if user_id in users:
            cleaned_emoji = re.sub(r'^\(\d+\)', '', selected_emoji).strip()
            users[user_id]['selected_emoji'] = cleaned_emoji

            coins = user_info.get('coins', 0)
            selected_emoji_display = users[user_id]['selected_emoji']

            inventory_string = format_inventory(user_info['items'])
            formatted_items = inventory_string.split("; ")
            keyboard = generate_inventory_buttons(formatted_items, page)

            await query.edit_message_text(
                text=f"Инвентарь {user_info['username']}\n"
                     f"Количество монет: {coins} 💲\n"
                     f"Выбранный эмодзи: {selected_emoji_display}",
                reply_markup=keyboard
            )
        return

    if query.data == 'next_page' and page < (len(items) - 1) // page_size:
        page += 1
    elif query.data == 'prev_page' and page > 0:
        page -= 1

    context.user_data['page'] = page

    inventory_string = format_inventory(user_info['items'])
    formatted_items = inventory_string.split("; ")
    keyboard = generate_inventory_buttons(formatted_items, page)

    await query.edit_message_reply_markup(reply_markup=keyboard)

# Команда /inventory для отображения инвентаря с кнопками
async def inventory(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    username = update.effective_user.username
    user_id = next((uid for uid, info in users.items() if username == info["username"]), None)

    if user_id is not None:
        user_info = users[user_id]
        context.user_data['user_id'] = user_id
        context.user_data['page'] = 0

        inventory_string = format_inventory(user_info['items'])

        formatted_items = inventory_string.split("; ")
        keyboard = generate_inventory_buttons(formatted_items, 0)

        coins = user_info.get('coins', 0)
        selected_emoji = user_info.get('selected_emoji', '😀')

        await update.message.reply_text(
            text=f"Инвентарь {user_info['username']}\n"
                 f"Количество монет: {coins} 💲\n"
                 f"Выбранный эмодзи: {selected_emoji}",
            reply_markup=keyboard
        )
    else:
        await update.message.reply_text("Ты не зарегистрирован. Используй /start для регистрации.")

# Обработка сообщений "чч" или "xx"
async def handle_message(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    text = update.message.text.lower()
    if text in ("чч", "xx"):
        username = update.effective_user.username
        user_id = next((uid for uid, info in users.items() if username == info["username"]), None)

        if user_id is not None:
            user_info = users[user_id]
            inventory_string = format_inventory(user_info["items"])
            coins = user_info.get('coins', 0)
            selected_emoji = user_info.get('selected_emoji', '😀')

            await update.message.reply_text(
                text=f"Инвентарь {user_info['username']}\n"
                     f"Количество монет: {coins} 💲\n"
                     f"Выбранный эмодзи: {selected_emoji}\n"
                     f"Инвентарь: {inventory_string}"
            )
        else:
            await update.message.reply_text("Ты не зарегистрирован. Используй /start для регистрации.")

    #elif text in ("cc","сс"):

# Команда /sail
async def sail(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    args = context.args
    if len(args) < 3:
        await update.message.reply_text("Неверный формат команды. Используйте /sail {эмодзи} {количество} {цена}.")
        return

    emoji = args[0]
    try:
        quantity = int(args[1])
        price = int(args[2])
    except ValueError:
        await update.message.reply_text("Количество и цена должны быть числами.")
        return

    telegram_user_id = update.effective_user.id
    username = update.effective_user.username

    user_info = next((info for info in users.values() if info.get('telegram_user_id') == telegram_user_id), None)

    if user_info is None:
        await update.message.reply_text("Ты не зарегистрирован. Используй /start для регистрации.")
        return

    items = user_info.get('items', [])

    for i, item in enumerate(items):
        match = re.match(r'^\((\d+)\)(.+)', item)
        if match:
            available_quantity = int(match.group(1))
            item_emoji = match.group(2).strip()
        else:
            available_quantity = items.count(emoji) if item.strip() == emoji else 0
            item_emoji = item.strip()

        if item_emoji == emoji:
            if available_quantity >= quantity:

                new_quantity = available_quantity - quantity
                if new_quantity > 1:
                    items[i] = f"({new_quantity}){emoji}"
                elif new_quantity == 1:
                    items[i] = emoji
                else:
                    items.pop(i)

                items = [item for item in items if not (item.strip() == emoji and items.count(item) > 1)]
                user_info['items'] = items

                market.append({
                    'seller_id': telegram_user_id,
                    'seller': username,
                    'emoji': emoji,
                    'quantity': quantity,
                    'price': price,
                })
                await update.message.reply_text(f"Вы выставили на продажу {quantity} {emoji} за {price} 💲.")
                return
            else:
                await update.message.reply_text("У тебя недостаточно эмодзи для продажи.")
                return

    await update.message.reply_text(f"У тебя нет {emoji} в инвентаре.")

# Обновление инвентаря
def update_inventory_after_sale(user_id, emoji, quantity):
    if user_id in users:
        items = users[user_id]['items']
        for i, item in enumerate(items):
            match = re.match(r'\((\d+)\)(.+)', item)
            if match:
                item_quantity = int(match.group(1))
                item_emoji = match.group(2).strip()

                if item_emoji == emoji:
                    new_quantity = item_quantity - quantity

                    if new_quantity > 0:
                        items[i] = f"({new_quantity}){item_emoji}"
                    else:
                        items.pop(i)
                    break

# Проверка на наличие емодзи
def has_enough_items(user_id, emoji, quantity):
    if user_id in users:
        items = users[user_id]['items']
        for item in items:
            match = re.match(r'\((\d+)\)(.+)', item)
            if match:
                item_quantity = int(match.group(1))
                item_emoji = match.group(2).strip()

                if item_emoji == emoji and item_quantity >= quantity:
                    return True
    return False

# Команда /market для просмотра рынка
async def market_view(update: Update, context: ContextTypes.DEFAULT_TYPE, page: int = 0):
    total_pages = (len(market) + page_size - 1) // page_size

    if not market:
        if update.callback_query:
            await update.callback_query.message.reply_text("Рынок пуст.")
        else:
            await update.message.reply_text("Рынок пуст.")
        return

    page = max(0, min(page, total_pages - 1))

    keyboard = []
    start = page * page_size
    end = min(start + page_size, len(market))

    for i in range(start, end):
        item = market[i]
        if i % 5 == 0:
            keyboard.append([])
        keyboard[-1].append(InlineKeyboardButton(
            f"{item['quantity']}{item['emoji']} - {item['price']}💲",
            callback_data=f"market_{i}"))

    navigation_buttons = []

    if page > 0:
        navigation_buttons.append(InlineKeyboardButton("⏪", callback_data="previous_market_page"))
    else:
        navigation_buttons.append(InlineKeyboardButton("❌", callback_data="no_action"))

    navigation_buttons.append(InlineKeyboardButton(f"{page + 1}/{total_pages}", callback_data="current_page"))

    if page < total_pages - 1:
        navigation_buttons.append(InlineKeyboardButton("⏩", callback_data="next_market_page"))
    else:
        navigation_buttons.append(InlineKeyboardButton("❌", callback_data="no_action"))

    keyboard.append(navigation_buttons)

    if update.callback_query:
        await update.callback_query.message.edit_text("Рынок:", reply_markup=InlineKeyboardMarkup(keyboard))
    else:
        await update.message.reply_text("Рынок:", reply_markup=InlineKeyboardMarkup(keyboard))

# Навигация в рынке
async def handle_market_navigation(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    current_page = context.user_data.get('market_page', 0)

    if query.data == "previous_market_page":
        current_page -= 1
    elif query.data == "next_market_page":
        current_page += 1

    context.user_data['market_page'] = current_page

    await market_view(update, context, page=current_page)
    await query.answer()

# Кнопки в рынке
async def handle_market_buttons(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    data = query.data

    if data.startswith('market_'):
        await handle_market_click(update, context)
    elif data in ["next_market_page", "prev_market_page"]:
        await handle_market_navigation(update, context)
    else:
        await update.message.reply_text("Неизвестное действие.")

# Обработка нажатия кнопки на рынке
async def handle_market_click(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()

    index = int(query.data.split('_')[1])

    if index < 0 or index >= len(market):
        item = market[index] if index < len(market) else {"emoji": "неизвестен", "price": "неизвестна"}
        await query.edit_message_text(f"Товар {item['emoji']} по цене {item['price']}💲 не найден.")
        await handle_market_navigation(update, context)
        return

    item = market[index]

    buyer_username = update.effective_user.username
    buyer_id = next((uid for uid, info in users.items() if buyer_username == info["username"]), None)

    if buyer_id is not None:
        buyer_info = users[buyer_id]
        seller_info = next(info for uid, info in users.items() if info["username"] == item['seller'])

        if buyer_info['coins'] < item['price']:
            await update.effective_user.send_message("Недостаточно средств для покупки.")
            return

        buyer_info['coins'] -= item['price']
        seller_info['coins'] += item['price']

        market.pop(index)

        emoji_entry = f"({item['quantity']}){item['emoji']}"

        existing_item = next((i for i in buyer_info['items'] if i.endswith(item['emoji'])), None)

        if existing_item:
            current_quantity = int(existing_item.split(')')[0][1:])
            new_quantity = current_quantity + item['quantity']
            buyer_info['items'].remove(existing_item)
            buyer_info['items'].append(f"({new_quantity}){item['emoji']}")
        else:
            buyer_info['items'].append(emoji_entry)

        message_text = (
            f"Вы успешно купили {item['quantity']} {item['emoji']} "
            f"за {item['price']}💲 у пользователя {seller_info['username']}."
        )
        await update.effective_user.send_message(message_text)

        await handle_market_navigation(update, context)
    else:
        await query.edit_message_text("Ты не зарегистрирован. Используй /start для регистрации.")

# Функция для добавления смайликов в инвентарь
async def add_emoji(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    emoji_input = context.args[0] if context.args else None
    if not emoji_input:
        await update.message.reply_text("Укажи смайлик после команды.")
        return

    match = re.match(r'(\d+)?(.+)', emoji_input)
    if match:
        count = int(match.group(1)) if match.group(1) else 1
        emoji = match.group(2)
    else:
        await update.message.reply_text("Неверный формат. Используй, например: /m 10☢️")
        return

    user_id = next((uid for uid, info in users.items() if update.effective_user.username == info["username"]), None)

    if user_id is not None:
        user_items = users[user_id]["items"]
        found = False

        for i in range(len(user_items)):
            match_item = re.match(r'(\d+)(.+)', user_items[i])
            if match_item and match_item.group(2) == emoji:
                current_count = int(match_item.group(1))
                user_items[i] = f"{current_count + count}{emoji}"
                found = True
                break

        if not found:
            user_items.append(f"{count}{emoji}" if count > 1 else emoji)

        await update.message.reply_text(f"{count} {emoji} добавлено в твой инвентарь!")
    else:
        await update.message.reply_text("Ты не зарегистрирован. Используй /start для регистрации.")

# Команда /get
async def get_users(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    file_path = 'users_info.txt'
    with open(file_path, 'w', encoding='utf-8') as file:
        for user_id, info in users.items():
            file.write(
                f"ID: {user_id}, Имя пользователя: {info['username']}, Дата захода: {info['join_date']}, "
                f"Баланс: {info['coins']} 💲, Инвентарь: {format_inventory(info['items'])}, "
                f"Telegram ID: {update.effective_user.id}\n"
            )

        file.write("\nРынок:\n")
        for index, item in enumerate(market):
            file.write(
                f"{index}: Эмодзи: {item['emoji']}, Количество: {item['quantity']}, "
                f"Цена: {item['price']}, Продавец: {item['seller']}\n"
            )

    with open(file_path, 'rb') as file:
        await update.message.reply_document(file, caption=f"Информация о пользователях и рынке в файле {file_path}")

# Команда /set
async def set_users(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    context.user_data['awaiting_file'] = True
    await update.message.reply_text("Пожалуйста, отправь файл с информацией о пользователях.")

# Обработка полученного файла
async def handle_document(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    if context.user_data.get('awaiting_file'):
        file = await context.bot.get_file(update.message.document.file_id)
        file_path = 'users_info.txt'
        await file.download_to_drive(file_path)

        with open(file_path, 'r', encoding='utf-8') as f:
            lines = f.readlines()

        users.clear()
        market.clear()
        in_market_section = False

        for line in lines:
            line = line.strip()

            if line.startswith("ID:"):
                parts = line.split(', ')
                user_id = int(parts[0].split(': ')[1])
                username = parts[1].split(': ')[1]
                join_date = parts[2].split(': ')[1]
                coins = int(parts[3].split(': ')[1].replace(' 💲', ''))
                items = parts[4].split(': ')[1].split('; ')
                telegram_id = int(parts[5].split(': ')[1])

                users[user_id] = {
                    "username": username,
                    "join_date": join_date,
                    "coins": coins,
                    "items": items,
                    "telegram_id": telegram_id
                }

            elif line.startswith("Рынок:"):
                in_market_section = True

            elif in_market_section and line:
                try:
                    index, details = line.split(': ', 1)
                    details_parts = details.split(', ')

                    emoji = details_parts[0].split(': ')[1]
                    quantity = int(details_parts[1].split(': ')[1])
                    price = int(details_parts[2].split(': ')[1])
                    seller = details_parts[3].split(': ')[1]

                    market.append({
                        "emoji": emoji,
                        "quantity": quantity,
                        "price": price,
                        "seller": seller
                    })
                except ValueError as e:
                    print(f"Ошибка обработки строки рынка: {line} - {e}")

            elif not line:
                in_market_section = False

        await update.message.reply_text(f"Информация из файла {file_path} загружена.")
        context.user_data['awaiting_file'] = False

#Обработка ввода
def main():
    application = ApplicationBuilder().token("7385251830:AAH8zfW8fxJVdlM1ipU-zdGnRfk5z86L6ec").build()

    application.add_handler(CommandHandler("start", start))
    application.add_handler(CommandHandler("inventory", inventory))
    application.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_message))
    application.add_handler(CommandHandler("m", add_emoji))
    application.add_handler(CommandHandler("get", get_users))
    application.add_handler(CommandHandler("set", set_users))
    application.add_handler(CommandHandler("market", market_view))
    application.add_handler(CommandHandler("sail", sail))
    application.add_handler(MessageHandler(filters.Document.ALL, handle_document))
    application.add_handler(CallbackQueryHandler(handle_market_navigation, pattern=r'(previous_market_page|next_market_page)'))
    application.add_handler(CallbackQueryHandler(handle_market_click, pattern=r'market_.*'))
    application.add_handler(CallbackQueryHandler(handle_inventory_button_click, pattern=r'(emoji_|next_page|prev_page)'))


    application.run_polling()

#Старт
if __name__ == '__main__':
    main()
