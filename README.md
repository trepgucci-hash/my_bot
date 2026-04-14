import telebot
import random
import sqlite3
from datetime import datetime, timedelta
import time
import os
import json
import shutil
import threading
from functools import wraps
from collections import defaultdict

# ==================== КОНФИГУРАЦИЯ ====================
BOT_TOKEN = "8606718005:AAHU_m5jZU3nQFJwONA6-pyM-wg_bj3fVjM"
ADMIN_IDS = [5120305949]
START_BALANCE = 50000
WORK_COOLDOWN = 15 * 60
TRANSFER_COMMISSION = 0.05
DB_NAME = 'bandit_bot.db'
BACKUP_FOLDER = "backups"

# ==================== СОЗДАНИЕ БОТА ====================
bot = telebot.TeleBot(BOT_TOKEN)

# ==================== ПАПКИ ====================
START_IMAGES_FOLDER = "start_images"
ROULETTE_IMAGES_FOLDER = "roulette_images"
SKINS_FOLDER = "skins"

for folder in [START_IMAGES_FOLDER, ROULETTE_IMAGES_FOLDER, SKINS_FOLDER, BACKUP_FOLDER]:
    if not os.path.exists(folder):
        os.makedirs(folder)

default_skin_path = os.path.join(SKINS_FOLDER, "default")
if not os.path.exists(default_skin_path):
    os.makedirs(default_skin_path)

# ==================== СКИНЫ ====================
SKINS = {
    'default': {'price': 0, 'name': 'СТАНДАРТНЫЙ', 'folder': 'default'},
    'radioactive': {'price': 1000000, 'name': 'РЕАКТИВНЫЙ', 'folder': 'radioactive'},
    'pug': {'price': 2500000, 'name': 'МОПС', 'folder': 'pug'},
    'pug_two': {'price': 5000000, 'name': 'МОПС 2', 'folder': 'pug_two'},
    'stoner': {'price': 7600000, 'name': 'УКУРЫШЬ', 'folder': 'stoner'},
    'face': {'price': 10000000, 'name': 'ФЕЙС', 'folder': 'face'},
    'main': {'price': 14000000, 'name': 'МАИН', 'folder': 'main'},
    'Hacci': {'price': 23000000, 'name': 'ХАЧ', 'folder': 'Hacci'},
    'fuck': {'price': 50000000, 'name': 'ПИЗДЕЦ', 'folder': 'fuck'},
    'Apex Twin': {'price': 1000000000, 'name': 'APEX TWIN', 'folder': 'Apex Twin'},
    'reallyGucci': {'price': 10000000000, 'name': 'РИЛ ФИТ ГУЧИ', 'folder': 'reallyGucci'},
}

# ==================== БАЗА ДАННЫХ ====================
def init_db():
    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()
    
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS users (
            user_id INTEGER PRIMARY KEY,
            username TEXT,
            custom_name TEXT,
            linked_username TEXT,
            link_reward_given INTEGER DEFAULT 0,
            balance INTEGER DEFAULT 50000,
            currency_dollar INTEGER DEFAULT 0,
            has_laptop INTEGER DEFAULT 0,
            has_gun INTEGER DEFAULT 0,
            has_car TEXT,
            business_owned TEXT,
            business_level INTEGER DEFAULT 1,
            processing_end_time TEXT,
            daily_collected TEXT,
            total_earned INTEGER DEFAULT 0,
            total_spent INTEGER DEFAULT 0,
            register_date TEXT,
            casino_wins INTEGER DEFAULT 0,
            casino_losses INTEGER DEFAULT 0,
            credit_amount INTEGER DEFAULT 0,
            credit_due_date TEXT,
            job_level INTEGER DEFAULT 1,
            job_exp INTEGER DEFAULT 0,
            total_works INTEGER DEFAULT 0,
            is_banned INTEGER DEFAULT 0,
            last_work_date TEXT,
            is_registered INTEGER DEFAULT 0,
            vip_level INTEGER DEFAULT 0,
            referrer_id INTEGER DEFAULT 0,
            total_referrals INTEGER DEFAULT 0,
            reputation INTEGER DEFAULT 50,
            current_city TEXT DEFAULT 'МОСКВА',
            current_skin TEXT DEFAULT 'default',
            owned_skins TEXT DEFAULT '["default"]',
            daily_streak INTEGER DEFAULT 0,
            last_daily_bonus TEXT,
            last_tax_paid TEXT,
            tax_debt INTEGER DEFAULT 0,
            mortgage_amount INTEGER DEFAULT 0,
            mortgage_monthly INTEGER DEFAULT 0,
            mortgage_remaining INTEGER DEFAULT 0,
            mortgage_property TEXT
        )
    ''')
    
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS achievements (
            user_id INTEGER PRIMARY KEY,
            ach_millionaire INTEGER DEFAULT 0,
            ach_billionaire INTEGER DEFAULT 0,
            ach_gambler INTEGER DEFAULT 0,
            ach_worker INTEGER DEFAULT 0,
            ach_businessman INTEGER DEFAULT 0,
            ach_tycoon INTEGER DEFAULT 0,
            ach_winner INTEGER DEFAULT 0,
            ach_vip INTEGER DEFAULT 0
        )
    ''')
    
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS admins (
            user_id INTEGER PRIMARY KEY,
            added_by INTEGER,
            added_date TEXT
        )
    ''')
    
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS transfers (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            from_user INTEGER,
            to_user INTEGER,
            amount INTEGER,
            transfer_type TEXT,
            date TEXT,
            commission INTEGER DEFAULT 0
        )
    ''')
    
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS referrals (
            referrer_id INTEGER,
            referred_id INTEGER,
            date TEXT,
            PRIMARY KEY (referrer_id, referred_id)
        )
    ''')
    
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS daily_quests (
            user_id INTEGER,
            quest_date TEXT,
            quest1_progress INTEGER DEFAULT 0,
            quest2_progress INTEGER DEFAULT 0,
            quest3_progress INTEGER DEFAULT 0,
            quest1_completed INTEGER DEFAULT 0,
            quest2_completed INTEGER DEFAULT 0,
            quest3_completed INTEGER DEFAULT 0,
            PRIMARY KEY (user_id, quest_date)
        )
    ''')
    
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS stocks (
            user_id INTEGER,
            stock_name TEXT,
            shares INTEGER DEFAULT 0,
            buy_price REAL,
            PRIMARY KEY (user_id, stock_name)
        )
    ''')
    
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS tournaments (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            game_type TEXT,
            start_date TEXT,
            end_date TEXT,
            entry_fee INTEGER,
            prize_pool INTEGER,
            status TEXT DEFAULT 'active',
            winner_id INTEGER,
            second_id INTEGER,
            third_id INTEGER
        )
    ''')
    
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS tournament_participants (
            tournament_id INTEGER,
            user_id INTEGER,
            score INTEGER DEFAULT 0,
            registered_at TEXT,
            PRIMARY KEY (tournament_id, user_id)
        )
    ''')
    
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS logs (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            user_id INTEGER,
            action TEXT,
            amount INTEGER,
            timestamp TEXT
        )
    ''')
    
    cursor.execute('CREATE INDEX IF NOT EXISTS idx_users_balance ON users(balance)')
    cursor.execute('CREATE INDEX IF NOT EXISTS idx_users_username ON users(username)')
    cursor.execute('CREATE INDEX IF NOT EXISTS idx_users_linked ON users(linked_username)')
    
    for aid in ADMIN_IDS:
        cursor.execute('INSERT OR IGNORE INTO admins VALUES (?, ?, ?)', 
                      (aid, aid, datetime.now().strftime("%Y-%m-%d %H:%M:%S")))
    
    conn.commit()
    conn.close()
    print("База данных готова")

# ==================== ФУНКЦИИ БД ====================
def get_user(user_id, username=None):
    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM users WHERE user_id = ?", (user_id,))
    user = cursor.fetchone()
    
    if not user:
        now = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        cursor.execute('''
            INSERT INTO users (user_id, username, register_date, last_tax_paid, owned_skins) 
            VALUES (?, ?, ?, ?, ?)
        ''', (user_id, username, now, now, '["default"]'))
        conn.commit()
        cursor.execute("SELECT * FROM users WHERE user_id = ?", (user_id,))
        user = cursor.fetchone()
        cursor.execute("INSERT OR IGNORE INTO achievements (user_id) VALUES (?)", (user_id,))
        conn.commit()
    elif username:
        cursor.execute("UPDATE users SET username = ? WHERE user_id = ?", (username, user_id))
        conn.commit()
    
    columns = [description[0] for description in cursor.description]
    user_dict = dict(zip(columns, user))
    
    if user_dict.get('owned_skins'):
        try:
            user_dict['owned_skins'] = json.loads(user_dict['owned_skins'])
        except:
            user_dict['owned_skins'] = ['default']
    else:
        user_dict['owned_skins'] = ['default']
    
    cursor.execute("SELECT * FROM achievements WHERE user_id = ?", (user_id,))
    ach = cursor.fetchone()
    if ach:
        ach_columns = [description[0] for description in cursor.description]
        user_dict.update(dict(zip(ach_columns, ach)))
    
    conn.close()
    return user_dict

def update_balance(user_id, amount):
    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()
    cursor.execute("UPDATE users SET balance = balance + ? WHERE user_id = ?", (amount, user_id))
    if amount > 0:
        cursor.execute("UPDATE users SET total_earned = total_earned + ? WHERE user_id = ?", (amount, user_id))
    elif amount < 0:
        cursor.execute("UPDATE users SET total_spent = total_spent + ? WHERE user_id = ?", (abs(amount), user_id))
    conn.commit()
    conn.close()

def update_dollars(user_id, amount):
    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()
    cursor.execute("UPDATE users SET currency_dollar = currency_dollar + ? WHERE user_id = ?", (amount, user_id))
    conn.commit()
    conn.close()

def update_exp(user_id, amount):
    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()
    cursor.execute("UPDATE users SET job_exp = job_exp + ? WHERE user_id = ?", (amount, user_id))
    
    cursor.execute("SELECT job_level, job_exp FROM users WHERE user_id = ?", (user_id,))
    level, exp = cursor.fetchone()
    exp_needed = level * 1000
    while exp >= exp_needed:
        exp -= exp_needed
        level += 1
        exp_needed = level * 1000
        level_bonus = level * 25000
        update_balance(user_id, level_bonus)
        try:
            bot.send_message(user_id, f"ПОВЫШЕНИЕ УРОВНЯ!\n\nВы достигли {level} уровня!\nБонус: {level_bonus}₽")
        except:
            pass
    cursor.execute("UPDATE users SET job_level = ?, job_exp = ? WHERE user_id = ?", (level, exp, user_id))
    conn.commit()
    conn.close()

def update_field(user_id, field, value):
    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()
    cursor.execute(f"UPDATE users SET {field} = ? WHERE user_id = ?", (value, user_id))
    conn.commit()
    conn.close()

def is_admin(user_id):
    if user_id in ADMIN_IDS:
        return True
    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()
    cursor.execute("SELECT 1 FROM admins WHERE user_id = ?", (user_id,))
    result = cursor.fetchone()
    conn.close()
    return result is not None

def get_user_by_name(name):
    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()
    cursor.execute("SELECT user_id, username FROM users WHERE username = ?", (name,))
    result = cursor.fetchone()
    conn.close()
    return result

def get_user_by_username(username):
    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()
    cursor.execute("SELECT user_id FROM users WHERE linked_username = ?", (username.lower(),))
    result = cursor.fetchone()
    conn.close()
    return result

def add_log(user_id, action, amount=0):
    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()
    cursor.execute("INSERT INTO logs (user_id, action, amount, timestamp) VALUES (?, ?, ?, ?)",
                  (user_id, action, amount, datetime.now().strftime("%Y-%m-%d %H:%M:%S")))
    conn.commit()
    conn.close()

# ==================== ФУНКЦИИ ДЛЯ КАРТИНОК ====================
def get_number_image(number):
    path = os.path.join(ROULETTE_IMAGES_FOLDER, f"{number}.jpg")
    if os.path.exists(path):
        return open(path, 'rb')
    return None

def get_skin_image(skin_name):
    skin = SKINS.get(skin_name, SKINS['default'])
    path = os.path.join(SKINS_FOLDER, skin['folder'], "start.jpg")
    if os.path.exists(path):
        return open(path, 'rb')
    default_path = os.path.join(SKINS_FOLDER, "default", "start.jpg")
    if os.path.exists(default_path):
        return open(default_path, 'rb')
    return None

# ==================== НАСТРОЙКИ ИГР ====================
RED_NUMBERS = [1,3,5,7,9,12,14,16,18,19,21,23,25,27,30,32,34,36]

