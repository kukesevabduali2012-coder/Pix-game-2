import telebot
from telebot import types
import random
import json
import os
from datetime import datetime
import threading
import time

TOKEN = '8507161577:AAFKpdJnZffLU03ybdehqTkJS2V_qTDsmMw'
bot = telebot.TeleBot(TOKEN)

ADMIN_ID = 7501838776
DATA_FILE = 'users_data.json'
PROMO_FILE = 'promo_codes.json'
ADMINS_FILE = 'admins.json'
EVENTS_FILE = 'events.json'
GIVEAWAYS_FILE = 'giveaways.json'
MARKET_FILE = 'market.json'

save_lock = threading.Lock()

def load_json(filename, default=None):
    try:
        if os.path.exists(filename):
            with open(filename, 'r', encoding='utf-8') as f:
                return json.load(f)
    except Exception as e:
        print(f"Ошибка загрузки {filename}: {e}")
    return default if default is not None else {}

def save_json(filename, data):
    try:
        with save_lock:
            with open(filename, 'w', encoding='utf-8') as f:
                json.dump(data, f, ensure_ascii=False, indent=2)
    except Exception as e:
        print(f"Ошибка сохранения {filename}: {e}")

users = load_json(DATA_FILE)
promo_codes = load_json(PROMO_FILE)
sub_admins = load_json(ADMINS_FILE, [])
events = load_json(EVENTS_FILE, {'active': None, 'history': [], 'claimed': {}})
giveaways = load_json(GIVEAWAYS_FILE, {'active': [], 'history': []})
market = load_json(MARKET_FILE, {'pixg_offers': [], 'pix_offers': []})

def save_data(data):
    save_json(DATA_FILE, data)

def save_promos(promos):
    save_json(PROMO_FILE, promos)

def save_admins():
    save_json(ADMINS_FILE, sub_admins)

def save_events():
    save_json(EVENTS_FILE, events)

def save_giveaways():
    save_json(GIVEAWAYS_FILE, giveaways)

def save_market():
    save_json(MARKET_FILE, market)

def init_user(user_id):
    user_id = str(user_id)
    if user_id not in users:
        users[user_id] = {
            'balance': 5000,
            'pixg': 0,
            'last_bonus': None,
            'activated_promos': [],
            'banned': False,
            'muted': False,
            'in_top3': False,
            'hilo_game': None,
            'blackjack_game': None,
            'wins': 0,
            'losses': 0,
            'total_won': 0,
            'total_lost': 0,
            'total_games': 0,
            'giveaway_entries': []
        }
        save_data(users)
    else:
        defaults = {
            'pixg': 0, 'wins': 0, 'losses': 0, 'total_won': 0,
            'total_lost': 0, 'total_games': 0, 'activated_promos': [],
            'giveaway_entries': []
        }
        for key, val in defaults.items():
            if key not in users[user_id]:
                users[user_id][key] = val

def format_num(num):
    return f"{num:,}".replace(',', ' ')

def is_admin(user_id):
    return user_id == ADMIN_ID or str(user_id) in sub_admins

def is_sub_admin(user_id):
    return str(user_id) in sub_admins

def main_menu(user_id=None):
    markup = types.ReplyKeyboardMarkup(resize_keyboard=True, row_width=2)
    buttons = [
        types.KeyboardButton('👤 Профиль'),
        types.KeyboardButton('💰 Баланс'),
        types.KeyboardButton('🎁 Бонус'),
        types.KeyboardButton('🎮 Игры'),
        types.KeyboardButton('🎫 Промокоды'),
        types.KeyboardButton('💎 Донаты'),
        types.KeyboardButton('🏆 Лидерборд'),
        types.KeyboardButton('🎉 Ивенты'),
        types.KeyboardButton('🛒 Продажа'),
        types.KeyboardButton('📞 Контакты'),
        types.KeyboardButton('❓ Помощь')
    ]
    
    if user_id == ADMIN_ID or str(user_id) in sub_admins:
        buttons.append(types.KeyboardButton('👑 Админка'))
    
    markup.add(*buttons)
    return markup

def admin_menu(user_id):
    markup = types.ReplyKeyboardMarkup(resize_keyboard=True, row_width=2)
    buttons = [
        types.KeyboardButton('👤 Управление игроками'),
        types.KeyboardButton('📊 Статистика бота'),
        types.KeyboardButton('🔙 Выйти')
    ]
    
    # Полный доступ только для главного админа
    if user_id == ADMIN_ID:
        buttons.insert(1, types.KeyboardButton('💰 Управление балансом'))
        buttons.insert(2, types.KeyboardButton('🎫 Админ промо'))
        buttons.insert(3, types.KeyboardButton('📢 Общее сообщение'))
        buttons.insert(4, types.KeyboardButton('🎉 Управление ивентами'))
        buttons.insert(5, types.KeyboardButton('🎁 Раздачи'))
        buttons.insert(6, types.KeyboardButton('👥 Зам админы'))
    
    markup.add(*buttons)
    return markup

def games_keyboard():
    markup = types.ReplyKeyboardMarkup(resize_keyboard=True, row_width=2)
    markup.add(
        types.KeyboardButton('🎲 HiLo'),
        types.KeyboardButton('🃏 Блэкджек'),
        types.KeyboardButton('🪙 Монетка'),
        types.KeyboardButton('🎡 X50'),
        types.KeyboardButton('🎯 Рулетка'),
        types.KeyboardButton('🎰 Слоты'),
        types.KeyboardButton('🦌 Охота'),
        types.KeyboardButton('🎪 Кости'),
        types.KeyboardButton('🍀 Удача'),
        types.KeyboardButton('🔙 В меню')
    )
    return markup

def start_giveaway_timer(giveaway_id, duration_minutes):
    def timer():
        time.sleep(duration_minutes * 60)
        finish_giveaway(giveaway_id)
    
    thread = threading.Thread(target=timer, daemon=True)
    thread.start()

def finish_giveaway(giveaway_id):
    for gw in giveaways['active']:
        if gw['id'] == giveaway_id:
            participants = gw['participants']
            winners_count = gw['winners_count']
            prize = gw['prize']
            
            if len(participants) == 0:
                giveaways['active'].remove(gw)
                save_giveaways()
                return
            
            weighted = []
            for uid in participants:
                init_user(uid)
                luck = 1 + (users[uid].get('pixg', 0) * 0.02)
                weighted.extend([uid] * int(luck * 10))
            
            winners = []
            for _ in range(min(winners_count, len(participants))):
                if weighted:
                    winner = random.choice(weighted)
                    if winner not in winners:
                        winners.append(winner)
                        weighted = [x for x in weighted if x != winner]
            
            for winner in winners:
                users[winner]['balance'] += prize
                try:
                    bot.send_message(int(winner), f"🎉 Победа в раздаче!\n💰 {format_num(prize)} Pix")
                except:
                    pass
            
            save_data(users)
            giveaways['active'].remove(gw)
            gw['winners'] = winners
            giveaways['history'].append(gw)
            save_giveaways()
            return

def start_event_timer(duration_minutes):
    def timer():
        time.sleep(duration_minutes * 60)
        finish_event()
    
    thread = threading.Thread(target=timer, daemon=True)
    thread.start()

def finish_event():
    if events['active']:
        events['history'].append(events['active'])
        events['active'] = None
        events['claimed'] = {}
        save_events()

@bot.message_handler(commands=['start'])
def start(message):
    user_id = str(message.from_user.id)
    init_user(user_id)
    
    if users[user_id].get('banned', False):
        bot.send_message(message.chat.id, "❌ Вы заблокированы!")
        return
    
    bot.send_message(
        message.chat.id,
        f"🎰 Glyph Game Bot\n\n"
        f"👤 {message.from_user.first_name}\n"
        f"💰 {format_num(users[user_id]['balance'])} Pix\n"
        f"💎 {format_num(users[user_id]['pixg'])} Pix.g",
        reply_markup=main_menu(message.from_user.id)
    )

@bot.message_handler(func=lambda m: m.text == '👤 Профиль')
def profile(message):
    user_id = str(message.from_user.id)
    init_user(user_id)
    
    wins = users[user_id]['wins']
    losses = users[user_id]['losses']
    total = wins + losses
    winrate = (wins / total * 100) if total > 0 else 0
    
    bot.send_message(
        message.chat.id,
        f"👤 Профиль\n\n"
        f"🆔 ID: `{user_id}`\n"
        f"👤 Имя: {message.from_user.first_name}\n\n"
        f"💰 Баланс: {format_num(users[user_id]['balance'])} Pix\n"
        f"💎 Pix.g: {format_num(users[user_id]['pixg'])}\n\n"
        f"📊 Статистика:\n"
        f"🎮 Игр: {users[user_id].get('total_games', 0)}\n"
        f"🏆 Побед: {wins}\n"
        f"💔 Поражений: {losses}\n"
        f"📈 Винрейт: {winrate:.1f}%\n\n"
        f"💵 Выиграно: {format_num(users[user_id]['total_won'])} Pix\n"
        f"💸 Проиграно: {format_num(users[user_id]['total_lost'])} Pix",
        parse_mode='Markdown'
    )

