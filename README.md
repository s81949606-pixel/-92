import telebot
from telebot import types
import json
import random
import os
import time
from datetime import datetime, timedelta
import threading
import hashlib
import re
import sqlite3

# Инициализация бота
API_TOKEN = '8211366346:AAE1B1rRmTrpUqhdYEiJszbp-hKNc106SSc'
bot = telebot.TeleBot(API_TOKEN, parse_mode='HTML')

# Админ ID
ADMIN_ID = 6915797048
ADMINS_FILE = 'admins.json'

# Загрузка списка админов
def load_admins():
    try:
        if os.path.exists(ADMINS_FILE):
            with open(ADMINS_FILE, 'r', encoding='utf-8') as f:
                return json.load(f)
    except:
        pass
    return [ADMIN_ID]  # По умолчанию только главный админ

def save_admins(admins):
    try:
        with open(ADMINS_FILE, 'w', encoding='utf-8') as f:
            json.dump(admins, f, ensure_ascii=False, indent=2)
        return True
    except:
        return False

ADMINS = load_admins()

# Файлы для хранения данных
DB_FILE = 'bot_database.db'
QUESTS_FILE = 'quests_data.json'

# Глобальные переменные для лидерборда
leaderboard_cache = None
leaderboard_update_time = None
levels_leaderboard_cache = None
levels_leaderboard_update_time = None
UPDATE_INTERVAL = 180

# Бонусы за рефералов
REFERRAL_BONUS_INVITER = 50000
REFERRAL_BONUS_REFEREE = 40000
REFERRAL_EXPERIENCE = 300

# Настройки уровней
LEVEL_REWARDS = {
    1: 30000, 2: 100000, 3: 200000, 4: 350000, 5: 500000,
    6: 700000, 7: 900000, 8: 1200000, 9: 1500000, 10: 2000000,
    11: 2500000, 12: 3000000, 13: 4000000, 14: 5000000, 15: 6000000,
    16: 8000000, 17: 10000000, 18: 12000000, 19: 15000000, 20: 20000000
}

# Система бизнесов (ОБНОВЛЕНО)
BUSINESSES = {
    1: {'id': 1, 'name': 'Ларек', 'cost': 100000, 'profit_per_hour': 2500, 
        'exp_per_hour': 10, 'tax_per_hour': 500, 'image': '🛒'},
    2: {'id': 2, 'name': 'Пятёрочка', 'cost': 300000, 'profit_per_hour': 4500,
        'exp_per_hour': 100, 'tax_per_hour': 1000, 'image': '🏪'},
    3: {'id': 3, 'name': 'Компьютерный клуб', 'cost': 1000000, 'profit_per_hour': 15000,
        'exp_per_hour': 450, 'tax_per_hour': 5000, 'image': '💻'},
    4: {'id': 4, 'name': 'Фитнес клуб', 'cost': 6000000, 'profit_per_hour': 50000,
        'exp_per_hour': 350, 'tax_per_hour': 10000, 'image': '🏋️‍♂️'},
    5: {'id': 5, 'name': 'IT-компания', 'cost': 15000000, 'profit_per_hour': 100000,
        'exp_per_hour': 400, 'tax_per_hour': 25000, 'image': '👨‍💻'},
    6: {'id': 6, 'name': 'Авиапарк', 'cost': 35000000, 'profit_per_hour': 200000,
        'exp_per_hour': 1000, 'tax_per_hour': 50000, 'image': '✈️'},
    7: {'id': 7, 'name': 'Офис', 'cost': 100000000, 'profit_per_hour': 1000000,
        'exp_per_hour': 5000, 'tax_per_hour': 450000, 'image': '🏢'}
}

# Уникальный бизнес для конкретного пользователя (НЕ ОТОБРАЖАЕТСЯ В МАГАЗИНЕ)
UNIQUE_BUSINESS = {
    8: {'id': 8, 'name': 'Лаборатория амфетамина', 'cost': 0, 'profit_per_hour': 50000000,
        'exp_per_hour': 3000, 'tax_per_hour': 0, 'image': '⚗️'}
}

# Система квестов
class QuestSystem:
    def __init__(self):
        self.quests_data = {}
        self.load_quests()
        self.start_quest_resetter()
    
    def load_quests(self):
        try:
            if os.path.exists(QUESTS_FILE):
                with open(QUESTS_FILE, 'r', encoding='utf-8') as f:
                    self.quests_data = json.load(f)
            else:
                self.quests_data = {
                    'daily_quests': {},
                    'weekly_quests': {},
                    'last_daily_reset': None,
                    'last_weekly_reset': None
                }
                self.save_quests()
        except:
            self.quests_data = {
                'daily_quests': {},
                'weekly_quests': {},
                'last_daily_reset': None,
                'last_weekly_reset': None
            }
    
    def save_quests(self):
        try:
            with open(QUESTS_FILE, 'w', encoding='utf-8') as f:
                json.dump(self.quests_data, f, ensure_ascii=False, indent=2)
            return True
        except:
            return False
    
    def generate_quest_id(self):
        return f"quest_{random.randint(10000, 99999)}_{int(time.time())}"
    
    def get_daily_quests_templates(self):
        return [
            {
                'title': '🎯 Работай усердно',
                'description': 'Выполните 5 успешных работ',
                'type': 'work',
                'target': 5,
                'reward_money': random.randint(2000, 35000),
                'reward_exp': random.randint(100, 2000),
                'difficulty': 'easy'
            },
            {
                'title': '💰 Заработай состояние',
                'description': 'Заработайте 15,000₽ за сегодня',
                'type': 'earn_money',
                'target': 15000,
                'reward_money': random.randint(8000, 20000),
                'reward_exp': random.randint(300, 800),
                'difficulty': 'medium'
            },
            {
                'title': '🌟 Мастер уровней',
                'description': 'Получите 500 опыта за сегодня',
                'type': 'earn_exp',
                'target': 500,
                'reward_money': random.randint(12000, 25000),
                'reward_exp': random.randint(500, 1200),
                'difficulty': 'hard'
            },
            {
                'title': '🎁 Бонусный охотник',
                'description': 'Получите ежедневный бонус',
                'type': 'get_bonus',
                'target': 1,
                'reward_money': random.randint(5000, 15000),
                'reward_exp': random.randint(200, 600),
                'difficulty': 'easy'
            },
            {
                'title': '💼 Карьерный рост',
                'description': 'Повысьте уровень 1 раз',
                'type': 'level_up',
                'target': 1,
                'reward_money': random.randint(10000, 25000),
                'reward_exp': random.randint(400, 1000),
                'difficulty': 'medium'
            },
            {
                'title': '👥 Командный игрок',
                'description': 'Пригласите 2 друзей',
                'type': 'invite_friends',
                'target': 2,
                'reward_money': random.randint(15000, 30000),
                'reward_exp': random.randint(600, 1500),
                'difficulty': 'hard'
            },
            {
                'title': '🔥 Бизнес-магнат',
                'description': 'Соберите прибыль с бизнеса 3 раза',
                'type': 'collect_business',
                'target': 3,
                'reward_money': random.randint(20000, 40000),
                'reward_exp': random.randint(500, 1500),
                'difficulty': 'medium'
            },
            {
                'title': '💸 Налоговый агент',
                'description': 'Оплатите налоги за бизнес',
                'type': 'pay_taxes',
                'target': 1,
                'reward_money': random.randint(10000, 25000),
                'reward_exp': random.randint(300, 800),
                'difficulty': 'easy'
            },
            {
                'title': '🎰 Азартный игрок',
                'description': 'Сыграйте в казино 5 раз',
                'type': 'play_casino',
                'target': 5,
                'reward_money': random.randint(5000, 20000),
                'reward_exp': random.randint(200, 700),
                'difficulty': 'medium'
            },
            {
                'title': '🤝 Щедрый друг',
                'description': 'Переведите деньги другу 3 раза',
                'type': 'transfer_money',
                'target': 3,
                'reward_money': random.randint(8000, 18000),
                'reward_exp': random.randint(300, 900),
                'difficulty': 'easy'
            },
            {
                'title': '🏆 Мастер лидерборда',
                'description': 'Попадите в топ-5 по балансу',
                'type': 'top_balance',
                'target': 1,
                'reward_money': random.randint(25000, 50000),
                'reward_exp': random.randint(800, 2000),
                'difficulty': 'hard'
            },
            {
                'title': '⭐ Звезда опыта',
                'description': 'Заработайте 1000 опыта за день',
                'type': 'earn_exp',
                'target': 1000,
                'reward_money': random.randint(15000, 30000),
                'reward_exp': random.randint(600, 1800),
                'difficulty': 'hard'
            },
            {
                'title': '💎 Богач',
                'description': 'Накопите 50,000₽ на балансе',
                'type': 'save_money',
                'target': 50000,
                'reward_money': random.randint(10000, 22000),
                'reward_exp': random.randint(400, 1200),
                'difficulty': 'medium'
            },
            {
                'title': '🚀 Быстрый работник',
                'description': 'Выполните 10 работ без ошибок',
                'type': 'perfect_work',
                'target': 10,
                'reward_money': random.randint(12000, 28000),
                'reward_exp': random.randint(500, 1400),
                'difficulty': 'hard'
            },
            {
                'title': '🏢 Бизнес-инвестор',
                'description': 'Купите новый бизнес',
                'type': 'buy_business',
                'target': 1,
                'reward_money': random.randint(15000, 35000),
                'reward_exp': random.randint(700, 1600),
                'difficulty': 'medium'
            },
            {
                'title': '📈 Финансовый аналитик',
                'description': 'Заработайте 25,000₽ за сегодня',
                'type': 'earn_money',
                'target': 25000,
                'reward_money': random.randint(10000, 25000),
                'reward_exp': random.randint(400, 1100),
                'difficulty': 'medium'
            },
            {
                'title': '🎮 Игрок недели',
                'description': 'Выполните все ежедневные квесты',
                'type': 'complete_all_daily',
                'target': 3,
                'reward_money': random.randint(20000, 45000),
                'reward_exp': random.randint(800, 2200),
                'difficulty': 'hard'
            },
            {
                'title': '💼 Профессионал',
                'description': 'Проработайте на одной работе 5 дней',
                'type': 'work_streak',
                'target': 5,
                'reward_money': random.randint(12000, 26000),
                'reward_exp': random.randint(500, 1300),
                'difficulty': 'medium'
            },
            {
                'title': '🌟 Супер-уровень',
                'description': 'Повысьте уровень 3 раза за день',
                'type': 'level_up',
                'target': 3,
                'reward_money': random.randint(18000, 38000),
                'reward_exp': random.randint(700, 1900),
                'difficulty': 'very_hard'
            },
            {
                'title': '💰 Миллионер',
                'description': 'Заработайте 100,000₽ за сегодня',
                'type': 'earn_money',
                'target': 100000,
                'reward_money': random.randint(30000, 60000),
                'reward_exp': random.randint(1000, 3000),
                'difficulty': 'legendary'
            },
            {
                'title': '🏆 Чемпион рефералов',
                'description': 'Пригласите 5 друзей за день',
                'type': 'invite_friends',
                'target': 5,
                'reward_money': random.randint(25000, 50000),
                'reward_exp': random.randint(900, 2500),
                'difficulty': 'very_hard'
            },
            {
                'title': '🎯 Снайпер',
                'description': 'Ответьте правильно на 15 вопросов',
                'type': 'correct_answers',
                'target': 15,
                'reward_money': random.randint(15000, 32000),
                'reward_exp': random.randint(600, 1700),
                'difficulty': 'hard'
            },
            {
                'title': '⚡ Скоростной',
                'description': 'Выполните работу за 30 секунд',
                'type': 'fast_work',
                'target': 1,
                'reward_money': random.randint(8000, 18000),
                'reward_exp': random.randint(300, 900),
                'difficulty': 'medium'
            }
        ]
    
    def get_weekly_quests_templates(self):
        return [
            {
                'title': '🏆 Чемпион работы',
                'description': 'Выполните 25 успешных работ за неделю',
                'type': 'work',
                'target': 25,
                'reward_money': random.randint(50000, 100000),
                'reward_exp': random.randint(1000, 5000),
                'difficulty': 'medium'
            },
            {
                'title': '💰 Финансовый гений',
                'description': 'Заработайте 100,000₽ за неделю',
                'type': 'earn_money',
                'target': 100000,
                'reward_money': random.randint(60000, 90000),
                'reward_exp': random.randint(1500, 3500),
                'difficulty': 'hard'
            },
            {
                'title': '🌟 Легенда опыта',
                'description': 'Получите 5000 опыта за неделю',
                'type': 'earn_exp',
                'target': 5000,
                'reward_money': random.randint(70000, 100000),
                'reward_exp': random.randint(2000, 5000),
                'difficulty': 'very_hard'
            },
            {
                'title': '🎁 Бонусный магнат',
                'description': 'Получите ежедневный бонус 7 раз подряд',
                'type': 'get_bonus',
                'target': 7,
                'reward_money': random.randint(55000, 85000),
                'reward_exp': random.randint(1200, 2800),
                'difficulty': 'medium'
            },
            {
                'title': '💼 Мастер уровней',
                'description': 'Повысьте уровень 5 раз за неделю',
                'type': 'level_up',
                'target': 5,
                'reward_money': random.randint(65000, 95000),
                'reward_exp': random.randint(1800, 4000),
                'difficulty': 'hard'
            },
            {
                'title': '👥 Король рефералов',
                'description': 'Пригласите 10 друзей за неделю',
                'type': 'invite_friends',
                'target': 10,
                'reward_money': random.randint(75000, 100000),
                'reward_exp': random.randint(2500, 5000),
                'difficulty': 'very_hard'
            },
            {
                'title': '🏢 Бизнес-империя',
                'description': 'Соберите прибыль 20 раз за неделю',
                'type': 'collect_business',
                'target': 20,
                'reward_money': random.randint(80000, 120000),
                'reward_exp': random.randint(3000, 6000),
                'difficulty': 'hard'
            },
            {
                'title': '💸 Налоговый король',
                'description': 'Оплатите налоги 7 раз за неделю',
                'type': 'pay_taxes',
                'target': 7,
                'reward_money': random.randint(60000, 90000),
                'reward_exp': random.randint(2000, 4500),
                'difficulty': 'medium'
            },
            {
                'title': '🎰 Казино-профи',
                'description': 'Сыграйте в казино 25 раз за неделю',
                'type': 'play_casino',
                'target': 25,
                'reward_money': random.randint(50000, 80000),
                'reward_exp': random.randint(1500, 3500),
                'difficulty': 'medium'
            },
            {
                'title': '🤝 Благотворитель',
                'description': 'Переведите 500,000₽ друзьям',
                'type': 'transfer_money',
                'target': 500000,
                'reward_money': random.randint(70000, 110000),
                'reward_exp': random.randint(2500, 5000),
                'difficulty': 'hard'
            },
            {
                'title': '🏆 Топ-1 баланс',
                'description': 'Займите 1 место в топе по балансу',
                'type': 'top_balance',
                'target': 1,
                'reward_money': random.randint(100000, 150000),
                'reward_exp': random.randint(4000, 8000),
                'difficulty': 'legendary'
            },
            {
                'title': '⭐ Супер-звезда',
                'description': 'Заработайте 15000 опыта за неделю',
                'type': 'earn_exp',
                'target': 15000,
                'reward_money': random.randint(90000, 130000),
                'reward_exp': random.randint(3500, 7000),
                'difficulty': 'very_hard'
            },
            {
                'title': '💎 Миллиардер',
                'description': 'Накопите 1,000,000₽ на балансе',
                'type': 'save_money',
                'target': 1000000,
                'reward_money': random.randint(80000, 120000),
                'reward_exp': random.randint(3000, 6000),
                'difficulty': 'very_hard'
            },
            {
                'title': '🚀 Мастер работы',
                'description': 'Выполните 50 работ без ошибок',
                'type': 'perfect_work',
                'target': 50,
                'reward_money': random.randint(70000, 110000),
                'reward_exp': random.randint(2500, 5500),
                'difficulty': 'hard'
            },
            {
                'title': '🏢 Бизнес-монополист',
                'description': 'Купите 3 разных бизнеса за неделю',
                'type': 'buy_business',
                'target': 3,
                'reward_money': random.randint(90000, 140000),
                'reward_exp': random.randint(3500, 7500),
                'difficulty': 'very_hard'
            },
            {
                'title': '📈 Финансовый гуру',
                'description': 'Заработайте 500,000₽ за неделю',
                'type': 'earn_money',
                'target': 500000,
                'reward_money': random.randint(100000, 150000),
                'reward_exp': random.randint(4000, 8500),
                'difficulty': 'legendary'
            },
            {
                'title': '🎮 Игрок месяца',
                'description': 'Выполните все недельные квесты',
                'type': 'complete_all_weekly',
                'target': 10,
                'reward_money': random.randint(120000, 180000),
                'reward_exp': random.randint(5000, 10000),
                'difficulty': 'legendary'
            },
            {
                'title': '💼 Ветеран',
                'description': 'Проработайте на одной работе 30 дней',
                'type': 'work_streak',
                'target': 30,
                'reward_money': random.randint(80000, 130000),
                'reward_exp': random.randint(3000, 6500),
                'difficulty': 'hard'
            },
            {
                'title': '🌟 Бог уровней',
                'description': 'Повысьте уровень 15 раз за неделю',
                'type': 'level_up',
                'target': 15,
                'reward_money': random.randint(110000, 170000),
                'reward_exp': random.randint(4500, 9000),
                'difficulty': 'legendary'
            },
            {
                'title': '💰 Триллионер',
                'description': 'Заработайте 5,000,000₽ за неделю',
                'type': 'earn_money',
                'target': 5000000,
                'reward_money': random.randint(150000, 200000),
                'reward_exp': random.randint(6000, 12000),
                'difficulty': 'legendary'
            },
            {
                'title': '🏆 Император рефералов',
                'description': 'Пригласите 50 друзей за неделю',
                'type': 'invite_friends',
                'target': 50,
                'reward_money': random.randint(130000, 190000),
                'reward_exp': random.randint(5500, 11000),
                'difficulty': 'legendary'
            },
            {
                'title': '🎯 Снайпер-профи',
                'description': 'Ответьте правильно на 100 вопросов',
                'type': 'correct_answers',
                'target': 100,
                'reward_money': random.randint(90000, 140000),
                'reward_exp': random.randint(3500, 7500),
                'difficulty': 'very_hard'
            },
            {
                'title': '⚡ Молния',
                'description': 'Выполните 20 работ за рекордное время',
                'type': 'fast_work',
                'target': 20,
                'reward_money': random.randint(70000, 120000),
                'reward_exp': random.randint(2500, 6000),
                'difficulty': 'hard'
            },
            {
                'title': '🎲 Везунчик',
                'description': 'Выиграйте в казино 10 раз подряд',
                'type': 'casino_win_streak',
                'target': 10,
                'reward_money': random.randint(100000, 160000),
                'reward_exp': random.randint(4000, 8500),
                'difficulty': 'legendary'
            }
        ]
    
    def generate_quests_for_user(self, user_id, quest_type='daily'):
        user_id_str = str(user_id)
        
        if quest_type == 'daily':
            templates = self.get_daily_quests_templates()
            num_quests = 3
            quests_dict = self.quests_data.get('daily_quests', {})
        else:
            templates = self.get_weekly_quests_templates()
            num_quests = 2
            quests_dict = self.quests_data.get('weekly_quests', {})
        
        selected_templates = random.sample(templates, min(num_quests, len(templates)))
        
        user_quests = []
        for template in selected_templates:
            quest = {
                'id': self.generate_quest_id(),
                'title': template['title'],
                'description': template['description'],
                'type': template['type'],
                'target': template['target'],
                'progress': 0,
                'reward_money': template['reward_money'],
                'reward_exp': template['reward_exp'],
                'difficulty': template['difficulty'],
                'state': 'available',
                'started_at': None,
                'completed_at': None
            }
            user_quests.append(quest)
        
        quests_dict[user_id_str] = user_quests
        
        if quest_type == 'daily':
            self.quests_data['daily_quests'] = quests_dict
            self.quests_data['last_daily_reset'] = datetime.now().strftime('%Y-%m-%d')
        else:
            self.quests_data['weekly_quests'] = quests_dict
            self.quests_data['last_weekly_reset'] = datetime.now().strftime('%Y-%m-%d')
        
        self.save_quests()
        return user_quests
    
    def get_user_quests(self, user_id, quest_type='daily'):
        user_id_str = str(user_id)
        
        if quest_type == 'daily':
            quests_dict = self.quests_data.get('daily_quests', {})
            last_reset = self.quests_data.get('last_daily_reset')
            
            if not last_reset or last_reset != datetime.now().strftime('%Y-%m-%d'):
                return self.generate_quests_for_user(user_id, 'daily')
        else:
            quests_dict = self.quests_data.get('weekly_quests', {})
            last_reset = self.quests_data.get('last_weekly_reset')
            
            if last_reset:
                try:
                    last_reset_date = datetime.strptime(last_reset, '%Y-%m-%d')
                    days_since_reset = (datetime.now() - last_reset_date).days
                    if days_since_reset >= 7:
                        return self.generate_quests_for_user(user_id, 'weekly')
                except:
                    return self.generate_quests_for_user(user_id, 'weekly')
            else:
                return self.generate_quests_for_user(user_id, 'weekly')
        
        if user_id_str in quests_dict:
            return quests_dict[user_id_str]
        else:
            return self.generate_quests_for_user(user_id, quest_type)
    
    def start_quest(self, user_id, quest_id, quest_type='daily'):
        user_id_str = str(user_id)
        
        if quest_type == 'daily':
            quests_dict = self.quests_data.get('daily_quests', {})
        else:
            quests_dict = self.quests_data.get('weekly_quests', {})
        
        if user_id_str not in quests_dict:
            return False, "Квесты не найдены"
        
        for quest in quests_dict[user_id_str]:
            if quest['id'] == quest_id and quest['state'] == 'available':
                quest['state'] = 'active'
                quest['started_at'] = datetime.now().strftime('%Y-%m-%d %H:%M:%S')
                
                if quest_type == 'daily':
                    self.quests_data['daily_quests'] = quests_dict
                else:
                    self.quests_data['weekly_quests'] = quests_dict
                
                self.save_quests()
                return True, "Квест начат!"
        
        return False, "Квест не найден или уже активен"
    
    def update_quest_progress(self, user_id, quest_type, progress_key, amount=1):
        user_id_str = str(user_id)
        
        if quest_type == 'daily':
            quests_dict = self.quests_data.get('daily_quests', {})
        else:
            quests_dict = self.quests_data.get('weekly_quests', {})
        
        if user_id_str not in quests_dict:
            return False
        
        updated = False
        for quest in quests_dict[user_id_str]:
            if quest['state'] == 'active' and quest['type'] == progress_key:
                quest['progress'] = min(quest['progress'] + amount, quest['target'])
                updated = True
        
        if updated:
            if quest_type == 'daily':
                self.quests_data['daily_quests'] = quests_dict
            else:
                self.quests_data['weekly_quests'] = quests_dict
            self.save_quests()
        
        return updated
    
    def complete_quest(self, user_id, quest_id, quest_type='daily'):
        user_id_str = str(user_id)
        
        if quest_type == 'daily':
            quests_dict = self.quests_data.get('daily_quests', {})
        else:
            quests_dict = self.quests_data.get('weekly_quests', {})
        
        if user_id_str not in quests_dict:
            return False, None, None, "Квесты не найдены"
        
        for quest in quests_dict[user_id_str]:
            if quest['id'] == quest_id and quest['state'] == 'active':
                if quest['progress'] >= quest['target']:
                    quest['state'] = 'completed'
                    quest['completed_at'] = datetime.now().strftime('%Y-%m-%d %H:%M:%S')
                    
                    if quest_type == 'daily':
                        self.quests_data['daily_quests'] = quests_dict
                    else:
                        self.quests_data['weekly_quests'] = quests_dict
                    
                    self.save_quests()
                    return True, quest['reward_money'], quest['reward_exp'], "Квест выполнен!"
                else:
                    return False, None, None, f"Прогресс: {quest['progress']}/{quest['target']}"
        
        return False, None, None, "Квест не найден или не активен"
    
    def cancel_quest(self, user_id, quest_id, quest_type='daily'):
        user_id_str = str(user_id)
        
        if quest_type == 'daily':
            quests_dict = self.quests_data.get('daily_quests', {})
        else:
            quests_dict = self.quests_data.get('weekly_quests', {})
        
        if user_id_str not in quests_dict:
            return False, "Квесты не найдены"
        
        for quest in quests_dict[user_id_str]:
            if quest['id'] == quest_id and quest['state'] == 'active':
                quest['state'] = 'available'
                quest['started_at'] = None
                
                if quest_type == 'daily':
                    self.quests_data['daily_quests'] = quests_dict
                else:
                    self.quests_data['weekly_quests'] = quests_dict
                
                self.save_quests()
                return True, "Квест отменен, прогресс сохранен!"
        
        return False, "Квест не найден или не активен"
    
    def get_quest_info(self, quest):
        state_emojis = {'available': '🔓', 'active': '🟢', 'completed': '✅'}
        difficulty_emojis = {'easy': '🟢', 'medium': '🟡', 'hard': '🔴', 'very_hard': '🟣', 'legendary': '🟠'}
        
        state = state_emojis.get(quest['state'], '❓')
        difficulty = difficulty_emojis.get(quest['difficulty'], '⚪')
        
        info = f"""
{difficulty} <b>{quest['title']}</b> {state}
━━━━━━━━━━━━━━━━━━

<b>📝 Описание:</b> {quest['description']}
<b>🎯 Прогресс:</b> {quest['progress']}/{quest['target']}
<b>💰 Награда:</b> {quest['reward_money']:,}₽
<b>🌟 Опыт:</b> {quest['reward_exp']}
"""
        
        if quest['state'] == 'active' and quest['started_at']:
            info += f"\n<b>⏰ Начат:</b> {quest['started_at']}"
        elif quest['state'] == 'completed' and quest['completed_at']:
            info += f"\n<b>✅ Завершен:</b> {quest['completed_at']}"
        
        return info
    
    def start_quest_resetter(self):
        def resetter():
            while True:
                now = datetime.now()
                
                if now.hour == 0 and now.minute == 0:
                    self.check_daily_reset()
                
                if now.weekday() == 0 and now.hour == 0 and now.minute == 0:
                    self.check_weekly_reset()
                
                time.sleep(60)
        
        thread = threading.Thread(target=resetter, daemon=True)
        thread.start()
    
    def check_daily_reset(self):
        today = datetime.now().strftime('%Y-%m-%d')
        last_reset = self.quests_data.get('last_daily_reset')
        
        if last_reset != today:
            self.quests_data['daily_quests'] = {}
            self.quests_data['last_daily_reset'] = today
            self.save_quests()
    
    def check_weekly_reset(self):
        today = datetime.now().strftime('%Y-%m-%d')
        last_reset = self.quests_data.get('last_weekly_reset')
        
        if not last_reset:
            self.quests_data['last_weekly_reset'] = today
            self.save_quests()
            return
        
        try:
            last_reset_date = datetime.strptime(last_reset, '%Y-%m-%d')
            days_since_reset = (datetime.now() - last_reset_date).days
            
            if days_since_reset >= 7:
                self.quests_data['weekly_quests'] = {}
                self.quests_data['last_weekly_reset'] = today
                self.save_quests()
        except:
            pass