def get_color(number):
    if number == 0:
        return "ЗЕЛЕНОЕ"
    return "КРАСНОЕ" if number in RED_NUMBERS else "ЧЁРНОЕ"

# ==================== РАБОТЫ ====================
JOBS = {
    'ГРУЗЧИК': {'min': 600, 'max': 900, 'req': None, 'exp': 50, 'level': 1},
    'ТАКСИСТ': {'min': 3100, 'max': 9500, 'req': None, 'exp': 80, 'level': 1},
    'ЭЛЕКТРИК': {'min': 3000, 'max': 3000, 'req': None, 'exp': 70, 'level': 1},
    'ДОСТАВЩИК': {'min': 2500, 'max': 4500, 'req': None, 'exp': 60, 'level': 1},
    'КУРЬЕР': {'min': 2000, 'max': 4000, 'req': None, 'exp': 55, 'level': 1},
    'ХАКЕР': {'min': 150000, 'max': 300000, 'req': 'has_laptop', 'exp': 150, 'level': 3},
    'БАНДИТ': {'min': 50000, 'max': 500000, 'req': 'has_gun', 'exp': 200, 'level': 3},
    'МЕХАНИК': {'min': 80000, 'max': 120000, 'req': None, 'exp': 120, 'level': 3},
    'СТРОИТЕЛЬ': {'min': 70000, 'max': 110000, 'req': None, 'exp': 110, 'level': 3},
    'ХУДОЖНИК': {'min': 90000, 'max': 150000, 'req': 'has_laptop', 'exp': 130, 'level': 3},
    'ФАРМАЦЕВТ': {'min': 200000, 'max': 350000, 'req': None, 'exp': 250, 'level': 5},
    'ЮРИСТ': {'min': 250000, 'max': 400000, 'req': None, 'exp': 280, 'level': 5},
    'ФИНАНСИСТ': {'min': 300000, 'max': 450000, 'req': 'has_laptop', 'exp': 300, 'level': 5},
    'УЧЁНЫЙ': {'min': 280000, 'max': 420000, 'req': None, 'exp': 270, 'level': 5},
    'ЛЁТЧИК': {'min': 350000, 'max': 500000, 'req': None, 'exp': 320, 'level': 5},
    'БАНКИР': {'min': 500000, 'max': 800000, 'req': None, 'exp': 400, 'level': 7},
    'УПРАВЛЯЮЩИЙ': {'min': 450000, 'max': 700000, 'req': 'has_laptop', 'exp': 380, 'level': 7},
    'ИНЖЕНЕР': {'min': 600000, 'max': 900000, 'req': None, 'exp': 450, 'level': 7},
    'РАЗВЕДЧИК': {'min': 550000, 'max': 850000, 'req': None, 'exp': 420, 'level': 7},
    'НАЁМНИК': {'min': 700000, 'max': 1000000, 'req': None, 'exp': 500, 'level': 7},
    'ОЛИГАРХ': {'min': 1500000, 'max': 2500000, 'req': None, 'exp': 800, 'level': 10},
    'АВТОРИТЕТ': {'min': 2000000, 'max': 3000000, 'req': 'has_gun', 'exp': 900, 'level': 10},
    'КИБЕРГЕНИЙ': {'min': 1800000, 'max': 2800000, 'req': 'has_laptop', 'exp': 850, 'level': 10},
    'ПРЕЗИДЕНТ': {'min': 3000000, 'max': 5000000, 'req': None, 'exp': 1000, 'level': 10},
}

# ==================== БИЗНЕСЫ ====================
BUSINESSES = {
    'ГАЗПРОМ': {'price': 500000, 'profit': 50000, 'level': 1},
    'ЛУКОЙЛ': {'price': 1000000, 'profit': 100000, 'level': 2},
    'РОСНЕФТЬ': {'price': 2500000, 'profit': 250000, 'level': 3},
    'СБЕРБАНК': {'price': 5000000, 'profit': 500000, 'level': 4},
    'ЯНДЕКС': {'price': 10000000, 'profit': 1000000, 'level': 5},
    'ВТБ': {'price': 20000000, 'profit': 2000000, 'level': 6},
    'НОРНИКЕЛЬ': {'price': 50000000, 'profit': 5000000, 'level': 7},
    'РЖД': {'price': 100000000, 'profit': 10000000, 'level': 8},
    'МТС': {'price': 250000000, 'profit': 25000000, 'level': 9},
    'РОСАТОМ': {'price': 500000000, 'profit': 50000000, 'level': 10},
}

# ==================== VIP СИСТЕМА ====================
VIP_LEVELS = {
    1: {'price': 500000, 'name': 'БРОНЗОВЫЙ', 'bonus': 1.1},
    2: {'price': 2000000, 'name': 'СЕРЕБРЯНЫЙ', 'bonus': 1.2},
    3: {'price': 5000000, 'name': 'ЗОЛОТОЙ', 'bonus': 1.3},
    4: {'price': 15000000, 'name': 'ПЛАТИНОВЫЙ', 'bonus': 1.5},
    5: {'price': 50000000, 'name': 'БРИЛЛИАНТОВЫЙ', 'bonus': 2.0},
}

# ==================== ГОРОДА ====================
CITIES = {
    'МОСКВА': {'bonus': 1.0, 'bonus_type': 'all'},
    'САНКТ-ПЕТЕРБУРГ': {'bonus': 1.1, 'bonus_type': 'creative'},
    'ЕКАТЕРИНБУРГ': {'bonus': 1.1, 'bonus_type': 'factory'},
    'ВЛАДИВОСТОК': {'bonus': 1.1, 'bonus_type': 'delivery'},
    'КРАСНОДАР': {'bonus': 1.1, 'bonus_type': 'rural'},
}

MOVE_COST = 50000

# ==================== АКЦИИ ====================
STOCKS = {
    'ГАЗПРОМ': {'price': 198.50, 'volatility': 0.04},
    'ЛУКОЙЛ': {'price': 7100.00, 'volatility': 0.05},
    'РОСНЕФТЬ': {'price': 540.00, 'volatility': 0.06},
    'СБЕРБАНК': {'price': 280.00, 'volatility': 0.03},
    'ЯНДЕКС': {'price': 4200.00, 'volatility': 0.07},
    'ВТБ': {'price': 0.025, 'volatility': 0.08},
    'НОРНИКЕЛЬ': {'price': 16800.00, 'volatility': 0.05},
    'РЖД': {'price': 120.00, 'volatility': 0.04},
    'МТС': {'price': 285.00, 'volatility': 0.04},
    'ИНТЕР РАО': {'price': 4.50, 'volatility': 0.07},
}

def update_stock_prices():
    for name in STOCKS:
        change = random.uniform(-STOCKS[name]['volatility'], STOCKS[name]['volatility'])
        STOCKS[name]['price'] = max(0.01, STOCKS[name]['price'] * (1 + change))

# ==================== АВТОМОБИЛИ ====================
CARS = {
    'LADA GRANTA': {'price': 500000, 'bonus': 1.05},
    'LADA VESTA': {'price': 800000, 'bonus': 1.07},
    'RENAULT LOGAN': {'price': 1000000, 'bonus': 1.10},
    'KIA RIO': {'price': 1200000, 'bonus': 1.12},
    'HYUNDAI SOLARIS': {'price': 1500000, 'bonus': 1.15},
    'TOYOTA CAMRY': {'price': 3000000, 'bonus': 1.20},
    'HONDA ACCORD': {'price': 3500000, 'bonus': 1.22},
    'BMW 3 SERIES': {'price': 4000000, 'bonus': 1.25},
    'MERCEDES C-CLASS': {'price': 4500000, 'bonus': 1.28},
    'AUDI A4': {'price': 5000000, 'bonus': 1.30},
    'BMW 5 SERIES': {'price': 6000000, 'bonus': 1.35},
    'MERCEDES E-CLASS': {'price': 7000000, 'bonus': 1.40},
    'AUDI A6': {'price': 8000000, 'bonus': 1.45},
    'PORSCHE PANAMERA': {'price': 12000000, 'bonus': 1.50},
    'MERCEDES S-CLASS': {'price': 15000000, 'bonus': 1.60},
    'PORSCHE 911': {'price': 20000000, 'bonus': 1.70},
    'LAMBORGHINI HURACAN': {'price': 30000000, 'bonus': 1.80},
    'FERRARI F8': {'price': 35000000, 'bonus': 1.90},
    'LAMBORGHINI AVENTADOR': {'price': 50000000, 'bonus': 2.00},
    'BUGATTI CHIRON': {'price': 100000000, 'bonus': 2.50},
}

# ==================== ИПОТЕКА ====================
MORTGAGE_PROPERTIES = {
    'КВАРТИРА-СТУДИЯ': {'price': 3000000, 'down_payment': 300000, 'monthly': 30000, 'months': 12, 'daily_income': 5000},
    'ОДНУШКА': {'price': 5000000, 'down_payment': 500000, 'monthly': 50000, 'months': 12, 'daily_income': 10000},
    'ДВУШКА': {'price': 10000000, 'down_payment': 1000000, 'monthly': 100000, 'months': 12, 'daily_income': 20000},
    'ТРЁШКА': {'price': 20000000, 'down_payment': 2000000, 'monthly': 200000, 'months': 12, 'daily_income': 40000},
    'КОТТЕДЖ': {'price': 50000000, 'down_payment': 5000000, 'monthly': 500000, 'months': 12, 'daily_income': 100000},
    'ОСОБНЯК': {'price': 100000000, 'down_payment': 10000000, 'monthly': 1000000, 'months': 12, 'daily_income': 200000},
}

# ==================== ВАЛЮТА ====================
currency_rate = 90.0

def update_currency_rate():
    global currency_rate
    change = random.uniform(-0.05, 0.05)
    currency_rate = max(50, min(150, currency_rate * (1 + change)))
    return currency_rate

# ==================== СЛУЧАЙНЫЕ СОБЫТИЯ ====================
RANDOM_EVENTS = [
    {'name': 'НАШЁЛ КОШЕЛЁК', 'effect': 10000, 'chance': 0.10},
    {'name': 'ШТРАФ ЗА ПАРКОВКУ', 'effect': -5000, 'chance': 0.08},
    {'name': 'ЗАКАЗАЛ ПИЦЦУ', 'effect': -3000, 'chance': 0.12},
    {'name': 'ДЕНЬ РОЖДЕНИЯ', 'effect': 50000, 'chance': 0.03},
    {'name': 'БОЛЬНИЦА', 'effect': -50000, 'chance': 0.05},
    {'name': 'ЛОТЕРЕЯ', 'effect': 15000, 'chance': 0.07},
]

def random_event(user_id):
    if random.random() < 0.3:
        event = random.choice(RANDOM_EVENTS)
        if random.random() < event['chance']:
            update_balance(user_id, event['effect'])
            try:
                if event['effect'] > 0:
                    bot.send_message(user_id, f"СЛУЧАЙНОЕ СОБЫТИЕ\n{event['name']}\n+{event['effect']:,}₽")
                else:
                    bot.send_message(user_id, f"СЛУЧАЙНОЕ СОБЫТИЕ\n{event['name']}\n{event['effect']:,}₽")
            except:
                pass
            return True
    return False

# ==================== ЕЖЕДНЕВНЫЙ СТРИК БОНУС ====================
def daily_streak_bonus(user_id):
    user = get_user(user_id)
    today = datetime.now().strftime("%Y-%m-%d")
    
    if user.get('daily_collected') == today:
        return False
    
    last_bonus = user.get('last_daily_bonus')
    if last_bonus and (datetime.now() - datetime.strptime(last_bonus, "%Y-%m-%d")).days == 1:
        streak = user.get('daily_streak', 0) + 1
    else:
        streak = 1
    
    bonus = min(10000 + (streak - 1) * 5000, 100000)
    update_balance(user_id, bonus)
    update_field(user_id, 'daily_streak', streak)
    update_field(user_id, 'last_daily_bonus', today)
    update_field(user_id, 'daily_collected', today)
    
    try:
        bot.send_message(user_id, f"БОНУС ЗА ВХОД!\nСТРИК: {streak} ДНЕЙ\n+{bonus:,} ₽")
    except:
        pass
    return True

# ==================== КОМАНДА "Я" ====================
@bot.message_handler(func=lambda m: m.text and m.text.lower() == "я")
def i_command(message):
    user = get_user(message.from_user.id)
    name = user.get('custom_name') or user.get('username') or 'ИГРОК'
    balance = user['balance']
    
    text = f"Здарово, {name}! У тебя на счету {balance:,} ₽"
    
    skin_name = user.get('current_skin', 'default')
    img = get_skin_image(skin_name)
    if img:
        bot.send_photo(message.chat.id, img, caption=text)
    else:
        bot.send_message(message.chat.id, text)

