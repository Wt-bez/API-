# API-дневника

class User(BaseModel):
    """Модель пользователя Telegram бота"""
    user_id: int = Field(..., description="ID пользователя в Telegram")
    username: Optional[str] = None
    first_name: str
    last_name: Optional[str] = None
    created_at: datetime = Field(default_factory=datetime.now)
    language_code: Optional[str] = "ru"
    
    class Config:
        from_attributes = True

        class HabitFrequency(str, Enum):
    """Частота выполнения привычки"""
    DAILY = "daily"
    WEEKLY = "weekly"
    MONTHLY = "monthly"

class HabitStatus(str, Enum):
    """Статус привычки"""
    ACTIVE = "active"
    COMPLETED = "completed"
    ARCHIVED = "archived"

class Habit(BaseModel):
    """Модель привычки пользователя"""
    habit_id: int = Field(..., description="ID привычки")
    user_id: int = Field(..., description="ID пользователя")
    title: str = Field(..., min_length=3, max_length=100)
    description: Optional[str] = None
    is_good: bool = Field(..., description="Полезная или вредная")
    frequency: HabitFrequency = Field(default=HabitFrequency.DAILY)
    reminder_time: Optional[time] = None
    status: HabitStatus = Field(default=HabitStatus.ACTIVE)
    created_at: datetime = Field(default_factory=datetime.now)
    updated_at: datetime = Field(default_factory=datetime.now)
    
    class Config:
        from_attributes = True

        logger = logging.getLogger(__name__)

