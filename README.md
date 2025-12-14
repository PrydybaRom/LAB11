
import tkinter as tk
from tkinter import ttk, messagebox
import sqlite3
import datetime

# --- 1. Керування Базою Даних (SQLite) ---
class DatabaseManager:
    """Клас для взаємодії з базою даних SQLite."""
    def __init__(self, db_name="trading_org.db"):
        self.conn = sqlite3.connect(db_name)
        self.cursor = self.conn.cursor()
        self._create_tables()

    def _create_tables(self, drop_tables=False):
        """Створення всіх необхідних таблиць."""
        if drop_tables:
            self.cursor.execute("DROP TABLE IF EXISTS payroll_log")
            self.cursor.execute("DROP TABLE IF EXISTS supplier_orders")
            self.cursor.execute("DROP TABLE IF EXISTS order_items")
            self.cursor.execute("DROP TABLE IF EXISTS orders")
            self.cursor.execute("DROP TABLE IF EXISTS products")
            self.cursor.execute("DROP TABLE IF EXISTS users")
        
        # Таблиця користувачів/працівників (додано поле salary)
        self.cursor.execute("""
            CREATE TABLE IF NOT EXISTS users (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                username TEXT UNIQUE NOT NULL,
                password TEXT NOT NULL,
                role TEXT NOT NULL,
                first_name TEXT, 
                last_name TEXT,
                phone_number TEXT,
                salary REAL DEFAULT 15000.0, -- Додано поле зарплати
                UNIQUE (username)
            )
        """)
        
        # Таблиця товарів
        self.cursor.execute("""
            CREATE TABLE IF NOT EXISTS products (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                name TEXT NOT NULL,
                description TEXT,
                price REAL NOT NULL,
                category TEXT NOT NULL,
                quantity INTEGER NOT NULL
            )
        """)
        
        # Таблиця замовлень клієнтів 
        self.cursor.execute("""
            CREATE TABLE IF NOT EXISTS orders (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                user_id INTEGER,
                order_date TEXT NOT NULL,
                status TEXT NOT NULL, 
                receiver_first_name TEXT,
                receiver_last_name TEXT,
                receiver_phone TEXT,
                city TEXT,
                post_office TEXT,
                total_amount REAL NOT NULL,
                FOREIGN KEY (user_id) REFERENCES users(id)
            )
        """)
        
        # Таблиця деталей замовлень
        self.cursor.execute("""
            CREATE TABLE IF NOT EXISTS order_items (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                order_id INTEGER,
                product_id INTEGER,
                quantity INTEGER NOT NULL,
                price_at_order REAL NOT NULL,
                FOREIGN KEY (order_id) REFERENCES orders(id),
                FOREIGN KEY (product_id) REFERENCES products(id)
            )
        """)
        
        # Таблиця замовлень постачальнику (додано price_at_supply)
        self.cursor.execute("""
            CREATE TABLE IF NOT EXISTS supplier_orders (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                product_id INTEGER,
                quantity INTEGER NOT NULL,
                order_date TEXT NOT NULL,
                status TEXT NOT NULL,
                price_at_supply REAL, -- Додано поле ціни закупівлі
                FOREIGN KEY (product_id) REFERENCES products(id)
            )
        """)
        
        # Таблиця: Журнал виплати зарплат
        self.cursor.execute("""
            CREATE TABLE IF NOT EXISTS payroll_log (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                payment_date TEXT NOT NULL,
                total_amount REAL NOT NULL,
                paid_by_user_id INTEGER,
                FOREIGN KEY (paid_by_user_id) REFERENCES users(id)
            )
        """)
        
        self.conn.commit()

    def register_user(self, username, password, role="customer", first_name="", last_name="", phone_number="", salary=15000.0):
        """Реєстрація нового користувача."""
        try:
            self.cursor.execute(
                "INSERT INTO users (username, password, role, first_name, last_name, phone_number, salary) VALUES (?, ?, ?, ?, ?, ?, ?)",
                (username, password, role, first_name, last_name, phone_number, salary)
            )
            self.conn.commit()
            return True
        except sqlite3.IntegrityError:
            return False 

    def login_user(self, username, password):
        """Перевірка даних для авторизації."""
        self.cursor.execute(
            "SELECT role, id FROM users WHERE username=? AND password=?", 
            (username, password)
        )
        result = self.cursor.fetchone()
        return result 

    def add_employee(self, username, password, role, first_name, last_name):
        """Додавання працівника адміністратором."""
        salary = 15000.0 if role != 'admin' else 10000.0
        return self.register_user(username, password, role, first_name, last_name, salary=salary)

    def add_product(self, name, description, price, category, quantity):
        """Додавання нового товару адміністратором."""
        try:
            self.cursor.execute(
                "INSERT INTO products (name, description, price, category, quantity) VALUES (?, ?, ?, ?, ?)",
                (name, description, price, category, quantity)
            )
            self.conn.commit()
            return True
        except Exception:
            return False
            
    def get_all_products(self):
        """Отримати повний список усіх товарів."""
        self.cursor.execute("SELECT id, name, description, price, category, quantity FROM products")
        return self.cursor.fetchall()

    def get_product_for_supply(self):
        """Отримати лише ID, Назву, Ціну для форми замовлення постачальнику."""
        self.cursor.execute("SELECT id, name, price FROM products")
        return self.cursor.fetchall()

    def delete_product(self, product_id):
        """Видалити товар за його ID."""
        try:
            self.cursor.execute("SELECT COUNT(*) FROM order_items WHERE product_id = ?", (product_id,))
            if self.cursor.fetchone()[0] > 0:
                 return False 
            self.cursor.execute("DELETE FROM products WHERE id = ?", (product_id,))
            self.conn.commit()
            return True
        except Exception:
            return False
            
    def get_employees(self):
        """Отримати список всіх працівників (роль НЕ customer)."""
        self.cursor.execute("""
            SELECT id, last_name, first_name, username, role 
            FROM users 
            WHERE role != 'customer'
            ORDER BY role, last_name
        """)
        return self.cursor.fetchall()

    def delete_user(self, user_id):
        """Видалити користувача за його ID."""
        try:
            self.cursor.execute("DELETE FROM users WHERE id = ?", (user_id,))
            self.conn.commit()
            return True
        except Exception:
            return False

    def get_products_for_customer(self):
        """Отримати список доступних товарів (кількість > 0)."""
        self.cursor.execute("SELECT id, name, description, price, category, quantity FROM products WHERE quantity > 0")
        return self.cursor.fetchall()

    def place_order(self, user_id, order_data, cart_items):
        """Створення нового замовлення та деталей замовлення."""
        date_now = datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        
        try:
            self.cursor.execute("""
                INSERT INTO orders (user_id, order_date, status, receiver_first_name, receiver_last_name, receiver_phone, city, post_office, total_amount)
                VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)
            """, (
                user_id, 
                date_now, 
                'нове', 
                order_data['first_name'], 
                order_data['last_name'], 
                order_data['phone'], 
                order_data['city'], 
                order_data['post_office'], 
                order_data['total']
            ))
            order_id = self.cursor.lastrowid
            
            for item in cart_items:
                product_id = item['id']
                quantity = item['quantity']
                price = item['price']
                
                self.cursor.execute("""
                    INSERT INTO order_items (order_id, product_id, quantity, price_at_order)
                    VALUES (?, ?, ?, ?)
                """, (order_id, product_id, quantity, price))
                
                self.cursor.execute("UPDATE products SET quantity = quantity - ? WHERE id = ?", (quantity, product_id))

            self.conn.commit()
            return True
        except Exception as e:
            self.conn.rollback()
            print(f"Помилка при оформленні замовлення: {e}")
            return False
            
    def get_user_orders(self, user_id):
        """Отримати всі замовлення для конкретного користувача."""
        self.cursor.execute("""
            SELECT id, order_date, status, total_amount, city, post_office
            FROM orders WHERE user_id = ?
            ORDER BY order_date DESC
        """, (user_id,))
        return self.cursor.fetchall()

    def get_all_new_orders(self):
        """Отримати всі активні замовлення для менеджера (з номером телефону)."""
        self.cursor.execute("""
            SELECT o.id, o.order_date, o.status, o.total_amount, o.receiver_first_name, o.receiver_last_name, o.receiver_phone, o.city
            FROM orders o
            WHERE o.status != 'виконано' 
            ORDER BY o.order_date
        """)
        return self.cursor.fetchall()

    def update_order_status(self, order_id, new_status):
        """Оновити статус замовлення менеджером."""
        try:
            self.cursor.execute("UPDATE orders SET status = ? WHERE id = ?", (new_status, order_id))
            self.conn.commit()
            return True
        except Exception:
            return False

    def create_supplier_order(self, product_id, quantity, price_at_supply):
        """Менеджер створює запит на постачання товару."""
        date_now = datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        try:
            self.cursor.execute("""
                INSERT INTO supplier_orders (product_id, quantity, order_date, status, price_at_supply)
                VALUES (?, ?, ?, ?, ?)
            """, (product_id, quantity, date_now, 'нове', price_at_supply))
            self.conn.commit()
            return True
        except Exception as e:
            print(f"Error creating supplier order: {e}")
            return False

    def get_supplier_orders(self):
        """Отримати всі замовлення постачальнику з назвою товару."""
        self.cursor.execute("""
            SELECT so.id, p.name, so.quantity, so.order_date, so.status, p.id, so.price_at_supply
            FROM supplier_orders so
            JOIN products p ON so.product_id = p.id
            ORDER BY so.order_date DESC
        """)
        return self.cursor.fetchall()

    def update_supplier_order_status(self, order_id, new_status):
        """Оновити статус замовлення постачальнику."""
        try:
            self.cursor.execute("UPDATE supplier_orders SET status = ? WHERE id = ?", (new_status, order_id))
            self.conn.commit()
            return True
        except Exception:
            return False
            
    def confirm_supply_arrival(self, order_id, product_id, quantity):
        """Підтвердити прихід товару, оновити статус та склад."""
        try:
            # 1. Оновити статус замовлення постачальнику
            self.cursor.execute("UPDATE supplier_orders SET status = 'доставлено' WHERE id = ?", (order_id,))
            
            # 2. Оновити кількість товару на складі
            self.cursor.execute("UPDATE products SET quantity = quantity + ? WHERE id = ?", (quantity, product_id))
            
            self.conn.commit()
            return True
        except Exception as e:
            self.conn.rollback()
            print(f"Помилка підтвердження приходу: {e}")
            return False
            
    # НОВІ МЕТОДИ ЗВІТНОСТІ ДЛЯ БУХГАЛТЕРА/ДИРЕКТОРА
    
    def get_sales_revenue(self):
        """Розрахунок загального доходу від продажів (сума всіх виконаних замовлень)."""
        self.cursor.execute("SELECT SUM(total_amount) FROM orders WHERE status = 'виконано'")
        result = self.cursor.fetchone()[0]
        return result if result else 0.0

    def get_supplier_expenses(self):
        """Розрахунок загальних витрат на постачальників (сума всіх доставлених замовлень)."""
        self.cursor.execute("SELECT SUM(quantity * price_at_supply) FROM supplier_orders WHERE status = 'доставлено'")
        result = self.cursor.fetchone()[0]
        return result if result else 0.0

    def get_employee_salaries(self):
        """Отримати список працівників та їх зарплат."""
        self.cursor.execute("""
            SELECT id, last_name, first_name, role, salary 
            FROM users 
            WHERE role != 'customer'
        """)
        return self.cursor.fetchall()
        
    def log_payroll_payment(self, paid_by_user_id, total_amount):
        """Записати факт виплати зарплат у журнал."""
        date_now = datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        try:
            self.cursor.execute("""
                INSERT INTO payroll_log (payment_date, total_amount, paid_by_user_id)
                VALUES (?, ?, ?)
            """, (date_now, total_amount, paid_by_user_id))
            self.conn.commit()
            return True
        except Exception:
            return False

    def get_total_payroll_expenses(self):
        """Отримати загальну суму виплачених зарплат (витрати)."""
        self.cursor.execute("SELECT SUM(total_amount) FROM payroll_log")
        result = self.cursor.fetchone()[0]
        return result if result else 0.0