# ==================== ТЕКСТОВАЯ РУЛЕТКА ====================
@bot.message_handler(func=lambda m: m.text and m.text.lower().startswith('рул '))
def roulette_text_command(message):
    parts = message.text.lower().split()
    if len(parts) < 3:
        bot.reply_to(message, "ПРИМЕР: рул красное 1000  или  рул красное алл")
        return
    
    bet_type = parts[1]
    user = get_user(message.from_user.id)
    
    try:
        if parts[2] == "алл":
            bet = user['balance']
        else:
            bet = int(parts[2])
        
        if bet < 1 or bet > user['balance']:
            bot.reply_to(message, f"СТАВКА ОТ 1 ДО {user['balance']:,} ₽")
            return
        
        result = random.randint(0, 36)
        color = get_color(result)
        
        win = False
        if bet_type in ['красное', 'red']:
            win = result != 0 and result in RED_NUMBERS
        elif bet_type in ['чёрное', 'черное', 'black']:
            win = result != 0 and result not in RED_NUMBERS
        elif bet_type in ['зеро', 'zero', '0']:
            win = result == 0
        elif bet_type.isdigit() and 0 <= int(bet_type) <= 36:
            win = result == int(bet_type)
        else:
            bot.reply_to(message, "КРАСНОЕ, ЧЁРНОЕ, ЗЕРО ИЛИ ЧИСЛО 0-36")
            return
        
        if win:
            mult = 36 if (bet_type in ['зеро', 'zero', '0'] or bet_type.isdigit()) else 2
            win_amount = bet * mult
            update_balance(message.from_user.id, win_amount)
            update_field(message.from_user.id, 'casino_wins', user.get('casino_wins', 0) + 1)
            user = get_user(message.from_user.id)
            text = f"РУЛЕТКА\nВЫПАЛО: {result} {color}\nВЫИГРЫШ! +{win_amount} ₽\nБАЛАНС: {user['balance']:,} ₽"
        else:
            update_balance(message.from_user.id, -bet)
            update_field(message.from_user.id, 'casino_losses', user.get('casino_losses', 0) + 1)
            user = get_user(message.from_user.id)
            text = f"РУЛЕТКА\nВЫПАЛО: {result} {color}\nПРОИГРЫШ! -{bet} ₽\nБАЛАНС: {user['balance']:,} ₽"
        
        img = get_number_image(result)
        if img:
            bot.send_photo(message.chat.id, img, caption=text)
        else:
            bot.send_message(message.chat.id, text)
            
    except ValueError:
        bot.reply_to(message, "ВВЕДИТЕ ЧИСЛО!")

# ==================== КЛАВИАТУРЫ ====================
def main_keyboard():
    kb = telebot.types.ReplyKeyboardMarkup(resize_keyboard=True, row_width=2)
    kb.row("ИГРЫ", "БИЗНЕС")
    kb.row("ПРОФИЛЬ", "РАБОТА")
    kb.row("КРЕДИТЫ", "ТОП")
    kb.row("БОНУС", "ПРИВЯЗАТЬ")
    kb.row("СКИНЫ", "ВАЛЮТА")
    kb.row("АКЦИИ", "АВТО")
    kb.row("ИПОТЕКА", "ГОРОДА")
    kb.row("ТУРНИРЫ", "ПОМОЩЬ")
    return kb

def admin_keyboard():
    kb = telebot.types.ReplyKeyboardMarkup(resize_keyboard=True, row_width=2)
    kb.row("ВЫДАТЬ ВАЛЮТУ", "ЗАБРАТЬ ВАЛЮТУ")
    kb.row("ВСЕ ИГРОКИ", "СТАТИСТИКА")
    kb.row("ЗАБАНИТЬ", "РАЗБАНИТЬ")
    kb.row("ДОБАВИТЬ АДМИНА", "УДАЛИТЬ АДМИНА")
    kb.row("СПИСОК АДМИНОВ", "БЭКАПЫ")
    kb.row("РАССЫЛКА", "ЛОГИ")
    kb.row("ГЛАВНОЕ МЕНЮ")
    return kb

def games_keyboard():
    kb = telebot.types.ReplyKeyboardMarkup(resize_keyboard=True, row_width=2)
    kb.row("СЛОТЫ", "КОСТИ")
    kb.row("РУЛЕТКА", "МИНЫ")
    kb.row("НАЗАД")
    return kb

def business_keyboard():
    kb = telebot.types.ReplyKeyboardMarkup(resize_keyboard=True, row_width=2)
    kb.row("КУПИТЬ БИЗНЕС", "КУПИТЬ СЫРЬЁ")
    kb.row("ЗАБРАТЬ ДОХОД", "МОЙ БИЗНЕС")
    kb.row("НАЗАД")
    return kb

def work_keyboard(user):
    kb = telebot.types.ReplyKeyboardMarkup(resize_keyboard=True, row_width=2)
    available_jobs = [job for job in JOBS if user['job_level'] >= JOBS[job]['level']]
    for job in available_jobs[:8]:
        kb.row(job)
    kb.row("НАЗАД")
    return kb

def skins_keyboard():
    kb = telebot.types.InlineKeyboardMarkup(row_width=2)
    kb.add(telebot.types.InlineKeyboardButton("МАГАЗИН СКИНОВ", callback_data="shop_skins"))
    kb.add(telebot.types.InlineKeyboardButton("МОИ СКИНЫ", callback_data="my_skins"))
    kb.add(telebot.types.InlineKeyboardButton("НАДЕТЬ СКИН", callback_data="equip_skin"))
    kb.add(telebot.types.InlineKeyboardButton("СНЯТЬ СКИН", callback_data="unequip_skin"))
    return kb

# ==================== ОСНОВНЫЕ КОМАНДЫ ====================
@bot.message_handler(commands=['start'])
def start_command(message):
    user_id = message.from_user.id
    username = message.from_user.username
    user = get_user(user_id, username)
    
    args = message.text.split()
    if len(args) > 1 and args[1].startswith('ref_'):
        try:
            referrer_id = int(args[1].replace('ref_', ''))
            if referrer_id != user_id and not user.get('referrer_id'):
                update_field(user_id, 'referrer_id', referrer_id)
                update_balance(user_id, 25000)
                update_balance(referrer_id, 35000)
                update_field(referrer_id, 'total_referrals', get_user(referrer_id).get('total_referrals', 0) + 1)
                try:
                    bot.send_message(user_id, "+25,000₽ ЗА РЕГИСТРАЦИЮ ПО РЕФЕРАЛЬНОЙ ССЫЛКЕ!")
                    bot.send_message(referrer_id, f"НОВЫЙ РЕФЕРАЛ!\n+35,000₽")
                except:
                    pass
        except:
            pass
    
    if not user.get('is_registered'):
        msg = bot.send_message(message.chat.id, "ДОБРО ПОЖАЛОВАТЬ\n\nПРИДУМАЙТЕ ИМЯ (2-20 СИМВОЛОВ):")
        bot.register_next_step_handler(msg, process_registration)
        return
    
    send_welcome(message.chat.id, user)
    
    if is_admin(user_id):
        bot.send_message(message.chat.id, "ГЛАВНОЕ МЕНЮ (АДМИН)", reply_markup=admin_keyboard())
    else:
        bot.send_message(message.chat.id, "ГЛАВНОЕ МЕНЮ", reply_markup=main_keyboard())

def process_registration(message):
    user_id = message.from_user.id
    name = message.text.strip().upper()
    
    if len(name) < 2 or len(name) > 20:
        bot.send_message(message.chat.id, "2-20 СИМВОЛОВ")
        return
    
    if get_user_by_name(name):
        bot.send_message(message.chat.id, "ИМЯ ЗАНЯТО")
        return
    
    update_field(user_id, 'custom_name', name)
    update_field(user_id, 'is_registered', 1)
    user = get_user(user_id)
    send_welcome(message.chat.id, user)
    
    if is_admin(user_id):
        bot.send_message(message.chat.id, "ГЛАВНОЕ МЕНЮ (АДМИН)", reply_markup=admin_keyboard())
    else:
        bot.send_message(message.chat.id, "ГЛАВНОЕ МЕНЮ", reply_markup=main_keyboard())

def send_welcome(chat_id, user):
    name = user.get('custom_name') or user.get('username') or 'ИГРОК'
    balance = user['balance']
    dollars = user.get('currency_dollar', 0)
    
    text = f"ЗДРАВСТВУЙ, {name}!\n\nНА СЧЕТУ У ТЯ {balance:,} ₽\nДОЛЛАРОВ: {dollars}$\nГОРОД: {user.get('current_city', 'МОСКВА')}"
    
    skin_name = user.get('current_skin', 'default')
    img = get_skin_image(skin_name)
    if img:
        bot.send_photo(chat_id, img, caption=text)
    else:
        bot.send_message(chat_id, text)

# ==================== КНОПКА НАЗАД ====================
@bot.message_handler(func=lambda m: m.text == "ГЛАВНОЕ МЕНЮ")
def back_to_main(message):
    user_id = message.from_user.id
    if is_admin(user_id):
        bot.send_message(message.chat.id, "ГЛАВНОЕ МЕНЮ (АДМИН)", reply_markup=admin_keyboard())
    else:
        bot.send_message(message.chat.id, "ГЛАВНОЕ МЕНЮ", reply_markup=main_keyboard())

@bot.message_handler(func=lambda m: m.text == "НАЗАД")
def back_to_main_from_menu(message):
    user_id = message.from_user.id
    if is_admin(user_id):
        bot.send_message(message.chat.id, "ГЛАВНОЕ МЕНЮ (АДМИН)", reply_markup=admin_keyboard())
    else:
        bot.send_message(message.chat.id, "ГЛАВНОЕ МЕНЮ", reply_markup=main_keyboard())

# ==================== ИГРЫ ====================
@bot.message_handler(func=lambda m: m.text == "ИГРЫ")
def games_menu(message):
    bot.send_message(message.chat.id, "ВЫБЕРИТЕ ИГРУ", reply_markup=games_keyboard())

@bot.message_handler(func=lambda m: m.text == "СЛОТЫ")
def slots_command(message):
    msg = bot.send_message(message.chat.id, "СТАВКА (ОТ 1 ₽):")
    bot.register_next_step_handler(msg, process_slots)

def process_slots(message):
    try:
        bet = int(message.text)
        user = get_user(message.from_user.id)
        
        if bet < 1 or bet > user['balance']:
            bot.send_message(message.chat.id, f"СТАВКА ОТ 1 ДО {user['balance']:,} ₽")
            return
        
        reels = [random.choice(["🍒","🍊","🍋","7️⃣"]) for _ in range(3)]
        win = 0
        
        if reels[0] == reels[1] == reels[2]:
            if reels[0] == "7️⃣":
                win = bet * 10
            else:
                win = bet * 2
        
        if win > 0:
            update_balance(message.from_user.id, win)
            update_field(message.from_user.id, 'casino_wins', user.get('casino_wins', 0) + 1)
            user = get_user(message.from_user.id)
            text = f"{reels[0]} {reels[1]} {reels[2]}\nВЫИГРЫШ! +{win} ₽\nБАЛАНС: {user['balance']:,} ₽"
        else:
            update_balance(message.from_user.id, -bet)
            update_field(message.from_user.id, 'casino_losses', user.get('casino_losses', 0) + 1)
            user = get_user(message.from_user.id)
            text = f"{reels[0]} {reels[1]} {reels[2]}\nПРОИГРЫШ! -{bet} ₽\nБАЛАНС: {user['balance']:,} ₽"
        
        bot.send_message(message.chat.id, text)
    except ValueError:
        bot.send_message(message.chat.id, "ВВЕДИТЕ ЧИСЛО!")

@bot.message_handler(func=lambda m: m.text == "КОСТИ")
def dice_command(message):
    msg = bot.send_message(message.chat.id, "СТАВКА (ОТ 1 ₽):")
    bot.register_next_step_handler(msg, process_dice)

