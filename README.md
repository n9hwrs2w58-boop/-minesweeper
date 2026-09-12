import random
import sqlite3
import time

from telegram import InlineKeyboardButton, InlineKeyboardMarkup, Update
from telegram.ext import (
    Application,
    CallbackQueryHandler,
    CommandHandler,
    ContextTypes,
    MessageHandler,
    filters,
)

# =========================
# НАСТРОЙКИ
# =========================

TOKEN = "ВСТАВЬ_СЮДА_ТОКЕН_БОТА"

START_BONUS = 1200
BONUS_AMOUNT = 1200
BONUS_COOLDOWN = 10 * 60 * 60  # 10 часов

BOARD_SIZE = 5
MULTIPLIER = 1.50

MIN_BET = 1
MAX_BET = 1_000_000

# Активные игры
games = {}

# Пользователи, которые вводят ставку
waiting_for_bet = set()


# =========================
# БАЗА ДАННЫХ
# =========================

db = sqlite3.connect("wzr_game.db", check_same_thread=False)
cursor = db.cursor()

cursor.execute("""
CREATE TABLE IF NOT EXISTS users (
    user_id INTEGER PRIMARY KEY,
    username TEXT,
    balance REAL DEFAULT 1200,
    last_bonus INTEGER DEFAULT 0,
    games INTEGER DEFAULT 0,
    wins INTEGER DEFAULT 0,
    losses INTEGER DEFAULT 0
)
""")

db.commit()


def get_user(user_id, username=""):
    cursor.execute(
        "SELECT * FROM users WHERE user_id = ?",
        (user_id,)
    )

    user = cursor.fetchone()

    if user is None:
        cursor.execute("""
            INSERT INTO users
            (user_id, username, balance, last_bonus, games, wins, losses)
            VALUES (?, ?, ?, ?, 0, 0, 0)
        """, (
            user_id,
            username,
            START_BONUS,
            int(time.time())
        ))

        db.commit()

    else:
        cursor.execute(
            "UPDATE users SET username = ? WHERE user_id = ?",
            (username, user_id)
        )
        db.commit()


def balance(user_id):
    cursor.execute(
        "SELECT balance FROM users WHERE user_id = ?",
        (user_id,)
    )

    result = cursor.fetchone()

    return float(result[0]) if result else 0


def change_balance(user_id, amount):
    cursor.execute(
        "UPDATE users SET balance = balance + ? WHERE user_id = ?",
        (amount, user_id)
    )

    db.commit()


# =========================
# ГЛАВНОЕ МЕНЮ
# =========================

def menu():
    keyboard = [
        [
            InlineKeyboardButton("🎮 Играть", callback_data="play"),
            InlineKeyboardButton("💰 Баланс