# --- 2. Основний Клас Додатку (Tkinter) ---
class TradingApp(tk.Tk):
    def __init__(self):
        super().__init__()
        self.title("Організація Торгівлі 🛒")
        self.geometry("850x650")
        
        self.db = DatabaseManager()
        self.current_user = None 
        
        self._check_and_create_admin()
        self._initialize_interface()

    def _check_and_create_admin(self):
        """Створює початкового адміністратора, якщо його немає."""
        self.db.cursor.execute("SELECT COUNT(*) FROM users WHERE role='admin'")
        if self.db.cursor.fetchone()[0] == 0:
            self.db.register_user("admin", "adminpass", "admin", "Іван", "Адмін", phone_number="0991234567", salary=10000.0)
            print("Створено початкового адміністратора: admin/adminpass")

    def _initialize_interface(self):
        """Налаштування контейнера для екранів."""
        self.container = ttk.Frame(self)
        self.container.pack(fill="both", expand=True)
        self.screens = {}

        for F in (LoginScreen, RegisterScreen, AdminDashboard, CustomerDashboard, ManagerDashboard, AccountantDashboard, DirectorDashboard, SupplierDashboard):
            page_name = F.__name__
            frame = F(parent=self.container, controller=self)
            self.screens[page_name] = frame
            frame.grid(row=0, column=0, sticky="nsew")

        self.switch_frame("LoginScreen")

    def switch_frame(self, page_name):
        """Показати вказаний екран (замість show_frame)."""
        frame = self.screens[page_name]
        frame.tkraise()

    def attempt_login(self, username, password):
        """Спроба авторизації користувача."""
        result = self.db.login_user(username, password)
        if result:
            role, user_id = result
            self.current_user = (user_id, role)
            messagebox.showinfo("Авторизація", f"Успішний вхід як: {role.capitalize()}")
            self.navigate_to_dashboard(role)
        else:
            messagebox.showerror("Помилка", "Невірне ім'я користувача або пароль")

    def navigate_to_dashboard(self, role):
        """Перенаправлення на відповідну панель за роллю."""
        if role == "admin":
            self.screens["AdminDashboard"].load_data()
            self.switch_frame("AdminDashboard")
        elif role == "customer":
            self.screens["CustomerDashboard"]._load_catalog() 
            self.switch_frame("CustomerDashboard")
        elif role == "manager":
            self.screens["ManagerDashboard"].load_data()
            self.switch_frame("ManagerDashboard")
        elif role == "accountant":
            self.screens["AccountantDashboard"].load_data()
            self.switch_frame("AccountantDashboard")
        elif role == "director":
            self.screens["DirectorDashboard"].load_data() # Оновлення даних для директора
            self.switch_frame("DirectorDashboard")
        elif role == "supplier":
            self.screens["SupplierDashboard"].load_data()
            self.switch_frame("SupplierDashboard")
        else:
            self.switch_frame("LoginScreen")

    def logout(self):
        """Вихід з системи."""
        self.current_user = None
        self.switch_frame("LoginScreen")

# --- 3. Екрани Tkinter ---

class LoginScreen(ttk.Frame):
    def __init__(self, parent, controller):
        super().__init__(parent)
        self.controller = controller
        
        ttk.Label(self, text="🚪 Авторизація", font=("Arial", 18, "bold")).pack(pady=40)

        self.username_var = tk.StringVar()
        self.password_var = tk.StringVar()

        ttk.Label(self, text="Ім'я користувача:").pack(pady=5)
        ttk.Entry(self, textvariable=self.username_var, width=30).pack(pady=5)

        ttk.Label(self, text="Пароль:").pack(pady=5)
        ttk.Entry(self, textvariable=self.password_var, show="*", width=30).pack(pady=5)

        ttk.Button(self, text="Увійти", command=self._login_command).pack(pady=20)
        ttk.Button(self, text="Зареєструватися (Покупець)", 
                   command=lambda: controller.switch_frame("RegisterScreen")).pack(pady=5)

    def _login_command(self):
        username = self.username_var.get()
        password = self.password_var.get()
        self.controller.attempt_login(username, password)
        self.username_var.set("")
        self.password_var.set("")