def process_dice(message):
    try:
        bet = int(message.text)
        user = get_user(message.from_user.id)
        
        if bet < 1 or bet > user['balance']:
            bot.send_message(message.chat.id, f"СТАВКА ОТ 1 ДО {user['balance']:,} ₽")
            return
        
        d1, d2 = random.randint(1, 6), random.randint(1, 6)
        s = d1 + d2
        win = 0
        
        if s == 7:
            win = bet * 3
        elif s in [2, 12]:
            win = bet * 5
        
        if win > 0:
            update_balance(message.from_user.id, win)
            update_field(message.from_user.id, 'casino_wins', user.get('casino_wins', 0) + 1)
            user = get_user(message.from_user.id)
            text = f"{d1}+{d2}={s}\nВЫИГРЫШ! +{win} ₽\nБАЛАНС: {user['balance']:,} ₽"
        else:
            update_balance(message.from_user.id, -bet)
            update_field(message.from_user.id, 'casino_losses', user.get('casino_losses', 0) + 1)
            user = get_user(message.from_user.id)
            text = f"{d1}+{d2}={s}\nПРОИГРЫШ! -{bet} ₽\nБАЛАНС: {user['balance']:,} ₽"
        
        bot.send_message(message.chat.id, text)
    except ValueError:
        bot.send_message(message.chat.id, "ВВЕДИТЕ ЧИСЛО!")

@bot.message_handler(func=lambda m: m.text == "РУЛЕТКА")
def roulette_command(message):
    kb = telebot.types.InlineKeyboardMarkup(row_width=2)
    kb.add(telebot.types.InlineKeyboardButton("КРАСНОЕ", callback_data="roulette_red"))
    kb.add(telebot.types.InlineKeyboardButton("ЧЁРНОЕ", callback_data="roulette_black"))
    kb.add(telebot.types.InlineKeyboardButton("ЗЕРО", callback_data="roulette_zero"))
    kb.add(telebot.types.InlineKeyboardButton("ЧИСЛО", callback_data="roulette_number"))
    bot.send_message(message.chat.id, "РУЛЕТКА", reply_markup=kb)

@bot.callback_query_handler(func=lambda call: call.data.startswith('roulette_'))
def roulette_callback(call):
    bet_type = call.data.replace('roulette_', '')
    if bet_type == 'number':
        msg = bot.send_message(call.message.chat.id, "ВВЕДИТЕ ЧИСЛО (0-36):")
        bot.register_next_step_handler(msg, process_roulette_number)
    else:
        msg = bot.send_message(call.message.chat.id, f"СТАВКА НА {bet_type}:")
        bot.register_next_step_handler(msg, process_roulette_bet, bet_type)

def process_roulette_number(message):
    try:
        number = int(message.text)
        if number < 0 or number > 36:
            bot.send_message(message.chat.id, "0-36")
            return
        msg = bot.send_message(message.chat.id, f"СТАВКА НА ЧИСЛО {number}:")
        bot.register_next_step_handler(msg, process_roulette_bet, f"number_{number}")
    except ValueError:
        bot.send_message(message.chat.id, "ВВЕДИТЕ ЧИСЛО!")

def process_roulette_bet(message, bet_type):
    try:
        bet = int(message.text)
        user = get_user(message.from_user.id)
        
        if bet < 1 or bet > user['balance']:
            bot.send_message(message.chat.id, f"СТАВКА ОТ 1 ДО {user['balance']:,} ₽")
            return
        
        result = random.randint(0, 36)
        color = get_color(result)
        
        win = False
        mult = 2
        
        if bet_type == 'red':
            win = result != 0 and result in RED_NUMBERS
        elif bet_type == 'black':
            win = result != 0 and result not in RED_NUMBERS
        elif bet_type == 'zero':
            win = result == 0
            mult = 36
        elif bet_type.startswith('number_'):
            target = int(bet_type.split('_')[1])
            win = result == target
            mult = 36
        
        if win:
            win_amount = bet * mult
            update_balance(message.from_user.id, win_amount)
            update_field(message.from_user.id, 'casino_wins', user.get('casino_wins', 0) + 1)
            user = get_user(message.from_user.id)
            text = f"РУЛЕТКА\nВЫПАЛО: {result} {color}\nВЫИГРЫШ! +{win_amount} ₽\nБАЛАНС: {user['balance']:,} ₽"
        else:
            update_balance(message.from_user.id, -bet)
            update_field(message.from_user.id, 'casino_losses', user.get('casino_losses', 0) + 1)
            user = get_user(message.from_user.id)
            text = f"РУЛЕТКА\nВЫПАЛО: {result} {color}\nПРОИГРЫШ! -{bet} ₽\nБАЛАНС: {user['balance']:,} ₽"
        
        img = get_number_image(result)
        if img:
            bot.send_photo(message.chat.id, img, caption=text)
        else:
            bot.send_message(message.chat.id, text)
    except ValueError:
        bot.send_message(message.chat.id, "ВВЕДИТЕ ЧИСЛО!")

@bot.message_handler(func=lambda m: m.text == "МИНЫ")
def mines_command(message):
    msg = bot.send_message(message.chat.id, "СТАВКА (ОТ 1 ₽):")
    bot.register_next_step_handler(msg, process_mines_bet)

mines_games = {}

def process_mines_bet(message):
    try:
        bet = int(message.text)
        user = get_user(message.from_user.id)
        
        if bet < 1 or bet > user['balance']:
            bot.send_message(message.chat.id, f"СТАВКА ОТ 1 ДО {user['balance']:,} ₽")
            return
        
        mines = random.sample(range(9), 3)
        game = {
            'bet': bet,
            'mines': mines,
            'opened': [],
            'mult': [1.5, 2.5, 4, 6, 8, 10],
            'timestamp': time.time()
        }
        mines_games[message.chat.id] = game
        show_mines_board(message.chat.id, game)
    except ValueError:
        bot.send_message(message.chat.id, "ВВЕДИТЕ ЧИСЛО!")

def show_mines_board(chat_id, game):
    kb = telebot.types.InlineKeyboardMarkup(row_width=3)
    for i in range(9):
        if i in game['opened']:
            text = "✅" if i not in game['mines'] else "💣"
        else:
            text = "❓"
        kb.add(telebot.types.InlineKeyboardButton(text, callback_data=f"mine_{i}"))
    kb.add(telebot.types.InlineKeyboardButton("ЗАБРАТЬ", callback_data="mine_cashout"))
    bot.send_message(chat_id, f"МИНЫ\nСТАВКА: {game['bet']} ₽\nОТКРЫТО: {len(game['opened'])}/6", reply_markup=kb)

@bot.callback_query_handler(func=lambda call: call.data.startswith('mine_'))
def mine_callback(call):
    game = mines_games.get(call.message.chat.id)
    if not game:
        bot.answer_callback_query(call.id, "ИГРА НЕ НАЙДЕНА")
        return
    
    if call.data == "mine_cashout":
        if len(game['opened']) == 0:
            bot.answer_callback_query(call.id, "НЕТ ОТКРЫТЫХ КЛЕТОК")
            return
        mult = game['mult'][min(len(game['opened'])-1, 5)]
        win = int(game['bet'] * mult)
        update_balance(call.message.chat.id, win)
        user = get_user(call.message.chat.id)
        bot.edit_message_text(f"ВЫИГРЫШ! +{win} ₽\nБАЛАНС: {user['balance']:,} ₽", call.message.chat.id, call.message.message_id)
        del mines_games[call.message.chat.id]
        return
    
    cell = int(call.data.split('_')[1])
    if cell in game['opened']:
        bot.answer_callback_query(call.id, "УЖЕ ОТКРЫТО")
        return
    if cell in game['mines']:
        update_balance(call.message.chat.id, -game['bet'])
        user = get_user(call.message.chat.id)
        bot.edit_message_text(f"МИНА! -{game['bet']} ₽\nБАЛАНС: {user['balance']:,} ₽", call.message.chat.id, call.message.message_id)
        del mines_games[call.message.chat.id]
        return
    
    game['opened'].append(cell)
    if len(game['opened']) == 6:
        win = int(game['bet'] * 10)
        update_balance(call.message.chat.id, win)
        user = get_user(call.message.chat.id)
        bot.edit_message_text(f"ДЖЕКПОТ! +{win} ₽\nБАЛАНС: {user['balance']:,} ₽", call.message.chat.id, call.message.message_id)
        del mines_games[call.message.chat.id]
        return
    show_mines_board(call.message.chat.id, game)
    bot.answer_callback_query(call.id, "БЕЗОПАСНО")

# ==================== БИЗНЕС ====================
@bot.message_handler(func=lambda m: m.text == "БИЗНЕС")
def business_menu(message):
    bot.send_message(message.chat.id, "УПРАВЛЕНИЕ БИЗНЕСОМ", reply_markup=business_keyboard())

@bot.message_handler(func=lambda m: m.text == "КУПИТЬ БИЗНЕС")
def buy_business_list(message):
    kb = telebot.types.InlineKeyboardMarkup(row_width=2)
    for name, info in BUSINESSES.items():
        kb.add(telebot.types.InlineKeyboardButton(f"{name} - {info['price']:,}₽", callback_data=f"biz_{name}"))
    bot.send_message(message.chat.id, "ВЫБЕРИТЕ БИЗНЕС", reply_markup=kb)

@bot.callback_query_handler(func=lambda call: call.data.startswith('biz_'))
def buy_biz_callback(call):
    biz_name = call.data[4:]
    user = get_user(call.message.chat.id)
    
    if user.get('business_owned'):
        bot.answer_callback_query(call.id, "У ВАС УЖЕ ЕСТЬ БИЗНЕС!")
        return
    
    info = BUSINESSES[biz_name]
    if user['balance'] < info['price']:
        bot.answer_callback_query(call.id, f"НУЖНО {info['price'] - user['balance']:,} ₽")
        return
    
    update_balance(call.message.chat.id, -info['price'])
    update_field(call.message.chat.id, 'business_owned', biz_name)
    bot.answer_callback_query(call.id, f"{biz_name} КУПЛЕН!")
    business_menu(call.message)

@bot.message_handler(func=lambda m: m.text == "КУПИТЬ СЫРЬЁ")
def buy_raw(message):
    user = get_user(message.from_user.id)
    
    if not user.get('business_owned'):
        bot.send_message(message.chat.id, "НЕТ БИЗНЕСА!")
        return
    
    if user.get('processing_end_time'):
        end = datetime.strptime(user['processing_end_time'], "%Y-%m-%d %H:%M:%S")
        if end > datetime.now():
            bot.send_message(message.chat.id, "ПЕРЕРАБОТКА УЖЕ ИДЁТ!")
            return
    
    if user['balance'] < 10000:
        bot.send_message(message.chat.id, "НУЖНО 10,000 ₽")
        return
    
    update_balance(message.from_user.id, -10000)
    end_time = (datetime.now() + timedelta(hours=1)).strftime("%Y-%m-%d %H:%M:%S")
    update_field(message.from_user.id, 'processing_end_time', end_time)
    bot.send_message(message.chat.id, "СЫРЬЁ КУПЛЕНО! ПЕРЕРАБОТКА 1 ЧАС.")

@bot.message_handler(func=lambda m: m.text == "ЗАБРАТЬ ДОХОД")
def collect_profit(message):
    user = get_user(message.from_user.id)
    
    if not user.get('business_owned'):
        bot.send_message(message.chat.id, "НЕТ БИЗНЕСА!")
        return
    
    if not user.get('processing_end_time'):
        bot.send_message(message.chat.id, "НЕТ ПЕРЕРАБОТКИ!")
        return
    
    end = datetime.strptime(user['processing_end_time'], "%Y-%m-%d %H:%M:%S")
    if end > datetime.now():
        bot.send_message(message.chat.id, "ПЕРЕРАБОТКА ЕЩЁ НЕ ЗАКОНЧЕНА!")
        return
    
    info = BUSINESSES.get(user['business_owned'], {})
    profit = info.get('profit', 50000)
    vip_bonus = VIP_LEVELS.get(user.get('vip_level', 0), {}).get('bonus', 1.0)
    profit = int(profit * vip_bonus)
    
    update_balance(message.from_user.id, profit)
    update_field(message.from_user.id, 'processing_end_time', None)
    bot.send_message(message.chat.id, f"ДОХОД ПОЛУЧЕН! +{profit:,} ₽")

@bot.message_handler(func=lambda m: m.text == "МОЙ БИЗНЕС")
def my_business(message):
    user = get_user(message.from_user.id)
    if not user.get('business_owned'):
        bot.send_message(message.chat.id, "НЕТ БИЗНЕСА!")
        return
    
    info = BUSINESSES[user['business_owned']]
    text = f"{user['business_owned']}\nСЫРЬЁ: 10,000 ₽\nДОХОД: {info['profit']:,} ₽"
    bot.send_message(message.chat.id, text)