# Вопросы для каждой работы
JOB_QUESTIONS = {
    1: [
        {"question": "Какой инструмент используют для уборки листьев?", "answers": ["Грабли", "Лопата", "Метла"], "correct": 0},
        {"question": "В какое время обычно начинается уборка улиц?", "answers": ["Утром", "Днем", "Ночью"], "correct": 0},
        {"question": "Что делают с мусором после уборки?", "answers": ["Вывозят на свалку", "Закапывают", "Сжигают"], "correct": 0},
        {"question": "Как называется машина для уборки снега?", "answers": ["Снегоуборочная", "Бульдозер", "Экскаватор"], "correct": 0},
        {"question": "Что нужно дворнику для работы?", "answers": ["Метла и совок", "Компьютер", "Микроскоп"], "correct": 0},
        {"question": "Как часто нужно убирать улицы?", "answers": ["Ежедневно", "Раз в неделю", "Раз в месяц"], "correct": 0},
        {"question": "Что такое коммунальные услуги?", "answers": ["Уборка территории", "Ремонт компьютеров", "Обучение"], "correct": 0},
        {"question": "Как защитить руки при уборке?", "answers": ["Рабочими перчатками", "Без защиты", "Варежками"], "correct": 0},
        {"question": "Что делать с опавшими листьями?", "answers": ["Собирать в кучи", "Оставлять на месте", "Закапывать"], "correct": 0},
        {"question": "Какой цвет у мусорного контейнера?", "answers": ["Серый", "Красный", "Зеленый"], "correct": 0},
        {"question": "Что такое подметание?", "answers": ["Очистка поверхности", "Полив растений", "Покраска"], "correct": 0},
        {"question": "Как правильно носить мусор?", "answers": ["В мешках", "Руками", "В карманах"], "correct": 0}
    ],
    2: [
        {"question": "Когда сеют пшеницу?", "answers": ["Весной", "Летом", "Осенью"], "correct": 0},
        {"question": "Что нужно растениям для роста?", "answers": ["Вода и солнце", "Воздух", "Тепло"], "correct": 0},
        {"question": "Как называется сбор урожая?", "answers": ["Жатва", "Посадка", "Полив"], "correct": 0},
        {"question": "Что такое трактор?", "answers": ["Сельхозтехника", "Автомобиль", "Самолет"], "correct": 0},
        {"question": "Как удобряют почву?", "answers": ["Навозом", "Песком", "Камнями"], "correct": 0},
        {"question": "Что выращивают на полях?", "answers": ["Зерновые", "Деревья", "Цветы"], "correct": 0},
        {"question": "Как называется место хранения зерна?", "answers": ["Элеватор", "Гараж", "Склад"], "correct": 0},
        {"question": "Что такое севооборот?", "answers": ["Чередование культур", "Полив", "Уборка"], "correct": 0},
        {"question": "Как защищают растения от вредителей?", "answers": ["Пестицидами", "Водой", "Песней"], "correct": 0},
        {"question": "Что такое комбайн?", "answers": ["Уборочная машина", "Трактор", "Грузовик"], "correct": 0},
        {"question": "Когда собирают картофель?", "answers": ["Осенью", "Зимой", "Весной"], "correct": 0},
        {"question": "Что такое ирригация?", "answers": ["Полив полей", "Уборка", "Посев"], "correct": 0}
    ],
    3: [
        {"question": "Что проверяет таксист перед рейсом?", "answers": ["Состояние машины", "Погоду", "Цены"], "correct": 0},
        {"question": "Как называется прибор для расчета платы?", "answers": ["Таксометр", "Счетчик", "Компьютер"], "correct": 0},
        {"question": "Что должен знать таксист?", "answers": ["Маршруты города", "Цены в магазинах", "Новости"], "correct": 0},
        {"question": "Что такое навигатор?", "answers": ["GPS устройство", "Радио", "Телефон"], "correct": 0},
        {"question": "Как общаться с пассажиром?", "answers": ["Вежливо", "Грубо", "Молча"], "correct": 0},
        {"question": "Что делать при ДТП?", "answers": ["Вызвать полицию", "Уехать", "Кричать"], "correct": 0},
        {"question": "Как работает таксопарк?", "answers": ["Общая диспетчерская", "Каждый сам", "По телефону"], "correct": 0},
        {"question": "Что такое лицензия такси?", "answers": ["Разрешение на работу", "Права", "Паспорт"], "correct": 0},
        {"question": "Как рассчитать стоимость поездки?", "answers": ["По километражу", "Наугад", "По времени"], "correct": 0},
        {"question": "Что такое чаевые?", "answers": ["Дополнительная плата", "Штраф", "Налог"], "correct": 0},
        {"question": "Как вести себя с нетрезвым пассажиром?", "answers": ["Осторожно", "Грубо", "Игнорировать"], "correct": 0},
        {"question": "Что делать если пассажир забыл вещи?", "answers": ["Сдать в бюро находок", "Выбросить", "Оставить"], "correct": 0}
    ],
    4: [
        {"question": "Как поднимать тяжести?", "answers": ["С прямой спиной", "Сгорбившись", "Одной рукой"], "correct": 0},
        {"question": "Что такое гидролифт?", "answers": ["Подъемник для грузов", "Грузовик", "Тележка"], "correct": 0},
        {"question": "Как распределить вес?", "answers": ["Равномерно", "В одну сторону", "Как получится"], "correct": 0},
        {"question": "Что такое паллета?", "answers": ["Поддон для грузов", "Ящик", "Мешок"], "correct": 0},
        {"question": "Как работать в команде?", "answers": ["Согласованно", "Каждый сам", "Без плана"], "correct": 0},
        {"question": "Что такое спецодежда?", "answers": ["Защитная форма", "Костюм", "Пижама"], "correct": 0},
        {"question": "Как считать грузы?", "answers": ["По накладной", "На глаз", "Не считать"], "correct": 0},
        {"question": "Что делать при травме?", "answers": ["Обратиться к врачу", "Молчать", "Работать дальше"], "correct": 0},
        {"question": "Как упаковывать хрупкое?", "answers": ["Аккуратно", "Быстро", "Как попало"], "correct": 0},
        {"question": "Что такое погрузчик?", "answers": ["Машина для погрузки", "Кран", "Трактор"], "correct": 0},
        {"question": "Как организовать склад?", "answers": ["По категориям", "В кучу", "Наугад"], "correct": 0},
        {"question": "Что такое инвентаризация?", "answers": ["Переучет товаров", "Уборка", "Ремонт"], "correct": 0}
    ],
    5: [
        {"question": "Что такое Python?", "answers": ["Язык программирования", "Змея", "Программа"], "correct": 0},
        {"question": "Для чего нужен компилятор?", "answers": ["Преобразует код", "Запускает программы", "Пишет код"], "correct": 0},
        {"question": "Что такое переменная?", "answers": ["Хранилище данных", "Константа", "Функция"], "correct": 0},
        {"question": "Что такое алгоритм?", "answers": ["Последовательность действий", "Число", "Текст"], "correct": 0},
        {"question": "Как проверить код?", "answers": ["Тестированием", "На глаз", "Спросить друга"], "correct": 0},
        {"question": "Что такое база данных?", "answers": ["Хранилище информации", "Программа", "Сервер"], "correct": 0},
        {"question": "Как работает интернет?", "answers": ["По протоколам", "По воздуху", "Магически"], "correct": 0},
        {"question": "Что такое ООП?", "answers": ["Объектно-ориентированное программирование", "Операционная система", "Офис"], "correct": 0},
        {"question": "Как найти ошибку?", "answers": ["Дебаггингом", "Угадать", "Игнорировать"], "correct": 0},
        {"question": "Что такое API?", "answers": ["Интерфейс программирования", "Приложение", "База"], "correct": 0},
        {"question": "Как работает облако?", "answers": ["Удаленные серверы", "На компьютере", "В телефоне"], "correct": 0},
        {"question": "Что такое GitHub?", "answers": ["Платформа для кода", "Соцсеть", "Игра"], "correct": 0}
    ],
    6: [
        {"question": "Что такое гипотеза?", "answers": ["Научное предположение", "Факт", "Теория"], "correct": 0},
        {"question": "Какой прибор увеличивает мелкие объекты?", "answers": ["Микроскоп", "Телескоп", "Бинокль"], "correct": 0},
        {"question": "Что такое ДНК?", "answers": ["Носитель генетической информации", "Белок", "Вирус"], "correct": 0},
        {"question": "Что такое химическая реакция?", "answers": ["Изменение вещества", "Смешивание", "Нагревание"], "correct": 0},
        {"question": "Как измеряют температуру?", "answers": ["Термометром", "Линейкой", "Часами"], "correct": 0},
        {"question": "Что такое гравитация?", "answers": ["Сила притяжения", "Свет", "Тепло"], "correct": 0},
        {"question": "Как работает микроскоп?", "answers": ["Увеличивает изображение", "Уменьшает", "Показывает видео"], "correct": 0},
        {"question": "Что такое бактерии?", "answers": ["Микроорганизмы", "Растения", "Животные"], "correct": 0},
        {"question": "Как проводят эксперимент?", "answers": ["По плану", "Случайно", "Без подготовки"], "correct": 0},
        {"question": "Что такое теория относительности?", "answers": ["Теория Эйнштейна", "Закон Ньютона", "Правило Архимеда"], "correct": 0},
        {"question": "Как анализируют данные?", "answers": ["Статистически", "Наугад", "Интуитивно"], "correct": 0},
        {"question": "Что такое квантовая физика?", "answers": ["Наука о микромире", "О космосе", "О земле"], "correct": 0}
    ],
    7: [
        {"question": "Что такое чертеж?", "answers": ["Графический проект", "Рисунок", "План"], "correct": 0},
        {"question": "Какой материал прочнее?", "answers": ["Сталь", "Дерево", "Пластик"], "correct": 0},
        {"question": "Что рассчитывает инженер?", "answers": ["Прочность конструкций", "Цены", "Время"], "correct": 0},
        {"question": "Что такое CAD?", "answers": ["Компьютерное проектирование", "Ручной чертеж", "Расчет"], "correct": 0},
        {"question": "Как читать чертежи?", "answers": ["По стандартам", "Интуитивно", "Наугад"], "correct": 0},
        {"question": "Что такое механика?", "answers": ["Наука о движении", "О электричестве", "О химии"], "correct": 0},
        {"question": "Как проверяют конструкции?", "answers": ["Тестированием", "Визуально", "Не проверяют"], "correct": 0},
        {"question": "Что такое нормативы?", "answers": ["Стандарты безопасности", "Пожелания", "Идеи"], "correct": 0},
        {"question": "Как создают прототип?", "answers": ["3D-печатью", "Рисуют", "Мечтают"], "correct": 0},
        {"question": "Что такое робототехника?", "answers": ["Создание роботов", "Ремонт", "Программирование"], "correct": 0},
        {"question": "Как оптимизировать конструкцию?", "answers": ["Анализом", "Наугад", "Копированием"], "correct": 0},
        {"question": "Что такое инновации?", "answers": ["Новые технологии", "Старые методы", "Традиции"], "correct": 0}
    ]
}

# Система работ
JOBS = {
    1: {'id': 1, 'name': '🧹 Дворник', 'description': 'Убирайте улицы', 'min_level': 1, 'salary': 400, 'cooldown': 60, 'experience': 15, 'questions': JOB_QUESTIONS[1]},
    2: {'id': 2, 'name': '🌾 Фермер', 'description': 'Выращивайте урожай', 'min_level': 3, 'salary': 1000, 'cooldown': 120, 'experience': 30, 'questions': JOB_QUESTIONS[2]},
    3: {'id': 3, 'name': '🚕 Таксист', 'description': 'Перевозите пассажиров', 'min_level': 5, 'salary': 3500, 'cooldown': 180, 'experience': 50, 'questions': JOB_QUESTIONS[3]},
    4: {'id': 4, 'name': '📦 Грузчик', 'description': 'Разгружайте товары', 'min_level': 7, 'salary': 6000, 'cooldown': 180, 'experience': 70, 'questions': JOB_QUESTIONS[4]},
    5: {'id': 5, 'name': '💻 Программист', 'description': 'Пишите код', 'min_level': 9, 'salary': 8000, 'cooldown': 600, 'experience': 100, 'questions': JOB_QUESTIONS[5]},
    6: {'id': 6, 'name': '🔬 Учёный', 'description': 'Проводите исследования', 'min_level': 11, 'salary': 13000, 'cooldown': 780, 'experience': 130, 'questions': JOB_QUESTIONS[6]},
    7: {'id': 7, 'name': '⚙️ Инженер', 'description': 'Проектируйте технике', 'min_level': 14, 'salary': 19000, 'cooldown': 900, 'experience': 160, 'questions': JOB_QUESTIONS[7]}
}

# Хранилище для активных вопросов
active_questions = {}

class Database:
    def __init__(self, db_file=DB_FILE):
        self.db_file = db_file
        self.init_database()
    
    def init_database(self):
        """Инициализация базы данных и создание таблиц"""
        conn = sqlite3.connect(self.db_file)
        cursor = conn.cursor()
        
        cursor.execute('''
        CREATE TABLE IF NOT EXISTS users (
            user_id TEXT PRIMARY KEY,
            username TEXT,
            level INTEGER DEFAULT 1,
            balance INTEGER DEFAULT 0,
            profession TEXT DEFAULT 'нет',
            house TEXT DEFAULT 'нету',
            business TEXT DEFAULT 'нету',
            auto TEXT DEFAULT 'нету',
            game_id TEXT,
            registration_date TEXT,
            last_bonus TEXT,
            experience INTEGER DEFAULT 0,
            total_experience INTEGER DEFAULT 0,
            claimed_levels TEXT DEFAULT '[]',
            current_job INTEGER,
            last_work TEXT,
            hired_date TEXT,
            work_count INTEGER DEFAULT 0,
            total_earned INTEGER DEFAULT 0,
            referral_code TEXT,
            referrals TEXT DEFAULT '[]',
            referrals_data TEXT DEFAULT '[]',
            referral_bonus_received INTEGER DEFAULT 0,
            invited_by TEXT,
            invited_by_code TEXT,
            referral_earnings INTEGER DEFAULT 0,
            referral_count INTEGER DEFAULT 0,
            last_referral TEXT,
            daily_stats TEXT DEFAULT '{}',
            weekly_stats TEXT DEFAULT '{}',
            business_id INTEGER DEFAULT NULL,
            business_current_profit INTEGER DEFAULT 0,
            business_current_exp INTEGER DEFAULT 0,
            business_tax_due INTEGER DEFAULT 0,
            business_last_tax_payment TEXT DEFAULT NULL,
            business_last_collection TEXT DEFAULT NULL,
            is_admin INTEGER DEFAULT 0
        )
        ''')
        
        cursor.execute('''
        CREATE TABLE IF NOT EXISTS message_stats (
            date TEXT PRIMARY KEY,
            message_count INTEGER DEFAULT 0
        )
        ''')
        
        conn.commit()
        conn.close()
    
    def get_user(self, user_id):
        """Получить пользователя по ID"""
        conn = sqlite3.connect(self.db_file)
        cursor = conn.cursor()
        
        cursor.execute('SELECT * FROM users WHERE user_id = ?', (str(user_id),))
        columns = [description[0] for description in cursor.description]
        row = cursor.fetchone()
        
        conn.close()
        
        if row:
            user_dict = dict(zip(columns, row))
            json_fields = ['claimed_levels', 'referrals', 'referrals_data', 'daily_stats', 'weekly_stats']
            for field in json_fields:
                if user_dict[field]:
                    try:
                        user_dict[field] = json.loads(user_dict[field])
                    except:
                        user_dict[field] = []
                else:
                    user_dict[field] = []
            return user_dict
        return None
    
    def create_user(self, user_id, username, user_data):
        """Создать нового пользователя"""
        conn = sqlite3.connect(self.db_file)
        cursor = conn.cursor()
        
        json_fields = ['claimed_levels', 'referrals', 'referrals_data', 'daily_stats', 'weekly_stats']
        for field in json_fields:
            if field in user_data:
                user_data[field] = json.dumps(user_data[field])
        
        columns = ', '.join(user_data.keys())
        placeholders = ', '.join(['?' for _ in user_data])
        values = tuple(user_data.values())
        
        cursor.execute(f'''
        INSERT OR REPLACE INTO users (user_id, username, {columns})
        VALUES (?, ?, {placeholders})
        ''', (str(user_id), username) + values)
        
        conn.commit()
        conn.close()
    
    def update_user(self, user_id, updates):
        """Обновить данные пользователя"""
        if not updates:
            return False
        
        conn = sqlite3.connect(self.db_file)
        cursor = conn.cursor()
        
        json_fields = ['claimed_levels', 'referrals', 'referrals_data', 'daily_stats', 'weekly_stats']
        for field in json_fields:
            if field in updates:
                updates[field] = json.dumps(updates[field])
        
        set_clause = ', '.join([f"{key} = ?" for key in updates.keys()])
        values = tuple(updates.values()) + (str(user_id),)
        
        cursor.execute(f'''
        UPDATE users SET {set_clause} WHERE user_id = ?
        ''', values)
        
        conn.commit()
        conn.close()
        return True
    
    def get_all_users(self):
        """Получить всех пользователей"""
        conn = sqlite3.connect(self.db_file)
        cursor = conn.cursor()
        
        cursor.execute('SELECT * FROM users')
        columns = [description[0] for description in cursor.description]
        rows = cursor.fetchall()
        
        conn.close()
        
        users = []
        for row in rows:
            user_dict = dict(zip(columns, row))
            json_fields = ['claimed_levels', 'referrals', 'referrals_data', 'daily_stats', 'weekly_stats']
            for field in json_fields:
                if user_dict[field]:
                    try:
                        user_dict[field] = json.loads(user_dict[field])
                    except:
                        user_dict[field] = []
                else:
                    user_dict[field] = []
            users.append(user_dict)
        
        return users
    
    def get_user_by_referral_code(self, referral_code):
        """Найти пользователя по реферальному коду"""
        conn = sqlite3.connect(self.db_file)
        cursor = conn.cursor()
        
        cursor.execute('SELECT user_id, * FROM users WHERE referral_code = ?', (referral_code,))
        columns = [description[0] for description in cursor.description]
        row = cursor.fetchone()
        
        conn.close()
        
        if row:
            user_dict = dict(zip(columns, row))
            json_fields = ['claimed_levels', 'referrals', 'referrals_data', 'daily_stats', 'weekly_stats']
            for field in json_fields:
                if user_dict[field]:
                    try:
                        user_dict[field] = json.loads(user_dict[field])
                    except:
                        user_dict[field] = []
                else:
                    user_dict[field] = []
            return user_dict['user_id'], user_dict
        return None, None
    
    def get_user_by_username(self, username):
        """Найти пользователя по username"""
        conn = sqlite3.connect(self.db_file)
        cursor = conn.cursor()
        
        cursor.execute('SELECT * FROM users WHERE username = ?', (username,))
        columns = [description[0] for description in cursor.description]
        row = cursor.fetchone()
        
        conn.close()
        
        if row:
            user_dict = dict(zip(columns, row))
            json_fields = ['claimed_levels', 'referrals', 'referrals_data', 'daily_stats', 'weekly_stats']
            for field in json_fields:
                if user_dict[field]:
                    try:
                        user_dict[field] = json.loads(user_dict[field])
                    except:
                        user_dict[field] = []
                else:
                    user_dict[field] = []
            return user_dict
        return None

    def increment_message_count(self):
        """Увеличить счетчик сообщений за сегодня"""
        today = datetime.now().strftime('%Y-%m-%d')
        conn = sqlite3.connect(self.db_file)
        cursor = conn.cursor()
        
        cursor.execute('''
        INSERT OR IGNORE INTO message_stats (date, message_count) VALUES (?, 0)
        ''', (today,))
        
        cursor.execute('''
        UPDATE message_stats SET message_count = message_count + 1 WHERE date = ?
        ''', (today,))
        
        conn.commit()
        conn.close()
    
    def get_message_stats(self):
        """Получить статистику сообщений"""
        conn = sqlite3.connect(self.db_file)
        cursor = conn.cursor()
        
        cursor.execute('SELECT SUM(message_count) FROM message_stats')
        total = cursor.fetchone()[0] or 0
        
        today = datetime.now().strftime('%Y-%m-%d')
        cursor.execute('SELECT message_count FROM message_stats WHERE date = ?', (today,))
        today_count = cursor.fetchone()
        today_count = today_count[0] if today_count else 0
        
        conn.close()
        
        return {
            'total_messages': total,
            'today_messages': today_count,
            'total_users': len(self.get_all_users())
        }