class RegisterScreen(ttk.Frame):
    def __init__(self, parent, controller):
        super().__init__(parent)
        self.controller = controller
        
        ttk.Label(self, text="📝 Реєстрація Покупця", font=("Arial", 18, "bold")).pack(pady=10)
        
        self.username_var = tk.StringVar()
        self.password_var = tk.StringVar()
        self.first_name_var = tk.StringVar()
        self.last_name_var = tk.StringVar()
        self.phone_number_var = tk.StringVar()
        
        fields_frame = ttk.Frame(self)
        fields_frame.pack(pady=10)
        
        ttk.Label(fields_frame, text="Прізвище:").grid(row=0, column=0, sticky="w", padx=5, pady=2)
        ttk.Entry(fields_frame, textvariable=self.last_name_var, width=30).grid(row=0, column=1, padx=5, pady=2)

        ttk.Label(fields_frame, text="Ім'я:").grid(row=1, column=0, sticky="w", padx=5, pady=2)
        ttk.Entry(fields_frame, textvariable=self.first_name_var, width=30).grid(row=1, column=1, padx=5, pady=2)

        ttk.Label(fields_frame, text="Номер телефону:").grid(row=2, column=0, sticky="w", padx=5, pady=2)
        ttk.Entry(fields_frame, textvariable=self.phone_number_var, width=30).grid(row=2, column=1, padx=5, pady=2)
        
        ttk.Label(fields_frame, text="Логін:").grid(row=3, column=0, sticky="w", padx=5, pady=2)
        ttk.Entry(fields_frame, textvariable=self.username_var, width=30).grid(row=3, column=1, padx=5, pady=2)
        
        ttk.Label(fields_frame, text="Пароль:").grid(row=4, column=0, sticky="w", padx=5, pady=2)
        ttk.Entry(fields_frame, textvariable=self.password_var, show="*", width=30).grid(row=4, column=1, padx=5, pady=2)
        
        ttk.Button(self, text="Зареєструватися", command=self._register_command).pack(pady=20)
        ttk.Button(self, text="Назад до Входу", command=lambda: controller.switch_frame("LoginScreen")).pack(pady=5)

    def _register_command(self):
        username = self.username_var.get()
        password = self.password_var.get()
        first_name = self.first_name_var.get()
        last_name = self.last_name_var.get()
        phone = self.phone_number_var.get()

        if not all([username, password, first_name, last_name, phone]):
            messagebox.showerror("Помилка", "Заповніть всі поля.")
            return

        if self.controller.db.register_user(username, password, role="customer", first_name=first_name, last_name=last_name, phone_number=phone):
            messagebox.showinfo("Успіх", "Реєстрація пройшла успішно! Тепер можете увійти.")
            self.controller.switch_frame("LoginScreen")
        else:
            messagebox.showerror("Помилка", "Користувач з таким ім'ям вже існує.")


# --- Панель АДМІНІСТРАТОРА ---
class AdminDashboard(ttk.Frame):
    def __init__(self, parent, controller):
        super().__init__(parent)
        self.controller = controller
        
        notebook = ttk.Notebook(self)
        notebook.pack(expand=True, fill="both", padx=10, pady=10)

        employee_frame = ttk.Frame(notebook)
        notebook.add(employee_frame, text="🧑‍💼 Керування Працівниками")
        self._setup_employee_tab(employee_frame)
        
        product_frame = ttk.Frame(notebook)
        notebook.add(product_frame, text="💻 Керування Товарами")
        self._setup_product_tab(product_frame)
        
        notebook.bind("<<NotebookTabChanged>>", self._on_tab_change)

        ttk.Button(self, text="Вийти", command=controller.logout).pack(pady=5)
        
    def _on_tab_change(self, event):
        selected_tab = event.widget.tab(event.widget.select(), "text")
        if selected_tab == "💻 Керування Товарами":
            self._load_products_list()
        elif selected_tab == "🧑‍💼 Керування Працівниками":
            self._load_employees_list()

    def load_data(self):
        if hasattr(self, 'employees_tree'):
            self._load_employees_list()
        if hasattr(self, 'products_tree'):
            self._load_products_list()


    # --- Вкладка Працівників ---
    def _setup_employee_tab(self, frame):
        add_frame = ttk.LabelFrame(frame, text="➕ Додати Нового Працівника")
        add_frame.pack(padx=10, pady=5, fill='x')
        
        fields_frame = ttk.Frame(add_frame)
        fields_frame.pack(padx=10, pady=10)
        
        self.emp_first_name_var = tk.StringVar()
        self.emp_last_name_var = tk.StringVar()
        self.emp_username_var = tk.StringVar()
        self.emp_password_var = tk.StringVar()
        self.emp_role_var = tk.StringVar(value='manager')
        roles = ['manager', 'admin', 'accountant', 'supplier', 'director']
        
        ttk.Label(fields_frame, text="Прізвище:").grid(row=0, column=0, sticky="w", padx=5, pady=2)
        ttk.Entry(fields_frame, textvariable=self.emp_last_name_var, width=30).grid(row=0, column=1, padx=5, pady=2)
        
        ttk.Label(fields_frame, text="Ім'я:").grid(row=1, column=0, sticky="w", padx=5, pady=2)
        ttk.Entry(fields_frame, textvariable=self.emp_first_name_var, width=30).grid(row=1, column=1, padx=5, pady=2)
        
        ttk.Label(fields_frame, text="Логін:").grid(row=2, column=0, sticky="w", padx=5, pady=2)
        ttk.Entry(fields_frame, textvariable=self.emp_username_var, width=30).grid(row=2, column=1, padx=5, pady=2)
        
        ttk.Label(fields_frame, text="Пароль:").grid(row=3, column=0, sticky="w", padx=5, pady=2)
        ttk.Entry(fields_frame, textvariable=self.emp_password_var, show="*", width=30).grid(row=3, column=1, padx=5, pady=2)
        
        ttk.Label(fields_frame, text="Посада:").grid(row=4, column=0, sticky="w", padx=5, pady=2)
        ttk.Combobox(fields_frame, textvariable=self.emp_role_var, values=roles, state="readonly", width=28).grid(row=4, column=1, padx=5, pady=2)
        
        ttk.Button(add_frame, text="Додати Працівника", command=self._add_employee_command).pack(pady=5)
        
        list_frame = ttk.LabelFrame(frame, text="📋 Список Працівників")
        list_frame.pack(padx=10, pady=5, fill='both', expand=True)

        columns = ("id", "last_name", "first_name", "username", "role")
        self.employees_tree = ttk.Treeview(list_frame, columns=columns, show='headings')
        
        vsb = ttk.Scrollbar(list_frame, orient="vertical", command=self.employees_tree.yview)
        vsb.pack(side='right', fill='y')
        self.employees_tree.configure(yscrollcommand=vsb.set)
        
        self.employees_tree.pack(fill='both', expand=True, side='left')

        self.employees_tree.heading("id", text="ID", anchor=tk.CENTER)
        self.employees_tree.heading("last_name", text="Прізвище")
        self.employees_tree.heading("first_name", text="Ім'я")
        self.employees_tree.heading("username", text="Логін")
        self.employees_tree.heading("role", text="Посада")

        self.employees_tree.column("id", width=30, stretch=tk.NO)
        self.employees_tree.column("last_name", width=120)
        self.employees_tree.column("first_name", width=120)
        self.employees_tree.column("username", width=100)
        self.employees_tree.column("role", width=100)

        ttk.Button(list_frame, text="❌ Видалити Вибраного Працівника", command=self._delete_employee_command).pack(pady=5)
        
        self._load_employees_list() 

    def _load_employees_list(self):
        for item in self.employees_tree.get_children():
            self.employees_tree.delete(item)

        employees = self.controller.db.get_employees()
        
        for emp in employees:
            self.employees_tree.insert("", tk.END, values=emp)
            
    def _delete_employee_command(self):
        selected_item = self.employees_tree.selection()
        if not selected_item:
            messagebox.showwarning("Помилка", "Будь ласка, виберіть працівника для видалення.")
            return

        values = self.employees_tree.item(selected_item, 'values')
        user_id = values[0]
        username = values[3]
        role = values[4]
        
        admin_count = sum(1 for item in self.employees_tree.get_children() if self.employees_tree.item(item, 'values')[4] == 'admin')
        
        if role == 'admin' and admin_count == 1:
            messagebox.showerror("Помилка", "Неможливо видалити єдиного адміністратора системи.")
            return

        if messagebox.askyesno("Підтвердження", f"Ви впевнені, що хочете видалити працівника '{username}' ({role})?"):
            if self.controller.db.delete_user(user_id):
                messagebox.showinfo("Успіх", f"Працівника '{username}' успішно видалено.")
                self._load_employees_list() 
            else:
                messagebox.showerror("Помилка", "Не вдалося видалити працівника.")

    def _add_employee_command(self):
        last_name = self.emp_last_name_var.get()
        first_name = self.emp_first_name_var.get()
        username = self.emp_username_var.get()
        password = self.emp_password_var.get()
        role = self.emp_role_var.get()

        if not all([last_name, first_name, username, password, role]):
            messagebox.showerror("Помилка", "Заповніть всі поля.")
            return

        if self.controller.db.add_employee(username, password, role, first_name, last_name):
            messagebox.showinfo("Успіх", f"Працівника '{last_name} {first_name}' ({role}) успішно додано.")
            self.emp_last_name_var.set("")
            self.emp_first_name_var.set("")
            self.emp_username_var.set("")
            self.emp_password_var.set("")
            self._load_employees_list()
        else:
            messagebox.showerror("Помилка", "Користувач з таким логіном вже існує.")


    # --- Вкладка Товарів ---
    def _setup_product_tab(self, frame):
        
        # --- Фрейм Додавання Товарів ---
        add_frame = ttk.LabelFrame(frame, text="➕ Додати Новий Товар")
        add_frame.pack(padx=10, pady=5, fill='x')
        
        fields_frame = ttk.Frame(add_frame)
        fields_frame.pack(padx=10, pady=10)

        self.prod_name_var = tk.StringVar()
        self.prod_desc_var = tk.StringVar()
        self.prod_price_var = tk.DoubleVar(value=0.0)
        self.prod_category_var = tk.StringVar(value='ноутбук')
        self.prod_quantity_var = tk.IntVar(value=0)
        
        categories = ['ноутбук', 'пк', 'планшет', 'смартфон', 'аксесуари']

        ttk.Label(fields_frame, text="Назва:").grid(row=0, column=0, sticky="w", padx=5, pady=2)
        ttk.Entry(fields_frame, textvariable=self.prod_name_var, width=30).grid(row=0, column=1, padx=5, pady=2)
        
        ttk.Label(fields_frame, text="Опис:").grid(row=1, column=0, sticky="w", padx=5, pady=2)
        ttk.Entry(fields_frame, textvariable=self.prod_desc_var, width=30).grid(row=1, column=1, padx=5, pady=2)
        
        ttk.Label(fields_frame, text="Ціна:").grid(row=2, column=0, sticky="w", padx=5, pady=2)
        ttk.Entry(fields_frame, textvariable=self.prod_price_var, width=30).grid(row=2, column=1, padx=5, pady=2)
        
        ttk.Label(fields_frame, text="Категорія:").grid(row=3, column=0, sticky="w", padx=5, pady=2)
        ttk.Combobox(fields_frame, textvariable=self.prod_category_var, values=categories, state="readonly", width=28).grid(row=3, column=1, padx=5, pady=2)
        
        ttk.Label(fields_frame, text="Кількість на складі:").grid(row=4, column=0, sticky="w", padx=5, pady=2)
        ttk.Entry(fields_frame, textvariable=self.prod_quantity_var, width=30).grid(row=4, column=1, padx=5, pady=2)
        
        ttk.Button(add_frame, text="Додати Товар", command=self._add_product_command).pack(pady=10)

        # --- Фрейм Списку Товарів та Видалення ---
        list_frame = ttk.LabelFrame(frame, text="📋 Список Товарів")
        list_frame.pack(padx=10, pady=5, fill='both', expand=True)

        columns = ("id", "name", "price", "category", "quantity")
        self.products_tree = ttk.Treeview(list_frame, columns=columns, show='headings')
        
        vsb = ttk.Scrollbar(list_frame, orient="vertical", command=self.products_tree.yview)
        vsb.pack(side='right', fill='y')
        self.products_tree.configure(yscrollcommand=vsb.set)
        
        self.products_tree.pack(fill='both', expand=True, side='left')

        self.products_tree.heading("id", text="ID", anchor=tk.CENTER)
        self.products_tree.heading("name", text="Назва")
        self.products_tree.heading("price", text="Ціна, грн")
        self.products_tree.heading("category", text="Категорія")
        self.products_tree.heading("quantity", text="На складі")

        self.products_tree.column("id", width=30, stretch=tk.NO, anchor=tk.CENTER)
        self.products_tree.column("price", width=80, anchor=tk.E)
        self.products_tree.column("quantity", width=80, anchor=tk.CENTER)

        ttk.Button(list_frame, text="❌ Видалити Вибраний Товар", command=self._delete_product_command).pack(pady=5)
        
        self._load_products_list()

    def _load_products_list(self):
        """Завантажує та оновлює список усіх товарів у таблиці."""
        if not hasattr(self, 'products_tree'):
            return

        for item in self.products_tree.get_children():
            self.products_tree.delete(item)

        products = self.controller.db.get_all_products()
        
        for prod in products:
            self.products_tree.insert("", tk.END, values=(
                prod[0], 
                prod[1], 
                f"{prod[3]:.2f}", 
                prod[4], 
                prod[5]
            ))

    def _delete_product_command(self):
        """Видаляє вибраний товар."""
        selected_item = self.products_tree.selection()
        if not selected_item:
            messagebox.showwarning("Помилка", "Будь ласка, виберіть товар для видалення.")
            return

        values = self.products_tree.item(selected_item, 'values')
        product_id = values[0]
        product_name = values[1]
        
        if messagebox.askyesno("Підтвердження", f"Ви впевнені, що хочете видалити товар '{product_name}' (ID: {product_id})?"):
            if self.controller.db.delete_product(product_id):
                messagebox.showinfo("Успіх", f"Товар '{product_name}' успішно видалено.")
                self._load_products_list() 
            else:
                messagebox.showerror("Помилка", "Не вдалося видалити товар. Він може бути пов'язаний з існуючими замовленнями.")

    def _add_product_command(self):
        try:
            name = self.prod_name_var.get()
            description = self.prod_desc_var.get()
            price = self.prod_price_var.get()
            category = self.prod_category_var.get()
            quantity = self.prod_quantity_var.get()
            
            if not all([name, description, category]) or price <= 0 or quantity < 0:
                 messagebox.showerror("Помилка", "Перевірте всі поля. Ціна має бути більша за 0, а кількість не менша 0.")
                 return

            if self.controller.db.add_product(name, description, price, category, quantity):
                messagebox.showinfo("Успіх", f"Товар '{name}' успішно додано до складу.")
                self.prod_name_var.set("")
                self.prod_desc_var.set("")
                self.prod_price_var.set(0.0)
                self.prod_quantity_var.set(0)
                self._load_products_list()
            else:
                messagebox.showerror("Помилка", "Не вдалося додати товар.")
        except tk.TclError:
             messagebox.showerror("Помилка", "Ціна та Кількість мають бути числами.")