# ==================== РАБОТА ====================
@bot.message_handler(func=lambda m: m.text == "РАБОТА")
def work_menu(message):
    user = get_user(message.from_user.id)
    
    if user.get('last_work_date'):
        last = datetime.strptime(user['last_work_date'], "%Y-%m-%d %H:%M:%S")
        if (datetime.now() - last).total_seconds() < WORK_COOLDOWN:
            rem = int(WORK_COOLDOWN - (datetime.now() - last).total_seconds())
            bot.send_message(message.chat.id, f"ОТДЫХАЙ! ЧЕРЕЗ {rem//60} МИН")
            return
    
    bot.send_message(message.chat.id, "ВЫБЕРИТЕ РАБОТУ", reply_markup=work_keyboard(user))

@bot.message_handler(func=lambda m: m.text in JOBS)
def do_work(message):
    job = JOBS[message.text]
    user = get_user(message.from_user.id)
    
    if user['job_level'] < job['level']:
        bot.send_message(message.chat.id, f"НУЖЕН УРОВЕНЬ {job['level']}")
        return
    
    if job['req'] and not user.get(job['req']):
        bot.send_message(message.chat.id, f"НУЖЕН ПРЕДМЕТ ИЗ МАГАЗИНА")
        return
    
    vip_bonus = VIP_LEVELS.get(user.get('vip_level', 0), {}).get('bonus', 1.0)
    car_bonus = CARS.get(user.get('has_car', ''), {}).get('bonus', 1.0) if user.get('has_car') else 1.0
    city_bonus = CITIES.get(user.get('current_city', 'МОСКВА'), {}).get('bonus', 1.0)
    
    earn = int(random.randint(job['min'], job['max']) * vip_bonus * car_bonus * city_bonus)
    update_balance(message.from_user.id, earn)
    update_exp(message.from_user.id, job['exp'])
    update_field(message.from_user.id, 'last_work_date', datetime.now().strftime("%Y-%m-%d %H:%M:%S"))
    update_field(message.from_user.id, 'total_works', user.get('total_works', 0) + 1)
    random_event(message.from_user.id)
    
    bot.send_message(message.chat.id, f"{message.text}\n+{earn:,} ₽\n+{job['exp']} EXP")

# ==================== ПРОФИЛЬ ====================
@bot.message_handler(func=lambda m: m.text == "ПРОФИЛЬ")
def profile_menu(message):
    user = get_user(message.from_user.id)
    vip_name = VIP_LEVELS.get(user.get('vip_level', 0), {}).get('name', 'НЕТ')
    current_skin = SKINS.get(user.get('current_skin', 'default'), {}).get('name', 'СТАНДАРТНЫЙ')
    current_car = user.get('has_car', 'НЕТ') or 'НЕТ'
    car_bonus = CARS.get(current_car, {}).get('bonus', 1.0) if current_car != 'НЕТ' else 1.0
    
    text = f"ВАШ ПРОФИЛЬ\n\n"
    text += f"ИМЯ: {user.get('custom_name') or user['username']}\n"
    text += f"БАЛАНС: {user['balance']:,} ₽\n"
    text += f"ДОЛЛАРОВ: {user.get('currency_dollar', 0)}$\n"
    text += f"УРОВЕНЬ: {user['job_level']}\n"
    text += f"ОПЫТ: {user['job_exp']} / {user['job_level'] * 1000}\n"
    text += f"ПОБЕД: {user.get('casino_wins', 0)} | ПОРАЖЕНИЙ: {user.get('casino_losses', 0)}\n"
    text += f"ЗАРАБОТАНО: {user['total_earned']:,} ₽\n"
    text += f"ЮЗЕРНЕЙМ: @{user.get('linked_username') or 'НЕ ПРИВЯЗАН'}\n"
    text += f"СКИН: {current_skin}\n"
    text += f"АВТО: {current_car} (БОНУС: +{int((car_bonus-1)*100)}%)\n"
    text += f"VIP: {vip_name}\n"
    text += f"ГОРОД: {user.get('current_city', 'МОСКВА')}\n"
    text += f"РЕФЕРАЛОВ: {user.get('total_referrals', 0)}"
    
    bot.send_message(message.chat.id, text)

# ==================== КРЕДИТЫ ====================
@bot.message_handler(func=lambda m: m.text == "КРЕДИТЫ")
def credits_menu(message):
    kb = telebot.types.InlineKeyboardMarkup(row_width=2)
    kb.add(telebot.types.InlineKeyboardButton("ВЗЯТЬ КРЕДИТ", callback_data="credit_take"))
    kb.add(telebot.types.InlineKeyboardButton("ПОГАСИТЬ КРЕДИТ", callback_data="credit_pay"))
    kb.add(telebot.types.InlineKeyboardButton("МОЙ КРЕДИТ", callback_data="credit_my"))
    bot.send_message(message.chat.id, "КРЕДИТЫ\nДО 50,000,000₽\nСРОК: 7 ДНЕЙ\nПРОЦЕНТ: 7%", reply_markup=kb)

@bot.callback_query_handler(func=lambda call: call.data.startswith('credit_'))
def credit_callback(call):
    user = get_user(call.message.chat.id)
    
    if call.data == 'credit_take':
        if user.get('credit_amount', 0) > 0:
            bot.answer_callback_query(call.id, "У ВАС УЖЕ ЕСТЬ КРЕДИТ!")
            return
        msg = bot.send_message(call.message.chat.id, "ВВЕДИТЕ СУММУ (100,000 - 50,000,000 ₽):")
        bot.register_next_step_handler(msg, process_take_credit)
    
    elif call.data == 'credit_pay':
        if user.get('credit_amount', 0) == 0:
            bot.answer_callback_query(call.id, "НЕТ КРЕДИТА!")
            return
        debt = int(user['credit_amount'] * 1.07)
        if user['balance'] < debt:
            bot.answer_callback_query(call.id, f"НУЖНО {debt - user['balance']:,} ₽")
            return
        update_balance(call.message.chat.id, -debt)
        update_field(call.message.chat.id, 'credit_amount', 0)
        update_field(call.message.chat.id, 'credit_due_date', None)
        bot.answer_callback_query(call.id, f"КРЕДИТ ПОГАШЕН! -{debt:,} ₽")
    
    elif call.data == 'credit_my':
        if user.get('credit_amount', 0) == 0:
            bot.answer_callback_query(call.id, "НЕТ КРЕДИТА!")
            return
        debt = int(user['credit_amount'] * 1.07)
        days_left = (datetime.strptime(user['credit_due_date'], "%Y-%m-%d") - datetime.now()).days
        text = f"ИНФОРМАЦИЯ О КРЕДИТЕ\n\n"
        text += f"СУММА: {user['credit_amount']:,} ₽\n"
        text += f"ВЕРНУТЬ: {debt:,} ₽\n"
        text += f"СРОК: {user['credit_due_date']}\n"
        text += f"ОСТАЛОСЬ ДНЕЙ: {max(0, days_left)}"
        bot.send_message(call.message.chat.id, text)

def process_take_credit(message):
    try:
        amount = int(message.text.strip())
        if amount < 100000 or amount > 50000000:
            bot.send_message(message.chat.id, "СУММА ОТ 100,000 ДО 50,000,000 ₽")
            return
        
        user_id = message.from_user.id
        user = get_user(user_id)
        
        if user.get('credit_amount', 0) > 0:
            bot.send_message(message.chat.id, "У ВАС УЖЕ ЕСТЬ КРЕДИТ!")
            return
        
        due_date = (datetime.now() + timedelta(days=7)).strftime("%Y-%m-%d")
        update_balance(user_id, amount)
        update_field(user_id, 'credit_amount', amount)
        update_field(user_id, 'credit_due_date', due_date)
        
        debt = int(amount * 1.07)
        bot.send_message(message.chat.id, f"КРЕДИТ ОДОБРЕН!\n+{amount:,} ₽\nВЕРНУТЬ: {debt:,} ₽\nСРОК: {due_date}")
    except ValueError:
        bot.send_message(message.chat.id, "ВВЕДИТЕ ЧИСЛО!")

# ==================== СКИНЫ ====================
@bot.message_handler(func=lambda m: m.text == "СКИНЫ")
def skins_menu(message):
    bot.send_message(message.chat.id, "УПРАВЛЕНИЕ СКИНАМИ", reply_markup=skins_keyboard())

@bot.callback_query_handler(func=lambda call: call.data in ['shop_skins', 'my_skins', 'equip_skin', 'unequip_skin'])
def skins_callbacks(call):
    user = get_user(call.message.chat.id)
    
    if call.data == 'shop_skins':
        text = "МАГАЗИН СКИНОВ\n\n"
        for skin_id, skin in SKINS.items():
            owned = "✅" if skin_id in user['owned_skins'] else "❌"
            price_info = f"{skin['price']:,} ₽" if skin['price'] > 0 else "БЕСПЛАТНО"
            text += f"{owned} {skin['name']} — {price_info}\n"
        
        kb = telebot.types.InlineKeyboardMarkup(row_width=2)
        for skin_id, skin in SKINS.items():
            if skin_id not in user['owned_skins'] and skin_id != 'default':
                kb.add(telebot.types.InlineKeyboardButton(f"КУПИТЬ {skin['name']}", callback_data=f"buy_skin_{skin_id}"))
        kb.add(telebot.types.InlineKeyboardButton("НАЗАД", callback_data="back_to_skins"))
        bot.edit_message_text(text, call.message.chat.id, call.message.message_id, reply_markup=kb)
    
    elif call.data == 'my_skins':
        owned = [s for s in user['owned_skins'] if s in SKINS]
        if len(owned) == 1 and owned[0] == 'default':
            text = "У ВАС НЕТ СКИНОВ, КРОМЕ СТАНДАРТНОГО"
        else:
            text = "ВАШИ СКИНЫ\n\n"
            for skin_id in owned:
                current = "⭐" if user.get('current_skin') == skin_id else "📦"
                text += f"{current} {SKINS[skin_id]['name']}\n"
        
        kb = telebot.types.InlineKeyboardMarkup()
        kb.add(telebot.types.InlineKeyboardButton("НАЗАД", callback_data="back_to_skins"))
        bot.edit_message_text(text, call.message.chat.id, call.message.message_id, reply_markup=kb)
    
    elif call.data == 'equip_skin':
        owned = [s for s in user['owned_skins'] if s in SKINS and s != 'default']
        if not owned:
            bot.answer_callback_query(call.id, "У ВАС НЕТ СКИНОВ! КУПИТЕ В МАГАЗИНЕ.")
            return
        
        text = "ВЫБЕРИТЕ СКИН ДЛЯ НАДЕВАНИЯ\n\n"
        kb = telebot.types.InlineKeyboardMarkup(row_width=2)
        for skin_id in owned:
            kb.add(telebot.types.InlineKeyboardButton(SKINS[skin_id]['name'], callback_data=f"equip_skin_{skin_id}"))
        kb.add(telebot.types.InlineKeyboardButton("НАЗАД", callback_data="back_to_skins"))
        bot.edit_message_text(text, call.message.chat.id, call.message.message_id, reply_markup=kb)
    
    elif call.data == 'unequip_skin':
        if user.get('current_skin') == 'default':
            bot.answer_callback_query(call.id, "НА ВАС УЖЕ СТАНДАРТНЫЙ СКИН")
            return
        
        update_field(call.message.chat.id, 'current_skin', 'default')
        bot.answer_callback_query(call.id, "СКИН СНЯТ! НАДЕТ СТАНДАРТНЫЙ СКИН.")
        bot.edit_message_text("СКИН УСПЕШНО СНЯТ!\n\nТЕПЕРЬ НА ВАС СТАНДАРТНЫЙ СКИН.", call.message.chat.id, call.message.message_id)

@bot.callback_query_handler(func=lambda call: call.data.startswith('buy_skin_'))
def buy_skin_callback(call):
    skin_id = call.data.replace('buy_skin_', '')
    user = get_user(call.message.chat.id)
    skin = SKINS.get(skin_id)
    
    if not skin:
        bot.answer_callback_query(call.id, "СКИН НЕ НАЙДЕН")
        return
    
    if skin_id in user['owned_skins']:
        bot.answer_callback_query(call.id, "У ВАС УЖЕ ЕСТЬ ЭТОТ СКИН")
        return
    
    if user['balance'] < skin['price']:
        bot.answer_callback_query(call.id, f"НУЖНО {skin['price'] - user['balance']:,} ₽")
        return
    
    update_balance(call.message.chat.id, -skin['price'])
    user['owned_skins'].append(skin_id)
    update_field(call.message.chat.id, 'owned_skins', json.dumps(user['owned_skins']))
    update_field(call.message.chat.id, 'current_skin', skin_id)
    
    bot.answer_callback_query(call.id, f"КУПЛЕН И НАДЕТ СКИН {skin['name']}!")
    bot.send_message(call.message.chat.id, f"КУПЛЕН СКИН {skin['name']}\n-{skin['price']:,} ₽\nСКИН АВТОМАТИЧЕСКИ НАДЕТ!")