class DatabaseManager:
    """Менеджер базы данных SQLite"""
    
    def __init__(self, db_path: str = "habits.db"):
        self.db_path = db_path
        self._init_db()
    
    def _get_connection(self):
        """Получить соединение с БД"""
        conn = sqlite3.connect(self.db_path)
        conn.row_factory = sqlite3.Row
        return conn
    
    def _init_db(self):
        """Инициализация базы данных"""
        with self._get_connection() as conn:
            cursor = conn.cursor()
            
            # Таблица пользователей
            cursor.execute('''
                CREATE TABLE IF NOT EXISTS users (
                    user_id INTEGER PRIMARY KEY,
                    username TEXT,
                    first_name TEXT NOT NULL,
                    last_name TEXT,
                    language_code TEXT DEFAULT 'ru',
                    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
                )
            ''')
            
            # Таблица привычек
            cursor.execute('''
                CREATE TABLE IF NOT EXISTS habits (
                    habit_id INTEGER PRIMARY KEY AUTOINCREMENT,
                    user_id INTEGER NOT NULL,
                    title TEXT NOT NULL,
                    description TEXT,
                    is_good BOOLEAN NOT NULL,
                    frequency TEXT DEFAULT 'daily',
                    reminder_time TEXT,
                    status TEXT DEFAULT 'active',
                    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                    FOREIGN KEY (user_id) REFERENCES users (user_id)
                )
            ''')
            
            # Таблица напоминаний
            cursor.execute('''
                CREATE TABLE IF NOT EXISTS reminders (
                    reminder_id INTEGER PRIMARY KEY AUTOINCREMENT,
                    habit_id INTEGER NOT NULL,
                    user_id INTEGER NOT NULL,
                    scheduled_time TIMESTAMP NOT NULL,
                    sent BOOLEAN DEFAULT FALSE,
                    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                    FOREIGN KEY (habit_id) REFERENCES habits (habit_id),
                    FOREIGN KEY (user_id) REFERENCES users (user_id)
                )
            ''')
            
            # Таблица заметок (для поиска данных)
            cursor.execute('''
                CREATE TABLE IF NOT EXISTS notes (
                    note_id INTEGER PRIMARY KEY AUTOINCREMENT,
                    user_id INTEGER NOT NULL,
                    title TEXT NOT NULL,
                    content TEXT NOT NULL,
                    tags TEXT,
                    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                    FOREIGN KEY (user_id) REFERENCES users (user_id)
                )
            ''')
            
            # Индексы для быстрого поиска
            cursor.execute('CREATE INDEX IF NOT EXISTS idx_notes_user ON notes(user_id)')
            cursor.execute('CREATE INDEX IF NOT EXISTS idx_notes_search ON notes(title, content, tags)')
            cursor.execute('CREATE INDEX IF NOT EXISTS idx_habits_user ON habits(user_id)')
            cursor.execute('CREATE INDEX IF NOT EXISTS idx_reminders_time ON reminders(scheduled_time)')
            
            conn.commit()
        logger.info("База данных инициализирована")
    
    def add_user(self, user_id: int, username: str, first_name: str, 
                 last_name: str = None, language_code: str = "ru"):
        """Добавление пользователя"""
        with self._get_connection() as conn:
            cursor = conn.cursor()
            cursor.execute('''
                INSERT OR IGNORE INTO users 
                (user_id, username, first_name, last_name, language_code)
                VALUES (?, ?, ?, ?, ?)
            ''', (user_id, username, first_name, last_name, language_code))
            conn.commit()
            return cursor.lastrowid
    
    def add_habit(self, user_id: int, title: str, is_good: bool, 
                  description: str = None, frequency: str = "daily", 
                  reminder_time: str = None):
        """Добавление привычки"""
        with self._get_connection() as conn:
            cursor = conn.cursor()
            cursor.execute('''
                INSERT INTO habits 
                (user_id, title, description, is_good, frequency, reminder_time)
                VALUES (?, ?, ?, ?, ?, ?)
            ''', (user_id, title, description, is_good, frequency, reminder_time))
            conn.commit()
            return cursor.lastrowid
    
    def get_user_habits(self, user_id: int, status: str = "active"):
        """Получение привычек пользователя"""
        with self._get_connection() as conn:
            cursor = conn.cursor()
            cursor.execute('''
                SELECT * FROM habits 
                WHERE user_id = ? AND status = ?
                ORDER BY created_at DESC
            ''', (user_id, status))
            return cursor.fetchall()
    
    def get_habit_by_id(self, habit_id: int):
        """Получение привычки по ID"""
        with self._get_connection() as conn:
            cursor = conn.cursor()
            cursor.execute('SELECT * FROM habits WHERE habit_id = ?', (habit_id,))
            return cursor.fetchone()
    
    def add_note(self, user_id: int, title: str, content: str, tags: str = None):
        """Добавление заметки"""
        with self._get_connection() as conn:
            cursor = conn.cursor()
            cursor.execute('''
                INSERT INTO notes (user_id, title, content, tags)
                VALUES (?, ?, ?, ?)
            ''', (user_id, title, content, tags))
            conn.commit()
            return cursor.lastrowid
    
    def search_notes(self, user_id: int, query: str):
        """Поиск заметок по тексту или тегам"""
        with self._get_connection() as conn:
            cursor = conn.cursor()
            search_pattern = f"%{query}%"
            cursor.execute('''
                SELECT * FROM notes 
                WHERE user_id = ? 
                AND (title LIKE ? OR content LIKE ? OR tags LIKE ?)
                ORDER BY created_at DESC
            ''', (user_id, search_pattern, search_pattern, search_pattern))
            return cursor.fetchall()
    
    def get_user_notes(self, user_id: int, limit: int = 50):
        """Получение всех заметок пользователя"""
        with self._get_connection() as conn:
            cursor = conn.cursor()
            cursor.execute('''
                SELECT * FROM notes 
                WHERE user_id = ? 
                ORDER BY created_at DESC
                LIMIT ?
            ''', (user_id, limit))
            return cursor.fetchall()
    
    def get_note_by_id(self, note_id: int):
        """Получение заметки по ID"""
        with self._get_connection() as conn:
            cursor = conn.cursor()
            cursor.execute('SELECT * FROM notes WHERE note_id = ?', (note_id,))
            return cursor.fetchone()
    
    def update_note(self, note_id: int, title: str = None, content: str = None, tags: str = None):
        """Обновление заметки"""
        with self._get_connection() as conn:
            cursor = conn.cursor()
            updates = []
            params = []
            
            if title is not None:
                updates.append("title = ?")
                params.append(title)
            if content is not None:
                updates.append("content = ?")
                params.append(content)
            if tags is not None:
                updates.append("tags = ?")
                params.append(tags)
            
            if updates:
                updates.append("updated_at = CURRENT_TIMESTAMP")
                params.append(note_id)
                query = f"UPDATE notes SET {', '.join(updates)} WHERE note_id = ?"
                cursor.execute(query, params)
                conn.commit()
    
    def delete_note(self, note_id: int):
        """Удаление заметки"""
        with self._get_connection() as conn:
            cursor = conn.cursor()
            cursor.execute('DELETE FROM notes WHERE note_id = ?', (note_id,))
            conn.commit()
    
    def schedule_reminder(self, habit_id: int, user_id: int, scheduled_time: datetime):
        """Планирование напоминания"""
        with self._get_connection() as conn:
            cursor = conn.cursor()
            cursor.execute('''
                INSERT INTO reminders (habit_id, user_id, scheduled_time)
                VALUES (?, ?, ?)
            ''', (habit_id, user_id, scheduled_time.isoformat()))
            conn.commit()
            return cursor.lastrowid
    
    def get_pending_reminders(self):
        """Получение ожидающих напоминаний"""
        with self._get_connection() as conn:
            cursor = conn.cursor()
            cursor.execute('''
                SELECT r.*, h.title, h.description
                FROM reminders r
                JOIN habits h ON r.habit_id = h.habit_id
                WHERE r.sent = FALSE 
                AND r.scheduled_time <= datetime('now')
                LIMIT 10
            ''')
            return cursor.fetchall()
    
    def mark_reminder_sent(self, reminder_id: int):
        """Отметка напоминания как отправленного"""
        with self._get_connection() as conn:
            cursor = conn.cursor()
            cursor.execute('UPDATE reminders SET sent = TRUE WHERE reminder_id = ?', (reminder_id,))
            conn.commit()
    
    def get_user_stats(self, user_id: int):
        """Получение статистики пользователя"""
        with self._get_connection() as conn:
            cursor = conn.cursor()
            
            # Статистика привычек
            cursor.execute('''
                SELECT 
                    COUNT(*) as total_habits,
                    SUM(CASE WHEN is_good = 1 THEN 1 ELSE 0 END) as good_habits,
                    SUM(CASE WHEN is_good = 0 THEN 1 ELSE 0 END) as bad_habits
                FROM habits 
                WHERE user_id = ? AND status = 'active'
            ''', (user_id,))
            habits_stats = cursor.fetchone()
            
            # Статистика заметок
            cursor.execute('SELECT COUNT(*) as total_notes FROM notes WHERE user_id = ?', (user_id,))
            notes_stats = cursor.fetchone()
            
            return {
                'total_habits': habits_stats[0] if habits_stats else 0,
                'good_habits': habits_stats[1] if habits_stats else 0,
                'bad_habits': habits_stats[2] if habits_stats else 0,
                'total_notes': notes_stats[0] if notes_stats else 0
            }