# --- Панель КОРИСТУВАЧА ---
class CustomerDashboard(ttk.Frame):
    def __init__(self, parent, controller):
        super().__init__(parent)
        self.controller = controller
        
        self.cart = {}
        
        notebook = ttk.Notebook(self)
        notebook.pack(expand=True, fill="both", padx=10, pady=10)

        catalog_frame = ttk.Frame(notebook)
        notebook.add(catalog_frame, text="🛒 Каталог Товарів")
        self._setup_catalog_tab(catalog_frame)

        checkout_frame = ttk.Frame(notebook)
        notebook.add(checkout_frame, text="✅ Кошик та Оформлення")
        self._setup_checkout_tab(checkout_frame)
        
        orders_frame = ttk.Frame(notebook)
        notebook.add(orders_frame, text="📦 Мої Замовлення")
        self._setup_orders_tab(orders_frame)
        
        notebook.bind("<<NotebookTabChanged>>", self._on_tab_change)

        ttk.Button(self, text="Вийти", command=controller.logout).pack(pady=5)
        
    def _on_tab_change(self, event):
        selected_tab = event.widget.tab(event.widget.select(), "text")
        if selected_tab == "✅ Кошик та Оформлення":
            self._display_cart()
        elif selected_tab == "📦 Мої Замовлення":
            self._load_user_orders()
        elif selected_tab == "🛒 Каталог Товарів":
            self._load_catalog()

    # --- 1. Каталог ---
    def _setup_catalog_tab(self, frame):
        ttk.Label(frame, text="Доступні Товари (Електроніка)", font=("Arial", 14, "bold")).pack(pady=10)
        
        columns = ("name", "category", "price", "available")
        self.catalog_tree = ttk.Treeview(frame, columns=columns, show='headings')
        self.catalog_tree.pack(fill='both', expand=True, padx=10)

        self.catalog_tree.heading("name", text="Назва")
        self.catalog_tree.heading("category", text="Категорія")
        self.catalog_tree.heading("price", text="Ціна")
        self.catalog_tree.heading("available", text="В наявності")
        
        self.catalog_tree.column("price", width=100, anchor=tk.E)
        self.catalog_tree.column("available", width=100, anchor=tk.CENTER)

        self._load_catalog()
        
        add_frame = ttk.Frame(frame)
        add_frame.pack(pady=10)
        
        self.qty_var = tk.IntVar(value=1)
        ttk.Label(add_frame, text="Кількість:").pack(side=tk.LEFT, padx=5)
        ttk.Entry(add_frame, textvariable=self.qty_var, width=5).pack(side=tk.LEFT, padx=5)
        ttk.Button(add_frame, text="➕ Додати в Кошик", command=self._add_to_cart).pack(side=tk.LEFT, padx=10)

    def _load_catalog(self):
        for item in self.catalog_tree.get_children():
            self.catalog_tree.delete(item)

        products = self.controller.db.get_products_for_customer()
        
        for prod in products:
            self.catalog_tree.insert("", tk.END, iid=prod[0], 
                                     values=(prod[1], prod[4], f"{prod[3]:.2f} грн", prod[5]))

    def _add_to_cart(self):
        selected_item = self.catalog_tree.focus()
        if not selected_item:
            messagebox.showwarning("Помилка", "Будь ласка, оберіть товар.")
            return

        product_id = int(selected_item)
        quantity = self.qty_var.get()
        
        values = self.catalog_tree.item(selected_item, 'values')
        name = values[0]
        price = float(values[2].replace(' грн', ''))
        available = int(values[3])
        
        current_cart_qty = self.cart.get(product_id, {}).get('quantity', 0)
        
        if quantity <= 0:
            messagebox.showerror("Помилка", "Кількість має бути позитивною.")
            return
            
        if quantity + current_cart_qty > available:
            messagebox.showerror("Помилка", f"На складі доступно лише {available} од. У кошику вже {current_cart_qty}.")
            return

        if product_id in self.cart:
            self.cart[product_id]['quantity'] += quantity
        else:
            self.cart[product_id] = {'id': product_id, 'name': name, 'price': price, 'quantity': quantity}
            
        messagebox.showinfo("Кошик", f"{quantity} од. '{name}' додано до кошика.")
        self.qty_var.set(1)

    # --- 2. Кошик та Оформлення ---
    def _setup_checkout_tab(self, frame):
        
        cart_frame = ttk.LabelFrame(frame, text="🧺 Ваш Кошик")
        cart_frame.pack(padx=10, pady=5, fill='x')
        
        cart_columns = ("name", "price", "qty", "subtotal")
        self.checkout_tree = ttk.Treeview(cart_frame, columns=cart_columns, show='headings', height=5)
        self.checkout_tree.pack(fill='x', padx=5, pady=5)
        self.checkout_tree.heading("name", text="Товар")
        self.checkout_tree.heading("price", text="Ціна, грн")
        self.checkout_tree.heading("qty", text="Кількість")
        self.checkout_tree.heading("subtotal", text="Сума, грн")
        self.checkout_tree.column("price", width=120, anchor=tk.E)
        self.checkout_tree.column("qty", width=100, anchor=tk.CENTER)
        self.checkout_tree.column("subtotal", width=150, anchor=tk.E)
        
        self.total_label = ttk.Label(cart_frame, text="Загальна сума: 0.00 грн", font=("Arial", 12, "bold"))
        self.total_label.pack(side=tk.RIGHT, padx=5, pady=5)
        
        ttk.Button(cart_frame, text="❌ Очистити Кошик", command=self._clear_cart).pack(side=tk.LEFT, padx=5, pady=5)


        checkout_form = ttk.LabelFrame(frame, text="🚚 Дані Доставки та Оформлення")
        checkout_form.pack(padx=10, pady=10, fill='x')
        
        form_grid = ttk.Frame(checkout_form)
        form_grid.pack(padx=10, pady=10)
        
        self.rec_first_name_var = tk.StringVar()
        self.rec_last_name_var = tk.StringVar()
        self.rec_phone_var = tk.StringVar()
        self.city_var = tk.StringVar()
        self.post_office_var = tk.StringVar()
        
        ttk.Label(form_grid, text="Прізвище отримувача:").grid(row=0, column=0, sticky="w", padx=5, pady=2)
        ttk.Entry(form_grid, textvariable=self.rec_last_name_var, width=30).grid(row=0, column=1, padx=5, pady=2)
        
        ttk.Label(form_grid, text="Ім'я отримувача:").grid(row=1, column=0, sticky="w", padx=5, pady=2)
        ttk.Entry(form_grid, textvariable=self.rec_first_name_var, width=30).grid(row=1, column=1, padx=5, pady=2)
        
        ttk.Label(form_grid, text="Номер телефону:").grid(row=2, column=0, sticky="w", padx=5, pady=2)
        ttk.Entry(form_grid, textvariable=self.rec_phone_var, width=30).grid(row=2, column=1, padx=5, pady=2)
        
        ttk.Label(form_grid, text="Місто:").grid(row=3, column=0, sticky="w", padx=5, pady=2)
        ttk.Entry(form_grid, textvariable=self.city_var, width=30).grid(row=3, column=1, padx=5, pady=2)
        
        ttk.Label(form_grid, text="Відділення пошти:").grid(row=4, column=0, sticky="w", padx=5, pady=2)
        ttk.Entry(form_grid, textvariable=self.post_office_var, width=30).grid(row=4, column=1, padx=5, pady=2)
        
        ttk.Label(checkout_form, text="Спосіб оплати: **Оплата при отриманні**", foreground="blue").pack(pady=5)
        ttk.Button(checkout_form, text="💸 Оформити Замовлення", command=self._place_order_command).pack(pady=10)


    def _display_cart(self):
        for item in self.checkout_tree.get_children():
            self.checkout_tree.delete(item)
            
        total = 0
        for item_data in self.cart.values():
            subtotal = item_data['price'] * item_data['quantity']
            total += subtotal
            self.checkout_tree.insert("", tk.END, values=(
                item_data['name'], 
                f"{item_data['price']:.2f}", 
                item_data['quantity'], 
                f"{subtotal:.2f}"
            ))
            
        self.total_label.config(text=f"Загальна сума: {total:.2f} грн")
        self.current_total = total

    def _clear_cart(self):
        self.cart = {}
        self._display_cart()
        messagebox.showinfo("Кошик", "Кошик очищено.")

    def _place_order_command(self):
        if not self.cart:
            messagebox.showerror("Помилка", "Кошик порожній!")
            return

        last_name = self.rec_last_name_var.get()
        first_name = self.rec_first_name_var.get()
        phone = self.rec_phone_var.get()
        city = self.city_var.get()
        post_office = self.post_office_var.get()
        
        if not all([last_name, first_name, phone, city, post_office]):
            messagebox.showerror("Помилка", "Заповніть всі дані доставки.")
            return

        order_data = {
            'last_name': last_name,
            'first_name': first_name,
            'phone': phone,
            'city': city,
            'post_office': post_office,
            'total': self.current_total
        }
        
        user_id = self.controller.current_user[0]
        cart_items_list = list(self.cart.values())

        if self.controller.db.place_order(user_id, order_data, cart_items_list):
            messagebox.showinfo("Успіх", "Замовлення успішно оформлено! Статус: Нове.")
            self._load_catalog()
            self.cart = {} 
            self._display_cart()
        else:
            messagebox.showerror("Помилка", "Не вдалося оформити замовлення. Спробуйте пізніше.")


    # --- 3. Мої Замовлення ---
    def _setup_orders_tab(self, frame):
        ttk.Label(frame, text="Ваші Замовлення та Статуси", font=("Arial", 14, "bold")).pack(pady=10)
        
        order_columns = ("id", "date", "status", "total", "city", "post")
        self.orders_tree = ttk.Treeview(frame, columns=order_columns, show='headings')
        self.orders_tree.pack(fill='both', expand=True, padx=10)

        self.orders_tree.heading("id", text="№")
        self.orders_tree.heading("date", text="Дата")
        self.orders_tree.heading("status", text="Статус")
        self.orders_tree.heading("total", text="Сума")
        self.orders_tree.heading("city", text="Місто")
        self.orders_tree.heading("post", text="Відділення")
        
        self.orders_tree.column("id", width=50, stretch=tk.NO)
        self.orders_tree.column("total", width=100, anchor=tk.E)

    def _load_user_orders(self):
        for item in self.orders_tree.get_children():
            self.orders_tree.delete(item)
            
        user_id = self.controller.current_user[0]
        orders = self.controller.db.get_user_orders(user_id)
        
        for order in orders:
            self.orders_tree.insert("", tk.END, values=(
                order[0], 
                order[1].split(' ')[0], 
                order[2].capitalize(), 
                f"{order[3]:.2f} грн", 
                order[4], 
                order[5]
            ))