@bot.callback_query_handler(func=lambda call: call.data.startswith('equip_skin_'))
def equip_skin_callback(call):
    skin_id = call.data.replace('equip_skin_', '')
    user = get_user(call.message.chat.id)
    skin = SKINS.get(skin_id)
    
    if not skin:
        bot.answer_callback_query(call.id, "СКИН НЕ НАЙДЕН")
        return
    
    if skin_id not in user['owned_skins']:
        bot.answer_callback_query(call.id, "У ВАС НЕТ ЭТОГО СКИНА")
        return
    
    update_field(call.message.chat.id, 'current_skin', skin_id)
    bot.answer_callback_query(call.id, f"НАДЕТ СКИН {skin['name']}!")

@bot.callback_query_handler(func=lambda call: call.data == 'back_to_skins')
def back_to_skins(call):
    skins_menu(call.message)

# ==================== ВАЛЮТА ====================
@bot.message_handler(func=lambda m: m.text == "ВАЛЮТА")
def currency_menu(message):
    update_currency_rate()
    user = get_user(message.from_user.id)
    
    kb = telebot.types.InlineKeyboardMarkup(row_width=2)
    kb.add(telebot.types.InlineKeyboardButton("КУПИТЬ $", callback_data="buy_dollars"))
    kb.add(telebot.types.InlineKeyboardButton("ПРОДАТЬ $", callback_data="sell_dollars"))
    
    text = f"ОБМЕН ВАЛЮТЫ\n\nКУРС: 1$ = {currency_rate:.2f} ₽\nРУБЛИ: {user['balance']:,} ₽\nДОЛЛАРЫ: {user.get('currency_dollar', 0)}$"
    bot.send_message(message.chat.id, text, reply_markup=kb)

@bot.callback_query_handler(func=lambda call: call.data in ['buy_dollars', 'sell_dollars'])
def currency_callback(call):
    if call.data == 'buy_dollars':
        msg = bot.send_message(call.message.chat.id, f"ВВЕДИТЕ СУММУ В РУБЛЯХ (КУРС: {currency_rate:.2f} ₽):")
        bot.register_next_step_handler(msg, process_buy_dollars)
    elif call.data == 'sell_dollars':
        msg = bot.send_message(call.message.chat.id, f"ВВЕДИТЕ СУММУ В ДОЛЛАРАХ (КУРС: {currency_rate:.2f} ₽):")
        bot.register_next_step_handler(msg, process_sell_dollars)

def process_buy_dollars(message):
    try:
        rub = int(message.text)
        if rub < 1000:
            bot.send_message(message.chat.id, "МИНИМУМ 1,000 ₽")
            return
        
        user = get_user(message.from_user.id)
        if user['balance'] < rub:
            bot.send_message(message.chat.id, "НЕ ХВАТАЕТ РУБЛЕЙ")
            return
        
        dollars = int(rub / currency_rate)
        update_balance(message.from_user.id, -rub)
        update_dollars(message.from_user.id, dollars)
        bot.send_message(message.chat.id, f"КУПЛЕНО {dollars}$ ПО КУРСУ 1$ = {currency_rate:.2f} ₽")
    except ValueError:
        bot.send_message(message.chat.id, "ВВЕДИТЕ ЧИСЛО!")

def process_sell_dollars(message):
    try:
        dollars = int(message.text)
        if dollars < 1:
            bot.send_message(message.chat.id, "МИНИМУМ 1$")
            return
        
        user = get_user(message.from_user.id)
        if user.get('currency_dollar', 0) < dollars:
            bot.send_message(message.chat.id, "НЕ ХВАТАЕТ ДОЛЛАРОВ")
            return
        
        rub = int(dollars * currency_rate)
        update_dollars(message.from_user.id, -dollars)
        update_balance(message.from_user.id, rub)
        bot.send_message(message.chat.id, f"ПРОДАНО {dollars}$ ЗА {rub} ₽ ПО КУРСУ 1$ = {currency_rate:.2f} ₽")
    except ValueError:
        bot.send_message(message.chat.id, "ВВЕДИТЕ ЧИСЛО!")

# ==================== АКЦИИ ====================
@bot.message_handler(func=lambda m: m.text == "АКЦИИ")
def stocks_menu(message):
    update_stock_prices()
    
    text = "ФОНДОВЫЙ РЫНОК\n\n"
    for name, info in STOCKS.items():
        text += f"{name} — {info['price']:.2f} ₽\n"
    
    kb = telebot.types.InlineKeyboardMarkup(row_width=2)
    kb.add(telebot.types.InlineKeyboardButton("КУПИТЬ", callback_data="buy_stock"))
    kb.add(telebot.types.InlineKeyboardButton("ПРОДАТЬ", callback_data="sell_stock"))
    kb.add(telebot.types.InlineKeyboardButton("МОИ АКЦИИ", callback_data="my_stocks"))
    bot.send_message(message.chat.id, text, reply_markup=kb)

@bot.callback_query_handler(func=lambda call: call.data in ['buy_stock', 'sell_stock', 'my_stocks'])
def stocks_callback(call):
    if call.data == 'my_stocks':
        conn = sqlite3.connect(DB_NAME)
        cursor = conn.cursor()
        cursor.execute("SELECT stock_name, shares, buy_price FROM stocks WHERE user_id = ?", (call.message.chat.id,))
        stocks = cursor.fetchall()
        conn.close()
        
        if not stocks:
            bot.send_message(call.message.chat.id, "У ВАС НЕТ АКЦИЙ")
            return
        
        text = "МОИ АКЦИИ\n\n"
        total_value = 0
        for stock_name, shares, buy_price in stocks:
            current_price = STOCKS.get(stock_name, {}).get('price', buy_price)
            value = int(current_price * shares)
            profit = value - int(buy_price * shares)
            emoji = "📈" if profit >= 0 else "📉"
            text += f"{emoji} {stock_name}: {shares} ШТ.\n"
            text += f"   СТОИМОСТЬ: {value:,} ₽\n"
            text += f"   ПРИБЫЛЬ: {profit:+,} ₽\n\n"
            total_value += value
        text += f"ОБЩАЯ СТОИМОСТЬ: {total_value:,} ₽"
        bot.send_message(call.message.chat.id, text)
    
    else:
        msg = bot.send_message(call.message.chat.id, "ВВЕДИТЕ НАЗВАНИЕ АКЦИИ:")
        bot.register_next_step_handler(msg, process_stock_name, call.data)

def process_stock_name(message, action):
    stock_name = message.text.strip().upper()
    if stock_name not in STOCKS:
        bot.send_message(message.chat.id, "АКЦИЯ НЕ НАЙДЕНА")
        return
    
    msg = bot.send_message(message.chat.id, f"ВВЕДИТЕ КОЛИЧЕСТВО АКЦИЙ {stock_name}:")
        bot.register_next_step_handler(msg, process_stock_amount, action, stock_name)

def process_stock_amount(message, action, stock_name):
    try:
        shares = int(message.text)
        if shares < 1:
            bot.send_message(message.chat.id, "МИНИМУМ 1 АКЦИЯ")
            return
        
        user = get_user(message.from_user.id)
        price = STOCKS[stock_name]['price']
        total = int(price * shares)
        
        if action == 'buy_stock':
            if user['balance'] < total:
                bot.send_message(message.chat.id, f"НУЖНО {total - user['balance']:,} ₽")
                return
            
            update_balance(message.from_user.id, -total)
            
            conn = sqlite3.connect(DB_NAME)
            cursor = conn.cursor()
            cursor.execute('''
                INSERT INTO stocks (user_id, stock_name, shares, buy_price)
                VALUES (?, ?, ?, ?)
                ON CONFLICT(user_id, stock_name) DO UPDATE SET
                shares = shares + ?, buy_price = (buy_price * shares + ?) / (shares + ?)
            ''', (message.from_user.id, stock_name, shares, price, shares, total, shares))
            conn.commit()
            conn.close()
            
            bot.send_message(message.chat.id, f"КУПЛЕНО {shares} АКЦИЙ {stock_name} ЗА {total:,} ₽")
        
        elif action == 'sell_stock':
            conn = sqlite3.connect(DB_NAME)
            cursor = conn.cursor()
            cursor.execute("SELECT shares, buy_price FROM stocks WHERE user_id = ? AND stock_name = ?", 
                          (message.from_user.id, stock_name))
            result = cursor.fetchone()
            
            if not result or result[0] < shares:
                bot.send_message(message.chat.id, "У ВАС НЕТ СТОЛЬКО АКЦИЙ")
                conn.close()
                return
            
            total_sell = int(price * shares)
            update_balance(message.from_user.id, total_sell)
            
            new_shares = result[0] - shares
            if new_shares == 0:
                cursor.execute("DELETE FROM stocks WHERE user_id = ? AND stock_name = ?", 
                              (message.from_user.id, stock_name))
            else:
                cursor.execute("UPDATE stocks SET shares = ? WHERE user_id = ? AND stock_name = ?", 
                              (new_shares, message.from_user.id, stock_name))
            
            conn.commit()
            conn.close()
            
            profit = total_sell - int(result[1] * shares)
            bot.send_message(message.chat.id, f"ПРОДАНО {shares} АКЦИЙ {stock_name} ЗА {total_sell:,} ₽ (ПРИБЫЛЬ: {profit:+,} ₽)")
    
    except ValueError:
        bot.send_message(message.chat.id, "ВВЕДИТЕ ЧИСЛО!")

# ==================== АВТОМОБИЛИ ====================
@bot.message_handler(func=lambda m: m.text == "АВТО")
def cars_menu(message):
    user = get_user(message.from_user.id)
    text = "АВТОМОБИЛИ\n\n"
    for car, info in CARS.items():
        owned = "✅" if user.get('has_car') == car else "❌"
        text += f"{owned} {car} — {info['price']:,} ₽ (+{int((info['bonus']-1)*100)}%)\n"
    
    kb = telebot.types.InlineKeyboardMarkup(row_width=2)
    for car in CARS:
        if user.get('has_car') != car:
            kb.add(telebot.types.InlineKeyboardButton(car, callback_data=f"buy_car_{car}"))
    bot.send_message(message.chat.id, text, reply_markup=kb)

@bot.callback_query_handler(func=lambda call: call.data.startswith('buy_car_'))
def buy_car_callback(call):
    car_name = call.data.replace('buy_car_', '')
    user = get_user(call.message.chat.id)
    car = CARS.get(car_name)
    
    if not car:
        bot.answer_callback_query(call.id, "АВТО НЕ НАЙДЕНО")
        return
    
    if user['balance'] < car['price']:
        bot.answer_callback_query(call.id, f"НУЖНО {car['price'] - user['balance']:,} ₽")
        return
    
    old_car = user.get('has_car')
    if old_car and old_car != 'None':
        old_price = CARS.get(old_car, {}).get('price', 0)
        refund = int(old_price * 0.7)
        update_balance(call.message.chat.id, refund)
        bot.send_message(call.message.chat.id, f"СТАРАЯ МАШИНА ПРОДАНА ЗА {refund:,} ₽")
    
    update_balance(call.message.chat.id, -car['price'])
    update_field(call.message.chat.id, 'has_car', car_name)
    bot.answer_callback_query(call.id, f"КУПЛЕН {car_name}!")
    profile_menu(call.message)

# ==================== ИПОТЕКА ====================
@bot.message_handler(func=lambda m: m.text == "ИПОТЕКА")
def mortgage_menu(message):
    user = get_user(message.from_user.id)
    
    if user.get('mortgage_amount', 0) > 0:
        text = f"АКТИВНАЯ ИПОТЕКА\n\n"
        text += f"НЕДВИЖИМОСТЬ: {user.get('mortgage_property', 'НЕИЗВЕСТНО')}\n"
        text += f"ОСТАТОК ДОЛГА: {user['mortgage_amount']:,} ₽\n"
        text += f"ОСТАЛОСЬ МЕСЯЦЕВ: {user.get('mortgage_remaining', 0)}\n"
        text += f"ЕЖЕМЕСЯЧНЫЙ ПЛАТЁЖ: {user.get('mortgage_monthly', 0):,} ₽\n"
        text += f"\nПОСЛЕ ВЫПЛАТЫ БУДЕТЕ ПОЛУЧАТЬ {MORTGAGE_PROPERTIES.get(user.get('mortgage_property', ''), {}).get('daily_income', 0):,} ₽/ДЕНЬ"
        bot.send_message(message.chat.id, text)
        return
    
    text = "КУПИТЬ НЕДВИЖИМОСТЬ В ИПОТЕКУ\n\n"
    for prop, info in MORTGAGE_PROPERTIES.items():
        text += f"{prop}\n"
        text += f"   ЦЕНА: {info['price']:,} ₽\n"
        text += f"   ПЕРВЫЙ ВЗНОС: {info['down_payment']:,} ₽\n"
        text += f"   ПЛАТЁЖ: {info['monthly']:,} ₽/МЕС, {info['months']} МЕС\n"
        text += f"   ДОХОД ПОСЛЕ ВЫПЛАТЫ: {info['daily_income']:,} ₽/ДЕНЬ\n\n"
    
    kb = telebot.types.InlineKeyboardMarkup(row_width=2)
    for prop in MORTGAGE_PROPERTIES:
        kb.add(telebot.types.InlineKeyboardButton(prop, callback_data=f"mortgage_{prop}"))
    bot.send_message(message.chat.id, text, reply_markup=kb)