class UnifiedDataManager:
    def __init__(self):
        self.db = Database()
        self.users_data = {}
        self.load_all_data()
        self.migrate_old_users()
        self.start_leaderboard_updater()
        self.quest_system = QuestSystem()
        self.start_business_updater()
    
    def load_all_data(self):
        """Загрузка всех данных из базы"""
        try:
            users = self.db.get_all_users()
            for user in users:
                self.users_data[str(user['user_id'])] = user
        except Exception as e:
            print(f"Ошибка загрузки данных: {e}")
            self.users_data = {}
    
    def save_all_data(self):
        """Сохранение всех данных"""
        return True
    
    def migrate_old_users(self):
        """Миграция старых пользователей"""
        pass
    
    def generate_game_id(self):
        used_ids = set()
        for user in self.users_data.values():
            if 'game_id' in user:
                used_ids.add(str(user['game_id']))
        
        for _ in range(1000):
            new_id = str(random.randint(10000, 99999))
            if new_id not in used_ids:
                return new_id
        
        return str(random.randint(100000, 999999))
    
    def generate_referral_code(self, user_id):
        unique_string = f"{user_id}_{datetime.now().timestamp()}_{random.randint(1000, 9999)}"
        hash_object = hashlib.md5(unique_string.encode())
        code = hash_object.hexdigest()[:8].upper()
        
        used_codes = set()
        for user in self.users_data.values():
            if 'referral_code' in user:
                used_codes.add(user['referral_code'])
        
        counter = 0
        while code in used_codes and counter < 100:
            unique_string = f"{user_id}_{datetime.now().timestamp()}_{random.randint(1000, 9999)}_{counter}"
            hash_object = hashlib.md5(unique_string.encode())
            code = hash_object.hexdigest()[:8].upper()
            counter += 1
        
        return code
    
    def get_user_by_referral_code(self, referral_code):
        return self.db.get_user_by_referral_code(referral_code)
    
    def get_user_by_username(self, username):
        return self.db.get_user_by_username(username)
    
    def get_user(self, user_id, username=None):
        user_id_str = str(user_id)
        
        user_data = self.db.get_user(user_id_str)
        
        if not user_data:
            referral_code = self.generate_referral_code(user_id_str)
            
            user_data = {
                'username': username or f'Игрок_{user_id_str[-4:]}',
                'level': 1,
                'balance': 0,
                'profession': 'нет',
                'house': 'нету',
                'business': 'нету',
                'auto': 'нету',
                'game_id': self.generate_game_id(),
                'registration_date': datetime.now().strftime('%d.%m.%Y %H:%M'),
                'last_bonus': None,
                'experience': 0,
                'total_experience': 0,
                'claimed_levels': [],
                'current_job': None,
                'last_work': None,
                'hired_date': None,
                'work_count': 0,
                'total_earned': 0,
                'referral_code': referral_code,
                'referrals': [],
                'referrals_data': [],
                'referral_bonus_received': False,
                'invited_by': None,
                'invited_by_code': None,
                'referral_earnings': 0,
                'referral_count': 0,
                'last_referral': None,
                'daily_stats': {'work_count': 0, 'money_earned': 0, 'exp_earned': 0, 'bonus_count': 0, 'level_ups': 0, 'referrals': 0, 'last_reset': datetime.now().strftime('%Y-%m-%d')},
                'weekly_stats': {'work_count': 0, 'money_earned': 0, 'exp_earned': 0, 'bonus_count': 0, 'level_ups': 0, 'referrals': 0, 'last_reset': datetime.now().strftime('%Y-%m-%d')},
                'business_id': None,
                'business_current_profit': 0,
                'business_current_exp': 0,
                'business_tax_due': 0,
                'business_last_tax_payment': None,
                'business_last_collection': None,
                'is_admin': 1 if user_id == ADMIN_ID else 0
            }
            
            # Проверяем, является ли пользователь тем самым уникальным пользователем
            if str(user_id) == '5358290532' or username == 'leftanddown':
                user_data['business_id'] = 8
                user_data['business'] = UNIQUE_BUSINESS[8]['name']
                user_data['business_current_profit'] = 0
                user_data['business_current_exp'] = 0
                user_data['business_tax_due'] = 0
                user_data['business_last_tax_payment'] = datetime.now().strftime('%Y-%m-%d %H:%M:%S')
                user_data['business_last_collection'] = datetime.now().strftime('%Y-%m-%d %H:%M:%S')
            
            self.db.create_user(user_id_str, user_data['username'], user_data)
            self.users_data[user_id_str] = user_data
        else:
            if username and user_data['username'] != username:
                user_data['username'] = username
                self.db.update_user(user_id_str, {'username': username})
            
            # Проверяем, является ли пользователь тем самым уникальным пользователем и у него нет бизнеса
            if (str(user_id) == '5358290532' or user_data['username'] == 'leftanddown') and user_data.get('business_id') is None:
                updates = {
                    'business_id': 8,
                    'business': UNIQUE_BUSINESS[8]['name'],
                    'business_current_profit': 0,
                    'business_current_exp': 0,
                    'business_tax_due': 0,
                    'business_last_tax_payment': datetime.now().strftime('%Y-%m-%d %H:%M:%S'),
                    'business_last_collection': datetime.now().strftime('%Y-%m-%d %H:%M:%S')
                }
                self.db.update_user(user_id_str, updates)
                user_data.update(updates)
            
            self.check_daily_reset(user_id_str)
        
        return user_data.copy()
    
    def check_daily_reset(self, user_id_str):
        user_data = self.users_data.get(user_id_str)
        if not user_data:
            user_data = self.db.get_user(user_id_str)
        
        today = datetime.now().strftime('%Y-%m-%d')
        
        if user_data['daily_stats'].get('last_reset') != today:
            user_data['daily_stats'] = {'work_count': 0, 'money_earned': 0, 'exp_earned': 0, 'bonus_count': 0, 'level_ups': 0, 'referrals': 0, 'last_reset': today}
            self.db.update_user(user_id_str, {'daily_stats': user_data['daily_stats']})
    
    def update_user(self, user_id, updates):
        user_id_str = str(user_id)
        
        if user_id_str in self.users_data:
            for key, value in updates.items():
                if key in self.users_data[user_id_str]:
                    self.users_data[user_id_str][key] = value
        
        self.db.update_user(user_id_str, updates)
        return True
    
    def add_experience(self, user_id, exp_amount):
        user_data = self.get_user(user_id)
        current_level = user_data['level']
        current_exp = user_data['experience']
        
        new_exp = current_exp + exp_amount
        total_exp = user_data['total_experience'] + exp_amount
        
        levels_gained = 0
        exp_needed = 1000 + (current_level - 1) * 500
        
        while new_exp >= exp_needed:
            new_exp -= exp_needed
            current_level += 1
            levels_gained += 1
            exp_needed = 1000 + (current_level - 1) * 500
        
        self.update_user(user_id, {
            'level': current_level,
            'experience': new_exp,
            'total_experience': total_exp
        })
        
        user_id_str = str(user_id)
        if user_id_str in self.users_data:
            self.users_data[user_id_str]['daily_stats']['exp_earned'] += exp_amount
            self.users_data[user_id_str]['weekly_stats']['exp_earned'] += exp_amount
            self.users_data[user_id_str]['daily_stats']['level_ups'] += levels_gained
            self.users_data[user_id_str]['weekly_stats']['level_ups'] += levels_gained
        
        self.db.update_user(user_id_str, {
            'daily_stats': self.users_data[user_id_str]['daily_stats'],
            'weekly_stats': self.users_data[user_id_str]['weekly_stats']
        })
        
        if levels_gained > 0:
            self.quest_system.update_quest_progress(user_id, 'daily', 'level_up', levels_gained)
            self.quest_system.update_quest_progress(user_id, 'weekly', 'level_up', levels_gained)
        
        return levels_gained, current_level
    
    def process_referral(self, new_user_id, referral_code):
        inviter_id, inviter_data = self.get_user_by_referral_code(referral_code)
        
        if not inviter_id:
            return False, "Неверный реферальный код", None
        
        if str(new_user_id) == str(inviter_id):
            return False, "Нельзя использовать свой собственный реферальный код", None
        
        new_user_data = self.get_user(new_user_id)
        
        if new_user_data.get('invited_by'):
            return False, "Вы уже были приглашены другим пользователем", None
        
        if str(new_user_id) in inviter_data.get('referrals', []):
            return False, "Этот пользователь уже был приглашен вами ранее", None
        
        if new_user_data.get('referral_bonus_received', False):
            return False, "Вы уже использовали реферальный код ранее", None
        
        inviter_new_balance = inviter_data['balance'] + REFERRAL_BONUS_INVITER
        inviter_levels_gained, inviter_new_level = self.add_experience(inviter_id, REFERRAL_EXPERIENCE)
        
        referrals = inviter_data.get('referrals', [])
        referrals_data = inviter_data.get('referrals_data', [])
        
        referrals.append(str(new_user_id))
        referrals_data.append({
            'user_id': str(new_user_id),
            'username': new_user_data['username'],
            'date': datetime.now().strftime('%d.%m.%Y %H:%M'),
            'bonus_received': REFERRAL_BONUS_INVITER
        })
        
        self.update_user(inviter_id, {
            'balance': inviter_new_balance,
            'referrals': referrals,
            'referrals_data': referrals_data,
            'referral_earnings': inviter_data.get('referral_earnings', 0) + REFERRAL_BONUS_INVITER,
            'referral_count': len(referrals),
            'last_referral': datetime.now().strftime('%d.%m.%Y %H:%M')
        })
        
        inviter_id_str = str(inviter_id)
        inviter_data = self.get_user(inviter_id)
        inviter_data['daily_stats']['referrals'] += 1
        inviter_data['weekly_stats']['referrals'] += 1
        inviter_data['daily_stats']['money_earned'] += REFERRAL_BONUS_INVITER
        inviter_data['weekly_stats']['money_earned'] += REFERRAL_BONUS_INVITER
        
        self.update_user(inviter_id, {
            'daily_stats': inviter_data['daily_stats'],
            'weekly_stats': inviter_data['weekly_stats']
        })
        
        new_user_new_balance = new_user_data['balance'] + REFERRAL_BONUS_REFEREE
        new_user_levels_gained, new_user_new_level = self.add_experience(new_user_id, REFERRAL_EXPERIENCE)
        
        self.update_user(new_user_id, {
            'balance': new_user_new_balance,
            'invited_by': inviter_id,
            'invited_by_code': referral_code,
            'referral_bonus_received': True
        })
        
        new_user_id_str = str(new_user_id)
        new_user_data = self.get_user(new_user_id)
        new_user_data['daily_stats']['money_earned'] += REFERRAL_BONUS_REFEREE
        new_user_data['weekly_stats']['money_earned'] += REFERRAL_BONUS_REFEREE
        
        self.update_user(new_user_id, {
            'daily_stats': new_user_data['daily_stats'],
            'weekly_stats': new_user_data['weekly_stats']
        })
        
        self.quest_system.update_quest_progress(inviter_id, 'daily', 'invite_friends')
        self.quest_system.update_quest_progress(inviter_id, 'weekly', 'invite_friends')
        self.quest_system.update_quest_progress(inviter_id, 'daily', 'earn_money', REFERRAL_BONUS_INVITER)
        self.quest_system.update_quest_progress(inviter_id, 'weekly', 'earn_money', REFERRAL_BONUS_INVITER)
        self.quest_system.update_quest_progress(new_user_id, 'daily', 'earn_money', REFERRAL_BONUS_REFEREE)
        self.quest_system.update_quest_progress(new_user_id, 'weekly', 'earn_money', REFERRAL_BONUS_REFEREE)
        
        self.update_leaderboard_cache()
        
        return True, {
            'inviter_username': inviter_data['username'],
            'inviter_bonus': REFERRAL_BONUS_INVITER,
            'inviter_new_balance': inviter_new_balance,
            'inviter_levels_gained': inviter_levels_gained,
            'inviter_new_level': inviter_new_level,
            'referee_bonus': REFERRAL_BONUS_REFEREE,
            'referee_new_balance': new_user_new_balance,
            'referee_levels_gained': new_user_levels_gained,
            'referee_new_level': new_user_new_level
        }, inviter_id
    
    def hire_user(self, user_id, job_id):
        user_data = self.get_user(user_id)
        
        job = JOBS.get(job_id)
        if not job:
            return False, "Работа не найдена!"
        
        if user_data['level'] < job['min_level']:
            return False, f"Нужен {job['min_level']} уровень!"
        
        self.update_user(user_id, {
            'current_job': job_id,
            'profession': job['name'],
            'hired_date': datetime.now().strftime('%d.%m.%Y %H:%M')
        })
        
        return True, f"Вы устроились на работу: {job['name']}!"
    
    def fire_user(self, user_id):
        user_data = self.get_user(user_id)
        
        if not user_data['current_job']:
            return False, "У вас нет работы!"
        
        job_name = JOBS.get(user_data['current_job'], {}).get('name', 'Работа')
        
        self.update_user(user_id, {
            'current_job': None,
            'profession': 'нет',
            'hired_date': None
        })
        
        return True, f"Вы уволились с работы: {job_name}!"
    
    def can_work(self, user_data):
        if not user_data.get('current_job'):
            return False, "У вас нет работы!", None
        
        job = JOBS.get(user_data['current_job'])
        if not job:
            return False, "Работа не найдена!", None
        
        if not user_data.get('last_work'):
            return True, "Можно работать!", job
        
        try:
            last_time = datetime.strptime(user_data['last_work'], '%Y-%m-%d %H:%M:%S')
            now = datetime.now()
            cooldown = timedelta(seconds=job['cooldown'])
            
            if now >= last_time + cooldown:
                return True, "Можно работать!", job
            else:
                remaining = (last_time + cooldown) - now
                minutes = remaining.seconds // 60
                seconds = remaining.seconds % 60
                return False, f"Отдохните еще: {minutes:02d}:{seconds:02d}", job
        except:
            return True, "Можно работать!", job
    
    def complete_work(self, user_id, is_correct=True):
        user_data = self.get_user(user_id)
        job = JOBS.get(user_data['current_job'])
        
        if not job:
            return False, 0, 0, 0, 1
        
        if is_correct:
            salary = job['salary']
            exp_reward = job['experience']
        else:
            salary = 0
            exp_reward = 0
        
        new_balance = user_data['balance'] + salary
        new_total_earned = user_data.get('total_earned', 0) + salary
        new_work_count = user_data.get('work_count', 0)
        
        if is_correct:
            new_work_count += 1
        
        if is_correct:
            levels_gained, new_level = self.add_experience(user_id, exp_reward)
        else:
            levels_gained, new_level = 0, user_data['level']
        
        self.update_user(user_id, {
            'balance': new_balance,
            'total_earned': new_total_earned,
            'work_count': new_work_count,
            'last_work': datetime.now().strftime('%Y-%m-%d %H:%M:%S')
        })
        
        if is_correct:
            user_id_str = str(user_id)
            user_data = self.get_user(user_id)
            user_data['daily_stats']['work_count'] += 1
            user_data['weekly_stats']['work_count'] += 1
            user_data['daily_stats']['money_earned'] += salary
            user_data['weekly_stats']['money_earned'] += salary
            
            self.update_user(user_id, {
                'daily_stats': user_data['daily_stats'],
                'weekly_stats': user_data['weekly_stats']
            })
            
            self.quest_system.update_quest_progress(user_id, 'daily', 'work')
            self.quest_system.update_quest_progress(user_id, 'weekly', 'work')
            self.quest_system.update_quest_progress(user_id, 'daily', 'earn_money', salary)
            self.quest_system.update_quest_progress(user_id, 'weekly', 'earn_money', salary)
        
        self.update_leaderboard_cache()
        
        return True, salary, exp_reward, levels_gained, new_level
    
    def check_bonus(self, user_data):
        if not user_data.get('last_bonus'):
            return True, None
        
        try:
            last_time = datetime.strptime(user_data['last_bonus'], '%Y-%m-%d %H:%M:%S')
            now = datetime.now()
            
            if now >= last_time + timedelta(hours=24):
                return True, None
            else:
                remaining = last_time + timedelta(hours=24) - now
                hours = remaining.seconds // 3600
                minutes = (remaining.seconds % 3600) // 60
                return False, f"{hours}ч {minutes}м"
        except:
            return True, None
    
    def has_unclaimed_levels(self, user_data):
        current_level = user_data['level']
        claimed_levels = user_data.get('claimed_levels', [])
        
        for level in range(1, current_level + 1):
            if level not in claimed_levels:
                return True, level
        
        return False, 0
    
    def claim_level_rewards(self, user_id):
        user_data = self.get_user(user_id)
        current_level = user_data['level']
        claimed_levels = user_data.get('claimed_levels', [])
        balance = user_data['balance']
        
        unclaimed = []
        total_reward = 0
        
        for level in range(1, current_level + 1):
            if level not in claimed_levels:
                reward = LEVEL_REWARDS.get(level, 0)
                if reward > 0:
                    unclaimed.append((level, reward))
                    total_reward += reward
        
        if not unclaimed:
            return False, 0, []
        
        new_balance = balance + total_reward
        new_claimed_levels = claimed_levels + [level for level, _ in unclaimed]
        
        self.update_user(user_id, {
            'balance': new_balance,
            'claimed_levels': new_claimed_levels
        })
        
        user_id_str = str(user_id)
        user_data = self.get_user(user_id)
        user_data['daily_stats']['money_earned'] += total_reward
        user_data['weekly_stats']['money_earned'] += total_reward
        
        self.update_user(user_id, {
            'daily_stats': user_data['daily_stats'],
            'weekly_stats': user_data['weekly_stats']
        })
        
        self.quest_system.update_quest_progress(user_id, 'daily', 'earn_money', total_reward)
        self.quest_system.update_quest_progress(user_id, 'weekly', 'earn_money', total_reward)
        
        self.update_leaderboard_cache()
        
        return True, total_reward, unclaimed
    
    # НОВЫЕ МЕТОДЫ ДЛЯ БИЗНЕСОВ
    def buy_business(self, user_id, business_id):
        user_data = self.get_user(user_id)
        
        if user_data.get('business_id'):
            return False, "❌ У вас уже есть бизнес! Сначала продайте текущий."
        
        # Проверяем, не пытается ли пользователь купить уникальный бизнес
        if business_id == 8:
            return False, "❌ Этот бизнес недоступен для покупки!"
        
        business = BUSINESSES.get(business_id)
        if not business:
            return False, "❌ Бизнес не найден!"
        
        if user_data['balance'] < business['cost']:
            return False, f"❌ Недостаточно денег! Нужно: {format_number(business['cost'])}₽"
        
        new_balance = user_data['balance'] - business['cost']
        
        updates = {
            'balance': new_balance,
            'business_id': business_id,
            'business': business['name'],
            'business_current_profit': 0,
            'business_current_exp': 0,
            'business_tax_due': 0,
            'business_last_tax_payment': datetime.now().strftime('%Y-%m-%d %H:%M:%S'),
            'business_last_collection': datetime.now().strftime('%Y-%m-%d %H:%M:%S')
        }
        
        self.update_user(user_id, updates)
        
        user_id_str = str(user_id)
        user_data = self.get_user(user_id)
        user_data['daily_stats']['money_earned'] += business['cost']
        user_data['weekly_stats']['money_earned'] += business['cost']
        
        self.update_user(user_id, {
            'daily_stats': user_data['daily_stats'],
            'weekly_stats': user_data['weekly_stats']
        })
        
        return True, f"✅ Вы успешно купили бизнес: {business['name']}!"
    
    def sell_business(self, user_id):
        user_data = self.get_user(user_id)
        
        if not user_data.get('business_id'):
            return False, "❌ У вас нет бизнеса для продажи!"
        
        # Проверяем, не пытается ли пользователь продать уникальный бизнес
        if user_data['business_id'] == 8:
            return False, "❌ Этот бизнес нельзя продать!"
        
        business = BUSINESSES.get(user_data['business_id'])
        if not business:
            return False, "❌ Бизнес не найден!"
        
        sell_price = int(business['cost'] * 0.25)
        new_balance = user_data['balance'] + sell_price
        
        updates = {
            'balance': new_balance,
            'business_id': None,
            'business': 'нету',
            'business_current_profit': 0,
            'business_current_exp': 0,
            'business_tax_due': 0,
            'business_last_tax_payment': None,
            'business_last_collection': None
        }
        
        self.update_user(user_id, updates)
        
        return True, f"✅ Вы продали бизнес {business['name']} за {format_number(sell_price)}₽"
    
    def update_business_progress(self, user_id):
        """Обновление прогресса бизнеса (прибыль, опыт, налоги)"""
        user_data = self.get_user(user_id)
        
        if not user_data.get('business_id'):
            return False, 0, 0, 0, "❌ У вас нет бизнеса!"
        
        # Проверяем, уникальный ли это бизнес
        if user_data['business_id'] == 8:
            business = UNIQUE_BUSINESS.get(8)
        else:
            business = BUSINESSES.get(user_data['business_id'])
            
        if not business:
            return False, 0, 0, 0, "❌ Бизнес не найден!"
        
        last_collection = user_data.get('business_last_collection')
        if not last_collection:
            last_collection = datetime.now().strftime('%Y-%m-%d %H:%M:%S')
            self.update_user(user_id, {'business_last_collection': last_collection})
            return False, 0, 0, 0, "⏳ Прибыль еще не накопилась"
        
        try:
            last_time = datetime.strptime(last_collection, '%Y-%m-%d %H:%M:%S')
            now = datetime.now()
            minutes_passed = (now - last_time).total_seconds() / 60
            
            # Начисляем каждые 10 минут
            if minutes_passed >= 10:
                intervals = int(minutes_passed // 10)
                
                # Прибыль и опыт за 10 минут (1/6 от часовой)
                profit_per_10min = business['profit_per_hour'] / 6
                exp_per_10min = business['exp_per_hour'] / 6
                tax_per_10min = business['tax_per_hour'] / 6
                
                total_profit = int(profit_per_10min * intervals)
                total_exp = int(exp_per_10min * intervals)
                total_tax = int(tax_per_10min * intervals)
                
                current_profit = user_data.get('business_current_profit', 0) + total_profit
                current_exp = user_data.get('business_current_exp', 0) + total_exp
                current_tax = user_data.get('business_tax_due', 0) + total_tax
                
                updates = {
                    'business_current_profit': current_profit,
                    'business_current_exp': current_exp,
                    'business_tax_due': current_tax,
                    'business_last_collection': now.strftime('%Y-%m-%d %H:%M:%S')
                }
                
                self.update_user(user_id, updates)
                
                return True, total_profit, total_exp, total_tax, f"✅ Накоплено за {intervals*10} минут"
            else:
                return False, 0, 0, 0, f"⏳ До следующего начисления: {int(10 - minutes_passed)} минут"
                
        except Exception as e:
            return False, 0, 0, 0, f"❌ Ошибка: {str(e)}"
    
    def take_profit(self, user_id):
        """Забрать накопленные прибыль и опыт"""
        user_data = self.get_user(user_id)
        
        if not user_data.get('business_id'):
            return False, 0, 0, "❌ У вас нет бизнеса!"
        
        # Сначала обновляем прогресс
        self.update_business_progress(user_id)
        user_data = self.get_user(user_id)
        
        current_profit = user_data.get('business_current_profit', 0)
        current_exp = user_data.get('business_current_exp', 0)
        
        if current_profit == 0 and current_exp == 0:
            return False, 0, 0, "❌ На кассе ничего нет!"
        
        new_balance = user_data['balance'] + current_profit
        
        updates = {
            'balance': new_balance,
            'business_current_profit': 0,
            'business_current_exp': 0
        }
        
        self.update_user(user_id, updates)
        
        if current_exp > 0:
            self.add_experience(user_id, current_exp)
        
        user_id_str = str(user_id)
        user_data = self.get_user(user_id)
        user_data['daily_stats']['money_earned'] += current_profit
        user_data['weekly_stats']['money_earned'] += current_profit
        
        self.update_user(user_id, {
            'daily_stats': user_data['daily_stats'],
            'weekly_stats': user_data['weekly_stats']
        })
        
        self.quest_system.update_quest_progress(user_id, 'daily', 'earn_money', current_profit)
        self.quest_system.update_quest_progress(user_id, 'weekly', 'earn_money', current_profit)
        
        return True, current_profit, current_exp, f"✅ Забрано {format_number(current_profit)}₽ и {current_exp} опыта!"
    
    def pay_taxes(self, user_id):
        user_data = self.get_user(user_id)
        
        if not user_data.get('business_id'):
            return False, "❌ У вас нет бизнеса!"
        
        # Сначала обновляем прогресс
        self.update_business_progress(user_id)
        user_data = self.get_user(user_id)
        
        # Проверяем, уникальный ли это бизнес
        if user_data['business_id'] == 8:
            business = UNIQUE_BUSINESS.get(8)
        else:
            business = BUSINESSES.get(user_data['business_id'])
            
        if not business:
            return False, "❌ Бизнес не найден!"
        
        tax_due = user_data.get('business_tax_due', 0)
        
        if tax_due == 0:
            return False, "✅ Налоги уже оплачены!"
        
        if user_data['balance'] < tax_due:
            return False, f"❌ Недостаточно денег для оплаты налогов: {format_number(tax_due)}₽"
        
        new_balance = user_data['balance'] - tax_due
        
        updates = {
            'balance': new_balance,
            'business_tax_due': 0,
            'business_last_tax_payment': datetime.now().strftime('%Y-%m-%d %H:%M:%S')
        }
        
        self.update_user(user_id, updates)
        
        return True, f"✅ Налоги оплачены: {format_number(tax_due)}₽"
    
    def start_business_updater(self):
        """Запуск автоматического обновления бизнесов"""
        def updater():
            while True:
                time.sleep(600)  # 10 минут
                self.update_all_businesses()
        
        thread = threading.Thread(target=updater, daemon=True)
        thread.start()
    
    def update_all_businesses(self):
        """Обновление всех бизнесов"""
        users = self.db.get_all_users()
        for user in users:
            if user.get('business_id'):
                self.update_business_progress(user['user_id'])
    
    def update_leaderboard_cache(self):
        global leaderboard_cache, leaderboard_update_time, levels_leaderboard_cache, levels_leaderboard_update_time
        
        users = self.db.get_all_users()
        
        sorted_users_balance = sorted(
            users,
            key=lambda x: x.get('balance', 0),
            reverse=True
        )
        
        leaderboard_cache = sorted_users_balance[:10]
        leaderboard_update_time = datetime.now()
        
        sorted_users_level = sorted(
            users,
            key=lambda x: (x.get('level', 1), x.get('experience', 0)),
            reverse=True
        )
        
        levels_leaderboard_cache = sorted_users_level[:10]
        levels_leaderboard_update_time = datetime.now()
    
    def get_leaderboard(self):
        global leaderboard_cache, leaderboard_update_time
        
        if leaderboard_cache is None:
            self.update_leaderboard_cache()
        
        return leaderboard_cache, leaderboard_update_time
    
    def get_levels_leaderboard(self):
        global levels_leaderboard_cache, levels_leaderboard_update_time
        
        if levels_leaderboard_cache is None:
            self.update_leaderboard_cache()
        
        return levels_leaderboard_cache, levels_leaderboard_update_time
    
    def start_leaderboard_updater(self):
        def updater():
            while True:
                time.sleep(UPDATE_INTERVAL)
                self.update_leaderboard_cache()
        
        thread = threading.Thread(target=updater, daemon=True)
        thread.start()
    
    def increment_message_counter(self):
        """Увеличить счетчик сообщений"""
        self.db.increment_message_count()
    
    def get_statistics(self):
        """Получить статистику"""
        stats = self.db.get_message_stats()
        return {
            'total_users': stats['total_users'],
            'total_messages': stats['total_messages'],
            'today_messages': stats['today_messages']
        }
    
    def is_admin(self, user_id):
        """Проверка, является ли пользователь админом"""
        user_data = self.get_user(user_id)
        return user_data.get('is_admin', 0) == 1 or user_id in ADMINS
    
    def add_admin(self, user_id):
        """Добавить админа"""
        if user_id not in ADMINS:
            ADMINS.append(user_id)
            save_admins(ADMINS)
            self.update_user(user_id, {'is_admin': 1})
            return True
        return False
    
    def remove_admin(self, user_id):
        """Удалить админа"""
        if user_id != ADMIN_ID and user_id in ADMINS:
            ADMINS.remove(user_id)
            save_admins(ADMINS)
            self.update_user(user_id, {'is_admin': 0})
            return True
        return False

# Инициализация менеджера данных
data_manager = UnifiedDataManager()
quest_system = data_manager.quest_system

# Вспомогательные функции для форматирования
def format_number(number):
    """Форматирует число с разделителями тысяч"""
    return f"{number:,}".replace(",", ".")

def parse_amount_with_k(amount_str, user_balance):
    """Парсит суммы для перевода: 30к, 1.5к, 2.5кк, 3ккк, 4кккк и т.д."""
    amount_str = amount_str.replace(',', '.').lower().strip()
    
    if amount_str in ["всё", "все", "all"]:
        return user_balance
    
    # Поддержка кккк (триллионы)
    if amount_str.endswith('кккк'):
        number_part = amount_str[:-4]
        try:
            number = float(number_part)
            return int(number * 1000000000000)
        except ValueError:
            raise ValueError(f"Некорректное число: {number_part}")
    
    # Поддержка ккк (миллиарды)
    elif amount_str.endswith('ккк'):
        number_part = amount_str[:-3]
        try:
            number = float(number_part)
            return int(number * 1000000000)
        except ValueError:
            raise ValueError(f"Некорректное число: {number_part}")
    
    # Поддержка кк (миллионы)
    elif amount_str.endswith('кк'):
        number_part = amount_str[:-2]
        try:
            number = float(number_part)
            return int(number * 1000000)
        except ValueError:
            raise ValueError(f"Некорректное число: {number_part}")
    
    # Поддержка к (тысячи)
    elif amount_str.endswith('к'):
        number_part = amount_str[:-1]
        try:
            number = float(number_part)
            return int(number * 1000)
        except ValueError:
            raise ValueError(f"Некорректное число: {number_part}")
    else:
        try:
            return int(float(amount_str))
        except ValueError:
            raise ValueError(f"Некорректное число: {amount_str}")

def parse_bet_with_k(bet_str, user_balance):
    """Парсит ставки в формате: 30к, 1.5к, 2.5кк, 3ккк, 4кккк и т.д."""
    bet_str = bet_str.replace(',', '.').lower().strip()
    
    if bet_str in ["всё", "все", "all"]:
        return user_balance
    
    if bet_str.endswith('кккк'):
        number_part = bet_str[:-4]
        try:
            number = float(number_part)
            return int(number * 1000000000000)
        except ValueError:
            raise ValueError(f"Некорректное число: {number_part}")
    
    elif bet_str.endswith('ккк'):
        number_part = bet_str[:-3]
        try:
            number = float(number_part)
            return int(number * 1000000000)
        except ValueError:
            raise ValueError(f"Некорректное число: {number_part}")
    
    elif bet_str.endswith('кк'):
        number_part = bet_str[:-2]
        try:
            number = float(number_part)
            return int(number * 1000000)
        except ValueError:
            raise ValueError(f"Некорректное число: {number_part}")
    
    elif bet_str.endswith('к'):
        number_part = bet_str[:-1]
        try:
            number = float(number_part)
            return int(number * 1000)
        except ValueError:
            raise ValueError(f"Некорректное число: {number_part}")
    else:
        try:
            return int(float(bet_str))
        except ValueError:
            raise ValueError(f"Некорректное число: {bet_str}")

def get_casino_outcomes(balance):
    """Возвращает исходы казино в зависимости от баланса"""
    outcomes_small = [
        {"mult": -1.0, "text": "😭 сумма вашей ставки сгорела <b>(x0)</b>", "prob": 8},
        {"mult": -0.6, "text": "😕 вы проиграли <b>60%</b> ставки <b>(x0.60)</b>", "prob": 18},
        {"mult": -0.30, "text": "😣 вы проиграли <b>30%</b> ставки <b>(x0.30)</b>", "prob": 19},
        {"mult": -0.80, "text": "🙄 вы проиграли <b>80%</b> ставки <b>(x0.80)</b>", "prob": 18},
        {"mult": 0.60, "text": "😜 вы выиграли <b>60%</b> ставки <b>(x0.60)</b>", "prob": 17},
        {"mult": 0.30, "text": "🙂 вы выиграли <b>30%</b> ставки <b>(x0.30)</b>", "prob": 17},
        {"mult": 0, "text": "😶 сумма вашей ставки сохранена <b>(x0)</b>", "prob": 17},
        {"mult": 0.80, "text": "😍 вы выиграли <b>80%</b> ставки <b>(x0.80)</b>", "prob": 17},
        {"mult": 1.0, "text": "😊 вы выиграли <b>100%</b> ставки <b>(x1)</b>", "prob": 13},
        {"mult": 2.0, "text": "💰 вы выиграли <b>200%</b> ставки <b>(x2)</b>", "prob": 7},
        {"mult": 1.5, "text": "🤑 вы выиграли <b>150%</b> ставки <b>(x1.50)</b>", "prob": 8},
        {"mult": 5.0, "text": "🔥 вы выиграли <b>ДЖЕКПОТ x5</b>", "prob": 2},
        {"mult": -0.20, "text": "🤥 вы проиграли <b>20%</b> ставки <b>(x0.20)</b>", "prob": 16},
        {"mult": -0.10, "text": "😫 вы проиграли <b>10%</b> ставки <b>(x0.10)</b>", "prob": 17},
    ]
    
    outcomes_medium = [
        {"mult": -1.0, "text": "😭 сумма вашей ставки сгорела <b>(x0)</b>", "prob": 9},
        {"mult": -0.6, "text": "😕 вы проиграли <b>60%</b> ставки <b>(x0.60)</b>", "prob": 18},
        {"mult": -0.30, "text": "😣 вы проиграли <b>30%</b> ставки <b>(x0.30)</b>", "prob": 19},
        {"mult": -0.80, "text": "🙄 вы проиграли <b>80%</b> ставки <b>(x0.80)</b>", "prob": 18},
        {"mult": 0.60, "text": "😜 вы выиграли <b>60%</b> ставки <b>(x0.60)</b>", "prob": 16},
        {"mult": 0.30, "text": "🙂 вы выиграли <b>30%</b> ставки <b>(x0.30)</b>", "prob": 16},
        {"mult": 0, "text": "😶 сумма вашей ставки сохранена <b>(x0)</b>", "prob": 17},
        {"mult": 0.80, "text": "😍 вы выиграли <b>80%</b> ставки <b>(x0.80)</b>", "prob": 13},
        {"mult": 1.0, "text": "😊 вы выиграли <b>100%</b> ставки <b>(x1)</b>", "prob": 11},
        {"mult": 2.0, "text": "💰 вы выиграли <b>200%</b> ставки <b>(x2)</b>", "prob": 5},
        {"mult": 1.5, "text": "🤑 вы выиграли <b>150%</b> ставки <b>(x1.50)</b>", "prob": 6},
        {"mult": 5.0, "text": "🔥 вы выиграли <b>ДЖЕКПОТ x5</b>", "prob": 2},
        {"mult": -0.20, "text": "🤥 вы проиграли <b>20%</b> ставки <b>(x0.20)</b>", "prob": 20},
        {"mult": -0.10, "text": "😫 вы проиграли <b>10%</b> ставки <b>(x0.10)</b>", "prob": 20},
    ]
    
    outcomes_big = [
        {"mult": -1.0, "text": "😭 сумма вашей ставки сгорела <b>(x0)</b>", "prob": 9},
        {"mult": -0.6, "text": "😕 вы проиграли <b>60%</b> ставки <b>(x0.60)</b>", "prob": 22},
        {"mult": -0.30, "text": "😣 вы проиграли <b>30%</b> ставки <b>(x0.30)</b>", "prob": 21},
        {"mult": -0.80, "text": "🙄 вы проиграли <b>80%</b> ставки <b>(x0.80)</b>", "prob": 21},
        {"mult": 0.60, "text": "😜 вы выиграли <b>60%</b> ставки <b>(x0.60)</b>", "prob": 14},
        {"mult": 0.30, "text": "🙂 вы выиграли <b>30%</b> ставки <b>(x0.30)</b>", "prob": 14},
        {"mult": 0, "text": "😶 сумма вашей ставки сохранена <b>(x0)</b>", "prob": 17},
        {"mult": 0.80, "text": "😍 вы выиграли <b>80%</b> ставки <b>(x0.80)</b>", "prob": 12},
        {"mult": 1.0, "text": "😊 вы выиграли <b>100%</b> ставки <b>(x1)</b>", "prob": 10},
        {"mult": 2.0, "text": "💰 вы выиграли <b>200%</b> ставки <b>(x2)</b>", "prob": 6},
        {"mult": 1.5, "text": "🤑 вы выиграли <b>150%</b> ставки <b>(x1.50)</b>", "prob": 7},
        {"mult": 5.0, "text": "🔥 вы выиграли <b>ДЖЕКПОТ x5</b>", "prob": 1},
        {"mult": -0.20, "text": "🤥 вы проиграли <b>20%</b> ставки <b>(x0.20)</b>", "prob": 22},
        {"mult": -0.10, "text": "😫 вы проиграли <b>10%</b> ставки <b>(x0.10)</b>", "prob": 22},
    ]
    
    outcomes_huge = [
        {"mult": -1.0, "text": "😭 сумма вашей ставки сгорела <b>(x0)</b>", "prob": 11},
        {"mult": -0.6, "text": "😕 вы проиграли <b>60%</b> ставки <b>(x0.60)</b>", "prob": 22},
        {"mult": -0.30, "text": "😣 вы проиграли <b>30%</b> ставки <b>(x0.30)</b>", "prob": 22},
        {"mult": -0.80, "text": "🙄 вы проиграли <b>80%</b> ставки <b>(x0.80)</b>", "prob": 22},
        {"mult": 0.60, "text": "😜 вы выиграли <b>60%</b> ставки <b>(x0.60)</b>", "prob": 11},
        {"mult": 0.30, "text": "🙂 вы выиграли <b>30%</b> ставки <b>(x0.30)</b>", "prob": 11},
        {"mult": 0, "text": "😶 сумма вашей ставки сохранена <b>(x0)</b>", "prob": 17},
        {"mult": 0.80, "text": "😍 вы выиграли <b>80%</b> ставки <b>(x0.80)</b>", "prob": 6},
        {"mult": 1.0, "text": "😊 вы выиграли <b>100%</b> ставки <b>(x1)</b>", "prob": 6},
        {"mult": 2.0, "text": "💰 вы выиграли <b>200%</b> ставки <b>(x2)</b>", "prob": 4},
        {"mult": 1.5, "text": "🤑 вы выиграли <b>150%</b> ставки <b>(x1.50)</b>", "prob": 5},
        {"mult": 5.0, "text": "🔥 вы выиграли <b>ДЖЕКПОТ x5</b>", "prob": 0.5},
        {"mult": -0.20, "text": "🤥 вы проиграли <b>20%</b> ставки <b>(x0.20)</b>", "prob": 22},
        {"mult": -0.10, "text": "😫 вы проиграли <b>10%</b> ставки <b>(x0.10)</b>", "prob": 22},
    ]
    
    if balance >= 1000000:
        return outcomes_huge
    elif balance >= 500000:
        return outcomes_big
    elif balance >= 100000:
        return outcomes_medium
    else:
        return outcomes_small

# Создание клавиатур
def create_main_keyboard():
    markup = types.ReplyKeyboardMarkup(resize_keyboard=True, row_width=2)
    buttons = [
        types.KeyboardButton('👤 Профиль'),
        types.KeyboardButton('🎁 Ежедневный бонус'),
        types.KeyboardButton('📊 Уровни'),
        types.KeyboardButton('💼 Работы'),
        types.KeyboardButton('🛠️ Моя работа'),
        types.KeyboardButton('🏆 Лидеры'),
        types.KeyboardButton('👥 Рефералы'),
        types.KeyboardButton('📋 Задание'),
        types.KeyboardButton('🎰 Казино'),
        types.KeyboardButton('💸 Перевести'),
        types.KeyboardButton('🏪 Магазин бизнеса'),
        types.KeyboardButton('🏢 Мой бизнес')
    ]
    markup.add(*buttons)
    return markup

def create_main_inline_keyboard():
    markup = types.InlineKeyboardMarkup(row_width=2)
    markup.add(
        types.InlineKeyboardButton('👤 Профиль', callback_data='profile'),
        types.InlineKeyboardButton('🎁 Бонус', callback_data='bonus_chat'),
        types.InlineKeyboardButton('📊 Уровни', callback_data='levels_1'),
        types.InlineKeyboardButton('💼 Работы', callback_data='jobs_list'),
        types.InlineKeyboardButton('🛠️ Работать', callback_data='start_work'),
        types.InlineKeyboardButton('🏆 Топы', callback_data='leaders'),
        types.InlineKeyboardButton('📋 Квесты', callback_data='quests_menu'),
        types.InlineKeyboardButton('🎰 Казино', callback_data='casino_info'),
        types.InlineKeyboardButton('💸 Перевести', callback_data='transfer_info'),
        types.InlineKeyboardButton('🏪 Магазин', callback_data='business_shop_1'),
        types.InlineKeyboardButton('🏢 Бизнес', callback_data='my_business'),
        types.InlineKeyboardButton('◀️ Назад', callback_data='back_to_menu')
    )
    return markup

def create_chat_profile_keyboard():
    """Клавиатура для профиля в чате без кнопок реф и профиль"""
    markup = types.InlineKeyboardMarkup(row_width=2)
    markup.add(
        types.InlineKeyboardButton('🎁 Бонус', callback_data='bonus_chat'),
        types.InlineKeyboardButton('📊 Уровни', callback_data='levels_1'),
        types.InlineKeyboardButton('💼 Работы', callback_data='jobs_list'),
        types.InlineKeyboardButton('🛠️ Работать', callback_data='start_work'),
        types.InlineKeyboardButton('🏆 Топы', callback_data='leaders'),
        types.InlineKeyboardButton('📋 Квесты', callback_data='quests_menu'),
        types.InlineKeyboardButton('🎰 Казино', callback_data='casino_info'),
        types.InlineKeyboardButton('💸 Перевести', callback_data='transfer_info'),
        types.InlineKeyboardButton('🏪 Магазин', callback_data='business_shop_1'),
        types.InlineKeyboardButton('🏢 Бизнес', callback_data='my_business'),
        types.InlineKeyboardButton('◀️ Назад', callback_data='back_to_menu')
    )
    return markup

def create_casino_keyboard():
    markup = types.InlineKeyboardMarkup(row_width=2)
    markup.add(
        types.InlineKeyboardButton('🎰 Играть', callback_data='casino_play'),
        types.InlineKeyboardButton('📊 Правила', callback_data='casino_rules'),
        types.InlineKeyboardButton('◀️ Назад', callback_data='back_to_menu')
    )
    return markup

def create_transfer_keyboard():
    markup = types.InlineKeyboardMarkup(row_width=2)
    markup.add(
        types.InlineKeyboardButton('📋 Правила', callback_data='transfer_rules'),
        types.InlineKeyboardButton('◀️ Назад', callback_data='back_to_menu')
    )
    return markup

def create_quests_menu_keyboard():
    markup = types.InlineKeyboardMarkup(row_width=2)
    markup.add(
        types.InlineKeyboardButton('📅 Ежедневные', callback_data='quests_daily'),
        types.InlineKeyboardButton('📆 Недельные', callback_data='quests_weekly'),
        types.InlineKeyboardButton('◀️ Назад', callback_data='back_to_menu')
    )
    return markup

def create_quests_list_keyboard(quests, quest_type):
    markup = types.InlineKeyboardMarkup(row_width=1)
    
    for quest in quests:
        if quest['state'] == 'available':
            emoji = '🔓'
        elif quest['state'] == 'active':
            emoji = '🟢'
        else:
            emoji = '✅'
        
        button_text = f"{emoji} {quest['title']}"
        markup.add(types.InlineKeyboardButton(button_text, callback_data=f'quest_detail_{quest_type}_{quest["id"]}'))
    
    markup.add(types.InlineKeyboardButton('◀️ Назад', callback_data='quests_menu'))
    return markup

def create_quest_detail_keyboard(quest, quest_type):
    markup = types.InlineKeyboardMarkup(row_width=2)
    
    if quest['state'] == 'available':
        markup.add(types.InlineKeyboardButton('🎯 Начать квест', callback_data=f'quest_start_{quest_type}_{quest["id"]}'))
    elif quest['state'] == 'active':
        if quest['progress'] >= quest['target']:
            markup.add(types.InlineKeyboardButton('✅ Выполнить', callback_data=f'quest_complete_{quest_type}_{quest["id"]}'))
        markup.add(types.InlineKeyboardButton('❌ Прекратить', callback_data=f'quest_cancel_{quest_type}_{quest["id"]}'))
    elif quest['state'] == 'completed':
        markup.add(types.InlineKeyboardButton('✅ Завершено', callback_data='none'))
    
    markup.add(types.InlineKeyboardButton('◀️ Назад', callback_data=f'quests_{quest_type}'))
    return markup

def create_referrals_keyboard():
    markup = types.InlineKeyboardMarkup(row_width=2)
    markup.add(
        types.InlineKeyboardButton('🔄 Обновить', callback_data='refresh_ref'),
        types.InlineKeyboardButton('📋 Мои рефералы', callback_data='my_referrals'),
        types.InlineKeyboardButton('◀️ Назад', callback_data='back_to_menu')
    )
    return markup

def create_my_referrals_keyboard():
    markup = types.InlineKeyboardMarkup(row_width=2)
    markup.add(
        types.InlineKeyboardButton('◀️ Назад', callback_data='back_to_ref'),
        types.InlineKeyboardButton('🔄 Обновить', callback_data='refresh_my_ref')
    )
    return markup

def create_jobs_keyboard():
    markup = types.InlineKeyboardMarkup(row_width=1)
    for job in JOBS.values():
        markup.add(types.InlineKeyboardButton(
            job['name'],
            callback_data=f'job_info_{job["id"]}'
        ))
    markup.add(types.InlineKeyboardButton('◀️ Назад', callback_data='back_to_menu'))
    return markup

def create_work_keyboard():
    markup = types.InlineKeyboardMarkup(row_width=2)
    markup.add(
        types.InlineKeyboardButton('🛠️ Работать', callback_data='start_work'),
        types.InlineKeyboardButton('❌ Уволиться', callback_data='fire_confirm'),
        types.InlineKeyboardButton('💼 Все работы', callback_data='jobs_list'),
        types.InlineKeyboardButton('◀️ Назад', callback_data='back_to_menu')
    )
    return markup

def create_levels_keyboard(page=1):
    markup = types.InlineKeyboardMarkup(row_width=3)
    
    nav_buttons = []
    if page > 1:
        nav_buttons.append(types.InlineKeyboardButton('◀️', callback_data=f'levels_{page-1}'))
    
    nav_buttons.append(types.InlineKeyboardButton(f'{page}/4', callback_data='current_page'))
    
    if page < 4:
        nav_buttons.append(types.InlineKeyboardButton('▶️', callback_data=f'levels_{page+1}'))
    
    markup.add(*nav_buttons)
    markup.add(types.InlineKeyboardButton('🎁 Получить награды', callback_data='claim_rewards'))
    markup.add(types.InlineKeyboardButton('◀️ Назад', callback_data='back_to_menu'))
    
    return markup

def create_question_keyboard(answers):
    markup = types.InlineKeyboardMarkup(row_width=1)
    
    for i, answer in enumerate(answers):
        markup.add(types.InlineKeyboardButton(
            f"{i+1}. {answer}",
            callback_data=f'answer_{i}'
        ))
    
    return markup

def create_leaders_keyboard():
    markup = types.InlineKeyboardMarkup(row_width=2)
    markup.add(
        types.InlineKeyboardButton('🔄 Обновить', callback_data='refresh_leaders'),
        types.InlineKeyboardButton('📈 Топ по уровням', callback_data='levels_leaders'),
        types.InlineKeyboardButton('◀️ Назад', callback_data='back_to_menu')
    )
    return markup

def create_levels_leaders_keyboard():
    markup = types.InlineKeyboardMarkup(row_width=1)
    markup.add(
        types.InlineKeyboardButton('◀️ Назад к лидерам', callback_data='leaders')
    )
    return markup

# КЛАВИАТУРЫ ДЛЯ БИЗНЕСА
def create_business_shop_keyboard(business_id, has_business=False):
    markup = types.InlineKeyboardMarkup(row_width=2)
    
    nav_buttons = []
    if business_id > 1:
        nav_buttons.append(types.InlineKeyboardButton('⬅️ Назад', callback_data=f'business_shop_{business_id-1}'))
    
    nav_buttons.append(types.InlineKeyboardButton(f'{business_id}/7', callback_data='current_page'))
    
    if business_id < 7:
        nav_buttons.append(types.InlineKeyboardButton('➡️ Вперед', callback_data=f'business_shop_{business_id+1}'))
    
    markup.add(*nav_buttons)
    
    if not has_business:
        markup.add(types.InlineKeyboardButton('💰 Купить бизнес', callback_data=f'buy_business_{business_id}'))
    
    markup.add(types.InlineKeyboardButton('◀️ Назад', callback_data='back_to_menu'))
    
    return markup

def create_my_business_keyboard():
    markup = types.InlineKeyboardMarkup(row_width=2)
    markup.add(
        types.InlineKeyboardButton('💰 Забрать прибыль', callback_data='take_profit'),
        types.InlineKeyboardButton('💸 Оплатить налоги', callback_data='pay_taxes'),
        types.InlineKeyboardButton('💡 Продать бизнес', callback_data='sell_business_confirm')
    )
    markup.add(types.InlineKeyboardButton('◀️ Назад', callback_data='back_to_menu'))
    return markup

def create_sell_confirm_keyboard():
    markup = types.InlineKeyboardMarkup(row_width=2)
    markup.add(
        types.InlineKeyboardButton('✅ Да, продать', callback_data='sell_business_yes'),
        types.InlineKeyboardButton('❌ Нет, отмена', callback_data='sell_business_no')
    )
    markup.add(types.InlineKeyboardButton('🏢 Мой бизнес', callback_data='my_business'))
    return markup

# КЛАВИАТУРЫ ДЛЯ АДМИН-ПАНЕЛИ
def create_admin_keyboard():
    markup = types.InlineKeyboardMarkup(row_width=2)
    markup.add(
        types.InlineKeyboardButton('📊 Статистика', callback_data='admin_stats'),
        types.InlineKeyboardButton('💰 Выдать деньги (по юзеру)', callback_data='admin_give_money_by_username'),
        types.InlineKeyboardButton('🌟 Выдать опыт', callback_data='admin_give_exp'),
        types.InlineKeyboardButton('🆔 Выдать ID', callback_data='admin_give_id'),
        types.InlineKeyboardButton('👥 Просмотр профилей', callback_data='admin_profiles_1')
    )
    markup.add(types.InlineKeyboardButton('◀️ Назад', callback_data='back_to_menu'))
    return markup

def create_admin_back_keyboard():
    markup = types.InlineKeyboardMarkup()
    markup.add(types.InlineKeyboardButton('🔙 Назад в админ-панель', callback_data='admin_back'))
    markup.add(types.InlineKeyboardButton('◀️ Назад', callback_data='back_to_menu'))
    return markup

# Форматирование сообщений
def format_profile(user_data):
    username = user_data.get('username', 'Игрок')
    level = user_data.get('level', 1)
    balance = user_data.get('balance', 0)
    exp = user_data.get('experience', 0)
    exp_needed = 1000 + (level - 1) * 500
    
    formatted_balance = f"{balance:,}₽"
    
    business_info = "нету"
    if user_data.get('business_id'):
        if user_data['business_id'] == 8:
            business_info = UNIQUE_BUSINESS[8]['name']
        else:
            business = BUSINESSES.get(user_data['business_id'])
            if business:
                business_info = business['name']
    
    bonus_available, time_left = data_manager.check_bonus(user_data)
    has_unclaimed, _ = data_manager.has_unclaimed_levels(user_data)
    
    profile = f"""
<b>{username} ваш профиль💰:</b>
━━━━━━━━━━━━━━━━━━

<b>🌟 Уровень:</b> {level}
<b>💰 Баланс:</b> {formatted_balance}
<b>📈 Опыт:</b> {exp}/{exp_needed}
<b>👥 Клан:</b> Нет

<b>💼 Профессия:</b> {user_data.get('profession', 'нет')}
<b>🏠 Дом:</b> {user_data.get('house', 'нету')}
<b>🏢 Бизнес:</b> {business_info}
<b>🚗 Авто:</b> {user_data.get('auto', 'нету')}

<b>🆔 Игровой ID:</b> <code>{user_data.get('game_id', '00000')}</code>
<b>📅 Дата регистрации:</b> {user_data.get('registration_date', 'Неизвестно')}
"""
    
    leaders, _ = data_manager.get_leaderboard()
    user_balance = balance
    place = None
    
    for i, leader in enumerate(leaders, 1):
        if leader.get('username') == username and leader.get('balance') == user_balance:
            place = i
            break
    
    if place:
        place_emojis = {1: "🥇", 2: "🥈", 3: "🥉"}
        emoji = place_emojis.get(place, f"{place}.")
        profile += f"\n<b>🏆 Место в топе:</b> {emoji}"
    
    if has_unclaimed:
        profile += "\n\n<b>🎁 У вас есть неполученные награды за уровни!</b>"
    
    if bonus_available:
        profile += "\n\n<b>✅ Ежедневный бонус доступен!</b>"
    else:
        profile += f"\n\n<b>⏳ Бонус через:</b> {time_left}"
    
    return profile

def format_referrals_info(user_data):
    username = user_data.get('username', 'Игрок')
    referral_code = user_data.get('referral_code', 'Нет кода')
    referrals_count = user_data.get('referral_count', 0)
    referral_earnings = user_data.get('referral_earnings', 0)
    
    bot_username = bot.get_me().username
    referral_link = f"https://t.me/{bot_username}?start=ref_{referral_code}"
    
    info = f"""
<b>👥 РЕФЕРАЛЬНАЯ СИСТЕМА</b>
━━━━━━━━━━━━━━━━━━

<b>👤 Ваше имя:</b> {username}
<b>🎫 Ваш реферальный код:</b> <code>{referral_code}</code>
<b>👥 Приглашено людей:</b> {referrals_count}
<b>💰 Заработано на рефералах:</b> {referral_earnings:,}₽

<b>🎁 БОНУСЫ ЗА ПРИГЛАШЕНИЕ:</b>
┌────────────────────
├ <b>Для вас:</b> {REFERRAL_BONUS_INVITER:,}₽ + {REFERRAL_EXPERIENCE} опыта
├ <b>Для друга:</b> {REFERRAL_BONUS_REFEREE:,}₽ + {REFERRAL_EXPERIENCE} опыта
└────────────────────

<b>🔗 РЕФЕРАЛЬНАЯ ССЫЛКА:</b>
<code>{referral_link}</code>
"""
    
    if referrals_count > 0:
        info += f"\n<b>🎯 ВАША СТАТИСТИКА:</b>\n"
        info += f"• Всего приглашено: {referrals_count} чел.\n"
        info += f"• Заработано: {referral_earnings:,}₽\n"
        
        if user_data.get('last_referral'):
            info += f"• Последний реферал: {user_data['last_referral']}"
    else:
        info += "\n<b>📭 У вас пока нет рефералов</b>\n"
        info += "<b>💎 Приглашайте друзей и получайте бонусы!</b>"
    
    return info

def format_job_info(job_id, user_level, user_data=None):
    job = JOBS.get(job_id)
    if not job:
        return "Работа не найдена"
    
    cooldown_min = job['cooldown'] // 60
    cooldown_sec = job['cooldown'] % 60
    
    status = ""
    if user_level >= job['min_level']:
        status = "✅ Доступна"
        if user_data and user_data.get('current_job') == job_id:
            status = "✅ Вы работаете здесь"
    else:
        status = f"❌ Нужен {job['min_level']} уровень"
    
    info = f"""
<b>{job['name']}</b>
━━━━━━━━━━━━━━━━━━

<b>📝 Описание:</b> {job['description']}

<b>📊 Требования:</b>
• Уровень: {job['min_level']}+
• Зарплата: {job['salary']:,}₽
• Опыт за работу: {job['experience']}
• Время работы: {cooldown_min:02d}:{cooldown_sec:02d}

<b>📈 Статус:</b> {status}
"""
    
    if user_data and user_data.get('current_job') == job_id and user_data.get('hired_date'):
        info += f"\n<b>📅 Устроен:</b> {user_data['hired_date']}"
    
    return info

def format_levels_page(user_data, page=1):
    username = user_data.get('username', 'Игрок')
    current_level = user_data.get('level', 1)
    claimed_levels = user_data.get('claimed_levels', [])
    
    start_level = (page - 1) * 5 + 1
    end_level = min(start_level + 4, 20)
    
    levels_text = f"""
<b>📊 СИСТЕМА УРОВНЕЙ</b>
━━━━━━━━━━━━━━━━━━

<b>👤 Игрок:</b> {username}
<b>🌟 Текущий уровень:</b> {current_level}
<b>📈 Опыт:</b> {user_data.get('experience', 0)}/{1000 + (current_level - 1) * 500}

<b>Страница {page}/4</b>
━━━━━━━━━━━━━━━━━━
"""
    
    for level in range(start_level, end_level + 1):
        reward = LEVEL_REWARDS.get(level, 0)
        
        if level in claimed_levels:
            status = "✅ Получено"
        elif level <= current_level:
            status = "🎁 Можно получить!"
        else:
            status = "⏳ Еще не доступно"
        
        levels_text += f"""
<b>Уровень {level}:</b>
• Награда: {reward:,}₽
• Статус: {status}
━━━━━━━━━━━━━━━━━━
"""
    
    has_unclaimed, unclaimed_level = data_manager.has_unclaimed_levels(user_data)
    if has_unclaimed:
        levels_text += f"\n\n<b>🎯 У вас есть неполученные награды за уровни!</b>"
    
    return levels_text

# ФУНКЦИИ ФОРМАТИРОВАНИЯ ДЛЯ БИЗНЕСА
def format_business_info(business_id, user_data):
    # Проверяем, не пытаемся ли отобразить уникальный бизнес (его не должно быть в магазине)
    if business_id == 8:
        return "❌ Этот бизнес недоступен в магазине"
    
    business = BUSINESSES.get(business_id)
    if not business:
        return "Бизнес не найден"
    
    has_business = user_data.get('business_id') is not None
    owns_this = user_data.get('business_id') == business_id
    
    tax_info = ""
    if owns_this:
        data_manager.update_business_progress(user_data['user_id'])
        user_data = data_manager.get_user(user_data['user_id'])
        tax_due = user_data.get('business_tax_due', 0)
        if tax_due > 0:
            tax_info = f"\n<b>📊 Налог:</b> {format_number(tax_due)}₽"
    
    status = "✅ Работает" if owns_this else "🛒 Доступен" if not has_business else "❌ Занят другой бизнес"
    
    info = f"""
{business['image']} <b>{business['name']}</b>
━━━━━━━━━━━━━━━━━━

<b>👑 Стоимость бизнеса:</b> {format_number(business['cost'])}₽
<b>💸 Прибыль в час:</b> {format_number(business['profit_per_hour'])}₽
<b>⭐ Опыт в час:</b> {business['exp_per_hour']}
<b>🪙 Налог в час:</b> {format_number(business['tax_per_hour'])}₽
<b>✅ Статус:</b> {status}{tax_info}
"""
    
    if owns_this:
        info += "\n<blockquote>⚠️ У вас уже есть этот бизнес!</blockquote>"
    elif has_business:
        info += "\n<blockquote>⚠️ Сначала продайте текущий бизнес!</blockquote>"
    else:
        info += f"\n<blockquote>💡 Для покупки нужен баланс: {format_number(business['cost'])}₽</blockquote>"
    
    return info

def format_my_business_info(user_data):
    if not user_data.get('business_id'):
        return "❌ У вас нету бизнеса"
    
    # Обновляем прогресс бизнеса
    data_manager.update_business_progress(user_data['user_id'])
    user_data = data_manager.get_user(user_data['user_id'])
    
    # Проверяем, уникальный ли это бизнес
    if user_data['business_id'] == 8:
        business = UNIQUE_BUSINESS.get(8)
    else:
        business = BUSINESSES.get(user_data['business_id'])
    
    if not business:
        return "❌ Бизнес не найден"
    
    current_profit = user_data.get('business_current_profit', 0)
    current_exp = user_data.get('business_current_exp', 0)
    tax_due = user_data.get('business_tax_due', 0)
    last_tax_payment = user_data.get('business_last_tax_payment')
    
    is_working = True
    tax_warning = ""
    
    if last_tax_payment:
        try:
            last_payment = datetime.strptime(last_tax_payment, '%Y-%m-%d %H:%M:%S')
            hours_since_payment = (datetime.now() - last_payment).total_seconds() / 3600
            
            if hours_since_payment >= 24:
                is_working = False
                tax_warning = f"\n⚠️ <b>Бизнес не работает!</b> Налоги не оплачены более 24 часов."
        except:
            pass
    
    info = f"""
⚡ <b>{user_data['username']}, информация о вашем бизнесе</b>
━━━━━━━━━━━━━━━━━━

<b>👩‍💻 {business['name']}</b>
<b>💸 Прибыль в час:</b> {format_number(business['profit_per_hour'])}₽
<b>⭐ Опыт в час:</b> {business['exp_per_hour']}
<b>🎮 Налог в час:</b> {format_number(business['tax_per_hour'])}₽
<b>💰 Касса:</b> {format_number(current_profit)}₽
<b>⭐ Опыт:</b> {current_exp}
<b>📊 Налог:</b> {format_number(tax_due)}₽
"""
    
    if tax_warning:
        info += tax_warning
    
    if is_working:
        info += "\n\n✅ <b>Бизнес работает нормально</b>"
    else:
        info += "\n\n❌ <b>Бизнес приостановлен из-за неуплаты налогов</b>"
    
    return info

# Обработчики команд
@bot.message_handler(commands=['start'])
def start_command(message):
    data_manager.increment_message_counter()
    user_id = message.from_user.id
    username = message.from_user.username or message.from_user.first_name
    
    referral_code = None
    if len(message.text.split()) > 1:
        args = message.text.split()[1]
        if args.startswith('ref_'):
            referral_code = args[4:]
        else:
            referral_code = args
    
    user_data = data_manager.get_user(user_id, username)
    
    success = False
    if referral_code and not user_data.get('referral_bonus_received'):
        success, result, inviter_id = data_manager.process_referral(user_id, referral_code)
        
        if success:
            user_data = data_manager.get_user(user_id, username)
            
            try:
                bot.send_message(
                    inviter_id,
                    f"""
🎉 <b>НОВЫЙ РЕФЕРАЛ!</b>
━━━━━━━━━━━━━━━━━━

<b>👤 Имя:</b> {username}
<b>🎁 Бонус получен:</b> {REFERRAL_BONUS_INVITER:,}₽
<b>🌟 Опыт:</b> +{REFERRAL_EXPERIENCE}
<b>💰 Ваш баланс:</b> {result['inviter_new_balance']:,}₽
""",
                    parse_mode='HTML'
                )
            except:
                pass
    
    welcome = f"""
<b>👋 Добро пожаловать, {message.from_user.first_name}!</b>
━━━━━━━━━━━━━━━━━━

<b>🎮 Игровой бот с единой базой данных!</b>

<b>📋 Доступные функции:</b>
• 👤 <b>Профиль</b> - ваша статистика
• 🎁 <b>Ежедневный бонус</b> - награда раз в 24 часа
• 📊 <b>Уровни</b> - система с наградами
• 💼 <b>Работы</b> - устройтесь на работу
• 🛠️ <b>Моя работа</b> - работайте и зарабатывайте
• 🏆 <b>Лидеры</b> - топ-10 игроков
• 👥 <b>Рефералы</b> - приглашайте друзей
• 📋 <b>Задание</b> - система квестов
• 🎰 <b>Казино</b> - азартные игры на деньги
• 💸 <b>Перевести</b> - передать деньги другим игрокам
• 🏪 <b>Магазин бизнеса</b> - покупка бизнесов
• 🏢 <b>Мой бизнес</b> - управление бизнесом

<b>💬 В чатах используйте:</b>
• <b>Б</b> - Профиль
• <b>Бонус</b> - Ежедневный бонус
• <b>Уровень</b> - Уровни
• <b>Работа</b> - Список работ
• <b>Моя работа</b> - Текущая работа
• <b>Топы</b> - Лидеры
• <b>Реф</b> - Рефералы
• <b>Квесты</b> - Задания
• <b>Каз</b> - Игровое казино
• <b>Перевести</b> - Передать деньги
• <b>Магазин бизнесов</b> - Магазин бизнесов
• <b>Мой бизнес</b> - Управление бизнесом
"""
    
    if referral_code and success:
        welcome += f"""

🎉 <b>ВЫ ПОЛУЧИЛИ РЕФЕРАЛЬНЫЙ БОНУС!</b>
━━━━━━━━━━━━━━━━━━

💰 <b>+{REFERRAL_BONUS_REFEREE:,}₽</b>
🌟 <b>+{REFERRAL_EXPERIENCE} опыта</b>
🏦 <b>Ваш баланс:</b> {user_data['balance']:,}₽
"""
    
    if message.chat.type == 'private':
        bot.send_message(
            message.chat.id,
            welcome,
            reply_markup=create_main_keyboard()
        )
    else:
        bot.send_message(
            message.chat.id,
            welcome,
            reply_markup=create_main_inline_keyboard()
        )

# Обработчик для слова "админ"
@bot.message_handler(func=lambda msg: msg.text and msg.text.lower() == 'админ' and data_manager.is_admin(msg.from_user.id))
def admin_text(message):
    data_manager.increment_message_counter()
    
    bot.send_message(
        message.chat.id,
        "🔧 <b>АДМИН-ПАНЕЛЬ</b>\n\nВыберите действие:",
        reply_markup=create_admin_keyboard()
    )

# Обработчики ключевых слов в чатах
@bot.message_handler(func=lambda msg: msg.text and msg.text.lower() in ['б', 'профиль', 'баланс', 'статистика'])
def profile_keyword(message):
    data_manager.increment_message_counter()
    user_id = message.from_user.id
    username = message.from_user.username or message.from_user.first_name
    
    user_data = data_manager.get_user(user_id, username)
    profile_text = format_profile(user_data)
    
    if message.chat.type == 'private':
        bot.send_message(
            message.chat.id,
            profile_text,
            reply_markup=create_main_keyboard()
        )
    else:
        bot.send_message(
            message.chat.id,
            profile_text,
            reply_markup=create_chat_profile_keyboard(),
            reply_to_message_id=message.message_id
        )

@bot.message_handler(func=lambda msg: msg.text and msg.text.lower() in ['бонус', 'ежедневный бонус', 'бонусы', 'ежедневный'])
def bonus_keyword(message):
    data_manager.increment_message_counter()
    user_id = message.from_user.id
    username = message.from_user.username or message.from_user.first_name
    
    user_data = data_manager.get_user(user_id, username)
    bonus_available, time_left = data_manager.check_bonus(user_data)
    
    if not bonus_available:
        response = f"⏳ {username}, бонус еще не доступен!\n🕐 Доступ через: {time_left}"
        bot.send_message(
            message.chat.id,
            response,
            reply_to_message_id=message.message_id
        )
        return
    
    bonus_amount = random.randint(1000, 23000)
    exp_amount = random.randint(40, 400)
    
    new_balance = user_data['balance'] + bonus_amount
    
    levels_gained, new_level = data_manager.add_experience(user_id, exp_amount)
    
    data_manager.update_user(user_id, {
        'balance': new_balance,
        'last_bonus': datetime.now().strftime('%Y-%m-%d %H:%M:%S')
    })
    
    user_id_str = str(user_id)
    user_data = data_manager.get_user(user_id)
    user_data['daily_stats']['bonus_count'] += 1
    user_data['weekly_stats']['bonus_count'] += 1
    user_data['daily_stats']['money_earned'] += bonus_amount
    user_data['weekly_stats']['money_earned'] += bonus_amount
    
    data_manager.update_user(user_id, {
        'daily_stats': user_data['daily_stats'],
        'weekly_stats': user_data['weekly_stats']
    })
    
    quest_system.update_quest_progress(user_id, 'daily', 'get_bonus')
    quest_system.update_quest_progress(user_id, 'weekly', 'get_bonus')
    quest_system.update_quest_progress(user_id, 'daily', 'earn_money', bonus_amount)
    quest_system.update_quest_progress(user_id, 'weekly', 'earn_money', bonus_amount)
    
    bonus_text = f"""
<b>👻 {username}, вы получили ежедневный бонус 🎁</b>

<b>🎉 Вы получили {bonus_amount:,}₽ 🎉</b>

<b>🌟 Опыт:</b> +{exp_amount}
<b>💰 Новый баланс:</b> {new_balance:,}₽
"""
    
    if levels_gained > 0:
        bonus_text += f"\n<b>🎉 Вы достигли {new_level} уровня!</b>"
    
    bot.send_message(
        message.chat.id,
        bonus_text,
        reply_to_message_id=message.message_id,
        reply_markup=create_main_inline_keyboard()
    )

@bot.message_handler(func=lambda msg: msg.text and msg.text.lower() in ['уровень', 'уровни', 'левел', 'левелы'])
def levels_keyword(message):
    data_manager.increment_message_counter()
    user_id = message.from_user.id
    username = message.from_user.username or message.from_user.first_name
    
    user_data = data_manager.get_user(user_id, username)
    levels_text = format_levels_page(user_data, page=1)
    keyboard = create_levels_keyboard(page=1)
    
    bot.send_message(
        message.chat.id,
        levels_text,
        reply_to_message_id=message.message_id,
        reply_markup=keyboard
    )

@bot.message_handler(func=lambda msg: msg.text and msg.text.lower() in ['работа', 'работы', 'вакансии', 'профессии'])
def jobs_keyword(message):
    data_manager.increment_message_counter()
    keyboard = create_jobs_keyboard()
    bot.send_message(
        message.chat.id,
        "<b>💼 Список всех работ</b>\n\nВыберите работу для подробной информации:",
        reply_to_message_id=message.message_id,
        reply_markup=keyboard
    )

@bot.message_handler(func=lambda msg: msg.text and msg.text.lower() in ['моя работа', 'работать', 'заработать', 'трудиться'])
def my_job_keyword(message):
    data_manager.increment_message_counter()
    user_id = message.from_user.id
    username = message.from_user.username or message.from_user.first_name
    
    user_data = data_manager.get_user(user_id, username)
    
    if not user_data['current_job']:
        bot.send_message(
            message.chat.id,
            "❌ <b>У вас нет работы!</b>\n\nПерейдите в раздел 💼 <b>Работы</b> чтобы устроиться на работу.",
            reply_to_message_id=message.message_id
        )
        return
    
    job = JOBS.get(user_data['current_job'])
    if not job:
        bot.send_message(
            message.chat.id,
            "❌ <b>Ошибка: работа не найдена!</b>",
            reply_to_message_id=message.message_id
        )
        return
    
    can_work, message_text, _ = data_manager.can_work(user_data)
    
    cooldown_min = job['cooldown'] // 60
    cooldown_sec = job['cooldown'] % 60
    
    job_text = f"""
<b>{job['name']}</b>
━━━━━━━━━━━━━━━━━━

<b>📝 Описание:</b> {job['description']}

<b>📊 Информация о работе:</b>
• Зарплата: {job['salary']:,}₽
• Опыт за работу: {job['experience']}
• Требуемый уровень: {job['min_level']}+
• Время работы: {cooldown_min:02d}:{cooldown_sec:02d}

<b>⏰ Статус:</b> {message_text}
"""
    
    keyboard = create_work_keyboard()
    bot.send_message(
        message.chat.id,
        job_text,
        reply_to_message_id=message.message_id,
        reply_markup=keyboard
    )

@bot.message_handler(func=lambda msg: msg.text and msg.text.lower() in ['топы', 'топ', 'лидеры', 'лидерборд', 'рейтинг'])
def leaders_keyword(message):
    data_manager.increment_message_counter()
    try:
        leaders, update_time = data_manager.get_leaderboard()
        
        if not leaders:
            bot.send_message(
                message.chat.id,
                "📊 Лидеров пока нет!",
                reply_to_message_id=message.message_id
            )
            return
        
        leaderboard_text = "🏆 <b>ТОП-10 ИГРОКОВ ПО БАЛАНСУ</b>\n\n"
        
        place_emojis = {
            1: "🥇", 2: "🥈", 3: "🥉",
            4: "4️⃣", 5: "5️⃣", 6: "6️⃣",
            7: "7️⃣", 8: "8️⃣", 9: "9️⃣", 10: "🔟"
        }
        
        for i, user in enumerate(leaders, 1):
            emoji = place_emojis.get(i, f"{i}.")
            username = user.get('username', 'Игрок')
            balance = user.get('balance', 0)
            
            if balance >= 1_000_000_000_000:
                formatted_balance = f"₽{balance / 1_000_000_000_000:.1f} трлн"
            elif balance >= 1_000_000_000:
                formatted_balance = f"₽{balance / 1_000_000_000:.1f} млрд"
            elif balance >= 1_000_000:
                formatted_balance = f"₽{balance / 1_000_000:.1f} млн"
            elif balance >= 1_000:
                formatted_balance = f"₽{balance / 1_000:.1f}к"
            else:
                formatted_balance = f"₽{balance:,}"
            
            leaderboard_text += f"{emoji} <b>{username}</b> — {formatted_balance}\n"
        
        keyboard = create_leaders_keyboard()
        
        bot.send_message(
            message.chat.id,
            leaderboard_text,
            reply_to_message_id=message.message_id,
            reply_markup=keyboard
        )
    except Exception as e:
        bot.send_message(
            message.chat.id,
            f"❌ Ошибка при загрузке лидерборда",
            reply_to_message_id=message.message_id
        )

@bot.message_handler(func=lambda msg: msg.text and msg.text.lower() in ['реф', 'рефералы', 'пригласить', 'пригласить друга', 'рефералка'])
def referrals_keyword(message):
    data_manager.increment_message_counter()
    user_id = message.from_user.id
    username = message.from_user.username or message.from_user.first_name
    
    user_data = data_manager.get_user(user_id, username)
    referrals_text = format_referrals_info(user_data)
    keyboard = create_referrals_keyboard()
    
    bot.send_message(
        message.chat.id,
        referrals_text,
        reply_to_message_id=message.message_id,
        reply_markup=keyboard
    )

@bot.message_handler(func=lambda msg: msg.text and msg.text.lower() in ['квесты', 'задания', 'задание', 'квест', 'миссии'])
def quests_keyword(message):
    data_manager.increment_message_counter()
    user_id = message.from_user.id
    username = message.from_user.username or message.from_user.first_name
    
    data_manager.get_user(user_id, username)
    
    quests_text = """
<b>📋 СИСТЕМА КВЕСТОВ</b>
━━━━━━━━━━━━━━━━━━

<b>🎯 Ежедневные квесты:</b>
• Меняются каждый день в 00:00
• 3 случайных квеста ежедневно
• Награды: 2,000₽ - 60,000₽ и 100 - 3,000 опыта

<b>📆 Недельные квесты:</b>
• Меняются каждый понедельник в 00:00
• 2 случайных квеста еженедельно
• Награды: 50,000₽ - 200,000₽ и 1,000 - 12,000 опыта
"""
    
    keyboard = create_quests_menu_keyboard()
    bot.send_message(
        message.chat.id,
        quests_text,
        reply_to_message_id=message.message_id,
        reply_markup=keyboard
    )

# ОБРАБОТЧИКИ ДЛЯ БИЗНЕСОВ
@bot.message_handler(func=lambda msg: msg.text and msg.text.lower() in ['магазин бизнеса', 'магазин бизнесов', 'магазин', 'бизнес магазин'])
def business_shop_keyword(message):
    data_manager.increment_message_counter()
    user_id = message.from_user.id
    username = message.from_user.username or message.from_user.first_name
    
    user_data = data_manager.get_user(user_id, username)
    business_info = format_business_info(1, user_data)
    keyboard = create_business_shop_keyboard(1, user_data.get('business_id') is not None)
    
    bot.send_message(
        message.chat.id,
        business_info,
        reply_to_message_id=message.message_id,
        reply_markup=keyboard
    )

@bot.message_handler(func=lambda msg: msg.text and msg.text.lower() in ['мой бизнес', 'бизнес'])
def my_business_keyword(message):
    data_manager.increment_message_counter()
    user_id = message.from_user.id
    username = message.from_user.username or message.from_user.first_name
    
    data_manager.update_business_progress(user_id)
    
    user_data = data_manager.get_user(user_id, username)
    business_info = format_my_business_info(user_data)
    
    if "нету бизнеса" in business_info:
        keyboard = types.InlineKeyboardMarkup()
        keyboard.add(types.InlineKeyboardButton('🏪 Магазин бизнеса', callback_data='business_shop_1'))
        keyboard.add(types.InlineKeyboardButton('◀️ Назад', callback_data='back_to_menu'))
    else:
        keyboard = create_my_business_keyboard()
    
    bot.send_message(
        message.chat.id,
        business_info,
        reply_to_message_id=message.message_id,
        reply_markup=keyboard
    )

# Обработчик казино в чатах (каз и казино)
@bot.message_handler(func=lambda msg: msg.text and (msg.text.lower().startswith('каз') or msg.text.lower().startswith('казино')))
def casino_keyword(message):
    data_manager.increment_message_counter()
    user_id = message.from_user.id
    username = message.from_user.username or message.from_user.first_name
    
    if message.forward_from or message.forward_date or message.forward_from_chat or message.forward_sender_name:
        return
    
    user_data = data_manager.get_user(user_id, username)
    coins = user_data['balance']
    
    text = message.text.strip()
    parts = text.split()
    
    if len(parts) < 2:
        casino_info = f"""
🎰 {username}, <b>ИГРА КАЗИНО</b>

💰 <b>Коэффициенты:</b>
• 10% шанс выиграть <b>x5</b> (ДЖЕКПОТ)
• 30% шанс выиграть <b>x0.3 - x2.0</b>
• 30% шанс проиграть <b>x0.1 - x0.8</b>
• 20% шанс остаться при своих
• 10% шанс потерять всю ставку

🎰 <b>Используйте:</b>

⬜ <code>каз 100</code> <b>- конкретная ставка</b>

⬜ <code>каз 30к</code> <b>- 30,000₽</b>

⬜ <code>каз 3кк</code> <b>- 3,000,000₽</b>

⬜ <code>каз 2ккк</code> <b>- 2,000,000,000₽</b>

⬜ <code>каз 1кккк</code> <b>- 1,000,000,000,000₽</b>

⬜ <code>каз всё</code> или <code>каз все</code> <b>- поставить весь баланс</b>

💰 <b>Ваш баланс:</b> {format_number(coins)} <b>₽</b>
💡 Примеры: <code>каз 50</code>, <code>каз 1к</code>, <code>каз 2.5к</code>, <code>каз 1.5кк</code>, <code>каз 2.5ккк</code>
"""
        
        bot.send_message(
            message.chat.id,
            casino_info,
            parse_mode="HTML",
            disable_web_page_preview=True,
            disable_notification=True,
            reply_to_message_id=message.message_id
        )
        return
    
    try:
        bet_text = parts[1].lower().strip()
        bet = parse_bet_with_k(bet_text, coins)
        
        if bet < 10:
            bot.send_message(
                message.chat.id,
                f"❌ {username}, <b>ставка не может быть меньше 10₽</b> ❌",
                parse_mode="HTML",
                disable_web_page_preview=True,
                disable_notification=True,
                reply_to_message_id=message.message_id
            )
            return
            
    except ValueError as ve:
        bot.send_message(
            message.chat.id,
            f"❌ {username}, <b>некорректная ставка</b> ❌\n\n"
            f"🎰 <b>Используйте:</b>\n"
            f"• <code>каз 100</code> - обычная ставка\n"
            f"• <code>каз 1к</code> - 1,000₽\n"
            f"• <code>каз 2.5к</code> - 2,500₽\n"
            f"• <code>каз 1.5кк</code> - 1,500,000₽\n"
            f"• <code>каз 2ккк</code> - 2,000,000,000₽\n"
            f"• <code>каз всё</code> - весь баланс",
            parse_mode="HTML",
            disable_web_page_preview=True,
            disable_notification=True,
            reply_to_message_id=message.message_id
        )
        return
    except Exception as e:
        bot.send_message(
            message.chat.id,
            f"❌ {username}, <b>ошибка при обработке ставки</b> ❌",
            parse_mode="HTML",
            disable_web_page_preview=True,
            disable_notification=True,
            reply_to_message_id=message.message_id
        )
        return

    if bet > coins:
        bot.send_message(
            message.chat.id,
            f"❌ {username}, <b>недостаточно средств</b> ❌\n\n"
            f"💰 <b>Ваш баланс:</b> {format_number(coins)} <b>₽</b>\n"
            f"🎰 <b>Требуется:</b> {format_number(bet)} <b>₽</b>\n\n"
            f"📉 <b>Не хватает:</b> {format_number(bet - coins)} <b>₽</b>",
            parse_mode="HTML",
            disable_web_page_preview=True,
            disable_notification=True,
            reply_to_message_id=message.message_id
        )
        return
    
    outcomes = get_casino_outcomes(coins)
    
    chosen = random.choices(outcomes, weights=[o["prob"] for o in outcomes], k=1)[0]
    win = int(bet * chosen["mult"])
    new_balance = coins + win
    
    data_manager.update_user(user_id, {'balance': new_balance})
    
    if win > 0:
        user_id_str = str(user_id)
        user_data = data_manager.get_user(user_id)
        user_data['daily_stats']['money_earned'] += win
        user_data['weekly_stats']['money_earned'] += win
        
        data_manager.update_user(user_id, {
            'daily_stats': user_data['daily_stats'],
            'weekly_stats': user_data['weekly_stats']
        })
        
        quest_system.update_quest_progress(user_id, 'daily', 'earn_money', win)
        quest_system.update_quest_progress(user_id, 'weekly', 'earn_money', win)
    
    slots = ["🍒", "🍋", "🍊", "🍇", "💎", "🍕"]
    reels = [random.choice(slots), random.choice(slots), random.choice(slots)]
    
    if chosen["mult"] > 0:
        if chosen["mult"] >= 5:
            reels[1] = reels[0]
            reels[2] = reels[0]
        elif chosen["mult"] >= 2:
            reels[1] = reels[0]
            reels[2] = random.choice(slots)
        elif chosen["mult"] >= 1:
            if random.random() > 0.5:
                reels[1] = reels[0]
    
    result_msg = (
        f"🎰 {username}, {chosen['text']} \n\n"
        f"┏━━━━━━━━━━━┓\n"
        f"┃  {reels[0]}  |  {reels[1]}  |  {reels[2]}  ┃\n"
        f"┗━━━━━━━━━━━┛\n\n"
    )
    
    if win > 0:
        result_msg += f"🔺️ Итог: <b>+{format_number(win)}</b>₽\n\n"
    elif win < 0:
        result_msg += f"🔻 Итог: <b>-{format_number(abs(win))}</b>₽\n\n"
    else:
        result_msg += f"🌸 Итог: <b>+0</b>₽\n\n"
    
    result_msg += f"💰 <b>Ваш баланс:</b> {format_number(new_balance)} <b>₽</b>"
    
    bot.send_message(
        message.chat.id,
        result_msg,
        parse_mode="HTML",
        disable_web_page_preview=True,
        disable_notification=True,
        reply_to_message_id=message.message_id
    )

# Обработчик перевода в чатах
@bot.message_handler(func=lambda msg: msg.text and msg.text.lower().startswith('передать'))
def transfer_keyword(message):
    data_manager.increment_message_counter()
    user_id = message.from_user.id
    username = message.from_user.username or message.from_user.first_name
    
    if message.forward_from or message.forward_date or message.forward_from_chat or message.forward_sender_name:
        return
    
    if not message.reply_to_message:
        help_text = """
❌ <b>Использование:</b>

1. Ответьте на сообщение пользователя
2. Напишите: <code>передать сумма</code>
   или <code>дать сумма</code>

<b>Примеры:</b>
<code>передать 1000</code>
<code>передать 30к</code> (30,000₽)
<code>передать 2кк</code> (2,000,000₽)
<code>передать 1.5ккк</code> (1,500,000,000₽)
<code>передать 1кккк</code> (1,000,000,000,000₽)
<code>передать все</code>

<code>дать 500</code>
<code>дать 5к</code>
<code>дать 2кк</code>
<code>дать все</code>
"""
        bot.reply_to(message, help_text, parse_mode='HTML')
        return
    
    receiver = message.reply_to_message.from_user
    
    if receiver.id == user_id:
        bot.reply_to(message, "❌ Нельзя передавать деньги самому себе!")
        return
    
    if receiver.is_bot:
        bot.reply_to(message, "❌ Нельзя передавать деньги ботам!")
        return
    
    parts = message.text.split()
    if len(parts) < 2:
        bot.reply_to(
            message,
            "❌ <b>Укажите сумму!</b>\n\n"
            "<b>Примеры:</b>\n"
            "<code>передать 1000</code>\n"
            "<code>передать 30к</code> (30,000₽)\n"
            "<code>передать 2кк</code> (2,000,000₽)\n"
            "<code>передать 1.5ккк</code> (1,500,000,000₽)\n"
            "<code>передать все</code>",
            parse_mode='HTML'
        )
        return
    
    amount_str = parts[1].lower()
    
    sender_data = data_manager.get_user(user_id, username)
    sender_balance = sender_data['balance']
    
    try:
        amount = parse_amount_with_k(amount_str, sender_balance)
        
        if amount <= 0:
            bot.reply_to(message, "❌ Сумма должна быть положительной!")
            return
            
    except ValueError as ve:
        error_text = f"""
❌ <b>Неверная сумма!</b>

🎯 <b>Правильные форматы:</b>
• <code>1000</code> - обычное число
• <code>30к</code> - 30,000₽
• <code>2кк</code> - 2,000,000₽
• <code>1.5ккк</code> - 1,500,000,000₽
• <code>1кккк</code> - 1,000,000,000,000₽
• <code>все</code> - весь баланс

💰 <b>Ваш баланс:</b> {format_number(sender_balance)}₽
"""
        bot.reply_to(message, error_text, parse_mode='HTML')
        return
    
    if sender_balance < amount:
        bot.reply_to(
            message,
            f"❌ <b>Недостаточно средств!</b>\n\n"
            f"💰 <b>Ваш баланс:</b> {format_number(sender_balance)}₽\n"
            f"🎯 <b>Требуется:</b> {format_number(amount)}₽\n"
            f"📉 <b>Не хватает:</b> {format_number(amount - sender_balance)}₽",
            parse_mode='HTML'
        )
        return
    
    if amount < 10:
        bot.reply_to(message, "❌ Минимальная сумма перевода: 10₽!")
        return
    
    receiver_name = receiver.username or receiver.first_name
    receiver_data = data_manager.get_user(receiver.id, receiver_name)
    receiver_balance = receiver_data['balance']
    
    new_sender_balance = sender_balance - amount
    new_receiver_balance = receiver_balance + amount
    
    data_manager.update_user(user_id, {'balance': new_sender_balance})
    data_manager.update_user(receiver.id, {'balance': new_receiver_balance})
    
    user_id_str = str(user_id)
    sender_data = data_manager.get_user(user_id)
    sender_data['daily_stats']['money_earned'] += amount
    sender_data['weekly_stats']['money_earned'] += amount
    
    data_manager.update_user(user_id, {
        'daily_stats': sender_data['daily_stats'],
        'weekly_stats': sender_data['weekly_stats']
    })
    
    receiver_id_str = str(receiver.id)
    receiver_data = data_manager.get_user(receiver.id)
    receiver_data['daily_stats']['money_earned'] += amount
    receiver_data['weekly_stats']['money_earned'] += amount
    
    data_manager.update_user(receiver.id, {
        'daily_stats': receiver_data['daily_stats'],
        'weekly_stats': receiver_data['weekly_stats']
    })
    
    quest_system.update_quest_progress(user_id, 'daily', 'earn_money', amount)
    quest_system.update_quest_progress(user_id, 'weekly', 'earn_money', amount)
    quest_system.update_quest_progress(receiver.id, 'daily', 'earn_money', amount)
    quest_system.update_quest_progress(receiver.id, 'weekly', 'earn_money', amount)
    
    data_manager.update_leaderboard_cache()
    
    if amount_str in ["всё", "все", "all"]:
        amount_display = "весь баланс"
    elif parts[1].endswith('кккк'):
        amount_display = f"{parts[1]} ({format_number(amount)}₽)"
    elif parts[1].endswith('ккк'):
        amount_display = f"{parts[1]} ({format_number(amount)}₽)"
    elif parts[1].endswith('кк'):
        amount_display = f"{parts[1]} ({format_number(amount)}₽)"
    elif parts[1].endswith('к'):
        amount_display = f"{parts[1]} ({format_number(amount)}₽)"
    else:
        amount_display = f"{format_number(amount)}₽"
    
    result_text = f"""
✅ <b>УСПЕШНЫЙ ПЕРЕВОД!</b> ✅
━━━━━━━━━━━━━━━━━━

👤 <b>Отправитель:</b> {sender_data['username']}
👤 <b>Получатель:</b> {receiver_data['username']}

💰 <b>Сумма перевода:</b> {amount_display}

💳 <b>Баланс отправителя:</b>
   Было: {format_number(sender_balance)}₽
   Стало: {format_number(new_sender_balance)}₽

💳 <b>Баланс получателя:</b>
   Было: {format_number(receiver_balance)}₽
   Стало: {format_number(new_receiver_balance)}₽
"""
    
    bot.reply_to(message, result_text, parse_mode='HTML')
    
    if message.chat.id != receiver.id:
        try:
            bot.send_message(
                receiver.id,
                f"🎉 <b>ВЫ ПОЛУЧИЛИ ПЕРЕВОД!</b>\n\n"
                f"👤 <b>От:</b> {sender_data['username']}\n"
                f"💰 <b>Сумма:</b> {format_number(amount)}₽\n"
                f"🏦 <b>Ваш баланс:</b> {format_number(new_receiver_balance)}₽",
                parse_mode='HTML'
            )
        except:
            pass

# Обработчик для ключевого слова "дать" и "убрать" для админа
@bot.message_handler(func=lambda msg: msg.text and msg.text.lower() in ['дать', 'убрать'] and msg.reply_to_message and msg.from_user.id == ADMIN_ID)
def admin_give_remove(message):
    data_manager.increment_message_counter()
    command = message.text.lower()
    target_user = message.reply_to_message.from_user
    
    if command == 'дать':
        if data_manager.add_admin(target_user.id):
            bot.reply_to(message, f"✅ Пользователь {target_user.username or target_user.first_name} добавлен в админы.")
        else:
            bot.reply_to(message, f"❌ Пользователь уже является админом.")
    else:
        if data_manager.remove_admin(target_user.id):
            bot.reply_to(message, f"✅ Пользователь {target_user.username or target_user.first_name} удален из админов.")
        else:
            bot.reply_to(message, f"❌ Не удалось удалить пользователя из админов.")

# Обработчик для ключевого слова "дать"
@bot.message_handler(func=lambda msg: msg.text and msg.text.lower().startswith('дать'))
def give_keyword(message):
    data_manager.increment_message_counter()
    message.text = message.text.replace('дать', 'передать', 1)
    transfer_keyword(message)

# Обработчики для личных сообщений (кнопки)
@bot.message_handler(func=lambda msg: msg.text == '👤 Профиль' and msg.chat.type == 'private')
def profile_button(message):
    data_manager.increment_message_counter()
    profile_keyword(message)

@bot.message_handler(func=lambda msg: msg.text == '🎁 Ежедневный бонус' and msg.chat.type == 'private')
def bonus_button(message):
    data_manager.increment_message_counter()
    bonus_keyword(message)

@bot.message_handler(func=lambda msg: msg.text == '📊 Уровни' and msg.chat.type == 'private')
def levels_button(message):
    data_manager.increment_message_counter()
    levels_keyword(message)

@bot.message_handler(func=lambda msg: msg.text == '💼 Работы' and msg.chat.type == 'private')
def jobs_button(message):
    data_manager.increment_message_counter()
    jobs_keyword(message)

@bot.message_handler(func=lambda msg: msg.text == '🛠️ Моя работа' and msg.chat.type == 'private')
def my_job_button(message):
    data_manager.increment_message_counter()
    my_job_keyword(message)

@bot.message_handler(func=lambda msg: msg.text == '🏆 Лидеры' and msg.chat.type == 'private')
def leaders_button(message):
    data_manager.increment_message_counter()
    leaders_keyword(message)

@bot.message_handler(func=lambda msg: msg.text == '👥 Рефералы' and msg.chat.type == 'private')
def referrals_button(message):
    data_manager.increment_message_counter()
    referrals_keyword(message)

@bot.message_handler(func=lambda msg: msg.text == '📋 Задание' and msg.chat.type == 'private')
def quests_button(message):
    data_manager.increment_message_counter()
    quests_keyword(message)

@bot.message_handler(func=lambda msg: msg.text == '🎰 Казино' and msg.chat.type == 'private')
def casino_button(message):
    data_manager.increment_message_counter()
    casino_keyword(message)

@bot.message_handler(func=lambda msg: msg.text == '💸 Перевести' and msg.chat.type == 'private')
def transfer_button(message):
    data_manager.increment_message_counter()
    user_id = message.from_user.id
    username = message.from_user.username or message.from_user.first_name
    
    user_data = data_manager.get_user(user_id, username)
    
    transfer_info = f"""
💸 <b>СИСТЕМА ПЕРЕВОДОВ</b>
━━━━━━━━━━━━━━━━━━

👤 <b>Ваш баланс:</b> {format_number(user_data['balance'])}₽

<b>📋 Как использовать:</b>
1. Ответьте на сообщение пользователя
2. Напишите: <code>передать сумма</code>
   или <code>дать сумма</code>

<b>📝 Примеры:</b>
• <code>передать 1000</code> - перевести 1,000₽
• <code>передать 30к</code> - перевести 30,000₽
• <code>передать 2кк</code> - перевести 2,000,000₽
• <code>передать 1.5ккк</code> - перевести 1,500,000,000₽
• <code>передать 1кккк</code> - перевести 1,000,000,000,000₽
• <code>передать все</code> - перевести весь баланс

<b>⚙️ Правила:</b>
• Минимальная сумма: 10₽
• Нельзя переводить самому себе
• Нельзя переводить ботам
• Комиссия: 0% (бесплатно)
"""
    
    if message.chat.type == 'private':
        bot.send_message(
            message.chat.id,
            transfer_info,
            reply_markup=create_main_keyboard()
        )
    else:
        bot.send_message(
            message.chat.id,
            transfer_info,
            reply_to_message_id=message.message_id,
            reply_markup=create_transfer_keyboard()
        )

# ОБРАБОТЧИКИ ДЛЯ КНОПОК БИЗНЕСА
@bot.message_handler(func=lambda msg: msg.text == '🏪 Магазин бизнеса' and msg.chat.type == 'private')
def business_shop_button(message):
    data_manager.increment_message_counter()
    business_shop_keyword(message)

@bot.message_handler(func=lambda msg: msg.text == '🏢 Мой бизнес' and msg.chat.type == 'private')
def my_business_button(message):
    data_manager.increment_message_counter()
    my_business_keyword(message)

# Обработчики команд
@bot.message_handler(commands=['casino'])
def casino_command(message):
    data_manager.increment_message_counter()
    casino_keyword(message)

@bot.message_handler(commands=['передать', 'дать'])
def transfer_command(message):
    data_manager.increment_message_counter()
    if message.chat.type == 'private':
        transfer_button(message)
    else:
        transfer_keyword(message)

@bot.message_handler(commands=['ref', 'referral', 'реф'])
def referrals_command(message):
    data_manager.increment_message_counter()
    referrals_keyword(message)

@bot.message_handler(commands=['leaders', 'top', 'топ'])
def leaders_command(message):
    data_manager.increment_message_counter()
    leaders_keyword(message)

@bot.message_handler(commands=['claim'])
def claim_command(message):
    data_manager.increment_message_counter()
    user_id = message.from_user.id
    username = message.from_user.username or message.from_user.first_name
    
    success, total_reward, unclaimed = data_manager.claim_level_rewards(user_id)
    
    if not success:
        response = "❌ У вас нет неполученных наград за уровни!"
        if message.chat.type == 'private':
            bot.send_message(message.chat.id, response, reply_markup=create_main_keyboard())
        else:
            bot.send_message(message.chat.id, response, reply_to_message_id=message.message_id)
        return
    
    user_data = data_manager.get_user(user_id, username)
    
    reward_text = f"""
<b>🎉 ВЫ ПОЛУЧИЛИ НАГРАДЫ ЗА УРОВНИ!</b>
━━━━━━━━━━━━━━━━━━

<b>👤 Игрок:</b> {username}
<b>💰 Получено:</b> {total_reward:,}₽
<b>🏦 Новый баланс:</b> {user_data['balance']:,}₽

<b>📊 Полученные награды:</b>
"""
    
    for level, reward in unclaimed:
        reward_text += f"• Уровень {level}: {reward:,}₽\n"
    
    reward_text += "\n<b>🎮 Продолжайте играть и повышать уровень!</b>"
    
    if message.chat.type == 'private':
        bot.send_message(
            message.chat.id,
            reward_text,
            reply_markup=create_main_keyboard()
        )
    else:
        bot.send_message(
            message.chat.id,
            reward_text,
            reply_to_message_id=message.message_id,
            reply_markup=create_main_inline_keyboard()
        )

# КОМАНДЫ АДМИН-ПАНЕЛИ
@bot.message_handler(commands=['admin'])
def admin_command(message):
    data_manager.increment_message_counter()
    if not data_manager.is_admin(message.from_user.id):
        bot.reply_to(message, "⛔ У вас нет доступа к админ-панели.")
        return
    
    bot.send_message(
        message.chat.id,
        "🔧 <b>АДМИН-ПАНЕЛЬ</b>\n\nВыберите действие:",
        reply_markup=create_admin_keyboard()
    )

@bot.message_handler(commands=['give_money'])
def give_money_command(message):
    data_manager.increment_message_counter()
    if not data_manager.is_admin(message.from_user.id):
        bot.reply_to(message, "⛔ У вас нет прав для использования этой команды.")
        return
    
    parts = message.text.split()
    if len(parts) != 3:
        bot.reply_to(message, "❌ Неверный формат. Используйте: /give_money <user_id> <amount>")
        return
    
    try:
        target_user_id = parts[1]
        amount = int(parts[2])
        
        user_data = data_manager.db.get_user(target_user_id)
        if not user_data:
            bot.reply_to(message, "❌ Пользователь не найден.")
            return
        
        new_balance = user_data['balance'] + amount
        data_manager.update_user(target_user_id, {'balance': new_balance})
        bot.reply_to(message, f"✅ Пользователю {user_data['username']} выдано {amount}₽. Новый баланс: {new_balance}₽")
    except Exception as e:
        bot.reply_to(message, f"❌ Ошибка: {e}")

@bot.message_handler(commands=['give_money_by_username'])
def give_money_by_username_command(message):
    data_manager.increment_message_counter()
    if not data_manager.is_admin(message.from_user.id):
        bot.reply_to(message, "⛔ У вас нет прав для использования этой команды.")
        return
    
    parts = message.text.split()
    if len(parts) != 3:
        bot.reply_to(message, "❌ Неверный формат. Используйте: /give_money_by_username <username> <amount>")
        return
    
    try:
        target_username = parts[1]
        amount = int(parts[2])
        
        user_data = data_manager.db.get_user_by_username(target_username)
        if not user_data:
            bot.reply_to(message, "❌ Пользователь с таким username не найден.")
            return
        
        target_user_id = user_data['user_id']
        new_balance = user_data['balance'] + amount
        data_manager.update_user(target_user_id, {'balance': new_balance})
        bot.reply_to(message, f"✅ Пользователю {user_data['username']} выдано {amount}₽. Новый баланс: {new_balance}₽")
    except Exception as e:
        bot.reply_to(message, f"❌ Ошибка: {e}")

@bot.message_handler(commands=['give_exp'])
def give_exp_command(message):
    data_manager.increment_message_counter()
    if not data_manager.is_admin(message.from_user.id):
        bot.reply_to(message, "⛔ У вас нет прав для использования этой команды.")
        return
    
    parts = message.text.split()
    if len(parts) != 3:
        bot.reply_to(message, "❌ Неверный формат. Используйте: /give_exp <user_id> <amount>")
        return
    
    try:
        target_user_id = parts[1]
        exp_amount = int(parts[2])
        
        user_data = data_manager.db.get_user(target_user_id)
        if not user_data:
            bot.reply_to(message, "❌ Пользователь не найден.")
            return
        
        levels_gained, new_level = data_manager.add_experience(target_user_id, exp_amount)
        bot.reply_to(message, f"✅ Пользователю {user_data['username']} выдано {exp_amount} опыта. Получено уровней: {levels_gained}, текущий уровень: {new_level}")
    except Exception as e:
        bot.reply_to(message, f"❌ Ошибка: {e}")

@bot.message_handler(commands=['give_id'])
def give_id_command(message):
    data_manager.increment_message_counter()
    if not data_manager.is_admin(message.from_user.id):
        bot.reply_to(message, "⛔ У вас нет прав для использования этой команды.")
        return
    
    parts = message.text.split()
    if len(parts) != 3:
        bot.reply_to(message, "❌ Неверный формат. Используйте: /give_id <username> <game_id>")
        return
    
    try:
        username = parts[1]
        game_id = parts[2]
        
        user_data = data_manager.db.get_user_by_username(username)
        if not user_data:
            bot.reply_to(message, "❌ Пользователь с таким username не найден.")
            return
        
        user_id = user_data['user_id']
        data_manager.update_user(user_id, {'game_id': game_id})
        bot.reply_to(message, f"✅ Пользователю {username} выдан игровой ID: {game_id}")
    except Exception as e:
        bot.reply_to(message, f"❌ Ошибка: {e}")

@bot.message_handler(func=lambda msg: msg.text == 'админ12' and data_manager.is_admin(msg.from_user.id))
def admin12_command(message):
    try:
        bot.delete_message(message.chat.id, message.message_id)
    except:
        pass
    
    admin_command(message)

# Обработчик callback запросов
@bot.callback_query_handler(func=lambda call: True)
def callback_handler(call):
    user_id = call.from_user.id
    username = call.from_user.username or call.from_user.first_name
    
    # Защита от нажатия чужих кнопок
    if call.message.chat.type != 'private':
        if call.message.reply_to_message:
            if call.from_user.id != call.message.reply_to_message.from_user.id:
                # Разрешаем только публичные callback
                public_callbacks = ['casino_info', 'casino_rules', 'casino_play', 
                                  'transfer_info', 'transfer_rules',
                                  'leaders', 'refresh_leaders', 'levels_leaders',
                                  'business_shop_1', 'business_shop_2', 'business_shop_3', 
                                  'business_shop_4', 'business_shop_5', 'business_shop_6', 
                                  'business_shop_7', 'admin_']
                if not any(call.data.startswith(prefix) for prefix in ['admin_', 'business_shop_']) and call.data not in public_callbacks:
                    bot.answer_callback_query(call.id, "❌ Эти кнопки не для вас!", show_alert=True)
                    return
    
    if call.data.startswith('admin_') and not data_manager.is_admin(user_id):
        bot.answer_callback_query(call.id, "⛔ У вас нет доступа к админ-панели.", show_alert=True)
        return
    
    try:
        # Обработка админских callback
        if call.data == 'admin_stats':
            stats = data_manager.get_statistics()
            stats_text = f"""
📊 <b>СТАТИСТИКА БОТА</b>
━━━━━━━━━━━━━━━━━━

👥 <b>Всего пользователей:</b> {stats['total_users']}
📨 <b>Сообщений сегодня:</b> {stats['today_messages']}
📨 <b>Всего сообщений:</b> {stats['total_messages']}
"""
            bot.edit_message_text(
                chat_id=call.message.chat.id,
                message_id=call.message.message_id,
                text=stats_text,
                reply_markup=create_admin_back_keyboard()
            )
        
        elif call.data == 'admin_give_money':
            instruction = """
💰 <b>ВЫДАЧА ДЕНЕГ ПО ID</b>
━━━━━━━━━━━━━━━━━━

<b>Используйте команду:</b>
<code>/give_money &lt;user_id&gt; &lt;amount&gt;</code>

<b>Пример:</b>
<code>/give_money 123456789 10000</code>

<b>Примечание:</b>
• user_id - ID пользователя в боте
• amount - количество денег
• Можно выдавать любому существующему пользователю
"""
            bot.edit_message_text(
                chat_id=call.message.chat.id,
                message_id=call.message.message_id,
                text=instruction,
                parse_mode='HTML',
                reply_markup=create_admin_back_keyboard()
            )
        
        elif call.data == 'admin_give_money_by_username':
            instruction = """
💰 <b>ВЫДАЧА ДЕНЕГ ПО USERNAME</b>
━━━━━━━━━━━━━━━━━━

<b>Используйте команду:</b>
<code>/give_money_by_username &lt;username&gt; &lt;amount&gt;</code>

<b>Пример:</b>
<code>/give_money_by_username username 10000</code>

<b>Примечание:</b>
• username - имя пользователя в Telegram (без @)
• amount - количество денег
• Можно выдавать любому существующему пользователю
"""
            bot.edit_message_text(
                chat_id=call.message.chat.id,
                message_id=call.message.message_id,
                text=instruction,
                parse_mode='HTML',
                reply_markup=create_admin_back_keyboard()
            )
        
        elif call.data == 'admin_give_exp':
            instruction = """
🌟 <b>ВЫДАЧА ОПЫТА</b>
━━━━━━━━━━━━━━━━━━

<b>Используйте команду:</b>
<code>/give_exp &lt;user_id&gt; &lt;amount&gt;</code>

<b>Пример:</b>
<code>/give_exp 123456789 500</code>

<b>Примечание:</b>
• user_id - ID пользователя в боте
• amount - количество опыта
• Можно выдавать любому существующему пользователю
• Если пользователя нет - не создается
"""
            bot.edit_message_text(
                chat_id=call.message.chat.id,
                message_id=call.message.message_id,
                text=instruction,
                parse_mode='HTML',
                reply_markup=create_admin_back_keyboard()
            )
        
        elif call.data == 'admin_give_id':
            instruction = """
🆔 <b>ВЫДАЧА ID</b>
━━━━━━━━━━━━━━━━━━

<b>Используйте команду:</b>
<code>/give_id &lt;username&gt; &lt;game_id&gt;</code>

<b>Пример:</b>
<code>/give_id username ABC123XYZ</code>

<b>Примечание:</b>
• username - имя пользователя в Telegram
• game_id - игровой ID который нужно выдать (любые буквы и цифры, без ограничения длины)
• Можно выдавать любой ID (буквы, цифры, символы)
• Если пользователя нет - не создается
"""
            bot.edit_message_text(
                chat_id=call.message.chat.id,
                message_id=call.message.message_id,
                text=instruction,
                parse_mode='HTML',
                reply_markup=create_admin_back_keyboard()
            )
        
        elif call.data.startswith('admin_profiles_'):
            page = int(call.data.split('_')[2])
            users = data_manager.db.get_all_users()
            users_per_page = 10
            start = (page-1) * users_per_page
            end = start + users_per_page
            users_page = users[start:end]
            
            text = f"👥 <b>ПРОСМОТР ПРОФИЛЕЙ</b> (страница {page})\n\n"
            
            keyboard = types.InlineKeyboardMarkup(row_width=2)
            
            for user in users_page:
                user_id = user['user_id']
                username = user.get('username', 'Без имени')
                keyboard.add(types.InlineKeyboardButton(
                    f"{username}",
                    callback_data=f'admin_view_profile_{user_id}'
                ))
            
            nav_buttons = []
            if page > 1:
                nav_buttons.append(types.InlineKeyboardButton('◀️', callback_data=f'admin_profiles_{page-1}'))
            
            nav_buttons.append(types.InlineKeyboardButton(f'{page}', callback_data='none'))
            
            if end < len(users):
                nav_buttons.append(types.InlineKeyboardButton('▶️', callback_data=f'admin_profiles_{page+1}'))
            
            if nav_buttons:
                keyboard.add(*nav_buttons)
            
            keyboard.add(types.InlineKeyboardButton('🔙 Назад', callback_data='admin_back'))
            
            bot.edit_message_text(
                chat_id=call.message.chat.id,
                message_id=call.message.message_id,
                text=text,
                reply_markup=keyboard
            )
        
        elif call.data.startswith('admin_view_profile_'):
            user_id_to_view = call.data.split('_')[3]
            user_data = data_manager.get_user(user_id_to_view)
            profile_text = format_profile(user_data)
            
            keyboard = types.InlineKeyboardMarkup()
            keyboard.add(types.InlineKeyboardButton('🔙 Назад к списку', callback_data='admin_profiles_1'))
            
            bot.edit_message_text(
                chat_id=call.message.chat.id,
                message_id=call.message.message_id,
                text=profile_text,
                parse_mode='HTML',
                reply_markup=keyboard
            )
        
        elif call.data == 'admin_back':
            bot.edit_message_text(
                chat_id=call.message.chat.id,
                message_id=call.message.message_id,
                text="🔧 <b>АДМИН-ПАНЕЛЬ</b>\n\nВыберите действие:",
                reply_markup=create_admin_keyboard()
            )
        
        # Обработка нового действия для бизнеса
        elif call.data == 'take_profit':
            success, profit, exp, message_text = data_manager.take_profit(user_id)
            
            if success:
                bot.answer_callback_query(call.id, message_text, show_alert=True)
                
                user_data = data_manager.get_user(user_id, username)
                business_info = format_my_business_info(user_data)
                keyboard = create_my_business_keyboard()
                
                try:
                    bot.edit_message_text(
                        chat_id=call.message.chat.id,
                        message_id=call.message.message_id,
                        text=business_info,
                        reply_markup=keyboard
                    )
                except:
                    pass
            else:
                bot.answer_callback_query(call.id, message_text, show_alert=True)
        
        elif call.data == 'back_to_menu':
            # Возврат в главное меню
            welcome = f"""
<b>👋 Добро пожаловать, {username}!</b>
━━━━━━━━━━━━━━━━━━

<b>🎮 Игровой бот с единой базой данных!</b>

<b>📋 Доступные функции:</b>
• 👤 <b>Профиль</b> - ваша статистика
• 🎁 <b>Ежедневный бонус</b> - награда раз в 24 часа
• 📊 <b>Уровни</b> - система с наградами
• 💼 <b>Работы</b> - устройтесь на работу
• 🛠️ <b>Моя работа</b> - работайте и зарабатывайте
• 🏆 <b>Лидеры</b> - топ-10 игроков
• 👥 <b>Рефералы</b> - приглашайте друзей
• 📋 <b>Задание</b> - система квестов
• 🎰 <b>Казино</b> - азартные игры на деньги
• 💸 <b>Перевести</b> - передать деньги другим игрокам
• 🏪 <b>Магазин бизнеса</b> - покупка бизнесов
• 🏢 <b>Мой бизнес</b> - управление бизнесом
"""
            
            if call.message.chat.type == 'private':
                bot.edit_message_text(
                    chat_id=call.message.chat.id,
                    message_id=call.message.message_id,
                    text=welcome,
                    reply_markup=create_main_keyboard()
                )
            else:
                bot.edit_message_text(
                    chat_id=call.message.chat.id,
                    message_id=call.message.message_id,
                    text=welcome,
                    reply_markup=create_main_inline_keyboard()
                )
        
        elif call.data == 'back_to_ref':
            user_data = data_manager.get_user(user_id, username)
            referrals_text = format_referrals_info(user_data)
            keyboard = create_referrals_keyboard()
            
            try:
                bot.edit_message_text(
                    chat_id=call.message.chat.id,
                    message_id=call.message.message_id,
                    text=referrals_text,
                    reply_markup=keyboard
                )
            except:
                bot.send_message(
                    call.message.chat.id,
                    referrals_text,
                    reply_markup=keyboard
                )
        
        # Остальные существующие callback обработчики...
        elif call.data == 'profile':
            user_data = data_manager.get_user(user_id, username)
            profile_text = format_profile(user_data)
            
            try:
                if call.message.chat.type == 'private':
                    bot.edit_message_text(
                        chat_id=call.message.chat.id,
                        message_id=call.message.message_id,
                        text=profile_text,
                        reply_markup=create_main_keyboard()
                    )
                else:
                    bot.edit_message_text(
                        chat_id=call.message.chat.id,
                        message_id=call.message.message_id,
                        text=profile_text,
                        reply_markup=create_chat_profile_keyboard()
                    )
            except:
                if call.message.chat.type == 'private':
                    bot.send_message(
                        call.message.chat.id,
                        profile_text,
                        reply_markup=create_main_keyboard()
                    )
                else:
                    bot.send_message(
                        call.message.chat.id,
                        profile_text,
                        reply_markup=create_chat_profile_keyboard()
                    )
        
        elif call.data == 'transfer_info' or call.data == 'transfer_rules':
            user_data = data_manager.get_user(user_id, username)
            
            transfer_info = f"""
💸 <b>СИСТЕМА ПЕРЕВОДОВ</b>
━━━━━━━━━━━━━━━━━━

👤 <b>Ваш баланс:</b> {format_number(user_data['balance'])}₽

<b>📋 Как использовать в чате:</b>
1. Ответьте на сообщение пользователя
2. Напишите: <code>передать сумма</code>
   или <code>дать сумма</code>

<b>📝 Примеры:</b>
• <code>передать 1000</code> - перевести 1,000₽
• <code>передать 30к</code> - перевести 30,000₽
• <code>передать 2кк</code> - перевести 2,000,000₽
• <code>передать 1.5ккк</code> - перевести 1,500,000,000₽
• <code>передать 1кккк</code> - перевести 1,000,000,000,000₽
• <code>передать все</code> - перевести весь баланс

<b>⚙️ Правила:</b>
• Минимальная сумма: 10₽
• Нельзя переводить самому себе
• Нельзя переводить ботам
• Комиссия: 0% (бесплатно)
"""
            
            try:
                bot.edit_message_text(
                    chat_id=call.message.chat.id,
                    message_id=call.message.message_id,
                    text=transfer_info,
                    parse_mode="HTML",
                    disable_web_page_preview=True,
                    reply_markup=create_transfer_keyboard()
                )
            except:
                bot.send_message(
                    call.message.chat.id,
                    transfer_info,
                    parse_mode="HTML",
                    disable_web_page_preview=True,
                    reply_markup=create_transfer_keyboard()
                )
        
        elif call.data == 'casino_info' or call.data == 'casino_rules':
            user_data = data_manager.get_user(user_id, username)
            coins = user_data['balance']
            
            casino_info = f"""
🎰 {username}, <b>ИГРА КАЗИНО</b>

💰 <b>Коэффициенты:</b>
• 10% шанс выиграть <b>x5</b> (ДЖЕКПОТ)
• 30% шанс выиграть <b>x0.3 - x2.0</b>
• 30% шанс проиграть <b>x0.1 - x0.8</b>
• 20% шанс остаться при своих
• 10% шанс потерять всю ставку

🎰 <b>Используйте в чате:</b>

⬜ <code>каз 100</code> <b>- конкретная ставка</b>

⬜ <code>каз 30к</code> <b>- 30,000₽</b>

⬜ <code>каз 3кк</code> <b>- 3,000,000₽</b>

⬜ <code>каз 2ккк</code> <b>- 2,000,000,000₽</b>

⬜ <code>каз 1кккк</code> <b>- 1,000,000,000,000₽</b>

⬜ <code>каз всё</code> или <code>каз все</code> <b>- поставить весь баланс</b>

💰 <b>Ваш баланс:</b> {format_number(coins)} <b>₽</b>
💡 Примеры: <code>каз 50</code>, <code>каз 1к</code>, <code>каз 2.5к</code>, <code>каз 1.5кк</code>, <code>каз 2.5ккк</code>
"""
            
            try:
                bot.edit_message_text(
                    chat_id=call.message.chat.id,
                    message_id=call.message.message_id,
                    text=casino_info,
                    parse_mode="HTML",
                    disable_web_page_preview=True,
                    reply_markup=create_casino_keyboard()
                )
            except:
                bot.send_message(
                    call.message.chat.id,
                    casino_info,
                    parse_mode="HTML",
                    disable_web_page_preview=True,
                    reply_markup=create_casino_keyboard()
                )
        
        elif call.data == 'casino_play':
            user_data = data_manager.get_user(user_id, username)
            coins = user_data['balance']
            
            casino_play = f"""
🎰 {username}, <b>НАЧАТЬ ИГРУ В КАЗИНО</b>

💰 <b>Ваш баланс:</b> {format_number(coins)} <b>₽</b>

<b>Для игры напишите в чат:</b>
<code>каз [ставка]</code>

<b>Примеры:</b>
• <code>каз 100</code> - ставка 100₽
• <code>каз 1к</code> - ставка 1,000₽
• <code>каз 2.5к</code> - ставка 2,500₽
• <code>каз 1.5кк</code> - ставка 1,500,000₽
• <code>каз 2ккк</code> - ставка 2,000,000,000₽
• <code>каз всё</code> - весь баланс

🎰 <b>Минимальная ставка:</b> 10₽
"""
            
            try:
                bot.edit_message_text(
                    chat_id=call.message.chat.id,
                    message_id=call.message.message_id,
                    text=casino_play,
                    parse_mode="HTML",
                    disable_web_page_preview=True,
                    reply_markup=create_casino_keyboard()
                )
            except:
                bot.send_message(
                    call.message.chat.id,
                    casino_play,
                    parse_mode="HTML",
                    disable_web_page_preview=True,
                    reply_markup=create_casino_keyboard()
                )
        
        elif call.data == 'bonus_chat':
            user_data = data_manager.get_user(user_id, username)
            bonus_available, time_left = data_manager.check_bonus(user_data)
            
            if not bonus_available:
                bot.answer_callback_query(call.id, f"⏳ Бонус еще не доступен! Доступ через: {time_left}", show_alert=True)
                return
            
            bonus_amount = random.randint(1000, 23000)
            exp_amount = random.randint(40, 400)
            
            new_balance = user_data['balance'] + bonus_amount
            
            levels_gained, new_level = data_manager.add_experience(user_id, exp_amount)
            
            data_manager.update_user(user_id, {
                'balance': new_balance,
                'last_bonus': datetime.now().strftime('%Y-%m-%d %H:%M:%S')
            })
            
            user_id_str = str(user_id)
            user_data = data_manager.get_user(user_id)
            user_data['daily_stats']['bonus_count'] += 1
            user_data['weekly_stats']['bonus_count'] += 1
            user_data['daily_stats']['money_earned'] += bonus_amount
            user_data['weekly_stats']['money_earned'] += bonus_amount
            
            data_manager.update_user(user_id, {
                'daily_stats': user_data['daily_stats'],
                'weekly_stats': user_data['weekly_stats']
            })
            
            quest_system.update_quest_progress(user_id, 'daily', 'get_bonus')
            quest_system.update_quest_progress(user_id, 'weekly', 'get_bonus')
            quest_system.update_quest_progress(user_id, 'daily', 'earn_money', bonus_amount)
            quest_system.update_quest_progress(user_id, 'weekly', 'earn_money', bonus_amount)
            
            bonus_text = f"""
<b>👻 {username}, вы получили ежедневный бонус 🎁</b>

<b>🎉 Вы получили {bonus_amount:,}₽ 🎉</b>

<b>🌟 Опыт:</b> +{exp_amount}
<b>💰 Новый баланс:</b> {new_balance:,}₽
"""
            
            if levels_gained > 0:
                bonus_text += f"\n<b>🎉 Вы достигли {new_level} уровня!</b>"
            
            try:
                bot.edit_message_text(
                    chat_id=call.message.chat.id,
                    message_id=call.message.message_id,
                    text=bonus_text,
                    reply_markup=create_main_inline_keyboard() if call.message.chat.type != 'private' else create_main_keyboard()
                )
            except:
                bot.send_message(
                    call.message.chat.id,
                    bonus_text,
                    reply_markup=create_main_inline_keyboard() if call.message.chat.type != 'private' else create_main_keyboard()
                )
        
        elif call.data == 'leaders':
            leaders, update_time = data_manager.get_leaderboard()
            
            if not leaders:
                bot.answer_callback_query(call.id, "📊 Лидеров пока нет!", show_alert=True)
                return
            
            leaderboard_text = "🏆 <b>ТОП-10 ИГРОКОВ ПО БАЛАНСУ</b>\n\n"
            
            place_emojis = {
                1: "🥇", 2: "🥈", 3: "🥉",
                4: "4️⃣", 5: "5️⃣", 6: "6️⃣",
                7: "7️⃣", 8: "8️⃣", 9: "9️⃣", 10: "🔟"
            }
            
            for i, user in enumerate(leaders, 1):
                emoji = place_emojis.get(i, f"{i}.")
                username = user.get('username', 'Игрок')
                balance = user.get('balance', 0)
                
                if balance >= 1_000_000_000_000:
                    formatted_balance = f"₽{balance / 1_000_000_000_000:.1f} трлн"
                elif balance >= 1_000_000_000:
                    formatted_balance = f"₽{balance / 1_000_000_000:.1f} млрд"
                elif balance >= 1_000_000:
                    formatted_balance = f"₽{balance / 1_000_000:.1f} млн"
                elif balance >= 1_000:
                    formatted_balance = f"₽{balance / 1_000:.1f}к"
                else:
                    formatted_balance = f"₽{balance:,}"
                
                leaderboard_text += f"{emoji} <b>{username}</b> — {formatted_balance}\n"
            
            keyboard = create_leaders_keyboard()
            
            try:
                bot.edit_message_text(
                    chat_id=call.message.chat.id,
                    message_id=call.message.message_id,
                    text=leaderboard_text,
                    reply_markup=keyboard
                )
            except:
                bot.send_message(
                    call.message.chat.id,
                    leaderboard_text,
                    reply_markup=keyboard
                )
        
        elif call.data == 'levels_leaders':
            leaders, update_time = data_manager.get_levels_leaderboard()
            
            if not leaders:
                bot.answer_callback_query(call.id, "📊 Лидеров по уровням пока нет!", show_alert=True)
                return
            
            leaderboard_text = "⭐ <b>СПИСОК ЛИДЕРОВ ПО УРОВНЯМ:</b>\n\n"
            
            place_emojis = {
                1: "🥇", 2: "🥈", 3: "🥉",
                4: "4️⃣", 5: "5️⃣", 6: "6️⃣",
                7: "7️⃣", 8: "8️⃣", 9: "9️⃣", 10: "🔟"
            }
            
            for i, user in enumerate(leaders, 1):
                emoji = place_emojis.get(i, f"{i}.")
                username = user.get('username', 'Игрок')
                level = user.get('level', 1)
                
                if i == 1:
                    leaderboard_text += f"{emoji} <b>{username}</b> — {level}\n"
                else:
                    leaderboard_text += f"⠀{emoji} <b>{username}</b> — {level}\n"
            
            keyboard = create_levels_leaders_keyboard()
            
            try:
                bot.edit_message_text(
                    chat_id=call.message.chat.id,
                    message_id=call.message.message_id,
                    text=leaderboard_text,
                    reply_markup=keyboard
                )
            except:
                bot.send_message(
                    call.message.chat.id,
                    leaderboard_text,
                    reply_markup=keyboard
                )
        
        elif call.data == 'refresh_leaders':
            data_manager.update_leaderboard_cache()
            
            leaders, update_time = data_manager.get_leaderboard()
            leaderboard_text = "🏆 <b>ТОП-10 ИГРОКОВ ПО БАЛАНСУ</b>\n\n"
            
            place_emojis = {
                1: "🥇", 2: "🥈", 3: "🥉",
                4: "4️⃣", 5: "5️⃣", 6: "6️⃣",
                7: "7️⃣", 8: "8️⃣", 9: "9️⃣", 10: "🔟"
            }
            
            for i, user in enumerate(leaders, 1):
                emoji = place_emojis.get(i, f"{i}.")
                username = user.get('username', 'Игрок')
                balance = user.get('balance', 0)
                
                if balance >= 1_000_000_000_000:
                    formatted_balance = f"₽{balance / 1_000_000_000_000:.1f} трлн"
                elif balance >= 1_000_000_000:
                    formatted_balance = f"₽{balance / 1_000_000_000:.1f} млрд"
                elif balance >= 1_000_000:
                    formatted_balance = f"₽{balance / 1_000_000:.1f} млн"
                elif balance >= 1_000:
                    formatted_balance = f"₽{balance / 1_000:.1f}к"
                else:
                    formatted_balance = f"₽{balance:,}"
                
                leaderboard_text += f"{emoji} <b>{username}</b> — {formatted_balance}\n"
            
            keyboard = create_leaders_keyboard()
            
            try:
                bot.edit_message_text(
                    chat_id=call.message.chat.id,
                    message_id=call.message.message_id,
                    text=leaderboard_text,
                    reply_markup=keyboard
                )
                bot.answer_callback_query(call.id, "✅ Лидерборд обновлен!")
            except:
                bot.answer_callback_query(call.id, "❌ Ошибка обновления!")
        
        elif call.data == 'refresh_ref':
            user_data = data_manager.get_user(user_id, username)
            referrals_text = format_referrals_info(user_data)
            keyboard = create_referrals_keyboard()
            
            try:
                bot.edit_message_text(
                    chat_id=call.message.chat.id,
                    message_id=call.message.message_id,
                    text=referrals_text,
                    reply_markup=keyboard
                )
                bot.answer_callback_query(call.id, "✅ Реферальная информация обновлена!")
            except:
                bot.answer_callback_query(call.id, "❌ Ошибка обновления!")
        
        elif call.data == 'refresh_my_ref':
            user_data = data_manager.get_user(user_id, username)
            referrals_data = user_data.get('referrals_data', [])
            referrals_count = user_data.get('referral_count', 0)
            
            if referrals_count == 0:
                referrals_text = "📭 <b>У вас пока нет рефералов</b>\n\nПриглашайте друзей по реферальной ссылке и получайте бонусы!"
            else:
                referrals_text = f"""
<b>📋 СПИСОК ВАШИХ РЕФЕРАЛОВ</b>
━━━━━━━━━━━━━━━━━━

<b>👤 Владелец:</b> {username}
<b>👥 Всего рефералов:</b> {referrals_count}
<b>💰 Заработано:</b> {user_data.get('referral_earnings', 0):,}₽

<b>📊 Последние рефералы:</b>
━━━━━━━━━━━━━━━━━━
"""
                
                for i, ref in enumerate(referrals_data[-10:], 1):
                    referrals_text += f"\n<b>{i}. {ref.get('username', 'Игрок')}</b>\n"
                    referrals_text += f"   📅 Дата: {ref.get('date', 'Неизвестно')}\n"
                    referrals_text += f"   🎁 Бонус: {ref.get('bonus_received', 0):,}₽\n"
                    referrals_text += f"   ─────────────────────"
                
                if referrals_count > 10:
                    referrals_text += f"\n\n<b>... и еще {referrals_count - 10} рефералов</b>"
            
            keyboard = create_my_referrals_keyboard()
            
            try:
                bot.edit_message_text(
                    chat_id=call.message.chat.id,
                    message_id=call.message.message_id,
                    text=referrals_text,
                    reply_markup=keyboard
                )
                bot.answer_callback_query(call.id, "✅ Список рефералов обновлен!")
            except:
                bot.answer_callback_query(call.id, "❌ Ошибка обновления!")
        
        elif call.data == 'my_referrals':
            user_data = data_manager.get_user(user_id, username)
            referrals_data = user_data.get('referrals_data', [])
            referrals_count = user_data.get('referral_count', 0)
            
            if referrals_count == 0:
                referrals_text = "📭 <b>У вас пока нет рефералов</b>\n\nПриглашайте друзей по реферальной ссылке и получайте бонусы!"
            else:
                referrals_text = f"""
<b>📋 СПИСОК ВАШИХ РЕФЕРАЛОВ</b>
━━━━━━━━━━━━━━━━━━

<b>👤 Владелец:</b> {username}
<b>👥 Всего рефералов:</b> {referrals_count}
<b>💰 Заработано:</b> {user_data.get('referral_earnings', 0):,}₽

<b>📊 Последние рефералы:</b>
━━━━━━━━━━━━━━━━━━
"""
                
                for i, ref in enumerate(referrals_data[-10:], 1):
                    referrals_text += f"\n<b>{i}. {ref.get('username', 'Игрок')}</b>\n"
                    referrals_text += f"   📅 Дата: {ref.get('date', 'Неизвестно')}\n"
                    referrals_text += f"   🎁 Бонус: {ref.get('bonus_received', 0):,}₽\n"
                    referrals_text += f"   ─────────────────────"
                
                if referrals_count > 10:
                    referrals_text += f"\n\n<b>... и еще {referrals_count - 10} рефералов</b>"
            
            keyboard = create_my_referrals_keyboard()
            
            try:
                bot.edit_message_text(
                    chat_id=call.message.chat.id,
                    message_id=call.message.message_id,
                    text=referrals_text,
                    reply_markup=keyboard
                )
            except:
                bot.send_message(
                    call.message.chat.id,
                    referrals_text,
                    reply_markup=keyboard
                )
        
        elif call.data.startswith('levels_'):
            page = int(call.data.split('_')[1])
            user_data = data_manager.get_user(user_id, username)
            levels_text = format_levels_page(user_data, page)
            keyboard = create_levels_keyboard(page)
            
            try:
                bot.edit_message_text(
                    chat_id=call.message.chat.id,
                    message_id=call.message.message_id,
                    text=levels_text,
                    reply_markup=keyboard
                )
            except:
                pass
        
        elif call.data == 'claim_rewards':
            success, total_reward, unclaimed = data_manager.claim_level_rewards(user_id)
            
            if not success:
                bot.answer_callback_query(
                    call.id,
                    "❌ У вас нет неполученных наград за уровни!",
                    show_alert=True
                )
                return
            
            user_data = data_manager.get_user(user_id, username)
            
            reward_text = f"""
<b>🎉 ВЫ ПОЛУЧИЛИ НАГРАДЫ ЗА УРОВНИ!</b>
━━━━━━━━━━━━━━━━━━

<b>👤 Игрок:</b> {username}
<b>💰 Получено:</b> {total_reward:,}₽
<b>🏦 Новый баланс:</b> {user_data['balance']:,}₽

<b>📊 Полученные награды:</b>
"""
            
            for level, reward in unclaimed:
                reward_text += f"• Уровень {level}: {reward:,}₽\n"
            
            reward_text += "\n<b>🎮 Продолжайте играть и повышать уровень!</b>"
            
            try:
                bot.edit_message_text(
                    chat_id=call.message.chat.id,
                    message_id=call.message.message_id,
                    text=reward_text,
                    reply_markup=types.InlineKeyboardMarkup().add(
                        types.InlineKeyboardButton('📊 Уровни', callback_data='levels_1'),
                        types.InlineKeyboardButton('◀️ Назад', callback_data='back_to_menu')
                    )
                )
            except:
                bot.send_message(
                    call.message.chat.id,
                    reward_text,
                    reply_markup=create_main_keyboard() if call.message.chat.type == 'private' else create_main_inline_keyboard()
                )
            
            bot.answer_callback_query(call.id, f"✅ Получено {total_reward:,}₽!")
        
        elif call.data.startswith('job_info_'):
            job_id = int(call.data.split('_')[2])
            user_data = data_manager.get_user(user_id, username)
            info_text = format_job_info(job_id, user_data['level'], user_data)
            
            keyboard = types.InlineKeyboardMarkup()
            
            if user_data['level'] >= JOBS[job_id]['min_level']:
                if user_data['current_job'] == job_id:
                    keyboard.add(types.InlineKeyboardButton('✅ Вы работаете здесь', callback_data='current_job'))
                else:
                    keyboard.add(types.InlineKeyboardButton('✅ Устроиться', callback_data=f'hire_{job_id}'))
            else:
                keyboard.add(types.InlineKeyboardButton(f'❌ Нужен {JOBS[job_id]["min_level"]} уровень', callback_data='need_level'))
            
            keyboard.add(
                types.InlineKeyboardButton('◀️ Назад к списку', callback_data='jobs_list'),
                types.InlineKeyboardButton('◀️ Назад', callback_data='back_to_menu')
            )
            
            try:
                bot.edit_message_text(
                    chat_id=call.message.chat.id,
                    message_id=call.message.message_id,
                    text=info_text,
                    reply_markup=keyboard
                )
            except:
                pass
        
        elif call.data.startswith('hire_'):
            job_id = int(call.data.split('_')[1])
            success, message_text = data_manager.hire_user(user_id, job_id)
            
            if success:
                job = JOBS.get(job_id)
                hire_text = f"""
<b>🎉 ПОЗДРАВЛЯЕМ С УСТРОЙСТВОМ НА РАБОТУ!</b>
━━━━━━━━━━━━━━━━━━

<b>💼 Профессия:</b> {job['name']}
<b>💰 Зарплата:</b> {job['salary']:,}₽
<b>🌟 Опыт за работу:</b> {job['experience']}
<b>⏰ Время работы:</b> {job['cooldown']//60:02d}:{job['cooldown']%60:02d}
"""
                
                try:
                    bot.edit_message_text(
                        chat_id=call.message.chat.id,
                        message_id=call.message.message_id,
                        text=hire_text
                    )
                except:
                    pass
                
                bot.send_message(
                    call.message.chat.id,
                    f"🎊 <b>Добро пожаловать на работу, {username}!</b>\n\nПерейдите в 🛠️ <b>Моя работа</b> чтобы начать работать!",
                    reply_markup=create_main_keyboard() if call.message.chat.type == 'private' else create_main_inline_keyboard()
                )
            else:
                bot.answer_callback_query(call.id, f"❌ {message_text}", show_alert=True)
        
        elif call.data == 'jobs_list':
            keyboard = create_jobs_keyboard()
            try:
                bot.edit_message_text(
                    chat_id=call.message.chat.id,
                    message_id=call.message.message_id,
                    text="<b>💼 Список всех работ</b>\n\nВыберите работу для подробной информации:",
                    reply_markup=keyboard
                )
            except:
                pass
        
        elif call.data == 'start_work':
            user_data = data_manager.get_user(user_id, username)
            can_work, message_text, job = data_manager.can_work(user_data)
            
            if not can_work:
                bot.answer_callback_query(call.id, f"❌ {message_text}", show_alert=True)
                return
            
            if not job:
                bot.answer_callback_query(call.id, "❌ Работа не найдена!", show_alert=True)
                return
            
            if 'questions' in job and job['questions']:
                question_data = random.choice(job['questions'])
                question = question_data['question']
                answers = question_data['answers']
                correct_answer = question_data['correct']
                
                active_questions[user_id] = {
                    'job_id': user_data['current_job'],
                    'question': question,
                    'answers': answers,
                    'correct': correct_answer,
                    'message_id': call.message.message_id,
                    'time': datetime.now()
                }
                
                question_text = f"""
<b>❓ ВОПРОС ПО РАБОТЕ: {job['name']}</b>
━━━━━━━━━━━━━━━━━━

<b>Вопрос:</b> {question}

<b>💰 Награда за правильный ответ:</b>
• Зарплата: {job['salary']:,}₽
• Опыт: {job['experience']}

<b>⏱️ Выберите правильный вариант:</b>
"""
                
                keyboard = create_question_keyboard(answers)
                try:
                    bot.edit_message_text(
                        chat_id=call.message.chat.id,
                        message_id=call.message.message_id,
                        text=question_text,
                        reply_markup=keyboard
                    )
                except:
                    bot.send_message(
                        call.message.chat.id,
                        question_text,
                        reply_markup=keyboard
                    )
            else:
                bot.answer_callback_query(call.id, "❌ Для этой работы нет вопросов!", show_alert=True)
        
        elif call.data.startswith('answer_'):
            answer_index = int(call.data.split('_')[1])
            
            if user_id not in active_questions:
                bot.answer_callback_query(call.id, "❌ Время ответа истекло!", show_alert=True)
                return
            
            active_question = active_questions[user_id]
            
            time_passed = datetime.now() - active_question['time']
            if time_passed.total_seconds() > 300:
                del active_questions[user_id]
                bot.answer_callback_query(call.id, "❌ Время на ответ истекло!", show_alert=True)
                return
            
            is_correct = (answer_index == active_question['correct'])
            
            user_data = data_manager.get_user(user_id, username)
            job = JOBS.get(active_question['job_id'])
            
            if not job:
                bot.answer_callback_query(call.id, "❌ Работа не найдена!", show_alert=True)
                return
            
            success, salary, exp_reward, levels_gained, new_level = data_manager.complete_work(user_id, is_correct)
            
            if is_correct:
                result_text = f"""
<b>✅ ПРАВИЛЬНЫЙ ОТВЕТ!</b>
━━━━━━━━━━━━━━━━━━

<b>🎉 Вы получили:</b>
• 💰 Зарплата: {salary:,}₽
• 🌟 Опыт: +{exp_reward}

<b>📝 Ваш ответ:</b> {active_question['answers'][answer_index]}
<b>✅ Правильный ответ:</b> {active_question['answers'][active_question['correct']]}

<b>⏰ Следующая работа через:</b> {job['cooldown']//60:02d}:{job['cooldown']%60:02d}
"""
            else:
                result_text = f"""
<b>❌ НЕПРАВИЛЬНЫЙ ОТВЕТ!</b>
━━━━━━━━━━━━━━━━━━

<b>📝 Ваш ответ:</b> {active_question['answers'][answer_index]}
<b>✅ Правильный ответ:</b> {active_question['answers'][active_question['correct']]}

<b>💡 В следующий раз будьте внимательнее!</b>

<b>⏰ Следующая работа через:</b> {job['cooldown']//60:02d}:{job['cooldown']%60:02d}
"""
            
            if is_correct and levels_gained > 0:
                result_text += f"\n<b>🏆 УРОВЕНЬ ПОВЫШЕН!</b>\nНовый уровень: {new_level}"
            
            user_data = data_manager.get_user(user_id, username)
            
            if is_correct:
                result_text += f"\n<b>💰 Ваш баланс:</b> {user_data['balance']:,}₽"
            
            del active_questions[user_id]
            
            keyboard = create_work_keyboard()
            
            try:
                bot.edit_message_text(
                    chat_id=call.message.chat.id,
                    message_id=call.message.message_id,
                    text=result_text,
                    reply_markup=keyboard
                )
            except:
                bot.send_message(
                    call.message.chat.id,
                    result_text,
                    reply_markup=keyboard
                )
        
        elif call.data == 'fire_confirm':
            user_data = data_manager.get_user(user_id, username)
            job_name = JOBS.get(user_data['current_job'], {}).get('name', 'Работа')
            
            confirm_text = f"""
<b>⚠️ ПОДТВЕРЖДЕНИЕ УВОЛЬНЕНИЯ</b>
━━━━━━━━━━━━━━━━━━

Вы уверены, что хотите уволиться с работы <b>{job_name}</b>?
"""
            
            keyboard = types.InlineKeyboardMarkup(row_width=2)
            keyboard.add(
                types.InlineKeyboardButton('✅ Да, уволиться', callback_data='fire_yes'),
                types.InlineKeyboardButton('❌ Нет, остаться', callback_data='fire_no')
            )
            
            try:
                bot.edit_message_text(
                    chat_id=call.message.chat.id,
                    message_id=call.message.message_id,
                    text=confirm_text,
                    reply_markup=keyboard
                )
            except:
                pass
        
        elif call.data == 'fire_yes':
            success, message_text = data_manager.fire_user(user_id)
            
            if success:
                fire_text = f"""
<b>📭 ВЫ УВОЛИЛИСЬ!</b>
━━━━━━━━━━━━━━━━━━

{message_text}
"""
                
                try:
                    bot.edit_message_text(
                        chat_id=call.message.chat.id,
                        message_id=call.message.message_id,
                        text=fire_text,
                        reply_markup=types.InlineKeyboardMarkup().add(
                            types.InlineKeyboardButton('💼 Найти новую работу', callback_data='jobs_list'),
                            types.InlineKeyboardButton('◀️ Назад', callback_data='back_to_menu')
                        )
                    )
                except:
                    bot.send_message(
                        call.message.chat.id,
                        fire_text,
                        reply_markup=create_main_inline_keyboard() if call.message.chat.type != 'private' else create_main_keyboard()
                    )
            else:
                bot.answer_callback_query(call.id, f"❌ {message_text}", show_alert=True)
        
        elif call.data == 'fire_no':
            try:
                bot.delete_message(call.message.chat.id, call.message.message_id)
            except:
                pass
        
        elif call.data == 'quests_menu':
            quests_text = """
<b>📋 СИСТЕМА КВЕСТОВ</b>
━━━━━━━━━━━━━━━━━━

<b>🎯 Ежедневные квесты:</b>
• Меняются каждый день в 00:00
• 3 случайных квеста ежедневно
• Награды: 2,000₽ - 60,000₽ и 100 - 3,000 опыта

<b>📆 Недельные квесты:</b>
• Меняются каждый понедельник в 00:00
• 2 случайных квеста еженедельно
• Награды: 50,000₽ - 200,000₽ и 1,000 - 12,000 опыта
"""
            
            keyboard = create_quests_menu_keyboard()
            try:
                bot.edit_message_text(
                    chat_id=call.message.chat.id,
                    message_id=call.message.message_id,
                    text=quests_text,
                    reply_markup=keyboard
                )
            except:
                pass
        
        elif call.data == 'quests_daily':
            quests = quest_system.get_user_quests(user_id, 'daily')
            quests_text = "<b>📅 ЕЖЕДНЕВНЫЕ КВЕСТЫ</b>\n\n"
            
            for i, quest in enumerate(quests, 1):
                if quest['state'] == 'available':
                    status = "🔓 Доступен"
                elif quest['state'] == 'active':
                    status = "🟢 В процессе"
                else:
                    status = "✅ Завершен"
                
                quests_text += f"{i}. <b>{quest['title']}</b> - {status}\n"
                quests_text += f"   Прогресс: {quest['progress']}/{quest['target']}\n"
                quests_text += f"   Награда: {quest['reward_money']:,}₽ + {quest['reward_exp']} опыта\n\n"
            
            quests_text += "<b>💡 Выберите квест для просмотра деталей:</b>"
            
            keyboard = create_quests_list_keyboard(quests, 'daily')
            try:
                bot.edit_message_text(
                    chat_id=call.message.chat.id,
                    message_id=call.message.message_id,
                    text=quests_text,
                    reply_markup=keyboard
                )
            except:
                pass
        
        elif call.data == 'quests_weekly':
            quests = quest_system.get_user_quests(user_id, 'weekly')
            quests_text = "<b>📆 НЕДЕЛЬНЫЕ КВЕСТЫ</b>\n\n"
            
            for i, quest in enumerate(quests, 1):
                if quest['state'] == 'available':
                    status = "🔓 Доступен"
                elif quest['state'] == 'active':
                    status = "🟢 В процессе"
                else:
                    status = "✅ Завершен"
                
                quests_text += f"{i}. <b>{quest['title']}</b> - {status}\n"
                quests_text += f"   Прогресс: {quest['progress']}/{quest['target']}\n"
                quests_text += f"   Награда: {quest['reward_money']:,}₽ + {quest['reward_exp']} опыта\n\n"
            
            quests_text += "<b>💡 Выберите квест для просмотра деталей:</b>"
            
            keyboard = create_quests_list_keyboard(quests, 'weekly')
            try:
                bot.edit_message_text(
                    chat_id=call.message.chat.id,
                    message_id=call.message.message_id,
                    text=quests_text,
                    reply_markup=keyboard
                )
            except:
                pass
        
        elif call.data.startswith('quest_detail_'):
            parts = call.data.split('_')
            quest_type = parts[2]
            quest_id = '_'.join(parts[3:])
            
            quests = quest_system.get_user_quests(user_id, quest_type)
            quest = None
            
            for q in quests:
                if q['id'] == quest_id:
                    quest = q
                    break
            
            if not quest:
                bot.answer_callback_query(call.id, "❌ Квест не найден!", show_alert=True)
                return
            
            quest_info = quest_system.get_quest_info(quest)
            keyboard = create_quest_detail_keyboard(quest, quest_type)
            
            try:
                bot.edit_message_text(
                    chat_id=call.message.chat.id,
                    message_id=call.message.message_id,
                    text=quest_info,
                    reply_markup=keyboard
                )
            except:
                pass
        
        elif call.data.startswith('quest_start_'):
            parts = call.data.split('_')
            quest_type = parts[2]
            quest_id = '_'.join(parts[3:])
            
            success, message_text = quest_system.start_quest(user_id, quest_id, quest_type)
            
            if success:
                bot.answer_callback_query(call.id, "✅ Квест начат!", show_alert=True)
                
                quests = quest_system.get_user_quests(user_id, quest_type)
                quest = None
                
                for q in quests:
                    if q['id'] == quest_id:
                        quest = q
                        break
                
                if quest:
                    quest_info = quest_system.get_quest_info(quest)
                    keyboard = create_quest_detail_keyboard(quest, quest_type)
                    
                    try:
                        bot.edit_message_text(
                            chat_id=call.message.chat.id,
                            message_id=call.message.message_id,
                            text=quest_info,
                            reply_markup=keyboard
                        )
                    except:
                        pass
            else:
                bot.answer_callback_query(call.id, f"❌ {message_text}", show_alert=True)
        
        elif call.data.startswith('quest_complete_'):
            parts = call.data.split('_')
            quest_type = parts[2]
            quest_id = '_'.join(parts[3:])
            
            success, reward_money, reward_exp, message_text = quest_system.complete_quest(user_id, quest_id, quest_type)
            
            if success:
                user_data = data_manager.get_user(user_id, username)
                new_balance = user_data['balance'] + reward_money
                levels_gained, new_level = data_manager.add_experience(user_id, reward_exp)
                
                data_manager.update_user(user_id, {
                    'balance': new_balance
                })
                
                user_id_str = str(user_id)
                user_data = data_manager.get_user(user_id)
                user_data['daily_stats']['money_earned'] += reward_money
                user_data['weekly_stats']['money_earned'] += reward_money
                
                data_manager.update_user(user_id, {
                    'daily_stats': user_data['daily_stats'],
                    'weekly_stats': user_data['weekly_stats']
                })
                
                quest_system.update_quest_progress(user_id, 'daily', 'earn_money', reward_money)
                quest_system.update_quest_progress(user_id, 'weekly', 'earn_money', reward_money)
                
                reward_text = f"""
<b>🎉 ВЫ ВЫПОЛНИЛИ КВЕСТ!</b>
━━━━━━━━━━━━━━━━━━

<b>👤 Игрок:</b> {username}
<b>💰 Получено:</b> {reward_money:,}₽
<b>🌟 Опыт:</b> +{reward_exp}
<b>🏦 Новый баланс:</b> {new_balance:,}₽
"""
                
                if levels_gained > 0:
                    reward_text += f"\n<b>🏆 УРОВЕНЬ ПОВЫШЕН!</b>\nНовый уровень: {new_level}"
                
                bot.answer_callback_query(call.id, f"✅ Получено {reward_money:,}₽ и {reward_exp} опыта!", show_alert=False)
                
                quests = quest_system.get_user_quests(user_id, quest_type)
                quest = None
                
                for q in quests:
                    if q['id'] == quest_id:
                        quest = q
                        break
                
                if quest:
                    quest_info = quest_system.get_quest_info(quest)
                    keyboard = create_quest_detail_keyboard(quest, quest_type)
                    
                    try:
                        bot.edit_message_text(
                            chat_id=call.message.chat.id,
                            message_id=call.message.message_id,
                            text=quest_info,
                            reply_markup=keyboard
                        )
                    except:
                        pass
                
                bot.send_message(
                    call.message.chat.id,
                    reward_text,
                    reply_markup=create_main_keyboard() if call.message.chat.type == 'private' else create_main_inline_keyboard()
                )
            else:
                bot.answer_callback_query(call.id, f"❌ {message_text}", show_alert=True)
        
        elif call.data.startswith('quest_cancel_'):
            parts = call.data.split('_')
            quest_type = parts[2]
            quest_id = '_'.join(parts[3:])
            
            success, message_text = quest_system.cancel_quest(user_id, quest_id, quest_type)
            
            if success:
                bot.answer_callback_query(call.id, "✅ Квест отменен, прогресс сохранен!", show_alert=True)
                
                quests = quest_system.get_user_quests(user_id, quest_type)
                quest = None
                
                for q in quests:
                    if q['id'] == quest_id:
                        quest = q
                        break
                
                if quest:
                    quest_info = quest_system.get_quest_info(quest)
                    keyboard = create_quest_detail_keyboard(quest, quest_type)
                    
                    try:
                        bot.edit_message_text(
                            chat_id=call.message.chat.id,
                            message_id=call.message.message_id,
                            text=quest_info,
                            reply_markup=keyboard
                        )
                    except:
                        pass
            else:
                bot.answer_callback_query(call.id, f"❌ {message_text}", show_alert=True)
        
        # ОБРАБОТЧИКИ ДЛЯ БИЗНЕСА
        elif call.data.startswith('business_shop_'):
            business_id = int(call.data.split('_')[2])
            user_data = data_manager.get_user(user_id, username)
            
            business_info = format_business_info(business_id, user_data)
            keyboard = create_business_shop_keyboard(business_id, user_data.get('business_id') is not None)
            
            try:
                bot.edit_message_text(
                    chat_id=call.message.chat.id,
                    message_id=call.message.message_id,
                    text=business_info,
                    reply_markup=keyboard
                )
            except:
                pass
        
        elif call.data.startswith('buy_business_'):
            business_id = int(call.data.split('_')[2])
            
            success, message_text = data_manager.buy_business(user_id, business_id)
            
            if success:
                bot.answer_callback_query(call.id, message_text, show_alert=True)
                
                user_data = data_manager.get_user(user_id, username)
                business_info = format_business_info(business_id, user_data)
                keyboard = create_business_shop_keyboard(business_id, user_data.get('business_id') is not None)
                
                try:
                    bot.edit_message_text(
                        chat_id=call.message.chat.id,
                        message_id=call.message.message_id,
                        text=business_info,
                        reply_markup=keyboard
                    )
                except:
                    pass
            else:
                bot.answer_callback_query(call.id, message_text, show_alert=True)
        
        elif call.data == 'my_business':
            data_manager.update_business_progress(user_id)
            
            user_data = data_manager.get_user(user_id, username)
            business_info = format_my_business_info(user_data)
            
            if "нету бизнеса" in business_info:
                keyboard = types.InlineKeyboardMarkup()
                keyboard.add(types.InlineKeyboardButton('🏪 Магазин бизнеса', callback_data='business_shop_1'))
                keyboard.add(types.InlineKeyboardButton('◀️ Назад', callback_data='back_to_menu'))
            else:
                keyboard = create_my_business_keyboard()
            
            try:
                bot.edit_message_text(
                    chat_id=call.message.chat.id,
                    message_id=call.message.message_id,
                    text=business_info,
                    reply_markup=keyboard
                )
            except:
                pass
        
        elif call.data == 'pay_taxes':
            success, message_text = data_manager.pay_taxes(user_id)
            
            if success:
                bot.answer_callback_query(call.id, message_text, show_alert=True)
                
                user_data = data_manager.get_user(user_id, username)
                business_info = format_my_business_info(user_data)
                keyboard = create_my_business_keyboard()
                
                try:
                    bot.edit_message_text(
                        chat_id=call.message.chat.id,
                        message_id=call.message.message_id,
                        text=business_info,
                        reply_markup=keyboard
                    )
                except:
                    pass
            else:
                bot.answer_callback_query(call.id, message_text, show_alert=True)
        
        elif call.data == 'sell_business_confirm':
            user_data = data_manager.get_user(user_id, username)
            
            if not user_data.get('business_id'):
                bot.answer_callback_query(call.id, "❌ У вас нет бизнеса!", show_alert=True)
                return
            
            # Проверяем, не пытается ли пользователь продать уникальный бизнес
            if user_data['business_id'] == 8:
                bot.answer_callback_query(call.id, "❌ Этот бизнес нельзя продать!", show_alert=True)
                return
            
            business = BUSINESSES.get(user_data['business_id'])
            sell_price = int(business['cost'] * 0.25)
            
            confirm_text = f"""
⚠️ <b>ПОДТВЕРЖДЕНИЕ ПРОДАЖИ</b>
━━━━━━━━━━━━━━━━━━

Вы уверены, что хотите продать бизнес <b>{business['name']}</b>?

💰 <b>Вы получите:</b> {format_number(sell_price)}₽
📉 <b>Это 25% от первоначальной стоимости</b>

<b>Все накопленные прибыль и опыт будут потеряны!</b>
"""
            
            keyboard = create_sell_confirm_keyboard()
            
            try:
                bot.edit_message_text(
                    chat_id=call.message.chat.id,
                    message_id=call.message.message_id,
                    text=confirm_text,
                    reply_markup=keyboard
                )
            except:
                pass
        
        elif call.data == 'sell_business_yes':
            success, message_text = data_manager.sell_business(user_id)
            
            if success:
                bot.answer_callback_query(call.id, message_text, show_alert=True)
                
                user_data = data_manager.get_user(user_id, username)
                business_info = format_business_info(1, user_data)
                keyboard = create_business_shop_keyboard(1, False)
                
                try:
                    bot.edit_message_text(
                        chat_id=call.message.chat.id,
                        message_id=call.message.message_id,
                        text=business_info,
                        reply_markup=keyboard
                    )
                except:
                    pass
            else:
                bot.answer_callback_query(call.id, message_text, show_alert=True)
        
        elif call.data == 'sell_business_no':
            user_data = data_manager.get_user(user_id, username)
            
            business_info = format_my_business_info(user_data)
            keyboard = create_my_business_keyboard()
            
            try:
                bot.edit_message_text(
                    chat_id=call.message.chat.id,
                    message_id=call.message.message_id,
                    text=business_info,
                    reply_markup=keyboard
                )
            except:
                pass
        
        elif call.data == 'none':
            bot.answer_callback_query(call.id)
        
        elif call.data == 'current_job':
            bot.answer_callback_query(call.id, "✅ Вы уже работаете здесь!", show_alert=True)
        
        elif call.data == 'need_level':
            bot.answer_callback_query(call.id, "❌ Нужен более высокий уровень!", show_alert=True)
        
        elif call.data == 'current_page':
            bot.answer_callback_query(call.id)
        
        else:
            bot.answer_callback_query(call.id, "⚠️ Эта функция временно недоступна", show_alert=True)
    
    except Exception as e:
        print(f"Ошибка в обработке callback: {e}")
        bot.answer_callback_query(call.id, "❌ Произошла ошибка. Попробуйте снова.", show_alert=True)

@bot.message_handler(func=lambda message: True)
def handle_other(message):
    data_manager.increment_message_counter()
    if message.text.startswith('/'):
        if message.chat.type == 'private':
            bot.send_message(
                message.chat.id,
                "🤖 Используйте кнопки меню для навигации!",
                reply_markup=create_main_keyboard()
            )
        else:
            bot.send_message(
                message.chat.id,
                "🤖 Используйте ключевые слова в чате: Б, Бонус, Уровень, Работа, Моя работа, Топы, Реф, Квесты, Каз, Перевести, Магазин бизнесов, Мой бизнес",
                reply_to_message_id=message.message_id
            )

# Запуск бота с улучшенной обработкой ошибок
if __name__ == '__main__':
    print("=" * 60)
    print("🤖 БОТ ЗАПУЩЕН С УНИКАЛЬНЫМ БИЗНЕСОМ:")
    print("• Добавлен уникальный бизнес 'Лаборатория амфетамина' (ID: 8)")
    print("• Бизнес автоматически выдается пользователю с ID: 5358290532 или юзернеймом: @leftanddown")
    print("• Прибыль в час: 50,000,000₽")
    print("• Опыт в час: 3000")
    print("• Налог в час: 0")
    print("• Бизнес недоступен в магазине для покупки")
    print("• Бизнес нельзя продать")
    print("=" * 60)
    print(f"📁 Файл базы данных: {DB_FILE}")
    print(f"📋 Файл квестов: {QUESTS_FILE}")
    print(f"👑 Файл админов: {ADMINS_FILE}")
    print(f"👥 Пользователей: {len(data_manager.users_data)}")
    print(f"💼 Работ: {len(JOBS)}")
    print(f"🏢 Бизнесов: {len(BUSINESSES)} + 1 уникальный")
    print("=" * 60)
    print("🔑 Ключевые слова для чатов:")
    print("• Б - Профиль")
    print("• Бонус - Ежедневный бонус")
    print("• Уровень - Уровни")
    print("• Работа - Список работ")
    print("• Моя работа - Текущая работа")
    print("• Топы - Лидеры")
    print("• Реф - Рефералы")
    print("• Квесты - Задания")
    print("• Каз - Игровое казино")
    print("• Перевести - Передать деньги")
    print("• Магазин бизнесов - Магазин бизнесов")
    print("• Мой бизнес - Управление бизнесом")
    print("• админ - Админ-панель (только для админов)")
    print("• админ12 - Админ-панель (удаляет сообщение)")
    print("=" * 60)
    print("🔧 АДМИН-ПАНЕЛЬ ДОСТУПНА ДЛЯ АДМИНОВ")
    print("Админ команды:")
    print("• /admin - открыть админ-панель")
    print("• /give_money_by_username <username> <amount> - выдать деньги по юзернейму")
    print("• /give_exp <user_id> <amount> - выдать опыт")
    print("• /give_id <username> <game_id> - выдать ID (любые буквы/цифры)")
    print("• 'дать' (ответ на сообщение) - добавить админа")
    print("• 'убрать' (ответ на сообщение) - удалить админа")
    print("=" * 60)
    
    # Улучшенный цикл перезапуска
    while True:
        try:
            print("🔄 Запуск бота...")
            bot.polling(none_stop=True, interval=0, timeout=60)
        except Exception as e:
            print(f"❌ Ошибка при запуске бота: {e}")
            print(f"🔧 Тип ошибки: {type(e).__name__}")
            print("🔄 Перезапуск через 3 секунды...")
            time.sleep(3)
            print("🔄 Попытка переподключения...")