# --- Панель МЕНЕДЖЕРА ---
class ManagerDashboard(ttk.Frame):
    def __init__(self, parent, controller):
        super().__init__(parent)
        self.controller = controller
        
        notebook = ttk.Notebook(self)
        notebook.pack(expand=True, fill="both", padx=10, pady=10)

        # 1. Обробка замовлень клієнтів
        orders_frame = ttk.Frame(notebook)
        notebook.add(orders_frame, text="📋 Обробка Замовлень Клієнтів")
        self._setup_orders_tab(orders_frame)

        # 2. Замовлення постачальнику (НОВА ФОРМА)
        supply_frame = ttk.Frame(notebook)
        notebook.add(supply_frame, text="🚚 Замовлення Постачальнику")
        self._setup_supply_tab(supply_frame)

        notebook.bind("<<NotebookTabChanged>>", self._on_tab_change)
        ttk.Button(self, text="Вийти", command=controller.logout).pack(pady=5)
    
    def load_data(self):
        """Оновлення даних при вході."""
        if hasattr(self, 'manager_orders_tree'):
            self._load_orders_list()
        if hasattr(self, 'supply_product_var'):
            self._load_supply_products()

    def _on_tab_change(self, event):
        selected_tab = event.widget.tab(event.widget.select(), "text")
        if selected_tab == "📋 Обробка Замовлень Клієнтів":
            self._load_orders_list()
        elif selected_tab == "🚚 Замовлення Постачальнику":
             self._load_supply_products()

    # --- 1. Обробка Замовлень ---
    def _setup_orders_tab(self, frame):
        ttk.Label(frame, text="Активні Замовлення", font=("Arial", 14, "bold")).pack(pady=10)
        
        order_columns = ("id", "date", "status", "total", "customer_name", "customer_phone", "city") 
        self.manager_orders_tree = ttk.Treeview(frame, columns=order_columns, show='headings')
        self.manager_orders_tree.pack(fill='both', expand=True, padx=10)

        self.manager_orders_tree.heading("id", text="№")
        self.manager_orders_tree.heading("date", text="Дата")
        self.manager_orders_tree.heading("status", text="Статус")
        self.manager_orders_tree.heading("total", text="Сума")
        self.manager_orders_tree.heading("customer_name", text="Отримувач (ПІБ)")
        self.manager_orders_tree.heading("customer_phone", text="Телефон")
        self.manager_orders_tree.heading("city", text="Місто")
        
        self.manager_orders_tree.column("id", width=40, stretch=tk.NO)
        self.manager_orders_tree.column("total", width=80, anchor=tk.E)
        self.manager_orders_tree.column("customer_phone", width=100) 

        self._load_orders_list()

        control_frame = ttk.Frame(frame)
        control_frame.pack(pady=10)
        
        self.status_var = tk.StringVar(value='взято в роботу')
        statuses = ['нове', 'взято в роботу', 'передано у службу доставки', 'виконано']
        
        ttk.Label(control_frame, text="Змінити статус на:").pack(side=tk.LEFT, padx=5)
        ttk.Combobox(control_frame, textvariable=self.status_var, values=statuses, state="readonly", width=30).pack(side=tk.LEFT, padx=5)
        ttk.Button(control_frame, text="✅ Змінити Статус", command=self._update_status_command).pack(side=tk.LEFT, padx=10)


    def _load_orders_list(self):
        for item in self.manager_orders_tree.get_children():
            self.manager_orders_tree.delete(item)
            
        orders = self.controller.db.get_all_new_orders()
        
        for order in orders:
            # order: (o.id, o.order_date, o.status, o.total_amount, o.receiver_first_name, o.receiver_last_name, o.receiver_phone, o.city)
            customer_name = f"{order[5]} {order[4]}" # Прізвище Ім'я
            self.manager_orders_tree.insert("", tk.END, values=(
                order[0], 
                order[1].split(' ')[0], 
                order[2].capitalize(), 
                f"{order[3]:.2f} грн", 
                customer_name, 
                order[6], # Телефон
                order[7] # Місто
            ))

    def _update_status_command(self):
        selected_item = self.manager_orders_tree.focus()
        if not selected_item:
            messagebox.showwarning("Помилка", "Будь ласка, оберіть замовлення.")
            return

        order_id = self.manager_orders_tree.item(selected_item, 'values')[0]
        new_status = self.status_var.get()
        
        if self.controller.db.update_order_status(order_id, new_status):
            messagebox.showinfo("Успіх", f"Статус замовлення №{order_id} змінено на '{new_status.capitalize()}'")
            self._load_orders_list()
        else:
            messagebox.showerror("Помилка", "Не вдалося оновити статус.")

    # --- 2. Замовлення Постачальнику ---
    def _setup_supply_tab(self, frame):
        
        form_frame = ttk.LabelFrame(frame, text="✍️ Створити Запит Постачальнику")
        form_frame.pack(padx=10, pady=10, fill='x')
        
        fields_grid = ttk.Frame(form_frame)
        fields_grid.pack(padx=10, pady=10)

        self.supply_product_var = tk.StringVar()
        self.supply_product_id = None
        self.supply_price = 0.0 # Зберігаємо ціну тут
        self.supply_price_label = tk.StringVar(value="Ціна закупівлі: 0.00 грн")
        self.supply_qty_var = tk.IntVar(value=1)
        
        ttk.Label(fields_grid, text="Товар (Назва):").grid(row=0, column=0, sticky="w", padx=5, pady=5)
        self.product_combobox = ttk.Combobox(fields_grid, textvariable=self.supply_product_var, state="readonly", width=50)
        self.product_combobox.grid(row=0, column=1, padx=5, pady=5)
        self.product_combobox.bind("<<ComboboxSelected>>", self._update_supply_price)

        ttk.Label(fields_grid, textvariable=self.supply_price_label, font=("Arial", 10, "bold")).grid(row=1, column=1, sticky="w", padx=5, pady=5)

        ttk.Label(fields_grid, text="Кількість для замовлення:").grid(row=2, column=0, sticky="w", padx=5, pady=5)
        ttk.Entry(fields_grid, textvariable=self.supply_qty_var, width=10).grid(row=2, column=1, sticky="w", padx=5, pady=5)

        ttk.Button(form_frame, text="🚀 Відправити Запит Постачальнику", command=self._send_supply_order).pack(pady=10)

        self._load_supply_products()

    def _load_supply_products(self):
        """Завантажує товари у комбобокс для замовлення постачальнику."""
        products = self.controller.db.get_product_for_supply()
        # Зберігаємо дані у форматі { "Назва": (ID, Ціна) }
        self.supply_products_data = {f"{p[1]}": (p[0], p[2]) for p in products} 
        
        if products:
            product_list = list(self.supply_products_data.keys())
            self.product_combobox['values'] = product_list
            self.product_combobox.set(product_list[0])
            self._update_supply_price()
        else:
            self.product_combobox['values'] = ["Немає доступних товарів"]
            self.product_combobox.set("Немає доступних товарів")
            self.supply_product_id = None
            self.supply_price = 0.0
            self.supply_price_label.set("Ціна закупівлі: 0.00 грн")

    def _update_supply_price(self, event=None):
        """Оновлює поле ціни та ID при виборі товару."""
        selected_text = self.supply_product_var.get()
        if selected_text in self.supply_products_data:
            product_id, price = self.supply_products_data[selected_text]
            self.supply_product_id = product_id
            self.supply_price = price
            self.supply_price_label.set(f"Ціна закупівлі: {price:.2f} грн")
        else:
            self.supply_product_id = None
            self.supply_price = 0.0
            self.supply_price_label.set("Ціна закупівлі: 0.00 грн")


    def _send_supply_order(self):
        """Відправляє запит на постачання."""
        if self.supply_product_id is None:
            messagebox.showerror("Помилка", "Будь ласка, виберіть товар для замовлення.")
            return

        quantity = self.supply_qty_var.get()

        if quantity <= 0:
            messagebox.showerror("Помилка", "Кількість має бути позитивною.")
            return

        product_name = self.supply_product_var.get()
        price_at_supply = self.supply_price # Використовуємо збережену ціну
        
        if messagebox.askyesno("Підтвердження", f"Створити замовлення постачальнику на {quantity} од. '{product_name}' (по {price_at_supply:.2f} грн за од.)?"):
            if self.controller.db.create_supplier_order(self.supply_product_id, quantity, price_at_supply):
                messagebox.showinfo("Успіх", f"Замовлення на {product_name} ({quantity} од.) успішно створено!")
            else:
                messagebox.showerror("Помилка", "Не вдалося створити замовлення постачальнику.")