# Создаем глобальный экземпляр менеджера БД
db_manager = DatabaseManager()

def init_db():
    """Инициализация базы данных (для main.py)"""
    return db_manager

    from .manager import DatabaseManager, init_db

__all__ = ['DatabaseManager', 'init_db']

logger = logging.getLogger(__name__)
router = Router()

# Клавиатуры
def get_main_keyboard():
    """Основная клавиатура"""
    keyboard = ReplyKeyboardMarkup(
        keyboard=[
            [KeyboardButton(text="➕ Добавить привычку"), KeyboardButton(text="📝 Добавить заметку")],
            [KeyboardButton(text="📋 Мои привычки"), KeyboardButton(text="📓 Мои заметки")],
            [KeyboardButton(text="🔍 Поиск заметок"), KeyboardButton(text="💡 Получить совет")],
            [KeyboardButton(text="📊 Статистика")]
        ],
        resize_keyboard=True
    )
    return keyboard

# Состояния FSM
class HabitForm(StatesGroup):
    """Состояния для формы добавления привычки"""
    title = State()
    description = State()
    is_good = State()
    frequency = State()
    reminder_time = State()

class NoteForm(StatesGroup):
    """Состояния для формы добавления заметки"""
    title = State()
    content = State()
    tags = State()

class SearchForm(StatesGroup):
    """Состояние для поиска"""
    query = State()