@bot.callback_query_handler(func=lambda call: call.data.startswith('mortgage_'))
def mortgage_callback(call):
    prop_name = call.data.replace('mortgage_', '')
    prop = MORTGAGE_PROPERTIES.get(prop_name)
    user = get_user(call.message.chat.id)
    
    if not prop:
        bot.answer_callback_query(call.id, "НЕ НАЙДЕНО")
        return
    
    if user.get('mortgage_amount', 0) > 0:
        bot.answer_callback_query(call.id, "У ВАС УЖЕ ЕСТЬ ИПОТЕКА!")
        return
    
    if user['balance'] < prop['down_payment']:
        bot.answer_callback_query(call.id, f"НУЖНО {prop['down_payment'] - user['balance']:,} ₽")
        return
    
    update_balance(call.message.chat.id, -prop['down_payment'])
    update_field(call.message.chat.id, 'mortgage_amount', prop['price'] - prop['down_payment'])
    update_field(call.message.chat.id, 'mortgage_monthly', prop['monthly'])
    update_field(call.message.chat.id, 'mortgage_remaining', prop['months'])
    update_field(call.message.chat.id, 'mortgage_property', prop_name)
    
    bot.answer_callback_query(call.id, f"ИПОТЕКА НА {prop_name} ОФОРМЛЕНА!")

# ==================== ГОРОДА ====================
@bot.message_handler(func=lambda m: m.text == "ГОРОДА")
def cities_menu(message):
    user = get_user(message.from_user.id)
    current_city = user.get('current_city', 'МОСКВА')
    
    text = f"ГОРОДА РОССИИ\n\nТЕКУЩИЙ ГОРОД: {current_city}\nСТОИМОСТЬ ПЕРЕЕЗДА: {MOVE_COST:,} ₽\n\n"
    for city, info in CITIES.items():
        bonus_text = "ВСЕ РАБОТЫ" if info['bonus_type'] == 'all' else f"{info['bonus_type']} РАБОТЫ"
        marker = "⭐" if city == current_city else "📍"
        text += f"{marker} {city} — +{int((info['bonus']-1)*100)}% К {bonus_text}\n"
    
    kb = telebot.types.InlineKeyboardMarkup(row_width=2)
    for city in CITIES:
        if city != current_city:
            kb.add(telebot.types.InlineKeyboardButton(city, callback_data=f"move_{city}"))
    bot.send_message(message.chat.id, text, reply_markup=kb)

@bot.callback_query_handler(func=lambda call: call.data.startswith('move_'))
def move_city_callback(call):
    city = call.data.replace('move_', '')
    user = get_user(call.message.chat.id)
    
    if user['balance'] < MOVE_COST:
        bot.answer_callback_query(call.id, f"НУЖНО {MOVE_COST - user['balance']:,} ₽")
        return
    
    update_balance(call.message.chat.id, -MOVE_COST)
    update_field(call.message.chat.id, 'current_city', city)
    bot.answer_callback_query(call.id, f"ПЕРЕЕХАЛИ В {city}!")
    cities_menu(call.message)

# ==================== ТУРНИРЫ ====================
@bot.message_handler(func=lambda m: m.text == "ТУРНИРЫ")
def tournaments_menu(message):
    text = "ЕЖЕНЕДЕЛЬНЫЕ ТУРНИРЫ\n\n"
    text += "ПОНЕДЕЛЬНИК: СЛОТЫ (10,000₽)\n"
    text += "СРЕДА: КОСТИ (10,000₽)\n"
    text += "ПЯТНИЦА: РУЛЕТКА (25,000₽)\n"
    text += "ВОСКРЕСЕНЬЕ: МИНЫ (15,000₽)\n\n"
    text += "ПРИЗОВОЙ ФОНД: 70% ОТ ВЗНОСОВ\n"
    text += "1 МЕСТО: 50%\n"
    text += "2 МЕСТО: 30%\n"
    text += "3 МЕСТО: 20%"
    
    kb = telebot.types.InlineKeyboardMarkup()
    kb.add(telebot.types.InlineKeyboardButton("АКТИВНЫЕ ТУРНИРЫ", callback_data="active_tournaments"))
    bot.send_message(message.chat.id, text, reply_markup=kb)

@bot.callback_query_handler(func=lambda call: call.data == 'active_tournaments')
def active_tournaments(call):
    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()
    cursor.execute("SELECT id, game_type, entry_fee, prize_pool FROM tournaments WHERE status = 'active'")
    tournaments = cursor.fetchall()
    conn.close()
    
    if not tournaments:
        bot.send_message(call.message.chat.id, "НЕТ АКТИВНЫХ ТУРНИРОВ")
        return
    
    text = "АКТИВНЫЕ ТУРНИРЫ\n\n"
    for t in tournaments:
        text += f"{t[1]}\n"
        text += f"ВЗНОС: {t[2]:,} ₽\n"
        text += f"ПРИЗОВОЙ ФОНД: {t[3]:,} ₽\n\n"
    
    bot.send_message(call.message.chat.id, text)

# ==================== БОНУС ====================
@bot.message_handler(func=lambda m: m.text == "БОНУС")
def bonus_command(message):
    daily_streak_bonus(message.from_user.id)

# ==================== ТОП ====================
@bot.message_handler(func=lambda m: m.text == "ТОП")
def top_command(message):
    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()
    cursor.execute("SELECT custom_name, username, balance FROM users ORDER BY balance DESC LIMIT 10")
    top = cursor.fetchall()
    conn.close()
    
    text = "ТОП-10 БОГАТЕЙШИХ\n\n"
    medals = ["🥇", "🥈", "🥉"]
    for i, (custom_name, username, balance) in enumerate(top, 1):
        name = custom_name or username or f"ИГРОК{i}"
        medal = medals[i-1] if i <= 3 else f"{i}."
        text += f"{medal} {name} — {balance:,} ₽\n"
    bot.send_message(message.chat.id, text)

# ==================== ПРИВЯЗКА ЮЗЕРНЕЙМА ====================
@bot.message_handler(func=lambda m: m.text == "ПРИВЯЗАТЬ")
def link_command(message):
    user = get_user(message.from_user.id)
    
    if user.get('linked_username'):
        bot.send_message(message.chat.id, f"У ВАС УЖЕ ПРИВЯЗАН ЮЗЕРНЕЙМ @{user['linked_username']}")
        return
    
    if user.get('link_reward_given', 0) == 1:
        bot.send_message(message.chat.id, "ВЫ УЖЕ ПОЛУЧАЛИ НАГРАДУ ЗА ПРИВЯЗКУ!")
        return
    
    msg = bot.send_message(message.chat.id, "ВВЕДИТЕ ВАШ ЮЗЕРНЕЙМ (БЕЗ @):")
    bot.register_next_step_handler(msg, process_link)

def process_link(message):
    username = message.text.strip().lower()
    user_id = message.from_user.id
    
    if len(username) < 3:
        bot.send_message(message.chat.id, "МИНИМУМ 3 СИМВОЛА")
        return
    
    if get_user_by_username(username):
        bot.send_message(message.chat.id, "ЮЗЕРНЕЙМ УЖЕ ЗАНЯТ")
        return
    
    update_field(user_id, 'linked_username', username)
    update_field(user_id, 'link_reward_given', 1)
    update_balance(user_id, 25000)
    bot.send_message(message.chat.id, f"ПРИВЯЗАН @{username}\n+25,000 ₽")

# ==================== ПОМОЩЬ ====================
@bot.message_handler(func=lambda m: m.text == "ПОМОЩЬ")
def help_command(message):
    text = """БОТ БАНДИТ - ПОМОЩЬ

ОСНОВНЫЕ КОМАНДЫ:
/start - НАЧАТЬ ИГРУ
/balance - БАЛАНС
/myid - МОЙ ID
/setname - СМЕНИТЬ ИМЯ
/referral - РЕФЕРАЛЬНАЯ ССЫЛКА

ПЕРЕВОДЫ (КОМИССИЯ 5%):
/transfer @username СУММА - РУБЛИ
/transfer_dollar @username СУММА - ДОЛЛАРЫ
/transfer_exp @username СУММА - ОПЫТ

ИГРЫ:
СЛОТЫ - 3 БАРАБАНА
КОСТИ - 2 КУБИКА
РУЛЕТКА - КРАСНОЕ/ЧЁРНОЕ/ЗЕРО
МИНЫ - ПОЛЕ 3x3

ТЕКСТОВАЯ РУЛЕТКА:
рул красное 1000
рул красное алл
рул чёрное 500
рул зеро 10000
рул 7 5000

ЭКОНОМИКА:
РАБОТА - 24 ПРОФЕССИИ
БИЗНЕС - 10 РОССИЙСКИХ КОМПАНИЙ
КРЕДИТЫ - ДО 50,000,000 ₽ (7%)
ВАЛЮТА - ПОКУПКА/ПРОДАЖА ДОЛЛАРОВ
АКЦИИ - 10 РОССИЙСКИХ АКЦИЙ
АВТО - 20 МАШИН С БОНУСАМИ
ИПОТЕКА - КУПИТЬ НЕДВИЖИМОСТЬ
ГОРОДА - 5 ГОРОДОВ С БОНУСАМИ

БОНУСЫ:
БОНУС - ЕЖЕДНЕВНЫЙ СТРИК БОНУС
СКИНЫ - МЕНЯЙ ВНЕШНОСТЬ
ТУРНИРЫ - ЕЖЕНЕДЕЛЬНЫЕ СОРЕВНОВАНИЯ

УДАЧИ!"""
    bot.send_message(message.chat.id, text)

# ==================== АДМИН-ПАНЕЛЬ (КНОПКИ) ====================
@bot.message_handler(func=lambda m: m.text == "ВЫДАТЬ ВАЛЮТУ")
def admin_give_menu(message):
    if not is_admin(message.from_user.id):
        return
    msg = bot.send_message(message.chat.id, "ВВЕДИТЕ ID ПОЛЬЗОВАТЕЛЯ И СУММУ:\nПРИМЕР: 123456789 100000")
    bot.register_next_step_handler(msg, process_admin_give)

def process_admin_give(message):
    if not is_admin(message.from_user.id):
        return
    try:
        parts = message.text.split()
        if len(parts) != 2:
            bot.send_message(message.chat.id, "НЕВЕРНЫЙ ФОРМАТ! ИСПОЛЬЗУЙТЕ: ID СУММА")
            return
        
        user_id = int(parts[0])
        amount = int(parts[1])
        
        update_balance(user_id, amount)
        add_log(message.from_user.id, f"admin_give_to_{user_id}", amount)
        bot.send_message(message.chat.id, f"ВЫДАНО {amount:,} ₽ ПОЛЬЗОВАТЕЛЮ {user_id}")
        try:
            bot.send_message(user_id, f"👑 ВАМ ВЫДАНО {amount:,} ₽ АДМИНИСТРАТОРОМ!")
        except:
            pass
    except:
        bot.send_message(message.chat.id, "ОШИБКА! ФОРМАТ: ID СУММА")

@bot.message_handler(func=lambda m: m.text == "ЗАБРАТЬ ВАЛЮТУ")
def admin_take_menu(message):
    if not is_admin(message.from_user.id):
        return
    msg = bot.send_message(message.chat.id, "ВВЕДИТЕ ID ПОЛЬЗОВАТЕЛЯ И СУММУ:\nПРИМЕР: 123456789 50000\nИЛИ: 123456789 ВСЕ")
    bot.register_next_step_handler(msg, process_admin_take)