# --- Панель ПОСТАЧАЛЬНИКА ---
class SupplierDashboard(ttk.Frame):
    def __init__(self, parent, controller):
        super().__init__(parent)
        self.controller = controller
        
        ttk.Label(self, text="🚚 Панель Постачальника", font=("Arial", 16, "bold")).pack(pady=10)
        
        # Список замовлень від менеджера
        list_frame = ttk.LabelFrame(self, text="📋 Замовлення Товарів від Менеджера")
        list_frame.pack(padx=10, pady=5, fill='both', expand=True)

        columns = ("id", "product_name", "quantity", "price_supply", "date", "status", "product_id_hidden")
        self.supplier_orders_tree = ttk.Treeview(list_frame, columns=columns, show='headings')
        self.supplier_orders_tree.pack(fill='both', expand=True, padx=5, pady=5)

        self.supplier_orders_tree.heading("id", text="№")
        self.supplier_orders_tree.heading("product_name", text="Товар")
        self.supplier_orders_tree.heading("quantity", text="Кількість")
        self.supplier_orders_tree.heading("price_supply", text="Ціна закупівлі")
        self.supplier_orders_tree.heading("date", text="Дата Замовлення")
        self.supplier_orders_tree.heading("status", text="Статус")
        
        self.supplier_orders_tree.column("id", width=50, stretch=tk.NO, anchor=tk.CENTER)
        self.supplier_orders_tree.column("quantity", width=80, anchor=tk.CENTER)
        self.supplier_orders_tree.column("price_supply", width=120, anchor=tk.E)
        self.supplier_orders_tree.column("product_id_hidden", width=0, stretch=tk.NO) 

        # Блок керування статусом
        control_frame = ttk.Frame(self)
        control_frame.pack(pady=10)
        
        self.status_var = tk.StringVar(value='в дорозі')
        statuses = ['нове', 'в дорозі']
        
        ttk.Label(control_frame, text="Змінити статус на:").pack(side=tk.LEFT, padx=5)
        ttk.Combobox(control_frame, textvariable=self.status_var, values=statuses, state="readonly", width=15).pack(side=tk.LEFT, padx=5)
        ttk.Button(control_frame, text="🔄 Оновити Статус", command=self._update_status_command).pack(side=tk.LEFT, padx=10)
        ttk.Button(control_frame, text="✅ Підтвердити Надходження (Оприбуткувати)", command=self._confirm_arrival).pack(side=tk.LEFT, padx=10)


        self.load_data()
        ttk.Button(self, text="Вийти", command=controller.logout).pack(pady=20)
        
    def load_data(self):
        self._load_supplier_orders()
        
    def _load_supplier_orders(self):
        for item in self.supplier_orders_tree.get_children():
            self.supplier_orders_tree.delete(item)
            
        orders = self.controller.db.get_supplier_orders()
        
        for order in orders:
            self.supplier_orders_tree.insert("", tk.END, values=(
                order[0], # id
                order[1], # product_name
                order[2], # quantity
                f"{order[6]:.2f} грн", # Ціна закупівлі
                order[3].split(' ')[0], # date
                order[4].capitalize(), # status
                order[5] # product_id (hidden)
            ))

    def _update_status_command(self):
        selected_item = self.supplier_orders_tree.focus()
        if not selected_item:
            messagebox.showwarning("Помилка", "Будь ласка, оберіть замовлення.")
            return

        values = self.supplier_orders_tree.item(selected_item, 'values')
        order_id = values[0]
        new_status = self.status_var.get()
        
        if new_status == 'доставлено':
            messagebox.showwarning("Увага", "Статус 'доставлено' має бути встановлений бухгалтером/адміном після оприбуткування.")
            return

        if self.controller.db.update_supplier_order_status(order_id, new_status):
            messagebox.showinfo("Успіх", f"Статус замовлення №{order_id} змінено на '{new_status.capitalize()}'")
            self._load_supplier_orders()
        else:
            messagebox.showerror("Помилка", "Не вдалося оновити статус.")
            
    # Заглушка, якщо постачальник хоче оприбуткувати сам (потрібна роль бухгалтера/адміна)
    def _confirm_arrival(self):
        messagebox.showinfo("Обмеження", "Для оприбуткування товару на склад (статус 'доставлено' та оновлення залишків) потрібна роль **Бухгалтера** або **Адміністратора**.")