@router.message(CommandStart())
async def cmd_start(message: Message):
    """Обработчик команды /start"""
    # Добавляем пользователя в БД
    db_manager.add_user(
        user_id=message.from_user.id,
        username=message.from_user.username,
        first_name=message.from_user.first_name,
        last_name=message.from_user.last_name,
        language_code=message.from_user.language_code
    )
    
    welcome_text = (
        f"👋 Привет, {message.from_user.first_name}!\n\n"
        "Я бот для отслеживания привычек и ведения заметок.\n\n"
        "Я помогу тебе:\n"
        "• 📝 Добавлять и отслеживать привычки\n"
        "• 💡 Получать персональные советы от ИИ\n"
        "• 🏷️ Создавать заметки с тегами\n"
        "• 🔍 Искать информацию в заметках\n"
        "• ⏰ Напоминать о важном\n\n"
        "Используй меню ниже для навигации!"
    )
    
    await message.answer(welcome_text, reply_markup=get_main_keyboard())

@router.message(F.text == "➕ Добавить привычку")
async def add_habit_start(message: Message, state: FSMContext):
    """Начало добавления привычки"""
    await message.answer("Введите название привычки (например: 'Утренняя зарядка'):")
    await state.set_state(HabitForm.title)

@router.message(HabitForm.title)
async def process_habit_title(message: Message, state: FSMContext):
    """Обработка названия привычки"""
    await state.update_data(title=message.text)
    await message.answer("Введите описание привычки (или отправьте '-' для пропуска):")
    await state.set_state(HabitForm.description)

@router.message(HabitForm.description)
async def process_habit_description(message: Message, state: FSMContext):
    """Обработка описания привычки"""
    description = message.text if message.text != "-" else None
    await state.update_data(description=description)
    
    keyboard = ReplyKeyboardMarkup(
        keyboard=[
            [KeyboardButton(text="✅ Полезная"), KeyboardButton(text="❌ Вредная")]
        ],
        resize_keyboard=True
    )
    await message.answer("Это полезная или вредная привычка?", reply_markup=keyboard)
    await state.set_state(HabitForm.is_good)

@router.message(HabitForm.is_good)
async def process_habit_type(message: Message, state: FSMContext):
    """Обработка типа привычки"""
    is_good = message.text == "✅ Полезная"
    await state.update_data(is_good=is_good)
    
    keyboard = ReplyKeyboardMarkup(
        keyboard=[
            [KeyboardButton(text="Ежедневно"), KeyboardButton(text="Еженедельно")],
            [KeyboardButton(text="Ежемесячно")]
        ],
        resize_keyboard=True
    )
    await message.answer("Как часто вы хотите выполнять эту привычку?", reply_markup=keyboard)
    await state.set_state(HabitForm.frequency)

@router.message(HabitForm.frequency)
async def process_habit_frequency(message: Message, state: FSMContext):
    """Обработка частоты привычки"""
    frequency_map = {
        "Ежедневно": "daily",
        "Еженедельно": "weekly",
        "Ежемесячно": "monthly"
    }
    frequency = frequency_map.get(message.text, "daily")
    await state.update_data(frequency=frequency)
    
    keyboard = ReplyKeyboardMarkup(
        keyboard=[
            [KeyboardButton(text="🕗 8:00"), KeyboardButton(text="🕙 10:00"), KeyboardButton(text="🕛 12:00")],
            [KeyboardButton(text="🕑 14:00"), KeyboardButton(text="🕔 17:00"), KeyboardButton(text="🕗 20:00")],
            [KeyboardButton(text="Не нужно")]
        ],
        resize_keyboard=True
    )
    await message.answer("Выберите время напоминания (или 'Не нужно'):", reply_markup=keyboard)
    await state.set_state(HabitForm.reminder_time)

