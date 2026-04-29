# MishaScvortsov
import tkinter as tk
from tkinter import ttk, messagebox
import requests
import json
import os

# Конфигурация
API_KEY = "YOUR_API_KEY_HERE"  # Замените на ваш API-ключ
API_URL = "https://api.exchangerate-api.com/v4/latest/"
HISTORY_FILE = "conversion_history.json"

class CurrencyConverter:
    def __init__(self, root):
        self.root = root
        self.root.title("Currency Converter")
        self.root.geometry("600x500")

        # Загрузка истории
        self.history = self.load_history()

        # Инициализация переменных
        self.from_currency = tk.StringVar(value="USD")
        self.to_currency = tk.StringVar(value="EUR")
        self.amount = tk.StringVar(value="1")
        self.result = tk.StringVar()

        self.setup_ui()

    def setup_ui(self):
        # Основной фрейм
        main_frame = ttk.Frame(self.root, padding="10")
        main_frame.grid(row=0, column=0, sticky=(tk.W, tk.E, tk.N, tk.S))

        # Выбор валюты "из"
        ttk.Label(main_frame, text="Из:").grid(row=0, column=0, sticky=tk.W)
        from_combo = ttk.Combobox(main_frame, textvariable=self.from_currency,
                                   values=self.get_currencies(), width=10)
        from_combo.grid(row=0, column=1, padx=5, pady=5)

        # Выбор валюты "в"
        ttk.Label(main_frame, text="В:").grid(row=0, column=2, sticky=tk.W)
        to_combo = ttk.Combobox(main_frame, textvariable=self.to_currency,
                              values=self.get_currencies(), width=10)
        to_combo.grid(row=0, column=3, padx=5, pady=5)

        # Поле ввода суммы
        ttk.Label(main_frame, text="Сумма:").grid(row=1, column=0, sticky=tk.W)
        amount_entry = ttk.Entry(main_frame, textvariable=self.amount, width=15)
        amount_entry.grid(row=1, column=1, padx=5, pady=5)

        # Кнопка конвертации
        convert_btn = ttk.Button(main_frame, text="Конвертировать",
                                command=self.convert_currency)
        convert_btn.grid(row=1, column=2, columnspan=2, pady=5)

        # Результат
        ttk.Label(main_frame, text="Результат:").grid(row=2, column=0, sticky=tk.W)
        result_label = ttk.Label(main_frame, textvariable=self.result,
                             font=("Arial", 12, "bold"))
        result_label.grid(row=2, column=1, columnspan=3, pady=10)

        # Таблица истории
        ttk.Label(main_frame, text="История конвертаций:").grid(row=3, column=0,
                                                               columnspan=4, sticky=tk.W, pady=(20, 5))

        columns = ("ID", "Дата", "Сумма", "Из", "В", "Результат")
        self.history_tree = ttk.Treeview(main_frame, columns=columns, show="headings", height=8)

        for col in columns:
            self.history_tree.heading(col, text=col)
            self.history_tree.column(col, width=80)

        self.history_tree.grid(row=4, column=0, columnspan=4, pady=5, sticky=(tk.W, tk.E))

        # Кнопки управления историей
        btn_frame = ttk.Frame(main_frame)
        btn_frame.grid(row=5, column=0, columnspan=4, pady=10)

        clear_btn = ttk.Button(btn_frame, text="Очистить историю",
                           command=self.clear_history)
        clear_btn.pack(side=tk.LEFT, padx=5)

        refresh_btn = ttk.Button(btn_frame, text="Обновить историю",
                            command=self.refresh_history)
        refresh_btn.pack(side=tk.LEFT, padx=5)

        self.refresh_history()

    def get_currencies(self):
        """Получение списка валют (упрощённый вариант)"""
        return ["USD", "EUR", "GBP", "JPY", "CAD", "AUD", "CHF", "CNY", "RUB"]

    def convert_currency(self):
        try:
            # Проверка ввода
            amount = float(self.amount.get())
            if amount <= 0:
                messagebox.showerror("Ошибка", "Сумма должна быть положительным числом")
                return

            from_curr = self.from_currency.get()
            to_curr = self.to_currency.get()

            # Получение курса через API
            rate = self.get_exchange_rate(from_curr, to_curr)
            if rate is None:
                return

            result = amount * rate
            self.result.set(f"{result:.2f} {to_curr}")

            # Сохранение в историю
            self.save_to_history(amount, from_curr, to_curr, result)

        except ValueError:
            messagebox.showerror("Ошибка", "Введите корректную сумму")
        except Exception as e:
            messagebox.showerror("Ошибка", f"Произошла ошибка: {str(e)}")

    def get_exchange_rate(self, from_curr, to_curr):
        """Получение курса обмена через API"""
        try:
            response = requests.get(f"{API_URL}{from_curr}")
            data = response.json()

            if response.status_code == 200 and 'rates' in data:
                if to_curr in data['rates']:
                    return data['rates'][to_curr]
                else:
                    messagebox.showerror("Ошибка", f"Валюта {to_curr} не найдена")
                    return None
            else:
                messagebox.showerror("Ошибка API", "Не удалось получить данные курса")
                return None
        except requests.RequestException as e:
            messagebox.showerror("Ошибка сети", f"Ошибка подключения: {str(e)}")
            return None

    def save_to_history(self, amount, from_curr, to_curr, result):
        """Сохранение конвертации в историю"""
        import datetime
        entry = {
            "id": len(self.history) + 1,
            "date": datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
            "amount": amount,
            "from": from_curr,
            "to": to_curr,
            "result": result
        }
        self.history.append(entry)
        self.save_history()
        self.refresh_history()

    def load_history(self):
        """Загрузка истории из файла"""
        if os.path.exists(HISTORY_FILE):
            try:
                with open(HISTORY_FILE, 'r', encoding='utf-8') as f:
                    return json.load(f)
            except (json.JSONDecodeError, IOError):
                return []
        return []

    def save_history(self):
        """Сохранение истории в файл"""
        try:
            with open(HISTORY_FILE, 'w', encoding='utf-8') as f:
                json.dump(self.history, f, ensure_ascii=False, indent=2)
        except IOError as e:
            messagebox.showerror("Ошибка", f"Не удалось сохранить историю: {str(e)}")

    def refresh_history(self):
        """Обновление отображения истории"""
        # Очистка таблицы
        for item in self.history_tree.get_children():
            self.history_tree.delete(item)