# --- Панель БУХГАЛТЕРА ---
class AccountantDashboard(ttk.Frame):
    def __init__(self, parent, controller):
        super().__init__(parent)
        self.controller = controller
        
        notebook = ttk.Notebook(self)
        notebook.pack(expand=True, fill="both", padx=10, pady=10)
        
        # 1. Звітність та Доходи
        report_frame = ttk.Frame(notebook)
        notebook.add(report_frame, text="📊 Звітність та Доходи")
        self._setup_report_tab(report_frame)

        # 2. Керування зарплатами
        payroll_frame = ttk.Frame(notebook)
        notebook.add(payroll_frame, text="💰 Керування Зарплатами")
        self._setup_payroll_tab(payroll_frame)
        
        # 3. Замовлення постачальнику (для оприбуткування)
        supply_frame = ttk.Frame(notebook)
        notebook.add(supply_frame, text="📦 Прихід від Постачальника")
        self._setup_supply_tab(supply_frame)


        notebook.bind("<<NotebookTabChanged>>", self._on_tab_change)
        ttk.Button(self, text="Вийти", command=controller.logout).pack(pady=20)
        
    def load_data(self):
        """Оновлення всіх вкладок при вході."""
        self._load_reports()
        self._load_payroll_list()
        self._load_supplier_orders()

    def _on_tab_change(self, event):
        selected_tab = event.widget.tab(event.widget.select(), "text")
        if selected_tab == "📦 Прихід від Постачальника":
            self._load_supplier_orders()
        elif selected_tab == "📊 Звітність та Доходи":
            self._load_reports()
        elif selected_tab == "💰 Керування Зарплатами":
            self._load_payroll_list()
            
    # --- 1. Вкладка ЗВІТНОСТІ ---
    def _setup_report_tab(self, frame):
        ttk.Label(frame, text="Фінансова Звітність (Виконані операції)", font=("Arial", 16, "bold")).pack(pady=20)
        
        # Фрейм для відображення ключових показників
        summary_frame = ttk.LabelFrame(frame, text="Основні Показники")
        summary_frame.pack(padx=10, pady=10, fill='x')
        
        self.revenue_var = tk.StringVar(value="Дохід від продажів: 0.00 грн")
        self.supplier_cost_var = tk.StringVar(value="Витрати на постачальника (закупівля): 0.00 грн")
        self.payroll_cost_var = tk.StringVar(value="Витрати на зарплату (загально): 0.00 грн")
        self.profit_var = tk.StringVar(value="Чистий прибуток: 0.00 грн")
        
        ttk.Label(summary_frame, textvariable=self.revenue_var, font=("Arial", 12, "bold"), foreground="green").pack(pady=5, anchor='w', padx=10)
        ttk.Label(summary_frame, textvariable=self.supplier_cost_var, font=("Arial", 12, "bold"), foreground="red").pack(pady=5, anchor='w', padx=10)
        ttk.Label(summary_frame, textvariable=self.payroll_cost_var, font=("Arial", 12, "bold"), foreground="red").pack(pady=5, anchor='w', padx=10)
        ttk.Separator(summary_frame, orient='horizontal').pack(fill='x', pady=5, padx=10)
        ttk.Label(summary_frame, textvariable=self.profit_var, font=("Arial", 14, "bold"), foreground="blue").pack(pady=10, anchor='w', padx=10)

        self._load_reports()
        
    def _load_reports(self):
        """Отримує та відображає фінансові звіти."""
        # Перевірка наявності елементів інтерфейсу
        if not hasattr(self, 'revenue_var'): return

        revenue = self.controller.db.get_sales_revenue()
        supplier_cost = self.controller.db.get_supplier_expenses()
        payroll_cost = self.controller.db.get_total_payroll_expenses()
        
        total_costs = supplier_cost + payroll_cost
        profit = revenue - total_costs
        
        self.revenue_var.set(f"💰 Дохід від продажів (виконані замовлення): {revenue:.2f} грн")
        self.supplier_cost_var.set(f"⬇️ Витрати на постачальника (доставлені замовлення): {supplier_cost:.2f} грн")
        self.payroll_cost_var.set(f"⬇️ Витрати на зарплату (загально): {payroll_cost:.2f} грн")
        
        color = "green" if profit >= 0 else "red"
        self.profit_var.set(f"🎉 Чистий прибуток: {profit:.2f} грн")
        
        # Оновлення кольору тексту для прибутку
        try:
             # Отримуємо посилання на лейбл, що відображає profit_var
            for widget in self.winfo_children():
                if isinstance(widget, ttk.Notebook):
                    report_frame = widget.winfo_children()[0]
                    summary_frame = report_frame.winfo_children()[1]
                    # Це останній лейбл у summary_frame
                    summary_frame.winfo_children()[-1].config(foreground=color) 
        except:
            pass # Ігноруємо помилки, якщо структура UI ще не повністю завантажена
        
    # --- 2. Вкладка ЗАРПЛАТИ ---
    def _setup_payroll_tab(self, frame):
        ttk.Label(frame, text="Керування Виплатами Зарплат", font=("Arial", 16, "bold")).pack(pady=10)
        
        payroll_list_frame = ttk.LabelFrame(frame, text="Список Працівників та Зарплати")
        payroll_list_frame.pack(padx=10, pady=5, fill='x')

        columns = ("id", "last_name", "first_name", "role", "salary")
        self.payroll_tree = ttk.Treeview(payroll_list_frame, columns=columns, show='headings', height=10)
        self.payroll_tree.pack(fill='x', padx=5, pady=5)

        self.payroll_tree.heading("id", text="ID")
        self.payroll_tree.heading("last_name", text="Прізвище")
        self.payroll_tree.heading("first_name", text="Ім'я")
        self.payroll_tree.heading("role", text="Посада")
        self.payroll_tree.heading("salary", text="Ставка, грн")
        
        self.payroll_tree.column("id", width=50, stretch=tk.NO)
        self.payroll_tree.column("salary", width=120, anchor=tk.E)
        
        self.total_salary_var = tk.StringVar(value="Загальна сума до виплати: 0.00 грн")
        ttk.Label(frame, textvariable=self.total_salary_var, font=("Arial", 12, "bold")).pack(pady=10)

        ttk.Button(frame, text="💸 ЗАПЛАТИТИ ЗАРПЛАТУ (Зафіксувати Витрати)", command=self._pay_salaries).pack(pady=10)
        
        self._load_payroll_list()

    def _load_payroll_list(self):
        """Завантажує список працівників та розраховує загальну суму."""
        if not hasattr(self, 'payroll_tree'): return
        for item in self.payroll_tree.get_children():
            self.payroll_tree.delete(item)
            
        employees = self.controller.db.get_employee_salaries()
        total_salary = 0.0
        
        for emp in employees:
            salary = emp[4]
            total_salary += salary
            self.payroll_tree.insert("", tk.END, values=(
                emp[0], # id
                emp[1], # last_name
                emp[2], # first_name
                emp[3].capitalize(), # role
                f"{salary:.2f}" # salary
            ))
            
        self.total_salary_var.set(f"Загальна сума до виплати: {total_salary:.2f} грн")
        self.current_total_salary = total_salary
        
    def _pay_salaries(self):
        """Фіксує виплату зарплат як загальну витрату."""
        total_amount = self.current_total_salary
        user_id = self.controller.current_user[0] # Бухгалтер, який фіксує операцію
        
        if total_amount <= 0:
            messagebox.showwarning("Помилка", "Немає працівників для виплати або сума дорівнює 0.")
            return

        if messagebox.askyesno("Підтвердження Виплати", f"Ви впевнені, що хочете зафіксувати виплату зарплат на загальну суму {total_amount:.2f} грн? Ця сума буде додана до витрат."):
            if self.controller.db.log_payroll_payment(user_id, total_amount):
                messagebox.showinfo("Успіх", f"Виплата зарплат на суму {total_amount:.2f} грн успішно зафіксована у витратах.")
                self._load_reports() # Оновлюємо звіт про витрати
            else:
                messagebox.showerror("Помилка", "Не вдалося зафіксувати виплату.")

    # --- 3. Вкладка ПРИХІД ВІД ПОСТАЧАЛЬНИКА (Бухгалтер) ---
    def _setup_supply_tab(self, frame):
        ttk.Label(frame, text="Замовлення, які очікують оприбуткування", font=("Arial", 14, "bold")).pack(pady=10)
        
        columns = ("id", "product_name", "quantity", "price_supply", "date", "status", "product_id_hidden")
        self.accountant_supply_tree = ttk.Treeview(frame, columns=columns, show='headings')
        self.accountant_supply_tree.pack(fill='both', expand=True, padx=5, pady=5)

        self.accountant_supply_tree.heading("id", text="№")
        self.accountant_supply_tree.heading("product_name", text="Товар")
        self.accountant_supply_tree.heading("quantity", text="Кількість")
        self.accountant_supply_tree.heading("price_supply", text="Ціна закупівлі")
        self.accountant_supply_tree.heading("date", text="Дата Замовлення")
        self.accountant_supply_tree.heading("status", text="Статус")
        
        self.accountant_supply_tree.column("id", width=50, stretch=tk.NO, anchor=tk.CENTER)
        self.accountant_supply_tree.column("quantity", width=80, anchor=tk.CENTER)
        self.accountant_supply_tree.column("price_supply", width=120, anchor=tk.E)
        self.accountant_supply_tree.column("product_id_hidden", width=0, stretch=tk.NO)

        ttk.Button(frame, text="✅ Оприбуткувати Вибране (На Склад)", command=self._confirm_arrival).pack(pady=10)

        self._load_supplier_orders()

    def _load_supplier_orders(self):
        if not hasattr(self, 'accountant_supply_tree'): return
        for item in self.accountant_supply_tree.get_children():
            self.accountant_supply_tree.delete(item)
            
        orders = self.controller.db.get_supplier_orders()
        
        for order in orders:
             self.accountant_supply_tree.insert("", tk.END, values=(
                order[0], # id
                order[1], # product_name
                order[2], # quantity
                f"{order[6]:.2f} грн", # Ціна закупівлі
                order[3].split(' ')[0], # date
                order[4].capitalize(), # status
                order[5] # product_id (hidden, 5-й індекс)
            ))
            
    def _confirm_arrival(self):
        selected_item = self.accountant_supply_tree.focus()
        if not selected_item:
            messagebox.showwarning("Помилка", "Будь ласка, оберіть замовлення.")
            return

        values = self.accountant_supply_tree.item(selected_item, 'values')
        order_id = values[0]
        product_name = values[1]
        quantity = int(values[2])
        product_id = values[5]
        
        if values[5].lower() == 'доставлено': # статус знаходиться на індексі 5
            messagebox.showinfo("Інфо", "Цей товар вже був оприбуткований.")
            return

        if messagebox.askyesno("Підтвердження Приходу", 
                                f"ПІДТВЕРДИТИ надходження товару: '{product_name}', Кількість: {quantity}? (Склад буде оновлено, витрати зафіксовані)"):
            if self.controller.db.confirm_supply_arrival(order_id, product_id, quantity):
                messagebox.showinfo("Успіх", f"Прихід товару '{product_name}' оприбутковано, склад оновлено!")
                self._load_supplier_orders()
                self._load_reports() # Оновлюємо фінансовий звіт, оскільки витрати зросли
            else:
                messagebox.showerror("Помилка", "Не вдалося оприбуткувати товар.")