@router.message(HabitForm.reminder_time)
async def process_habit_reminder(message: Message, state: FSMContext):
    """Обработка времени напоминания"""
    reminder_time = None
    if message.text != "Не нужно":
        try:
            time_str = message.text.split()[-1]  # Извлекаем время из текста
            reminder_time = time.fromisoformat(time_str)
        except:
            await message.answer("Используйте одно из предложенных значений времени!")
            return
    
    user_data = await state.get_data()
    
    # Добавляем привычку в БД
    habit_id = db_manager.add_habit(
        user_id=message.from_user.id,
        title=user_data['title'],
        is_good=user_data['is_good'],
        description=user_data.get('description'),
        frequency=user_data['frequency'],
        reminder_time=reminder_time.isoformat() if reminder_time else None
    )
    
    # Если указано время напоминания, планируем его
    if reminder_time:
        # Случайное время в интервале от 3 до 6 часов после указанного
        hour_offset = random.randint(3, 6)
        scheduled_time = datetime.combine(datetime.now().date(), reminder_time) + timedelta(hours=hour_offset)
        db_manager.schedule_reminder(habit_id, message.from_user.id, scheduled_time)
    
    habit_type = "полезной" if user_data['is_good'] else "вредной"
    response = (
        f"✅ Привычка добавлена!\n\n"
        f"📝 Название: {user_data['title']}\n"
        f"📄 Описание: {user_data.get('description') or 'нет'}\n"
        f"🎯 Тип: {habit_type} привычка\n"
        f"📅 Частота: {message.text if message.text == 'Не нужно' else message.text.split()[0]}\n"
    )
    
    if reminder_time:
        response += f"⏰ Напоминание: {reminder_time.strftime('%H:%M')}\n"
    
    await message.answer(response, reply_markup=get_main_keyboard())
    await state.clear()

@router.message(F.text == "📝 Добавить заметку")
async def add_note_start(message: Message, state: FSMContext):
    """Начало добавления заметки"""
    await message.answer("Введите заголовок заметки:")
    await state.set_state(NoteForm.title)

@router.message(NoteForm.title)
async def process_note_title(message: Message, state: FSMContext):
    """Обработка заголовка заметки"""
    await state.update_data(title=message.text)
    await message.answer("Введите текст заметки:")
    await state.set_state(NoteForm.content)

@router.message(NoteForm.content)
async def process_note_content(message: Message, state: FSMContext):
    """Обработка содержания заметки"""
    await state.update_data(content=message.text)
    await message.answer("Введите теги через запятую (например: работа, идеи, важное):")
    await state.set_state(NoteForm.tags)

@router.message(NoteForm.tags)
async def process_note_tags(message: Message, state: FSMContext):
    """Обработка тегов заметки"""
    note_data = await state.get_data()
    
    # Добавляем заметку в БД
    note_id = db_manager.add_note(
        user_id=message.from_user.id,
        title=note_data['title'],
        content=note_data['content'],
        tags=message.text
    )
    
    response = (
        f"✅ Заметка #{note_id} добавлена!\n\n"
        f"📌 Заголовок: {note_data['title']}\n"
        f"📄 Содержание: {note_data['content'][:100]}...\n"
        f"🏷️ Теги: {message.text or 'нет'}\n"
        f"📅 Дата: {datetime.now().strftime('%d.%m.%Y %H:%M')}"
    )
    
    await message.answer(response, reply_markup=get_main_keyboard())
    await state.clear()

@router.message(F.text == "📋 Мои привычки")
async def show_habits(message: Message):
    """Показ привычек пользователя"""
    habits = db_manager.get_user_habits(message.from_user.id)
    
    if not habits:
        await message.answer("У вас пока нет привычек. Добавьте первую!")
        return
    
    response = "📋 Ваши привычки:\n\n"
    for habit in habits:
        habit_type = "✅" if habit['is_good'] else "❌"
        response += (
            f"{habit_type} {habit['title']}\n"
            f"   📅 Частота: {habit['frequency']}\n"
            f"   📝 Описание: {habit['description'] or 'нет'}\n"
            f"   📅 Создана: {habit['created_at']}\n\n"
        )
    
    await message.answer(response)

@router.message(F.text == "📓 Мои заметки")
async def show_notes(message: Message):
    """Показать все заметки пользователя"""
    notes = db_manager.get_user_notes(message.from_user.id)
    
    if not notes:
        await message.answer("У вас пока нет заметок. Добавьте первую!")
        return
    
    response = "📓 Ваши заметки:\n\n"
    for note in notes:
        response += (
            f"📌 {note['title']}\n"
            f"   📄 {note['content'][:50]}...\n"
            f"   🏷️ Теги: {note['tags'] or 'нет'}\n"
            f"   📅 {note['created_at'][:10]}\n\n"
        )
    
    await message.answer(response[:4000])  # Ограничение Telegram

