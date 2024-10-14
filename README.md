# Gamebot

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


def get_current_time_kiev():
    kiev_tz = pytz.timezone('Europe/Kiev')
    now = datetime.now(kiev_tz)
    return now.strftime("%H:%M; %d/%m/%Y")


# Команда /start
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    global next_user_id
    username = update.effective_user.username
    user_id = next_user_id
    next_user_id += 1

    users[user_id] = {
        "username": username,
        "join_date": get_current_time_kiev(),
        "coins": 10,
        "items": ["😀"],
        "user_bot_id":user_id
    }

    await update.message.reply_text(f"Добро пожаловать, {username}! Ты добавлен в систему. Твой ID: {user_id}")


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
    for i in range(0, len(items_on_page), 5):  # 5 items per row
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
    page = context.user_data.get('page', 0)  # Default page 0 if not found

    if user_id is not None and user_id in users:
        user_info = users[user_id]
        items = user_info['items']

    if query.data.startswith('emoji_'):
        selected_emoji = query.data.split('_')[1]
        if user_id in users:
            # Убираем все перед закрывающей скобкой ")", оставляем только сам эмодзи
            cleaned_emoji = re.sub(r'^\(\d+\)', '', selected_emoji).strip()
            users[user_id]['selected_emoji'] = cleaned_emoji

            coins = user_info.get('coins', 0)
            selected_emoji_display = users[user_id]['selected_emoji']  # Берем только эмодзи

            inventory_string = format_inventory(user_info['items'])
            formatted_items = inventory_string.split("; ")
            keyboard = generate_inventory_buttons(formatted_items, page)

            await query.edit_message_text(
                text=f"Инвентарь {user_info['username']}\n"
                     f"Количество монет: {coins} 💲\n"
                     f"Выбранный эмодзи: {selected_emoji_display}",  # Отображаем только выбранный эмодзи
                reply_markup=keyboard
            )
        return

    # Обработка кнопок навигации страниц
    if query.data == 'next_page' and page < (len(items) - 1) // page_size:
        print("<")
        page += 1
    elif query.data == 'prev_page' and page > 0:
        print(">")
        page -= 1

    context.user_data['page'] = page  # Обновляем текущую страницу в данных пользователя

    # Обновляем кнопки для новой страницы
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
        context.user_data['page'] = 0  # Начинаем с первой страницы

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
            inventory_string = format_inventory(user_info["items"])  # Получаем строку инвентаря
            coins = user_info.get('coins', 0)
            selected_emoji = user_info.get('selected_emoji', '😀')

            # Добавляем инвентарь к сообщению
            await update.message.reply_text(
                text=f"Инвентарь {user_info['username']}\n"
                     f"Количество монет: {coins} 💲\n"
                     f"Выбранный эмодзи: {selected_emoji}\n"
                     f"Инвентарь: {inventory_string}"  # Выводим инвентарь
            )
        else:
            await update.message.reply_text("Ты не зарегистрирован. Используй /start для регистрации.")

    #elif text in ("cc","сс"):



# Команда /sail
async def sail(update: Update, context: ContextTypes.DEFAULT_TYPE):
    username = update.effective_user.username
    user_id = next((uid for uid, info in users.items() if username == info["username"]), None)

    if user_id is not None:
        try:
            args = context.args
            if len(args) != 3:
                await update.message.reply_text("Использование: /sail [эмодзи] [количество] [цена за все]")
                return

            emoji = args[0]
            quantity = int(args[1])
            total_price = int(args[2])

            user_info = users[user_id]
            items = user_info['items']

            # Объединяем все эмодзи в словарь для проверки количества
            item_counts = {}
            for item in items:
                match = re.match(r'(\d+)(.+)', item)
                if match:
                    count = int(match.group(1))
                    item_emoji = match.group(2)
                    if item_emoji in item_counts:
                        item_counts[item_emoji] += count
                    else:
                        item_counts[item_emoji] = count
                else:
                    if item in item_counts:
                        item_counts[item] += 1
                    else:
                        item_counts[item] = 1

            # Проверка, есть ли у пользователя достаточное количество эмодзи
            if emoji not in item_counts or item_counts[emoji] < quantity:
                await update.message.reply_text(f"У вас недостаточно {emoji} для продажи.")
                return

            # Удаление проданных эмодзи из инвентаря
            new_items = []
            remaining_quantity = quantity
            for item in items:
                match = re.match(r'(\d+)(.+)', item)
                if match and match.group(2) == emoji:
                    count = int(match.group(1))
                    if count > remaining_quantity:
                        new_items.append(f"{count - remaining_quantity}{emoji}")
                        remaining_quantity = 0
                    else:
                        remaining_quantity -= count
                elif match or item != emoji:
                    new_items.append(item)

            user_info['items'] = new_items

            # Добавляем предмет на рынок
            market.append({
                'seller': username,
                'emoji': emoji,
                'quantity': quantity,
                'total_price': total_price
            })

            await update.message.reply_text(f"Вы успешно добавили на рынок {quantity}{emoji} за {total_price} монет.")
        except ValueError:
            await update.message.reply_text("Ошибка в количестве или цене. Проверьте формат.")
    else:
        await update.message.reply_text("Ты не зарегистрирован. Используй /start для регистрации.")