# --- Панель ДИРЕКТОРА (ОНОВЛЕНО: додана звітність) ---
class DirectorDashboard(ttk.Frame):
    def __init__(self, parent, controller):
        super().__init__(parent)
        self.controller = controller
        
        notebook = ttk.Notebook(self)
        notebook.pack(expand=True, fill="both", padx=10, pady=10)

        # 1. Звітність та Аналіз
        report_frame = ttk.Frame(notebook)
        notebook.add(report_frame, text="👑 Звітність та Аналіз")
        self._setup_report_tab(report_frame)
        
        notebook.bind("<<NotebookTabChanged>>", self._on_tab_change)
        
        ttk.Button(self, text="Вийти", command=controller.logout).pack(pady=20)

    def load_data(self):
        """Оновлення даних при вході."""
        self._load_reports()

    def _on_tab_change(self, event):
        selected_tab = event.widget.tab(event.widget.select(), "text")
        if selected_tab == "👑 Звітність та Аналіз":
            self._load_reports()

    def _setup_report_tab(self, frame):
        ttk.Label(frame, text="Фінансовий Аналіз (Доходи та Витрати)", font=("Arial", 16, "bold")).pack(pady=20)
        
        # Фрейм для відображення ключових показників (скопійовано з AccountantDashboard)
        summary_frame = ttk.LabelFrame(frame, text="Основні Показники")
        summary_frame.pack(padx=10, pady=10, fill='x')
        
        self.revenue_var = tk.StringVar(value="Дохід від продажів: 0.00 грн")
        self.supplier_cost_var = tk.StringVar(value="Витрати на постачальника: 0.00 грн")
        self.payroll_cost_var = tk.StringVar(value="Витрати на зарплату: 0.00 грн")
        self.profit_var = tk.StringVar(value="Чистий прибуток: 0.00 грн")
        
        ttk.Label(summary_frame, textvariable=self.revenue_var, font=("Arial", 12, "bold"), foreground="green").pack(pady=5, anchor='w', padx=10)
        ttk.Label(summary_frame, textvariable=self.supplier_cost_var, font=("Arial", 12, "bold"), foreground="red").pack(pady=5, anchor='w', padx=10)
        ttk.Label(summary_frame, textvariable=self.payroll_cost_var, font=("Arial", 12, "bold"), foreground="red").pack(pady=5, anchor='w', padx=10)
        ttk.Separator(summary_frame, orient='horizontal').pack(fill='x', pady=5, padx=10)
        ttk.Label(summary_frame, textvariable=self.profit_var, font=("Arial", 14, "bold"), foreground="blue").pack(pady=10, anchor='w', padx=10)

        self._load_reports()
        
    def _load_reports(self):
        """Отримує та відображає фінансові звіти."""
        if not hasattr(self, 'revenue_var'): return

        revenue = self.controller.db.get_sales_revenue()
        supplier_cost = self.controller.db.get_supplier_expenses()
        payroll_cost = self.controller.db.get_total_payroll_expenses()
        
        total_costs = supplier_cost + payroll_cost
        profit = revenue - total_costs
        
        self.revenue_var.set(f"💰 Дохід від продажів (виконані замовлення): {revenue:.2f} грн")
        self.supplier_cost_var.set(f"⬇️ Витрати на постачальника (доставлені замовлення): {supplier_cost:.2f} грн")
        self.payroll_cost_var.set(f"⬇️ Витрати на зарплату (загально): {payroll_cost:.2f} грн")
        
        color = "green" if profit >= 0 else "red"
        self.profit_var.set(f"🎉 Чистий прибуток: {profit:.2f} грн")
        
        # Оновлення кольору тексту для прибутку
        try:
            # Знаходимо останній лейбл (Чистий прибуток) у summary_frame
            summary_frame = self.winfo_children()[0].winfo_children()[0].winfo_children()[1]
            summary_frame.winfo_children()[-1].config(foreground=color) 
        except:
            pass # Ігноруємо помилки, якщо структура UI ще не повністю завантажена


if __name__ == "__main__":
    
    app = TradingApp()
    app.mainloop()
