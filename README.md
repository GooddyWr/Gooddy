from telegram import Update, InlineKeyboardMarkup, InlineKeyboardButton
from telegram.ext import Application, CommandHandler, MessageHandler, CallbackQueryHandler, filters, \
    ConversationHandler, ContextTypes

# Этапы опроса
NAME, SURNAME, GROUP, TEST = range(4)

# Ваш Telegram ID для получения результатов
YOUR_TELEGRAM_ID = -1002354545756  # Замените на ваш ID
TOKEN = '7989136336:AAGpMa-bCeQY5u3e3j9m31eDWvpSJiW8q_0'  # Замените на ваш токен

# Вопросы теста с вариантами ответов и фото
questions = [
    {
        "question": "В каком году началась Вторая мировая война?",
        "options": ["1937", "1938", "1939", "1940"],
        "answer": "1939",
        "photo": "https://t.me/grusstty/236"  # Замените на ссылку к фото
    },
    {
        "question": "Какая битва была поворотным моментом на Восточном фронте?",
        "options": ["Курская битва", "Сталинградская битва", "Ленинградская битва", "Битва за Москву"],
        "answer": "Сталинградская битва",
    },
    # Добавьте больше вопросов здесь
]


# Начало опроса
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    await update.message.reply_text("Добро пожаловать! Для начала, введите своё имя.")
    return NAME


# Сохранение имени
async def name(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    context.user_data['name'] = update.message.text
    await update.message.reply_text("Введите вашу фамилию.")
    return SURNAME


# Сохранение фамилии
async def surname(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    context.user_data['surname'] = update.message.text
    await update.message.reply_text("Введите вашу группу.")
    return GROUP


# Сохранение группы и начало теста
async def group(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    context.user_data['group'] = update.message.text
    context.user_data['score'] = 0
    context.user_data['question_index'] = 0
    await update.message.reply_text("Отлично! Начинаем тест по истории.")

    # Вызов функции ask_question напрямую, чтобы задать первый вопрос
    return await ask_question(update, context)


# Функция для задания вопроса с фото и вариантами ответов
async def ask_question(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    index = context.user_data['question_index']
    if index < len(questions):
        question_data = questions[index]
        question_text = question_data["question"]
        options = question_data["options"]
        photo = question_data.get("photo")

        # Создаем кнопки для вариантов ответов
        buttons = [[InlineKeyboardButton(option, callback_data=option)] for option in options]
        reply_markup = InlineKeyboardMarkup(buttons)

        # Отправляем фото, если оно есть
        if photo:
            await update.message.reply_photo(photo=photo)

        # Отправляем вопрос с кнопками
        await update.message.reply_text(question_text, reply_markup=reply_markup)
        return TEST
    else:
        return await finish_test(update, context)


# Проверка ответа
async def check_answer(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    query = update.callback_query
    await query.answer()  # Убираем уведомление о выбранном ответе
    answer = query.data  # Получаем ответ пользователя
    index = context.user_data['question_index']
    correct_answer = questions[index]["answer"]

    # Определяем, был ли ответ правильным
    if answer == correct_answer:
        context.user_data['score'] += 1
        response_text = "Верно! Это правильный ответ."
    else:
        response_text = f"Неправильно. Правильный ответ: {correct_answer}."

    # Отправляем пользователю сообщение с результатом текущего вопроса
    await query.message.reply_text(response_text)

    # Переходим к следующему вопросу
    context.user_data['question_index'] += 1
    return await ask_question(query, context)


# Завершение теста и отправка результатов
async def finish_test(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    name = context.user_data['name']
    surname = context.user_data['surname']
    group = context.user_data['group']
    score = context.user_data['score']
    total_questions = len(questions)

    result_message = (f"Результаты:\n"
                      f"Имя: {name}\nФамилия: {surname}\nГруппа: {group}\n"
                      f"Баллы: {score}/{total_questions}")
    await update.message.reply_text("Тест завершен! Спасибо за участие.")

    # Отправка результата вам в личные сообщения
    await context.bot.send_message(chat_id=YOUR_TELEGRAM_ID, text=result_message)
    return ConversationHandler.END


# Команда отмены
async def cancel(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    await update.message.reply_text("Опрос был отменён.")
    return ConversationHandler.END


# Главная функция
def main() -> None:
    application = Application.builder().token(TOKEN).build()

    # Определение последовательности этапов опроса
    conv_handler = ConversationHandler(
        entry_points=[CommandHandler('start', start)],
        states={
            NAME: [MessageHandler(filters.TEXT & ~filters.COMMAND, name)],
            SURNAME: [MessageHandler(filters.TEXT & ~filters.COMMAND, surname)],
            GROUP: [MessageHandler(filters.TEXT & ~filters.COMMAND, group)],
            TEST: [CallbackQueryHandler(check_answer)],
        },
        fallbacks=[CommandHandler('cancel', cancel)],
    )

    application.add_handler(conv_handler)
    application.run_polling()


if __name__ == '__main__':
    main()