def process_admin_take(message):
    if not is_admin(message.from_user.id):
        return
    try:
        parts = message.text.split()
        if len(parts) != 2:
            bot.send_message(message.chat.id, "НЕВЕРНЫЙ ФОРМАТ! ИСПОЛЬЗУЙТЕ: ID СУММА ИЛИ ID ВСЕ")
            return
        
        user_id = int(parts[0])
        
        if parts[1].upper() == "ВСЕ":
            user = get_user(user_id)
            amount = user['balance']
        else:
            amount = int(parts[1])
        
        update_balance(user_id, -amount)
        add_log(message.from_user.id, f"admin_take_from_{user_id}", amount)
        bot.send_message(message.chat.id, f"ЗАБРАНО {amount:,} ₽ У ПОЛЬЗОВАТЕЛЯ {user_id}")
        try:
            bot.send_message(user_id, f"👑 У ВАС ЗАБРАЛИ {amount:,} ₽ АДМИНИСТРАТОРОМ!")
        except:
            pass
    except:
        bot.send_message(message.chat.id, "ОШИБКА! ФОРМАТ: ID СУММА ИЛИ ID ВСЕ")

@bot.message_handler(func=lambda m: m.text == "ВСЕ ИГРОКИ")
def admin_list_players(message):
    if not is_admin(message.from_user.id):
        return
    
    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()
    cursor.execute("SELECT user_id, custom_name, username, balance FROM users ORDER BY balance DESC LIMIT 50")
    users = cursor.fetchall()
    conn.close()
    
    text = "СПИСОК ИГРОКОВ (ТОП-50)\n\n"
    for uid, custom_name, username, balance in users:
        name = custom_name or username or f"ID:{uid}"
        text += f"{name} — {balance:,} ₽ (ID: {uid})\n"
    
    if len(text) > 4000:
        text = text[:4000] + "\n\n... И ДРУГИЕ"
    
    bot.send_message(message.chat.id, text)

@bot.message_handler(func=lambda m: m.text == "СТАТИСТИКА")
def admin_stats(message):
    if not is_admin(message.from_user.id):
        return
    
    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()
    cursor.execute("SELECT COUNT(*) FROM users")
    total_users = cursor.fetchone()[0]
    cursor.execute("SELECT SUM(balance) FROM users")
    total_balance = cursor.fetchone()[0] or 0
    cursor.execute("SELECT COUNT(*) FROM users WHERE is_banned=1")
    banned_users = cursor.fetchone()[0]
    cursor.execute("SELECT COUNT(*) FROM admins")
    admins_count = cursor.fetchone()[0]
    conn.close()
    
    text = f"СТАТИСТИКА СЕРВЕРА\n\n"
    text += f"ВСЕГО ИГРОКОВ: {total_users}\n"
    text += f"ОБЩИЙ БАЛАНС: {total_balance:,} ₽\n"
    text += f"ЗАБАНОВ: {banned_users}\n"
    text += f"АДМИНОВ: {admins_count}\n"
    text += f"СТАТУС: АКТИВЕН"
    
    bot.send_message(message.chat.id, text)

@bot.message_handler(func=lambda m: m.text == "ЗАБАНИТЬ")
def admin_ban(message):
    if not is_admin(message.from_user.id):
        return
    msg = bot.send_message(message.chat.id, "ВВЕДИТЕ ID ПОЛЬЗОВАТЕЛЯ ДЛЯ БЛОКИРОВКИ:")
    bot.register_next_step_handler(msg, process_admin_ban)

def process_admin_ban(message):
    if not is_admin(message.from_user.id):
        return
    try:
        user_id = int(message.text)
        update_field(user_id, 'is_banned', 1)
        add_log(message.from_user.id, f"admin_banned_{user_id}", 0)
        bot.send_message(message.chat.id, f"ПОЛЬЗОВАТЕЛЬ {user_id} ЗАБЛОКИРОВАН")
        try:
            bot.send_message(user_id, "⛔ ВАШ АККАУНТ ЗАБЛОКИРОВАН!\n\nОБРАТИТЕСЬ К АДМИНИСТРАЦИИ.")
        except:
            pass
    except:
        bot.send_message(message.chat.id, "ОШИБКА! ВВЕДИТЕ ID ЧИСЛОМ")

@bot.message_handler(func=lambda m: m.text == "РАЗБАНИТЬ")
def admin_unban(message):
    if not is_admin(message.from_user.id):
        return
    msg = bot.send_message(message.chat.id, "ВВЕДИТЕ ID ПОЛЬЗОВАТЕЛЯ ДЛЯ РАЗБЛОКИРОВКИ:")
    bot.register_next_step_handler(msg, process_admin_unban)

def process_admin_unban(message):
    if not is_admin(message.from_user.id):
        return
    try:
        user_id = int(message.text)
        update_field(user_id, 'is_banned', 0)
        add_log(message.from_user.id, f"admin_unbanned_{user_id}", 0)
        bot.send_message(message.chat.id, f"ПОЛЬЗОВАТЕЛЬ {user_id} РАЗБЛОКИРОВАН")
        try:
            bot.send_message(user_id, "✅ ВАШ АККАУНТ РАЗБЛОКИРОВАН!\n\nПРИЯТНОЙ ИГРЫ!")
        except:
            pass
    except:
        bot.send_message(message.chat.id, "ОШИБКА! ВВЕДИТЕ ID ЧИСЛОМ")

@bot.message_handler(func=lambda m: m.text == "ДОБАВИТЬ АДМИНА")
def admin_add_admin(message):
    if not is_admin(message.from_user.id):
        return
    msg = bot.send_message(message.chat.id, "ВВЕДИТЕ ID НОВОГО АДМИНА:")
    bot.register_next_step_handler(msg, process_admin_add_admin)

def process_admin_add_admin(message):
    if not is_admin(message.from_user.id):
        return
    try:
        new_admin_id = int(message.text)
        conn = sqlite3.connect(DB_NAME)
        cursor = conn.cursor()
        cursor.execute("INSERT OR IGNORE INTO admins VALUES (?, ?, ?)", 
                      (new_admin_id, message.from_user.id, datetime.now().strftime("%Y-%m-%d %H:%M:%S")))
        conn.commit()
        conn.close()
        bot.send_message(message.chat.id, f"ПОЛЬЗОВАТЕЛЬ {new_admin_id} ДОБАВЛЕН В АДМИНЫ")
        try:
            bot.send_message(new_admin_id, "👑 ВАМ ВЫДАНЫ ПРАВА АДМИНИСТРАТОРА!")
        except:
            pass
    except:
        bot.send_message(message.chat.id, "ОШИБКА! ВВЕДИТЕ ID ЧИСЛОМ")

@bot.message_handler(func=lambda m: m.text == "УДАЛИТЬ АДМИНА")
def admin_remove_admin(message):
    if not is_admin(message.from_user.id):
        return
    msg = bot.send_message(message.chat.id, "ВВЕДИТЕ ID АДМИНА ДЛЯ УДАЛЕНИЯ:")
    bot.register_next_step_handler(msg, process_admin_remove_admin)

def process_admin_remove_admin(message):
    if not is_admin(message.from_user.id):
        return
    try:
        admin_id = int(message.text)
        if admin_id in ADMIN_IDS:
            bot.send_message(message.chat.id, "НЕЛЬЗЯ УДАЛИТЬ ГЛАВНОГО АДМИНА!")
            return
        
        conn = sqlite3.connect(DB_NAME)
        cursor = conn.cursor()
        cursor.execute("DELETE FROM admins WHERE user_id = ?", (admin_id,))
        conn.commit()
        conn.close()
        bot.send_message(message.chat.id, f"ПОЛЬЗОВАТЕЛЬ {admin_id} УДАЛЁН ИЗ АДМИНОВ")
        try:
            bot.send_message(admin_id, "👑 ВАШИ ПРАВА АДМИНИСТРАТОРА ОТОЗВАНЫ!")
        except:
            pass
    except:
        bot.send_message(message.chat.id, "ОШИБКА! ВВЕДИТЕ ID ЧИСЛОМ")

@bot.message_handler(func=lambda m: m.text == "СПИСОК АДМИНОВ")
def admin_list_admins(message):
    if not is_admin(message.from_user.id):
        return
    
    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()
    cursor.execute("SELECT user_id, added_by, added_date FROM admins")
    admins = cursor.fetchall()
    conn.close()
    
    text = "СПИСОК АДМИНИСТРАТОРОВ\n\n"
    for admin_id, added_by, added_date in admins:
        user = get_user(admin_id)
        name = user.get('custom_name') or user.get('username') or f"ID:{admin_id}"
        text += f"• {name} (ID: {admin_id})\n"
        if added_date:
            text += f"  ДОБАВЛЕН: {added_date}\n"
        text += "\n"
    
    bot.send_message(message.chat.id, text)

@bot.message_handler(func=lambda m: m.text == "БЭКАПЫ")
def admin_backup(message):
    if not is_admin(message.from_user.id):
        return
    
    # Создаём бэкап
    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
    backup_name = f"backup_{timestamp}.db"
    shutil.copy2(DB_NAME, os.path.join(BACKUP_FOLDER, backup_name))
    
    bot.send_message(message.chat.id, f"✅ БЭКАП СОЗДАН: {backup_name}")

@bot.message_handler(func=lambda m: m.text == "РАССЫЛКА")
def admin_mailing(message):
    if not is_admin(message.from_user.id):
        return
    msg = bot.send_message(message.chat.id, "ВВЕДИТЕ ТЕКСТ ДЛЯ РАССЫЛКИ:")
    bot.register_next_step_handler(msg, process_admin_mailing)

def process_admin_mailing(message):
    if not is_admin(message.from_user.id):
        return
    
    text = message.text
    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()
    cursor.execute("SELECT user_id FROM users WHERE is_banned = 0")
    users = cursor.fetchall()
    conn.close()
    
    success = 0
    fail = 0
    
    for user in users:
        try:
            bot.send_message(user[0], f"📢 *РАССЫЛКА ОТ АДМИНИСТРАЦИИ*\n\n{text}", parse_mode="Markdown")
            success += 1
        except:
            fail += 1
        time.sleep(0.05)
    
    bot.send_message(message.chat.id, f"РАССЫЛКА ЗАВЕРШЕНА!\n✅ ДОСТАВЛЕНО: {success}\n❌ НЕ ДОСТАВЛЕНО: {fail}")

@bot.message_handler(func=lambda m: m.text == "ЛОГИ")
def admin_logs(message):
    if not is_admin(message.from_user.id):
        return
    
    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()
    cursor.execute("SELECT id, user_id, action, amount, timestamp FROM logs ORDER BY id DESC LIMIT 50")
    logs = cursor.fetchall()
    conn.close()
    
    if not logs:
        bot.send_message(message.chat.id, "ЛОГОВ НЕТ")
        return
    
    text = "ПОСЛЕДНИЕ 50 ЛОГОВ\n\n"
    for log in logs:
        text += f"#{log[0]} | {log[4]} | {log[1]} | {log[2]} | {log[3]}\n"
    
    if len(text) > 4000:
        text = text[:4000] + "\n\n... И ДРУГИЕ"
    
    bot.send_message(message.chat.id, text)

# ==================== ЗАПУСК ====================
if __name__ == "__main__":
    init_db()
    
    def background_tasks():
        while True:
            try:
                time.sleep(3600)
                update_stock_prices()
                update_currency_rate()
                check_overdue_credits()
                print("Фоновые задачи выполнены")
            except Exception as e:
                print(f"Ошибка в фоновых задачах: {e}")
                time.sleep(60)
    
    background_thread = threading.Thread(target=background_tasks, daemon=True)
    background_thread.start()
    
    print("=" * 60)
    print("БОТ ЗАПУЩЕН")
    print("АДМИН ID: 5120305949")
    print("=" * 60)
    print("ДОСТУПНЫЕ ФУНКЦИИ:")
    print("   • 4 ИГРЫ (СЛОТЫ, КОСТИ, РУЛЕТКА, МИНЫ)")
    print("   • 24 РАБОТЫ")
    print("   • 10 БИЗНЕСОВ")
    print("   • 5 ГОРОДОВ С БОНУСАМИ")
    print("   • 10 РОССИЙСКИХ АКЦИЙ")
    print("   • 20 АВТОМОБИЛЕЙ")
    print("   • 6 ОБЪЕКТОВ ИПОТЕКИ")
    print("   • 5 VIP УРОВНЕЙ")
    print("   • 11 СКИНОВ")
    print("   • АДМИН-ПАНЕЛЬ (13 ФУНКЦИЙ)")
    print("   • КОМАНДА 'я' - ПРИВЕТСТВИЕ")
    print("   • ТЕКСТОВАЯ РУЛЕТКА (рул красное 1000)")
    print("=" * 60)
    
    while True:
        try:
            bot.infinity_polling(timeout=60)
        except Exception as e:
            print(f"ОШИБКА: {e}")
            time.sleep(10)