@bot.message_handler(func=lambda m: m.text == '💰 Баланс')
def balance(message):
    user_id = str(message.from_user.id)
    init_user(user_id)
    
    bot.send_message(
        message.chat.id,
        f"💰 Баланс:\n\n"
        f"💵 Pix: {format_num(users[user_id]['balance'])}\n"
        f"💎 Pix.g: {format_num(users[user_id]['pixg'])}\n\n"
        f"🏆 Побед: {users[user_id]['wins']}\n"
        f"💔 Поражений: {users[user_id]['losses']}"
    )

@bot.message_handler(func=lambda m: m.text == '🎁 Бонус')
def daily_bonus(message):
    user_id = str(message.from_user.id)
    init_user(user_id)
    
    now = datetime.now()
    last_bonus = users[user_id].get('last_bonus')
    
    if last_bonus:
        last_time = datetime.fromisoformat(last_bonus)
        if (now - last_time).total_seconds() < 86400:
            remaining = 86400 - (now - last_time).total_seconds()
            hours = int(remaining // 3600)
            minutes = int((remaining % 3600) // 60)
            bot.send_message(message.chat.id, f"⏰ Уже получен!\n⏳ Через: {hours}ч {minutes}м")
            return
    
    base_bonus = random.randint(1000, 3000)
    bonus = base_bonus
    
    if users[user_id].get('in_top3', False):
        bonus = int(bonus * 1.1)
        msg = f"🎁 Базовый: {format_num(base_bonus)} Pix\n🏆 Топ-3: +10%\n💰 Итого: {format_num(bonus)} Pix!"
    else:
        msg = f"🎁 Получено: {format_num(bonus)} Pix!"
    
    users[user_id]['balance'] += bonus
    users[user_id]['last_bonus'] = now.isoformat()
    save_data(users)
    
    bot.send_message(message.chat.id, msg)

@bot.message_handler(func=lambda m: m.text == '👑 Админка')
def admin_panel(message):
    if is_admin(message.from_user.id):
        bot.send_message(message.chat.id, "👑 Админ-панель", reply_markup=admin_menu(message.from_user.id))

@bot.message_handler(func=lambda m: m.text == '🔙 Выйти')
def exit_admin(message):
    if is_admin(message.from_user.id):
        bot.send_message(message.chat.id, "📱 Главное меню", reply_markup=main_menu(message.from_user.id))

@bot.message_handler(func=lambda m: m.text == '👥 Зам админы')
def sub_admin_panel(message):
    if message.from_user.id != ADMIN_ID:
        return
    
    msg = bot.send_message(
        message.chat.id,
        "👥 Управление:\n\n"
        "добавить [ID]\n"
        "удалить [ID]\n"
        "список\n\n"
        f"Зам админов: {len(sub_admins)}"
    )
    bot.register_next_step_handler(msg, process_sub_admin)

def process_sub_admin(message):
    if message.from_user.id != ADMIN_ID:
        return
    
    text = message.text.lower()
    
    if text.startswith('добавить '):
        uid = text.split()[1]
        if uid == str(ADMIN_ID):
            bot.send_message(message.chat.id, "❌ Нельзя!", reply_markup=admin_menu(ADMIN_ID))
            return
        if uid not in sub_admins:
            sub_admins.append(uid)
            save_admins()
            bot.send_message(message.chat.id, f"✅ {uid} добавлен", reply_markup=admin_menu(ADMIN_ID))
    
    elif text.startswith('удалить '):
        uid = text.split()[1]
        if uid in sub_admins:
            sub_admins.remove(uid)
            save_admins()
            bot.send_message(message.chat.id, f"✅ {uid} удалён", reply_markup=admin_menu(ADMIN_ID))
    
    elif text == 'список':
        if sub_admins:
            text = "👥 Зам админы:\n\n" + "\n".join([f"• {uid}" for uid in sub_admins])
        else:
            text = "Нет зам админов"
        bot.send_message(message.chat.id, text, reply_markup=admin_menu(ADMIN_ID))

@bot.message_handler(func=lambda m: m.text == '👤 Управление игроками')
def player_management(message):
    if not is_admin(message.from_user.id):
        return
    
    msg = bot.send_message(
        message.chat.id,
        "👤 Управление:\n\n"
        "бан [ID]\n"
        "разбан [ID]\n"
        "мут [ID]\n"
        "размут [ID]"
    )
    bot.register_next_step_handler(msg, process_player_management)

def process_player_management(message):
    if not is_admin(message.from_user.id):
        return
    
    text = message.text.lower()
    
    try:
        if text.startswith('бан '):
            tid = text.split()[1]
            # Защита от бана админов
            if tid == str(ADMIN_ID) or tid in sub_admins:
                bot.send_message(message.chat.id, "❌ Нельзя забанить админа!", reply_markup=admin_menu(message.from_user.id))
                return
            init_user(tid)
            users[tid]['banned'] = True
            save_data(users)
            bot.send_message(message.chat.id, f"✅ {tid} забанен", reply_markup=admin_menu(message.from_user.id))
        
        elif text.startswith('разбан '):
            tid = text.split()[1]
            if tid in users:
                users[tid]['banned'] = False
                save_data(users)
                bot.send_message(message.chat.id, f"✅ {tid} разбанен", reply_markup=admin_menu(message.from_user.id))
        
        elif text.startswith('мут '):
            tid = text.split()[1]
            # Защита от мута админов
            if tid == str(ADMIN_ID) or tid in sub_admins:
                bot.send_message(message.chat.id, "❌ Нельзя замутить админа!", reply_markup=admin_menu(message.from_user.id))
                return
            init_user(tid)
            users[tid]['muted'] = True
            save_data(users)
            bot.send_message(message.chat.id, f"✅ {tid} в муте", reply_markup=admin_menu(message.from_user.id))
        
        elif text.startswith('размут '):
            tid = text.split()[1]
            if tid in users:
                users[tid]['muted'] = False
                save_data(users)
                bot.send_message(message.chat.id, f"✅ {tid} размучен", reply_markup=admin_menu(message.from_user.id))
    except:
        bot.send_message(message.chat.id, "❌ Ошибка!", reply_markup=admin_menu(message.from_user.id))

@bot.message_handler(func=lambda m: m.text == '💰 Управление балансом')
def balance_management(message):
    # Только главный админ
    if message.from_user.id != ADMIN_ID:
        return
    
    msg = bot.send_message(
        message.chat.id,
        "💰 Управление:\n\n"
        "givemoney [ID] [сумма]\n"
        "takemoney [ID] [сумма]\n"
        "givepixg [ID] [сумма]\n"
        "takepixg [ID] [сумма]"
    )
    bot.register_next_step_handler(msg, process_balance_management)

def process_balance_management(message):
    if message.from_user.id != ADMIN_ID:
        return
    
    text = message.text.lower()
    
    try:
        parts = text.split()
        cmd = parts[0]
        tid = parts[1]
        amt = int(parts[2])
        
        init_user(tid)
        
        if cmd == 'givemoney':
            users[tid]['balance'] += amt
            save_data(users)
            bot.send_message(message.chat.id, f"✅ +{format_num(amt)} Pix для {tid}", reply_markup=admin_menu(message.from_user.id))
        
        elif cmd == 'takemoney':
            users[tid]['balance'] -= amt
            save_data(users)
            bot.send_message(message.chat.id, f"✅ -{format_num(amt)} Pix у {tid}", reply_markup=admin_menu(message.from_user.id))
        
        elif cmd == 'givepixg':
            users[tid]['pixg'] += amt
            save_data(users)
            bot.send_message(message.chat.id, f"✅ +{format_num(amt)} Pix.g для {tid}", reply_markup=admin_menu(message.from_user.id))
        
        elif cmd == 'takepixg':
            users[tid]['pixg'] -= amt
            save_data(users)
            bot.send_message(message.chat.id, f"✅ -{format_num(amt)} Pix.g у {tid}", reply_markup=admin_menu(message.from_user.id))
    except:
        bot.send_message(message.chat.id, "❌ Ошибка!", reply_markup=admin_menu(message.from_user.id))

@bot.message_handler(func=lambda m: m.text == '🎫 Админ промо')
def admin_create_promo(message):
    if message.from_user.id != ADMIN_ID:
        return
    
    msg = bot.send_message(message.chat.id, "📝 Формат: название сумма макс_использований\nПример: BONUS2026 5000 100")
    bot.register_next_step_handler(msg, process_admin_promo)

def process_admin_promo(message):
    if message.from_user.id != ADMIN_ID:
        return
    
    try:
        parts = message.text.split()
        pname = parts[0]
        rew = int(parts[1])
        max_uses = int(parts[2]) if len(parts) > 2 else 999999
        
        promo_codes[pname] = {
            'reward': rew,
            'creator': 'admin',
            'uses': 0,
            'max_uses': max_uses,
            'used_by': []
        }
        save_promos(promo_codes)
        bot.send_message(message.chat.id, f"✅ {pname}: {format_num(rew)} Pix (макс: {max_uses})", reply_markup=admin_menu(message.from_user.id))
    except:
        bot.send_message(message.chat.id, "❌ Ошибка!", reply_markup=admin_menu(message.from_user.id))

@bot.message_handler(func=lambda m: m.text == '📊 Статистика бота')
def bot_stats(message):
    if not is_admin(message.from_user.id):
        return
    
    total_users = len(users)
    total_balance = sum(u.get('balance', 0) for u in users.values())
    total_pixg = sum(u.get('pixg', 0) for u in users.values())
    total_promos = len(promo_codes)
    total_games = sum(u.get('total_games', 0) for u in users.values())
    active_giveaways = len(giveaways.get('active', []))
    
    bot.send_message(
        message.chat.id,
        f"📊 Статистика:\n\n"
        f"👥 Игроков: {total_users}\n"
        f"💰 Общий Pix: {format_num(total_balance)}\n"
        f"💎 Общий Pix.g: {format_num(total_pixg)}\n"
        f"🎫 Промокодов: {total_promos}\n"
        f"🎮 Игр сыграно: {format_num(total_games)}\n"
        f"🎁 Раздач активных: {active_giveaways}",
        reply_markup=admin_menu(message.from_user.id)
    )

@bot.message_handler(func=lambda m: m.text == '📢 Общее сообщение')
def broadcast(message):
    if message.from_user.id != ADMIN_ID:
        return
    
    msg = bot.send_message(message.chat.id, "📢 Введите сообщение:")
    bot.register_next_step_handler(msg, process_broadcast)

def process_broadcast(message):
    if message.from_user.id != ADMIN_ID:
        return
    
    text = message.text
    count = 0
    
    for uid in users:
        try:
            bot.send_message(int(uid), f"📢 Сообщение:\n\n{text}")
            count += 1
        except:
            pass
    
    bot.send_message(message.chat.id, f"✅ Отправлено {count} игрокам", reply_markup=admin_menu(message.from_user.id))

@bot.message_handler(func=lambda m: m.text == '🎉 Управление ивентами')
def event_management(message):
    if message.from_user.id != ADMIN_ID:
        return
    
    msg = bot.send_message(
        message.chat.id,
        "🎉 Создать ивент:\n\n"
        "Формат: ивент [описание] [сумма] [минуты]\n"
        "Пример: ивент Новый год! 5000 60"
    )
    bot.register_next_step_handler(msg, process_event)

def process_event(message):
    if message.from_user.id != ADMIN_ID:
        return
    
    try:
        parts = message.text.split()
        if parts[0].lower() != 'ивент':
            raise ValueError()
        
        prize = int(parts[-2])
        duration = int(parts[-1])
        description = ' '.join(parts[1:-2])
        
        events['active'] = {
            'description': description,
            'prize': prize,
            'duration': duration,
            'created': datetime.now().isoformat()
        }
        events['claimed'] = {}
        save_events()
        start_event_timer(duration)
        
        for uid in users:
            try:
                bot.send_message(int(uid), f"🎉 Ивент!\n\n{description}\n💰 {format_num(prize)} Pix\n⏰ {duration} мин")
            except:
                pass
        
        bot.send_message(message.chat.id, "✅ Ивент создан!", reply_markup=admin_menu(message.from_user.id))
    except:
        bot.send_message(message.chat.id, "❌ Ошибка!", reply_markup=admin_menu(message.from_user.id))

@bot.message_handler(func=lambda m: m.text == '🎁 Раздачи')
def giveaway_management(message):
    if message.from_user.id != ADMIN_ID:
        return
    
    markup = types.ReplyKeyboardMarkup(resize_keyboard=True)
    markup.add(
        types.KeyboardButton('➕ Создать раздачу'),
        types.KeyboardButton('🏁 Завершить раздачу'),
        types.KeyboardButton('🔙 Назад')
    )
    
    bot.send_message(message.chat.id, "🎁 Управление раздачами", reply_markup=markup)

@bot.message_handler(func=lambda m: m.text == '🔙 Назад')
def back_to_admin(message):
    if is_admin(message.from_user.id):
        bot.send_message(message.chat.id, "👑 Админ-панель", reply_markup=admin_menu(message.from_user.id))

@bot.message_handler(func=lambda m: m.text == '➕ Создать раздачу')
def create_giveaway(message):
    if message.from_user.id != ADMIN_ID:
        return
    
    if len([g for g in giveaways.get('active', []) if g]) >= 2:
        bot.send_message(message.chat.id, "❌ Максимум 2 раздачи!")
        return
    
    msg = bot.send_message(
        message.chat.id,
        "🎁 Создать:\n\n"
        "Формат: раздача [сумма] [участники] [минуты] [победители]\n"
        "Пример: раздача 10000 50 60 2"
    )
    bot.register_next_step_handler(msg, process_giveaway)

def process_giveaway(message):
    if message.from_user.id != ADMIN_ID:
        return
    
    try:
        parts = message.text.split()
        if parts[0].lower() != 'раздача':
            raise ValueError()
        
        prize = int(parts[1])
        max_participants = int(parts[2])
        duration = int(parts[3])
        winners_count = int(parts[4])
        
        gw_id = len(giveaways.get('active', [])) + len(giveaways.get('history', [])) + 1
        
        giveaway = {
            'id': gw_id,
            'prize': prize,
            'max_participants': max_participants,
            'duration': duration,
            'winners_count': winners_count,
            'participants': [],
            'created': datetime.now().isoformat()
        }
        
        if 'active' not in giveaways:
            giveaways['active'] = []
        giveaways['active'].append(giveaway)
        save_giveaways()
        start_giveaway_timer(gw_id, duration)
        
        for uid in users:
            try:
                bot.send_message(
                    int(uid),
                    f"🎁 Раздача!\n\n"
                    f"💰 {format_num(prize)} Pix\n"
                    f"👥 0/{max_participants}\n"
                    f"🏆 Победителей: {winners_count}\n"
                    f"⏰ {duration} мин"
                )
            except:
                pass
        
        bot.send_message(message.chat.id, "✅ Раздача создана!", reply_markup=admin_menu(message.from_user.id))
    except:
        bot.send_message(message.chat.id, "❌ Ошибка!", reply_markup=admin_menu(message.from_user.id))

@bot.message_handler(func=lambda m: m.text == '🏁 Завершить раздачу')
def finish_giveaway_manual(message):
    if message.from_user.id != ADMIN_ID:
        return
    
    if not giveaways.get('active'):
        bot.send_message(message.chat.id, "❌ Нет раздач!")
        return
    
    msg = bot.send_message(message.chat.id, "ID раздачи:")
    bot.register_next_step_handler(msg, process_finish_giveaway)

def process_finish_giveaway(message):
    if message.from_user.id != ADMIN_ID:
        return
    
    try:
        gw_id = int(message.text)
        finish_giveaway(gw_id)
        bot.send_message(message.chat.id, "✅ Завершена!", reply_markup=admin_menu(message.from_user.id))
    except:
        bot.send_message(message.chat.id, "❌ Ошибка!", reply_markup=admin_menu(message.from_user.id))

@bot.message_handler(func=lambda m: m.text == '🎉 Ивенты')
def events_menu(message):
    user_id = str(message.from_user.id)
    init_user(user_id)
    
    markup = types.ReplyKeyboardMarkup(resize_keyboard=True)
    markup.add(
        types.KeyboardButton('🎁 Участвовать'),
        types.KeyboardButton('🎉 Забрать'),
        types.KeyboardButton('🔙 В меню')
    )
    
    text = "🎉 Ивенты\n\n"
    
    if events.get('active'):
        evt = events['active']
        text += f"📌 {evt['description']}\n💰 {format_num(evt['prize'])} Pix\n\n"
    
    if giveaways.get('active'):
        text += "🎁 Раздачи:\n"
        for gw in giveaways['active']:
            text += f"#{gw['id']}: {format_num(gw['prize'])} Pix ({len(gw['participants'])}/{gw['max_participants']})\n"
    
    bot.send_message(message.chat.id, text, reply_markup=markup)

@bot.message_handler(func=lambda m: m.text == '🎁 Участвовать')
def join_giveaway(message):
    user_id = str(message.from_user.id)
    init_user(user_id)
    
    if not giveaways.get('active'):
        bot.send_message(message.chat.id, "❌ Нет раздач!")
        return
    
    text = "🎁 Раздачи:\n\n"
    for gw in giveaways['active']:
        text += f"#{gw['id']}: {format_num(gw['prize'])} Pix ({len(gw['participants'])}/{gw['max_participants']})\n"
    
    text += "\nID:"
    msg = bot.send_message(message.chat.id, text)
    bot.register_next_step_handler(msg, process_join_giveaway)

def process_join_giveaway(message):
    user_id = str(message.from_user.id)
    
    try:
        gw_id = int(message.text)
        
        for gw in giveaways.get('active', []):
            if gw['id'] == gw_id:
                if len(gw['participants']) >= gw['max_participants']:
                    bot.send_message(message.chat.id, "❌ Заполнено!")
                    return
                
                if user_id in gw['participants']:
                    bot.send_message(message.chat.id, "❌ Уже участвуете!")
                    return
                
                gw['participants'].append(user_id)
                save_giveaways()
                bot.send_message(message.chat.id, f"✅ Участвуете в #{gw_id}!")
                return
        
        bot.send_message(message.chat.id, "❌ Не найдена!")
    except:
        bot.send_message(message.chat.id, "❌ Ошибка!")

@bot.message_handler(func=lambda m: m.text == '🎉 Забрать')
def claim_event(message):
    user_id = str(message.from_user.id)
    init_user(user_id)
    
    if not events.get('active'):
        bot.send_message(message.chat.id, "❌ Нет ивентов!")
        return
    
    # Проверка на дубликат
    if 'claimed' not in events:
        events['claimed'] = {}
    
    if user_id in events['claimed']:
        bot.send_message(message.chat.id, "❌ Уже забрали награду!")
        return
    
    evt = events['active']
    users[user_id]['balance'] += evt['prize']
    events['claimed'][user_id] = True
    save_data(users)
    save_events()
    
    bot.send_message(message.chat.id, f"✅ Получено {format_num(evt['prize'])} Pix!")

@bot.message_handler(func=lambda m: m.text == '🛒 Продажа')
def market_menu(message):
    user_id = str(message.from_user.id)
    init_user(user_id)
    
    markup = types.ReplyKeyboardMarkup(resize_keyboard=True, row_width=2)
    markup.add(
        types.KeyboardButton('💎 Продать Pix.g'),
        types.KeyboardButton('💰 Продать Pix'),
        types.KeyboardButton('🛍️ Купить Pix.g'),
        types.KeyboardButton('💵 Купить Pix'),
        types.KeyboardButton('📋 Объявления'),
        types.KeyboardButton('🔙 В меню')
    )
    
    bot.send_message(
        message.chat.id,
        f"🛒 Рынок P2P\n\n"
        f"💰 {format_num(users[user_id]['balance'])} Pix\n"
        f"💎 {format_num(users[user_id]['pixg'])} Pix.g",
        reply_markup=markup
    )

@bot.message_handler(func=lambda m: m.text == '💎 Продать Pix.g')
def sell_pixg_market(message):
    user_id = str(message.from_user.id)
    init_user(user_id)
    
    if users[user_id]['pixg'] == 0:
        bot.send_message(message.chat.id, "❌ Нет Pix.g!")
        return
    
    msg = bot.send_message(
        message.chat.id,
        f"💎 У вас: {format_num(users[user_id]['pixg'])} Pix.g\n\n"
        f"Формат: [кол-во] [цена за 1]\nПример: 5 900000000"
    )
    bot.register_next_step_handler(msg, process_sell_pixg_market)

def process_sell_pixg_market(message):
    user_id = str(message.from_user.id)
    
    try:
        parts = message.text.split()
        amount = int(parts[0])
        price = int(parts[1])
        
        if amount <= 0 or users[user_id]['pixg'] < amount:
            bot.send_message(message.chat.id, "❌ Недостаточно!")
            return
        
        if price < 500000000:
            bot.send_message(message.chat.id, "❌ Мин: 500 млн Pix")
            return
        
        users[user_id]['pixg'] -= amount
        
        offer = {
            'id': len(market.get('pixg_offers', [])) + 1,
            'seller': user_id,
            'amount': amount,
            'price_per': price,
            'total': amount * price,
            'created': datetime.now().isoformat()
        }
        
        if 'pixg_offers' not in market:
            market['pixg_offers'] = []
        market['pixg_offers'].append(offer)
        save_market()
        save_data(users)
        
        bot.send_message(
            message.chat.id,
            f"✅ Создано!\n💎 {format_num(amount)} Pix.g\n💰 {format_num(amount * price)} Pix"
        )
    except:
        bot.send_message(message.chat.id, "❌ Ошибка!")

@bot.message_handler(func=lambda m: m.text == '💰 Продать Pix')
def sell_pix_market(message):
    user_id = str(message.from_user.id)
    init_user(user_id)
    
    if users[user_id]['balance'] < 100000:
        bot.send_message(message.chat.id, "❌ Мин 100к Pix!")
        return
    
    msg = bot.send_message(
        message.chat.id,
        f"💰 У вас: {format_num(users[user_id]['balance'])} Pix\n\n"
        f"Формат: [Pix] [цена Pix.g]\nПример: 1000000 1"
    )
    bot.register_next_step_handler(msg, process_sell_pix_market)

def process_sell_pix_market(message):
    user_id = str(message.from_user.id)
    
    try:
        parts = message.text.split()
        amount = int(parts[0])
        price_pixg = float(parts[1])
        
        if amount < 100000 or users[user_id]['balance'] < amount:
            bot.send_message(message.chat.id, "❌ Недостаточно!")
            return
        
        if price_pixg < 0.1:
            bot.send_message(message.chat.id, "❌ Мин: 0.1 Pix.g")
            return
        
        users[user_id]['balance'] -= amount
        
        offer = {
            'id': len(market.get('pix_offers', [])) + 1,
            'seller': user_id,
            'amount': amount,
            'price_pixg': price_pixg,
            'created': datetime.now().isoformat()
        }
        
        if 'pix_offers' not in market:
            market['pix_offers'] = []
        market['pix_offers'].append(offer)
        save_market()
        save_data(users)
        
        bot.send_message(message.chat.id, f"✅ Создано!\n💰 {format_num(amount)} Pix\n💎 {price_pixg} Pix.g")
    except:
        bot.send_message(message.chat.id, "❌ Ошибка!")

@bot.message_handler(func=lambda m: m.text == '🛍️ Купить Pix.g')
def buy_pixg_market(message):
    if not market.get('pixg_offers'):
        bot.send_message(message.chat.id, "❌ Нет предложений!")
        return
    
    text = "🛍️ Купить:\n\n"
    for offer in market['pixg_offers'][:10]:
        text += f"#{offer['id']}: {format_num(offer['amount'])} Pix.g за {format_num(offer['total'])} Pix\n"
    
    text += "\nID:"
    msg = bot.send_message(message.chat.id, text)
    bot.register_next_step_handler(msg, process_buy_pixg_market)

def process_buy_pixg_market(message):
    user_id = str(message.from_user.id)
    init_user(user_id)
    
    try:
        offer_id = int(message.text)
        
        offer = None
        for o in market.get('pixg_offers', []):
            if o['id'] == offer_id:
                offer = o
                break
        
        if not offer:
            bot.send_message(message.chat.id, "❌ Не найдено!")
            return
        
        if users[user_id]['balance'] < offer['total']:
            bot.send_message(message.chat.id, "❌ Недостаточно!")
            return
        
        seller_id = offer['seller']
        
        users[user_id]['balance'] -= offer['total']
        users[user_id]['pixg'] += offer['amount']
        users[seller_id]['balance'] += offer['total']
        
        market['pixg_offers'].remove(offer)
        save_market()
        save_data(users)
        
        bot.send_message(message.chat.id, f"✅ Куплено {format_num(offer['amount'])} Pix.g!")
        
        try:
            bot.send_message(int(seller_id), f"✅ Продано #{offer['id']}\n💰 +{format_num(offer['total'])} Pix")
        except:
            pass
    except:
        bot.send_message(message.chat.id, "❌ Ошибка!")

@bot.message_handler(func=lambda m: m.text == '💵 Купить Pix')
def buy_pix_market(message):
    if not market.get('pix_offers'):
        bot.send_message(message.chat.id, "❌ Нет предложений!")
        return
    
    text = "💵 Купить:\n\n"
    for offer in market['pix_offers'][:10]:
        text += f"#{offer['id']}: {format_num(offer['amount'])} Pix за {offer['price_pixg']} Pix.g\n"
    
    text += "\nID:"
    msg = bot.send_message(message.chat.id, text)
    bot.register_next_step_handler(msg, process_buy_pix_market)

def process_buy_pix_market(message):
    user_id = str(message.from_user.id)
    init_user(user_id)
    
    try:
        offer_id = int(message.text)
        
        offer = None
        for o in market.get('pix_offers', []):
            if o['id'] == offer_id:
                offer = o
                break
        
        if not offer:
            bot.send_message(message.chat.id, "❌ Не найдено!")
            return
        
        if users[user_id]['pixg'] < offer['price_pixg']:
            bot.send_message(message.chat.id, "❌ Недостаточно!")
            return
        
        seller_id = offer['seller']
        
        users[user_id]['pixg'] -= offer['price_pixg']
        users[user_id]['balance'] += offer['amount']
        users[seller_id]['pixg'] += offer['price_pixg']
        
        market['pix_offers'].remove(offer)
        save_market()
        save_data(users)
        
        bot.send_message(message.chat.id, f"✅ Куплено {format_num(offer['amount'])} Pix!")
        
        try:
            bot.send_message(int(seller_id), f"✅ Продано #{offer['id']}\n💎 +{offer['price_pixg']} Pix.g")
        except:
            pass
    except:
        bot.send_message(message.chat.id, "❌ Ошибка!")

@bot.message_handler(func=lambda m: m.text == '📋 Объявления')
def my_offers(message):
    user_id = str(message.from_user.id)
    
    my_pixg = [o for o in market.get('pixg_offers', []) if o['seller'] == user_id]
    my_pix = [o for o in market.get('pix_offers', []) if o['seller'] == user_id]
    
    text = "📋 Ваши:\n\n"
    
    if my_pixg:
        text += "💎 Pix.g:\n"
        for o in my_pixg:
            text += f"#{o['id']}: {format_num(o['amount'])} за {format_num(o['total'])}\n"
    
    if my_pix:
        text += "\n💰 Pix:\n"
        for o in my_pix:
            text += f"#{o['id']}: {format_num(o['amount'])} за {o['price_pixg']} Pix.g\n"
    
    if not my_pixg and not my_pix:
        text = "📋 Нет объявлений"
    
    bot.send_message(message.chat.id, text)

@bot.message_handler(func=lambda m: m.text == '🎫 Промокоды')
def promo_menu(message):
    user_id = str(message.from_user.id)
    init_user(user_id)
    
    markup = types.ReplyKeyboardMarkup(resize_keyboard=True)
    markup.add(
        types.KeyboardButton('✅ Активировать'),
        types.KeyboardButton('➕ Создать промо'),
        types.KeyboardButton('🔙 В меню')
    )
    
    bot.send_message(
        message.chat.id,
        "🎫 Промокоды\n\n"
        "✅ Активировать\n"
        "➕ Создать: 4 500 000 → 4 000 Pix",
        reply_markup=markup
    )

@bot.message_handler(func=lambda m: m.text == '✅ Активировать')
def activate_promo(message):
    msg = bot.send_message(message.chat.id, "📝 Промокод:")
    bot.register_next_step_handler(msg, process_promo)

def process_promo(message):
    user_id = str(message.from_user.id)
    init_user(user_id)
    promo = message.text.strip()
    
    if promo not in promo_codes:
        bot.send_message(message.chat.id, "❌ Не найден!")
        return
    
    # Проверка на повторную активацию
    if 'used_by' not in promo_codes[promo]:
        promo_codes[promo]['used_by'] = []
    
    if user_id in promo_codes[promo]['used_by']:
        bot.send_message(message.chat.id, "❌ Уже активирован!")
        return
    
    # Проверка лимита использований
    max_uses = promo_codes[promo].get('max_uses', 999999)
    if promo_codes[promo]['uses'] >= max_uses:
        bot.send_message(message.chat.id, "❌ Промокод исчерпан!")
        return
    
    reward = promo_codes[promo]['reward']
    users[user_id]['balance'] += reward
    promo_codes[promo]['uses'] += 1
    promo_codes[promo]['used_by'].append(user_id)
    
    save_data(users)
    save_promos(promo_codes)
    
    bot.send_message(message.chat.id, f"✅ Активирован!\n💰 +{format_num(reward)} Pix")

@bot.message_handler(func=lambda m: m.text == '➕ Создать промо')
def create_promo_user(message):
    user_id = str(message.from_user.id)
    init_user(user_id)
    
    if users[user_id]['balance'] < 4500000:
        bot.send_message(message.chat.id, f"❌ Нужно 4.5М Pix\nУ вас: {format_num(users[user_id]['balance'])}")
        return
    
    msg = bot.send_message(message.chat.id, "📝 Название:")
    bot.register_next_step_handler(msg, process_create_user_promo)

def process_create_user_promo(message):
    user_id = str(message.from_user.id)
    promo = message.text.strip()
    
    if not promo.isalnum():
        bot.send_message(message.chat.id, "❌ Только буквы/цифры!")
        return
    
    if promo in promo_codes:
        bot.send_message(message.chat.id, "❌ Существует!")
        return
    
    users[user_id]['balance'] -= 4500000
    promo_codes[promo] = {
        'reward': 4000,
        'creator': user_id,
        'uses': 0,
        'max_uses': 999999,
        'used_by': []
    }
    
    save_data(users)
    save_promos(promo_codes)
    
    bot.send_message(message.chat.id, f"✅ {promo} создан!\n💰 4К Pix\n💸 -4.5М Pix")

@bot.message_handler(func=lambda m: m.text in ['🎮 Игры', '🔙 В меню'])
def games_menu(message):
    user_id = str(message.from_user.id)
    init_user(user_id)
    
    if message.text == '🔙 В меню':
        bot.send_message(message.chat.id, "📱 Меню", reply_markup=main_menu(message.from_user.id))
        return
    
    bot.send_message(
        message.chat.id,
        "🎮 Игры (мин 100 Pix)\n\n"
        "🎲 хл [сумма]\n"
        "🃏 блэкджек [сумма]\n"
        "🪙 монетка [сумма] [орёл/решка]\n"
        "🎡 х50 [сумма]\n"
        "🎯 рулетка [сумма] [0-36]\n"
        "🎰 слоты [сумма]\n"
        "🦌 охота [сумма]\n"
        "🎪 кости [сумма]\n"
        "🍀 удача [сумма]",
        reply_markup=games_keyboard()
    )

# ИГРА: HiLo
@bot.message_handler(func=lambda m: m.text.lower().startswith('хл '))
def hilo_game(message):
    user_id = str(message.from_user.id)
    init_user(user_id)
    
    if users[user_id].get('banned') or users[user_id].get('muted'):
        return
    
    try:
        bet = int(message.text.split()[1])
        
        if bet < 100:
            bot.send_message(message.chat.id, "❌ Мин 100!")
            return
        
        if users[user_id]['balance'] < bet:
            bot.send_message(message.chat.id, "❌ Недостаточно!")
            return
        
        users[user_id]['balance'] -= bet
        save_data(users)
        
        cards = ['2','3','4','5','6','7','8','9','10','J','Q','K','A']
        suits = ['♠','♥','♦','♣']
        card = random.choice(cards) + random.choice(suits)
        
        users[user_id]['hilo_game'] = {
            'bet': bet,
            'card': card,
            'multiplier': 1.0,
            'round': 1
        }
        save_data(users)
        
        markup = types.InlineKeyboardMarkup(row_width=2)
        markup.add(
            types.InlineKeyboardButton("⬆️ Выше", callback_data="hilo_higher"),
            types.InlineKeyboardButton("⬇️ Ниже", callback_data="hilo_lower"),
            types.InlineKeyboardButton(f"💰 x{users[user_id]['hilo_game']['multiplier']:.1f}", callback_data="hilo_cash")
        )
        
        bot.send_message(
            message.chat.id,
            f"🎲 Раунд {users[user_id]['hilo_game']['round']}\n\n"
            f"🃏 {card}\n"
            f"📊 x{users[user_id]['hilo_game']['multiplier']:.1f}\n"
            f"💵 {format_num(int(bet * users[user_id]['hilo_game']['multiplier']))} Pix",
            reply_markup=markup
        )
    except:
        bot.send_message(message.chat.id, "❌ Формат: хл [сумма]")

@bot.callback_query_handler(func=lambda c: c.data.startswith('hilo_'))
def hilo_callback(call):
    user_id = str(call.from_user.id)
    init_user(user_id)
    
    if not users[user_id].get('hilo_game'):
        bot.answer_callback_query(call.id, "❌ Не найдена!")
        return
    
    game = users[user_id]['hilo_game']
    
    if call.data == "hilo_cash":
        win = int(game['bet'] * game['multiplier'])
        users[user_id]['balance'] += win
        users[user_id]['wins'] += 1
        users[user_id]['total_won'] += (win - game['bet'])
        users[user_id]['total_games'] += 1
        users[user_id]['hilo_game'] = None
        save_data(users)
        
        bot.edit_message_text(
            f"💰 Забрано!\n\n🎲 {game['round']} раундов\n📊 x{game['multiplier']:.1f}\n💵 {format_num(win)} Pix",
            call.message.chat.id,
            call.message.message_id
        )
        return
    
    cards_val = {'2':2,'3':3,'4':4,'5':5,'6':6,'7':7,'8':8,'9':9,'10':10,'J':11,'Q':12,'K':13,'A':14}
    current = game['card'][:-1]
    
    cards = ['2','3','4','5','6','7','8','9','10','J','Q','K','A']
    suits = ['♠','♥','♦','♣']
    new = random.choice(cards) + random.choice(suits)
    
    won = False
    if call.data == "hilo_higher" and cards_val[new[:-1]] > cards_val[current]:
        won = True
    elif call.data == "hilo_lower" and cards_val[new[:-1]] < cards_val[current]:
        won = True
    
    if won:
        game['multiplier'] += 0.5
        game['round'] += 1
        game['card'] = new
        users[user_id]['hilo_game'] = game
        save_data(users)
        
        markup = types.InlineKeyboardMarkup(row_width=2)
        markup.add(
            types.InlineKeyboardButton("⬆️ Выше", callback_data="hilo_higher"),
            types.InlineKeyboardButton("⬇️ Ниже", callback_data="hilo_lower"),
            types.InlineKeyboardButton(f"💰 x{game['multiplier']:.1f}", callback_data="hilo_cash")
        )
        
        bot.edit_message_text(
            f"✅ Угадали!\n\n🎲 Раунд {game['round']}\n🃏 {new}\n📊 x{game['multiplier']:.1f}\n💵 {format_num(int(game['bet'] * game['multiplier']))} Pix",
            call.message.chat.id,
            call.message.message_id,
            reply_markup=markup
        )
    else:
        users[user_id]['losses'] += 1
        users[user_id]['total_lost'] += game['bet']
        users[user_id]['total_games'] += 1
        users[user_id]['hilo_game'] = None
        save_data(users)
        
        bot.edit_message_text(
            f"❌ Проигрыш!\n\n🃏 {new}\n🎲 {game['round']} раундов\n💸 {format_num(game['bet'])} Pix",
            call.message.chat.id,
            call.message.message_id
        )

# ИГРА: Блэкджек
@bot.message_handler(func=lambda m: m.text.lower().startswith('блэкджек '))
def blackjack_game(message):
    user_id = str(message.from_user.id)
    init_user(user_id)
    
    if users[user_id].get('banned') or users[user_id].get('muted'):
        return
    
    try:
        bet = int(message.text.split()[1])
        
        if bet < 100:
            bot.send_message(message.chat.id, "❌ Мин 100!")
            return
        
        if users[user_id]['balance'] < bet:
            bot.send_message(message.chat.id, "❌ Недостаточно!")
            return
        
        users[user_id]['balance'] -= bet
        
        def get_card():
            return random.choice(['2','3','4','5','6','7','8','9','10','J','Q','K','A'])
        
        def card_value(cards):
            val = 0
            aces = 0
            for c in cards:
                if c in ['J','Q','K']:
                    val += 10
                elif c == 'A':
                    aces += 1
                    val += 11
                else:
                    val += int(c)
            
            while val > 21 and aces > 0:
                val -= 10
                aces -= 1
            
            return val
        
        player = [get_card(), get_card()]
        dealer = [get_card(), get_card()]
        
        users[user_id]['blackjack_game'] = {
            'bet': bet,
            'player': player,
            'dealer': dealer
        }
        save_data(users)
        
        p_val = card_value(player)
        d_val = card_value([dealer[0]])
        
        markup = types.InlineKeyboardMarkup(row_width=2)
        markup.add(
            types.InlineKeyboardButton("🃏 Ещё", callback_data="bj_hit"),
            types.InlineKeyboardButton("✋ Хватит", callback_data="bj_stand")
        )
        
        bot.send_message(
            message.chat.id,
            f"🃏 Блэкджек\n\n"
            f"Вы: {' '.join(player)} ({p_val})\n"
            f"Дилер: {dealer[0]} ?",
            reply_markup=markup
        )
    except:
        bot.send_message(message.chat.id, "❌ Формат: блэкджек [сумма]")

@bot.callback_query_handler(func=lambda c: c.data.startswith('bj_'))
def blackjack_callback(call):
    user_id = str(call.from_user.id)
    init_user(user_id)
    
    if not users[user_id].get('blackjack_game'):
        bot.answer_callback_query(call.id, "❌ Игра не найдена!")
        return
    
    game = users[user_id]['blackjack_game']
    
    def get_card():
        return random.choice(['2','3','4','5','6','7','8','9','10','J','Q','K','A'])
    
    def card_value(cards):
        val = 0
        aces = 0
        for c in cards:
            if c in ['J','Q','K']:
                val += 10
            elif c == 'A':
                aces += 1
                val += 11
            else:
                val += int(c)
        
        while val > 21 and aces > 0:
            val -= 10
            aces -= 1
        
        return val
    
    if call.data == "bj_hit":
        game['player'].append(get_card())
        users[user_id]['blackjack_game'] = game
        save_data(users)
        
        p_val = card_value(game['player'])
        
        if p_val > 21:
            users[user_id]['losses'] += 1
            users[user_id]['total_lost'] += game['bet']
            users[user_id]['total_games'] += 1
            users[user_id]['blackjack_game'] = None
            save_data(users)
            
            bot.edit_message_text(
                f"❌ Перебор!\n\n"
                f"Вы: {' '.join(game['player'])} ({p_val})\n"
                f"💸 -{format_num(game['bet'])} Pix",
                call.message.chat.id,
                call.message.message_id
            )
        else:
            markup = types.InlineKeyboardMarkup(row_width=2)
            markup.add(
                types.InlineKeyboardButton("🃏 Ещё", callback_data="bj_hit"),
                types.InlineKeyboardButton("✋ Хватит", callback_data="bj_stand")
            )
            
            bot.edit_message_text(
                f"🃏 Блэкджек\n\n"
                f"Вы: {' '.join(game['player'])} ({p_val})\n"
                f"Дилер: {game['dealer'][0]} ?",
                call.message.chat.id,
                call.message.message_id,
                reply_markup=markup
            )
    
    elif call.data == "bj_stand":
        while card_value(game['dealer']) < 17:
            game['dealer'].append(get_card())
        
        p_val = card_value(game['player'])
        d_val = card_value(game['dealer'])
        
        if d_val > 21 or p_val > d_val:
            win = game['bet'] * 2
            users[user_id]['balance'] += win
            users[user_id]['wins'] += 1
            users[user_id]['total_won'] += game['bet']
            result = f"✅ Победа!\n💰 +{format_num(win)} Pix"
        elif p_val == d_val:
            users[user_id]['balance'] += game['bet']
            result = f"🤝 Ничья!\n💰 {format_num(game['bet'])} Pix"
        else:
            users[user_id]['losses'] += 1
            users[user_id]['total_lost'] += game['bet']
            result = f"❌ Проигрыш!\n💸 -{format_num(game['bet'])} Pix"
        
        users[user_id]['total_games'] += 1
        users[user_id]['blackjack_game'] = None
        save_data(users)
        
        bot.edit_message_text(
            f"{result}\n\n"
            f"Вы: {' '.join(game['player'])} ({p_val})\n"
            f"Дилер: {' '.join(game['dealer'])} ({d_val})",
            call.message.chat.id,
            call.message.message_id
        )

# ИГРА: Монетка
@bot.message_handler(func=lambda m: m.text.lower().startswith('монетка '))
def coinflip_game(message):
    user_id = str(message.from_user.id)
    init_user(user_id)
    
    if users[user_id].get('banned') or users[user_id].get('muted'):
        return
    
    try:
        parts = message.text.lower().split()
        bet = int(parts[1])
        choice = parts[2]
        
        if choice not in ['орёл', 'решка']:
            bot.send_message(message.chat.id, "❌ Выберите: орёл/решка")
            return
        
        if bet < 100:
            bot.send_message(message.chat.id, "❌ Мин 100!")
            return
        
        if users[user_id]['balance'] < bet:
            bot.send_message(message.chat.id, "❌ Недостаточно!")
            return
        
        users[user_id]['balance'] -= bet
        
        result = random.choice(['орёл', 'решка'])
        
        if result == choice:
            win = bet * 2
            users[user_id]['balance'] += win
            users[user_id]['wins'] += 1
            users[user_id]['total_won'] += bet
            msg = f"🪙 {result.upper()}\n✅ Победа!\n💰 +{format_num(win)} Pix"
        else:
            users[user_id]['losses'] += 1
            users[user_id]['total_lost'] += bet
            msg = f"🪙 {result.upper()}\n❌ Проигрыш!\n💸 -{format_num(bet)} Pix"
        
        users[user_id]['total_games'] += 1
        save_data(users)
        
        bot.send_message(message.chat.id, msg)
    except:
        bot.send_message(message.chat.id, "❌ Формат: монетка [сумма] [орёл/решка]")

# ИГРА: X50
@bot.message_handler(func=lambda m: m.text.lower().startswith('х50 '))
def x50_game(message):
    user_id = str(message.from_user.id)
    init_user(user_id)
    
    if users[user_id].get('banned') or users[user_id].get('muted'):
        return
    
    try:
        bet = int(message.text.split()[1])
        
        if bet < 100:
            bot.send_message(message.chat.id, "❌ Мин 100!")
            return
        
        if users[user_id]['balance'] < bet:
            bot.send_message(message.chat.id, "❌ Недостаточно!")
            return
        
        users[user_id]['balance'] -= bet
        
        luck = 1 + (users[user_id].get('pixg', 0) * 0.02)
        chance = 2 * luck
        
        if random.random() * 100 < chance:
            win = bet * 50
            users[user_id]['balance'] += win
            users[user_id]['wins'] += 1
            users[user_id]['total_won'] += (win - bet)
            msg = f"🎡 X50!\n✅ ДЖЕКПОТ!\n💰 +{format_num(win)} Pix"
        else:
            users[user_id]['losses'] += 1
            users[user_id]['total_lost'] += bet
            msg = f"🎡 X50\n❌ Не повезло!\n💸 -{format_num(bet)} Pix"
        
        users[user_id]['total_games'] += 1
        save_data(users)
        
        bot.send_message(message.chat.id, msg)
    except:
        bot.send_message(message.chat.id, "❌ Формат: х50 [сумма]")

# ИГРА: Рулетка
@bot.message_handler(func=lambda m: m.text.lower().startswith('рулетка '))
def roulette_game(message):
    user_id = str(message.from_user.id)
    init_user(user_id)
    
    if users[user_id].get('banned') or users[user_id].get('muted'):
        return
    
    try:
        parts = message.text.split()
        bet = int(parts[1])
        choice = int(parts[2])
        
        if choice < 0 or choice > 36:
            bot.send_message(message.chat.id, "❌ Число 0-36!")
            return
        
        if bet < 100:
            bot.send_message(message.chat.id, "❌ Мин 100!")
            return
        
        if users[user_id]['balance'] < bet:
            bot.send_message(message.chat.id, "❌ Недостаточно!")
            return
        
        users[user_id]['balance'] -= bet
        
        result = random.randint(0, 36)
        
        if result == choice:
            win = bet * 36
            users[user_id]['balance'] += win
            users[user_id]['wins'] += 1
            users[user_id]['total_won'] += (win - bet)
            msg = f"🎯 Выпало: {result}\n✅ ВЫИГРЫШ!\n💰 +{format_num(win)} Pix"
        else:
            users[user_id]['losses'] += 1
            users[user_id]['total_lost'] += bet
            msg = f"🎯 Выпало: {result}\n❌ Не повезло!\n💸 -{format_num(bet)} Pix"
        
        users[user_id]['total_games'] += 1
        save_data(users)
        
        bot.send_message(message.chat.id, msg)
    except:
        bot.send_message(message.chat.id, "❌ Формат: рулетка [сумма] [0-36]")

# ИГРА: Слоты
@bot.message_handler(func=lambda m: m.text.lower().startswith('слоты '))
def slots_game(message):
    user_id = str(message.from_user.id)
    init_user(user_id)
    
    if users[user_id].get('banned') or users[user_id].get('muted'):
        return
    
    try:
        bet = int(message.text.split()[1])
        
        if bet < 100:
            bot.send_message(message.chat.id, "❌ Мин 100!")
            return
        
        if users[user_id]['balance'] < bet:
            bot.send_message(message.chat.id, "❌ Недостаточно!")
            return
        
        users[user_id]['balance'] -= bet
        
        symbols = ['🍒', '🍋', '🍊', '🍇', '💎', '7️⃣']
        slot1 = random.choice(symbols)
        slot2 = random.choice(symbols)
        slot3 = random.choice(symbols)
        
        if slot1 == slot2 == slot3:
            if slot1 == '💎':
                mult = 100
            elif slot1 == '7️⃣':
                mult = 50
            else:
                mult = 10
            
            win = bet * mult
            users[user_id]['balance'] += win
            users[user_id]['wins'] += 1
            users[user_id]['total_won'] += (win - bet)
            msg = f"🎰 {slot1} {slot2} {slot3}\n✅ ДЖЕКПОТ x{mult}!\n💰 +{format_num(win)} Pix"
        elif slot1 == slot2 or slot2 == slot3:
            win = bet * 2
            users[user_id]['balance'] += win
            users[user_id]['wins'] += 1
            users[user_id]['total_won'] += bet
            msg = f"🎰 {slot1} {slot2} {slot3}\n✅ Пара x2!\n💰 +{format_num(win)} Pix"
        else:
            users[user_id]['losses'] += 1
            users[user_id]['total_lost'] += bet
            msg = f"🎰 {slot1} {slot2} {slot3}\n❌ Не повезло!\n💸 -{format_num(bet)} Pix"
        
        users[user_id]['total_games'] += 1
        save_data(users)
        
        bot.send_message(message.chat.id, msg)
    except:
        bot.send_message(message.chat.id, "❌ Формат: слоты [сумма]")

# ИГРА: Охота
@bot.message_handler(func=lambda m: m.text.lower().startswith('охота '))
def hunt_game(message):
    user_id = str(message.from_user.id)
    init_user(user_id)
    
    if users[user_id].get('banned') or users[user_id].get('muted'):
        return
    
    try:
        bet = int(message.text.split()[1])
        
        if bet < 100:
            bot.send_message(message.chat.id, "❌ Мин 100!")
            return
        
        if users[user_id]['balance'] < bet:
            bot.send_message(message.chat.id, "❌ Недостаточно!")
            return
        
        users[user_id]['balance'] -= bet
        
        animals = [
            ('🦌', 5, 30),
            ('🐗', 3, 40),
            ('🦅', 10, 20),
            ('🐻', 15, 10),
            ('🦊', 2, 50)
        ]
        
        luck = 1 + (users[user_id].get('pixg', 0) * 0.02)
        caught = None
        
        for animal, mult, chance in animals:
            if random.random() * 100 < (chance * luck):
                caught = (animal, mult)
                break
        
        if caught:
            win = bet * caught[1]
            users[user_id]['balance'] += win
            users[user_id]['wins'] += 1
            users[user_id]['total_won'] += (win - bet)
            msg = f"🦌 Поймали: {caught[0]}\n✅ x{caught[1]}!\n💰 +{format_num(win)} Pix"
        else:
            users[user_id]['losses'] += 1
            users[user_id]['total_lost'] += bet
            msg = f"🦌 Промах!\n❌ Ничего не поймали\n💸 -{format_num(bet)} Pix"
        
        users[user_id]['total_games'] += 1
        save_data(users)
        
        bot.send_message(message.chat.id, msg)
    except:
        bot.send_message(message.chat.id, "❌ Формат: охота [сумма]")

# ИГРА: Кости
@bot.message_handler(func=lambda m: m.text.lower().startswith('кости '))
def dice_game(message):
    user_id = str(message.from_user.id)
    init_user(user_id)
    
    if users[user_id].get('banned') or users[user_id].get('muted'):
        return
    
    try:
        bet = int(message.text.split()[1])
        
        if bet < 100:
            bot.send_message(message.chat.id, "❌ Мин 100!")
            return
        
        if users[user_id]['balance'] < bet:
            bot.send_message(message.chat.id, "❌ Недостаточно!")
            return
        
        users[user_id]['balance'] -= bet
        
        player = random.randint(1, 6) + random.randint(1, 6)
        dealer = random.randint(1, 6) + random.randint(1, 6)
        
        if player > dealer:
            win = bet * 2
            users[user_id]['balance'] += win
            users[user_id]['wins'] += 1
            users[user_id]['total_won'] += bet
            msg = f"🎪 Вы: {player} | Дилер: {dealer}\n✅ Победа!\n💰 +{format_num(win)} Pix"
        elif player == dealer:
            users[user_id]['balance'] += bet
            msg = f"🎪 Вы: {player} | Дилер: {dealer}\n🤝 Ничья!\n💰 {format_num(bet)} Pix"
        else:
            users[user_id]['losses'] += 1
            users[user_id]['total_lost'] += bet
            msg = f"🎪 Вы: {player} | Дилер: {dealer}\n❌ Проигрыш!\n💸 -{format_num(bet)} Pix"
        
        users[user_id]['total_games'] += 1
        save_data(users)
        
        bot.send_message(message.chat.id, msg)
    except:
        bot.send_message(message.chat.id, "❌ Формат: кости [сумма]")

# ИГРА: Удача
@bot.message_handler(func=lambda m: m.text.lower().startswith('удача '))
def luck_game(message):
    user_id = str(message.from_user.id)
    init_user(user_id)
    
    if users[user_id].get('banned') or users[user_id].get('muted'):
        return
    
    try:
        bet = int(message.text.split()[1])
        
        if bet < 100:
            bot.send_message(message.chat.id, "❌ Мин 100!")
            return
        
        if users[user_id]['balance'] < bet:
            bot.send_message(message.chat.id, "❌ Недостаточно!")
            return
        
        users[user_id]['balance'] -= bet
        
        luck = 1 + (users[user_id].get('pixg', 0) * 0.02)
        chance = 50 * luck
        
        if random.random() * 100 < chance:
            multiplier = random.uniform(1.5, 5.0)
            win = int(bet * multiplier)
            users[user_id]['balance'] += win
            users[user_id]['wins'] += 1
            users[user_id]['total_won'] += (win - bet)
            msg = f"🍀 Удача!\n✅ x{multiplier:.2f}\n💰 +{format_num(win)} Pix"
        else:
            users[user_id]['losses'] += 1
            users[user_id]['total_lost'] += bet
            msg = f"🍀 Не повезло!\n❌ Проигрыш\n💸 -{format_num(bet)} Pix"
        
        users[user_id]['total_games'] += 1
        save_data(users)
        
        bot.send_message(message.chat.id, msg)
    except:
        bot.send_message(message.chat.id, "❌ Формат: удача [сумма]")

@bot.message_handler(func=lambda m: m.text == '💎 Донаты')
def donate_menu(message):
    user_id = str(message.from_user.id)
    init_user(user_id)
    
    markup = types.InlineKeyboardMarkup(row_width=2)
    markup.add(
        types.InlineKeyboardButton("💎 Купить", callback_data="buy_pixg"),
        types.InlineKeyboardButton("💸 Продать", callback_data="sell_pixg"),
        types.InlineKeyboardButton("💳 Донат", callback_data="donate_real")
    )
    
    bot.send_message(
        message.chat.id,
        f"💎 Донаты\n\n"
        f"💰 {format_num(users[user_id]['balance'])} Pix\n"
        f"💎 {format_num(users[user_id]['pixg'])} Pix.g\n\n"
        f"📊 1 Pix.g = 1 млрд Pix\n"
        f"🍀 1 Pix.g = +2% удачи",
        reply_markup=markup
    )

@bot.callback_query_handler(func=lambda c: c.data in ['buy_pixg', 'sell_pixg', 'donate_real'])
def donate_callback(call):
    if call.data == 'buy_pixg':
        msg = bot.send_message(call.message.chat.id, "💎 Сколько?")
        bot.register_next_step_handler(msg, process_buy_pixg)
    elif call.data == 'sell_pixg':
        msg = bot.send_message(call.message.chat.id, "💸 Сколько?")
        bot.register_next_step_handler(msg, process_sell_pixg)
    elif call.data == 'donate_real':
        bot.send_message(
            call.message.chat.id,
            "⚠️ ВНИМАНИЕ!\n\nДонаты НЕ возвращаются!\n\n💳 Каспи: 87026476683\n(К.Абдуали)\n👨‍💻 @abdu_ali1"
        )

def process_buy_pixg(message):
    user_id = str(message.from_user.id)
    init_user(user_id)
    
    try:
        amt = int(message.text)
        cost = amt * 1000000000
        
        if amt <= 0 or users[user_id]['balance'] < cost:
            bot.send_message(message.chat.id, "❌ Недостаточно!")
            return
        
        users[user_id]['balance'] -= cost
        users[user_id]['pixg'] += amt
        save_data(users)
        
        bot.send_message(message.chat.id, f"✅ Куплено!\n💎 +{format_num(amt)}\n💸 -{format_num(cost)}")
    except:
        bot.send_message(message.chat.id, "❌ Ошибка!")

def process_sell_pixg(message):
    user_id = str(message.from_user.id)
    init_user(user_id)
    
    try:
        amt = int(message.text)
        
        if amt <= 0 or users[user_id]['pixg'] < amt:
            bot.send_message(message.chat.id, "❌ Недостаточно!")
            return
        
        profit = int(amt * 1000000000 * 0.5)
        users[user_id]['pixg'] -= amt
        users[user_id]['balance'] += profit
        save_data(users)
        
        bot.send_message(message.chat.id, f"✅ Продано!\n💸 -{format_num(amt)}\n💰 +{format_num(profit)}")
    except:
        bot.send_message(message.chat.id, "❌ Ошибка!")

@bot.message_handler(func=lambda m: m.text == '🏆 Лидерборд')
def leaderboard(message):
    filtered = {k: v for k, v in users.items() if k != str(ADMIN_ID)}
    sorted_users = sorted(filtered.items(), key=lambda x: x[1].get('balance', 0), reverse=True)[:10]
    
    for uid in users:
        users[uid]['in_top3'] = False
    
    for i, (uid, _) in enumerate(sorted_users[:3]):
        users[uid]['in_top3'] = True
    
    save_data(users)
    
    text = "🏆 Топ-10:\n\n"
    medals = ['🥇', '🥈', '🥉']
    
    for i, (uid, data) in enumerate(sorted_users):
        try:
            user = bot.get_chat(int(uid))
            name = user.first_name
        except:
            name = f"User{uid[:4]}"
        
        medal = medals[i] if i < 3 else f"{i+1}."
        bonus = " 🎁" if i < 3 else ""
        text += f"{medal} {name}: {format_num(data.get('balance', 0))}{bonus}\n"
    
    bot.send_message(message.chat.id, text)

@bot.message_handler(func=lambda m: m.text == '📞 Контакты')
def contacts(message):
    markup = types.InlineKeyboardMarkup()
    markup.add(
        types.InlineKeyboardButton("📢 Канал", url="https://t.me/abdu1likanal"),
        types.InlineKeyboardButton("👨‍💻 Dev", url="https://t.me/abdu_ali1")
    )
    
    bot.send_message(message.chat.id, "📞 Контакты:\n\n📢 @abdu1likanal\n👨‍💻 @abdu_ali1", reply_markup=markup)

@bot.message_handler(func=lambda m: m.text == '❓ Помощь')
def help_menu(message):
    bot.send_message(
        message.chat.id,
        "❓ Команды (мин 100):\n\n"
        "🎲 хл [сумма]\n"
        "🃏 блэкджек [сумма]\n"
        "🪙 монетка [сумма] [орёл/решка]\n"
        "🎡 х50 [сумма]\n"
        "🎯 рулетка [сумма] [0-36]\n"
        "🎰 слоты [сумма]\n"
        "🦌 охота [сумма]\n"
        "🎪 кости [сумма]\n"
        "🍀 удача [сумма]\n\n"
        "💸 дать [ID] [сумма]\n\n"
        "💎 1 Pix.g = +2% удачи\n"
        "🛒 Рынок P2P\n"
        "🎉 Ивенты и раздачи"
    )

@bot.message_handler(func=lambda m: m.text.lower().startswith('дать '))
def transfer(message):
    user_id = str(message.from_user.id)
    init_user(user_id)
    
    if users[user_id].get('banned') or users[user_id].get('muted'):
        return
    
    try:
        parts = message.text.split()
        tid = parts[1]
        amt = int(parts[2])
        
        if tid not in users:
            bot.send_message(message.chat.id, "❌ Не найден!")
            return
        
        if amt <= 0 or users[user_id]['balance'] < amt or tid == user_id:
            bot.send_message(message.chat.id, "❌ Ошибка!")
            return
        
        users[user_id]['balance'] -= amt
        users[tid]['balance'] += amt
        save_data(users)
        
        bot.send_message(message.chat.id, f"✅ {format_num(amt)} Pix")
        
        try:
            bot.send_message(int(tid), f"💰 +{format_num(amt)} от {message.from_user.first_name}!")
        except:
            pass
    except:
        bot.send_message(message.chat.id, "❌ Формат: дать [ID] [сумма]")

if __name__ == '__main__':
    print("🎰 Glyph Game Bot запущен!")
    print(f"💎 Админ: {ADMIN_ID}")
    print(f"📊 Игроков: {len(users)}")
    bot.infinity_polling()