# Команда /market для просмотра рынка
async def market_view(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not market:
        await update.message.reply_text("Рынок пуст.")
        return

    keyboard = []
    for i in range(0, len(market), 5):
        keyboard.append([InlineKeyboardButton(f"{item['quantity']}{item['emoji']} - {item['total_price']} монет",
                                             callback_data=f"market_{i}") for item in market[i:i + 5]])

    await update.message.reply_text("Рынок:", reply_markup=InlineKeyboardMarkup(keyboard))


# Обработка нажатия кнопки на рынке
async def handle_market_click(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()  # Это обязательное действие для Telegram API.

    print("Вход в handle_market_click")  # Для проверки работы функции

    index = int(query.data.split('_')[1])
    item = market[index]

    # Проверка нажатия на кнопку для покупки
    buyer_username = update.effective_user.username
    buyer_id = next((uid for uid, info in users.items() if buyer_username == info["username"]), None)

    if buyer_id is not None:
        buyer_info = users[buyer_id]
        seller_info = next(info for uid, info in users.items() if info["username"] == item['seller'])

        # Отладочный вывод для проверки seller_info
        print(f"Информация о продавце: {seller_info}")  # Вывод информации о продавце

        # Проверка наличия монет у покупателя
        if buyer_info['coins'] < item['total_price']:
            await query.edit_message_text("Недостаточно средств для покупки.")
            return  # Возвращаемся, не удаляя товар

        # Списываем монеты с покупателя и добавляем их продавцу
        buyer_info['coins'] -= item['total_price']
        seller_info['coins'] += item['total_price']

        # Вывод информации в консоль
        print(f"Покупатель: {buyer_info['username']} (ID: {buyer_id})")
        print(f"Продавец: {seller_info['username']} (ID: {seller_info.get('user_id', 'нет user_id')})")  # Используем get для избежания KeyError
        print(f"Количество монет покупателя: {buyer_info['coins']} монет")
        print(f"Купленный эмодзи: {item['emoji']} (Количество: {item['quantity']})")
        print(f"Цена: {item['total_price']} монет")

        # Удаляем товар из рынка
        market.pop(index)

        # Добавляем эмодзи в инвентарь покупателя
        buyer_info['items'].append(f"{item['quantity']}{item['emoji']}")

        await query.edit_message_text(f"Вы купили {item['quantity']}{item['emoji']} за {item['total_price']} монет.")
    else:
        await query.edit_message_text("Ты не зарегистрирован. Используй /start для регистрации.")


# Обработка нажатий на кнопки
async def handle_market_buttons(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    data = query.data

    print(f"Вход в handle_market_buttons с данными: {data}")  # Для проверки работы функции

    # Проверка, к какому действию относится нажатие кнопки
    if data.startswith('market_'):
        await handle_market_click(update, context)  # Для обработки кликов на рынке
    elif data.startswith('emoji_'):
        print("Переход к инвентарю")  # Добавлено для отладки
        await generate_inventory_buttons(update, context)  # Вызов функции для отображения инвентаря
    else:
        await update.message.reply_text("Неизвестное действие.")


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
                f"Баланс: {info['coins']} 💲, Инвентарь: {format_inventory(info['items'])}\n")

    with open(file_path, 'rb') as file:
        await update.message.reply_document(file, caption=f"Информация о пользователях в файле {file_path}")


# Ожидание файла для команды /set
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
            for line in f:
                parts = line.strip().split(', ')
                user_id = int(parts[0].split(': ')[1])
                username = parts[1].split(': ')[1]
                join_date = parts[2].split(': ')[1]
                coins = int(parts[3].split(': ')[1].replace(' 💲', ''))
                items = parts[4].split(': ')[1].split('; ')

                users[user_id] = {
                    "username": username,
                    "join_date": join_date,
                    "coins": coins,
                    "items": items
                }

        await update.message.reply_text(f"Информация из файла {file_path} загружена.")
        context.user_data['awaiting_file'] = False


def main():
    application = ApplicationBuilder().token("7385251830:AAH8zfW8fxJVdlM1ipU-zdGnRfk5z86L6ec").build()

    application.add_handler(CommandHandler("start", start))
    application.add_handler(CommandHandler("inventory", inventory))
    application.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_message))  # Добавляем команду /cc
    application.add_handler(CommandHandler("m", add_emoji))
    application.add_handler(CommandHandler("get", get_users))
    application.add_handler(CommandHandler("set", set_users))
    application.add_handler(CommandHandler("market", market_view))  # Обработчик для команды /market
    application.add_handler(CommandHandler("sail", sail))  # Обработчик для команды /sail
    application.add_handler(MessageHandler(filters.Document.ALL, handle_document))  # Добавлен обработчик документов
    application.add_handler(CallbackQueryHandler(handle_market_buttons, pattern=r'market_.*'))
    application.add_handler(CallbackQueryHandler(handle_inventory_button_click, pattern=r'emoji_.*'))


    application.run_polling()


if __name__ == '__main__':
    main()