@router.message(F.text == "🔍 Поиск заметок")
async def search_notes_start(message: Message, state: FSMContext):
    """Начало поиска заметок"""
    await message.answer("Введите текст для поиска (по заголовку, содержанию или тегам):")
    await state.set_state(SearchForm.query)

@router.message(SearchForm.query)
async def process_search(message: Message, state: FSMContext):
    """Обработка поискового запроса"""
    notes = db_manager.search_notes(message.from_user.id, message.text)
    
    if not notes:
        await message.answer(f"По запросу '{message.text}' ничего не найдено 😔")
        await state.clear()
        return
    
    response = f"🔍 Найдено заметок: {len(notes)}\n\n"
    for note in notes:
        response += (
            f"📌 {note['title']}\n"
            f"   📄 {note['content'][:50]}...\n"
            f"   🏷️ Теги: {note['tags'] or 'нет'}\n"
            f"   📅 {note['created_at'][:10]}\n\n"
        )
    
    await message.answer(response[:4000])
    await state.clear()

@router.message(F.text == "💡 Получить совет")
async def get_advice(message: Message):
    """Получение совета от ИИ по привычкам"""
    habits = db_manager.get_user_habits(message.from_user.id)
    
    if not habits:
        await message.answer("Сначала добавьте привычки, чтобы получить персональные советы!")
        return
    
    # Берем последнюю привычку для примера
    last_habit = habits[0]
    
    agent = HabitAgent()
    advice = await agent.get_advice(
        habit_description=last_habit['title'],
        is_good=bool(last_habit['is_good'])
    )
    
    response = (
        f"💡 Совет по привычке '{last_habit['title']}':\n\n"
        f"{advice.advice}\n\n"
        f"🔥 Мотивация:\n{advice.motivation}\n\n"
        f"📋 Практические шаги:\n"
    )
    
    for i, tip in enumerate(advice.tips, 1):
        response += f"{i}. {tip}\n"
    
    await message.answer(response)

@router.message(F.text == "📊 Статистика")
async def show_stats(message: Message):
    """Показ статистики"""
    habits = db_manager.get_user_habits(message.from_user.id)
    notes = db_manager.get_user_notes(message.from_user.id)
    
    good_habits = [h for h in habits if h['is_good']]
    bad_habits = [h for h in habits if not h['is_good']]
    
    response = (
        f"📊 Ваша статистика:\n\n"
        f"📋 Привычки:\n"
        f"   • Всего: {len(habits)}\n"
        f"   • ✅ Полезных: {len(good_habits)}\n"
        f"   • ❌ Вредных: {len(bad_habits)}\n\n"
        f"📓 Заметки:\n"
        f"   • Всего: {len(notes)}\n"
        f"   • С тегами: {len([n for n in notes if n['tags']])}\n\n"
        f"🎯 Рекомендация: "
    )
    
    if len(good_habits) < 3:
        response += "Добавьте больше полезных привычек!"
    elif len(bad_habits) > len(good_habits):
        response += "Поработайте над уменьшением вредных привычек!"
    else:
        response += "Вы на правильном пути! Продолжайте в том же духе!"
    
    await message.answer(response)

@router.message(Command("help"))
async def cmd_help(message: Message):
    """Помощь по командам"""
    help_text = (
        "📖 Справка по командам:\n\n"
        "Основные функции:\n"
        "➕ Добавить привычку - Создание новой привычки\n"
        "📝 Добавить заметку - Создание заметки с тегами\n"
        "📋 Мои привычки - Просмотр всех привычек\n"
        "📓 Мои заметки - Просмотр всех заметок\n"
        "🔍 Поиск заметок - Поиск по заметкам\n"
        "💡 Получить совет - ИИ-совет по привычкам\n"
        "📊 Статистика - Ваша статистика\n\n"
        "Команды:\n"
        "/start - Перезапуск бота\n"
        "/help - Эта справка\n\n"
        "💡 Совет: Используйте теги в заметках для удобного поиска!"
    )
    await message.answer(help_text)7

    
