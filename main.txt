Перед началом: виртуальная машина с Альт Линукс 10, войти как root (su -)

================================================================================
ЧАСТЬ 1. НАСТРОЙКА СЕРВЕРА (LINUX) - МОДУЛЬ 1
================================================================================

# 1.1 Обновление пакетов
apt-get update

# 1.2 Установка MySQL-сервера (именно mysql-server, не mariadb)
apt-get install -y mysql-server

# 1.3 (Опционально) Установка MySQL Workbench на сервер (не обязательно, но можно)
apt-get install -y mysql-workbench-community

# 1.4 Включение и запуск сервера
systemctl enable --now mysqld
systemctl status mysqld.service   # проверить, что active

# 1.5 Разрешаем удалённые подключения (отключаем skip-networking)
sed -i "s/skip-networking/#skip-networking/g" /etc/my.cnf.d/server.cnf

# 1.6 Проверяем, какие порты слушает (должен быть 0.0.0.0:3306)
ss -tlpn

# 1.7 Вход в MySQL как root (первый раз без пароля)
mysql -u root

# --- Внутри MySQL ---
# Меняем пароль root
ALTER USER 'root'@'localhost' IDENTIFIED BY 'root';
FLUSH PRIVILEGES;
EXIT;
# -------------------

# 1.8 Перезапускаем MySQL
systemctl restart mysqld

# 1.9 Теперь заходим с паролем
mysql -u root -p   # пароль root

# --- Внутри MySQL ---
# Даём права root на удалённый доступ (на все хосты)
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'root' WITH GRANT OPTION;
# (Если ошибка, то сначала UPDATE mysql.user SET host='%' WHERE user='root';)
UPDATE mysql.user SET host='%' WHERE user='root';
FLUSH PRIVILEGES;

# Проверяем, что host изменился
SELECT user, host FROM mysql.user;

# Создаём специального пользователя 'sa' (как в задании) – номер места 19 заменить на свой
CREATE USER 'sa'@'%' IDENTIFIED BY 'De_19';
# Для совместимости с Workbench используем mysql_native_password
ALTER USER 'sa'@'%' IDENTIFIED WITH mysql_native_password BY 'De_19';
FLUSH PRIVILEGES;

# Проверяем плагин
SELECT user, host, plugin FROM mysql.user;

EXIT;
# -------------------

# 1.10 Меняем имя сервера (хоста) в ОС
hostnamectl set-hostname Server_19   # замените 19 на номер вашего места

# 1.11 Проверяем, что MySQL видит новое имя сервера (заходим в MySQL и выполняем)
mysql -u root -p
SHOW VARIABLES LIKE 'hostname';
EXIT;

# 1.12 Отключаем брандмауэр (если включён, чтобы не блокировал порт 3306)
systemctl stop firewalld   # или iptables -F

# 1.13 Узнаём IP-адрес сервера (понадобится для подключения из Workbench)
ip a   # запомните inet-адрес, например 192.168.1.100

================================================================================
ЧАСТЬ 2. ПОДКЛЮЧЕНИЕ ИЗ MYSQL WORKBENCH (на вашем основном ПК)
================================================================================

1. Установите MySQL Workbench на свой компьютер (Windows/Linux).
2. Запустите Workbench, нажмите "+" рядом с "MySQL Connections".
3. Заполните поля:
   - Connection Name: Экзамен
   - Hostname: IP-адрес сервера (из ip a)
   - Port: 3306
   - Username: sa
   - Password: De_19 (ваш номер)
4. Нажмите "Test Connection". Должно быть успешно.
5. Сохраните соединение.

================================================================================
ЧАСТЬ 3. ВЫПОЛНЕНИЕ ЗАДАНИЙ (SQL) В WORKBENCH
================================================================================

Внимание: Все следующие скрипты выполняются в среде MySQL Workbench после подключения к серверу.

-------------------------------------------------------------------------------
МОДУЛЬ 1 (продолжение): Автоматическое создание 10 пользователей, 10 БД и таблицы Users
-------------------------------------------------------------------------------

USE BD;   -- если BD уже создана, иначе сначала создадим

-- Создаём базу данных BD, если её нет
CREATE DATABASE IF NOT EXISTS BD;
USE BD;

-- Создаём таблицу Users (если её нет)
CREATE TABLE IF NOT EXISTS Users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50) NOT NULL,
    password_hash VARCHAR(255) NOT NULL
);

-- Скрипт для автоматического создания 10 пользователей MySQL, 10 БД и прав
-- Выполняется в цикле (хранимая процедура или直接用 WHILE)
DELIMITER $$
DROP PROCEDURE IF EXISTS create_users_dbs$$
CREATE PROCEDURE create_users_dbs()
BEGIN
    DECLARE i INT DEFAULT 1;
    DECLARE username VARCHAR(50);
    DECLARE pwd VARCHAR(5);
    DECLARE dbname VARCHAR(10);
    DECLARE sql_stmt TEXT;

    WHILE i <= 10 DO
        SET username = CONCAT('user', i);
        SET pwd = SUBSTRING(MD5(RAND()), 1, 5);
        SET dbname = CONCAT('BD', i);

        SET sql_stmt = CONCAT('CREATE USER IF NOT EXISTS ''', username, '''@''%'' IDENTIFIED BY ''', pwd, '''');
        PREPARE stmt FROM sql_stmt; EXECUTE stmt; DEALLOCATE PREPARE stmt;

        SET sql_stmt = CONCAT('CREATE DATABASE IF NOT EXISTS ', dbname);
        PREPARE stmt FROM sql_stmt; EXECUTE stmt; DEALLOCATE PREPARE stmt;

        SET sql_stmt = CONCAT('GRANT ALL PRIVILEGES ON ', dbname, '.* TO ''', username, '''@''%''');
        PREPARE stmt FROM sql_stmt; EXECUTE stmt; DEALLOCATE PREPARE stmt;

        -- Сохраняем логин и пароль в таблицу Users (пока в открытом виде)
        INSERT INTO Users (username, password_hash) VALUES (username, pwd);

        SET i = i + 1;
    END WHILE;
    FLUSH PRIVILEGES;
END$$
DELIMITER ;

-- Запускаем процедуру
CALL create_users_dbs();

-- Проверяем, что таблица Users заполнена
SELECT * FROM Users;

-------------------------------------------------------------------------------
МОДУЛЬ 2: Шифрование паролей (AES)
-------------------------------------------------------------------------------

-- Устанавливаем ключ шифрования (должен быть одинаков при шифровании и расшифровке)
SET @key_str = SHA2('MySecretKey2026', 512);

-- Шифруем все пароли в таблице Users (сохраняем в шестнадцатеричном виде)
UPDATE Users SET password_hash = HEX(AES_ENCRYPT(password_hash, @key_str));

-- Проверка расшифровки (для отладки)
SELECT id, username, CAST(AES_DECRYPT(UNHEX(password_hash), @key_str) AS CHAR) AS decrypted_password
FROM Users;

-------------------------------------------------------------------------------
МОДУЛЬ 3: Резервное копирование и восстановление базы данных BD
-------------------------------------------------------------------------------

-- Команды выполняются в терминале Linux (не в Workbench)
-- Замените 19 на номер вашего места

# Создание резервной копии
mysqldump -u sa -pDe_19 BD > /tmp/BD_backup_19.sql

# Восстановление из резервной копии (если потребуется)
mysql -u sa -pDe_19 BD < /tmp/BD_backup_19.sql

-------------------------------------------------------------------------------
МОДУЛЬ 4: Проектирование БД на основе ER-диаграммы (предметная область "Продукция-Заказчики")
-------------------------------------------------------------------------------

USE BD;

-- Таблица "Продукция"
CREATE TABLE IF NOT EXISTS Product (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL,
    material_cost DECIMAL(10,2) NOT NULL
);

-- Таблица "Заказчик"
CREATE TABLE IF NOT EXISTS Customer (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL
);

-- Таблица "Заказ"
CREATE TABLE IF NOT EXISTS Orders (
    id INT PRIMARY KEY AUTO_INCREMENT,
    customer_id INT NOT NULL,
    order_date DATE NOT NULL,
    total_amount DECIMAL(10,2) DEFAULT 0,
    FOREIGN KEY (customer_id) REFERENCES Customer(id) ON DELETE CASCADE
);

-- Таблица "Состав заказа" (связь многие-ко-многим)
CREATE TABLE IF NOT EXISTS OrderItem (
    order_id INT NOT NULL,
    product_id INT NOT NULL,
    quantity INT NOT NULL,
    price DECIMAL(10,2) NOT NULL,
    PRIMARY KEY (order_id, product_id),
    FOREIGN KEY (order_id) REFERENCES Orders(id) ON DELETE CASCADE,
    FOREIGN KEY (product_id) REFERENCES Product(id)
);

-- Добавление тестовых данных (не менее 3 записей в каждую таблицу)
INSERT INTO Product (name, material_cost) VALUES
('Стол', 500.00), ('Стул', 200.00), ('Шкаф', 1000.00);

INSERT INTO Customer (name) VALUES
('ООО "Ромашка"'), ('ИП Иванов'), ('ЗАО "Весна"');

INSERT INTO Orders (customer_id, order_date) VALUES
(1, '2025-01-15'), (2, '2025-02-20'), (3, '2025-03-10');

INSERT INTO OrderItem (order_id, product_id, quantity, price) VALUES
(1, 1, 2, 1500.00), (1, 2, 4, 600.00),
(2, 3, 1, 3000.00),
(3, 1, 1, 1600.00), (3, 2, 2, 650.00);

-- Инструкция по созданию ER-диаграммы в Workbench:
-- Меню Database → Reverse Engineer → выбрать BD → далее → готово.
-- Экспорт: File → Export → Save as PDF.

-------------------------------------------------------------------------------
МОДУЛЬ 5: Создание процедуры и триггера
-------------------------------------------------------------------------------

USE BD;

-- Процедура, выводящая отчёт за период (даты как параметры)
DELIMITER $$
CREATE PROCEDURE GetOrderReport(IN start_date DATE, IN end_date DATE)
BEGIN
    SELECT
        c.name AS customer,
        COUNT(DISTINCT o.id) AS total_orders,
        SUM(oi.quantity) AS total_products,
        SUM(oi.quantity * oi.price) AS total_sum
    FROM Orders o
    JOIN Customer c ON o.customer_id = c.id
    JOIN OrderItem oi ON o.id = oi.order_id
    WHERE o.order_date BETWEEN start_date AND end_date
    GROUP BY c.id;
END$$
DELIMITER ;

-- Триггер, автоматически рассчитывающий итоговую сумму заказа после добавления позиции
DELIMITER $$
CREATE TRIGGER update_order_total
AFTER INSERT ON OrderItem
FOR EACH ROW
BEGIN
    UPDATE Orders
    SET total_amount = (
        SELECT SUM(quantity * price)
        FROM OrderItem
        WHERE order_id = NEW.order_id
    )
    WHERE id = NEW.order_id;
END$$
DELIMITER ;

-- Проверка работы:
CALL GetOrderReport('2025-01-01', '2025-12-31');
-- Проверка триггера: добавим новый заказ и позицию, затем посмотрим Orders.total_amount

-------------------------------------------------------------------------------
МОДУЛЬ 6: Написание запросов (архивация заказов за месяц)
-------------------------------------------------------------------------------

USE BD;

-- Пример: архивируем заказы за январь 2025
SET @month = 1;
SET @year = 2025;
SET @table_archive = CONCAT('Orders_', @year, '_', LPAD(@month,2,'0'));

SET @sql_create = CONCAT('CREATE TABLE IF NOT EXISTS ', @table_archive, ' LIKE Orders');
PREPARE stmt FROM @sql_create; EXECUTE stmt; DEALLOCATE PREPARE stmt;

SET @sql_insert = CONCAT('INSERT INTO ', @table_archive, ' SELECT * FROM Orders WHERE MONTH(order_date) = ', @month, ' AND YEAR(order_date) = ', @year);
PREPARE stmt FROM @sql_insert; EXECUTE stmt; DEALLOCATE PREPARE stmt;

SET @sql_delete = CONCAT('DELETE FROM Orders WHERE MONTH(order_date) = ', @month, ' AND YEAR(order_date) = ', @year);
PREPARE stmt FROM @sql_delete; EXECUTE stmt; DEALLOCATE PREPARE stmt;

-- Посмотреть результат
SELECT 'Архив' AS source, * FROM Orders_2025_01
UNION ALL
SELECT 'Остались' AS source, * FROM Orders;

================================================================================
ЧАСТЬ 4. PYTHON-ОБЁРТКА (МОДУЛЬ 7) - СКЕЛЕТ ДЛЯ ЛЮБОЙ ПРЕДМЕТНОЙ ОБЛАСТИ
================================================================================

import sys
import pymysql
from PyQt6.QtWidgets import *
from PyQt6.QtCore import Qt
from PyQt6.QtGui import QColor

class ExamApp(QMainWindow):
    def __init__(self):
        super().__init__()
        self.setWindowTitle("ДЭ - Администратор БД")
        self.setGeometry(100, 100, 1100, 600)

        # Параметры подключения к MySQL
        self.connection = pymysql.connect(
            host='127.0.0.1',
            user='root',
            password='root',
            database='BD',
            charset='utf8mb4',
            cursorclass=pymysql.cursors.DictCursor
        )

        # Интерфейс
        central = QWidget()
        self.setCentralWidget(central)
        layout = QVBoxLayout(central)

        self.table = QTableWidget()
        layout.addWidget(self.table)

        # Панель управления
        panel = QHBoxLayout()

        self.sort_combo = QComboBox()
        self.sort_combo.addItems(['customer', 'order_date', 'total_amount'])
        self.sort_btn = QPushButton('Сортировать')
        self.sort_btn.clicked.connect(self.sort_data)

        self.filter_combo = QComboBox()
        self.filter_combo.addItem('Все', None)
        self.load_customers()
        self.filter_btn = QPushButton('Фильтровать')
        self.filter_btn.clicked.connect(self.filter_data)
        self.reset_btn = QPushButton('Показать все')
        self.reset_btn.clicked.connect(self.load_data)

        self.search_edit = QLineEdit()
        self.search_edit.setPlaceholderText('Поиск...')
        self.search_btn = QPushButton('Найти')
        self.search_btn.clicked.connect(self.search_data)

        panel.addWidget(QLabel('Сортировка:'))
        panel.addWidget(self.sort_combo)
        panel.addWidget(self.sort_btn)
        panel.addWidget(QLabel('Клиент:'))
        panel.addWidget(self.filter_combo)
        panel.addWidget(self.filter_btn)
        panel.addWidget(self.reset_btn)
        panel.addWidget(self.search_edit)
        panel.addWidget(self.search_btn)
        layout.addLayout(panel)

        self.status_label = QLabel()
        layout.addWidget(self.status_label)

        self.load_data()

    def load_customers(self):
        cursor = self.connection.cursor()
        cursor.execute("SELECT id, name FROM Customer")
        for row in cursor.fetchall():
            self.filter_combo.addItem(row['name'], row['id'])
        cursor.close()

    def load_data(self, filter_customer=None):
        query = """
            SELECT o.id, c.name AS customer, o.order_date, o.total_amount
            FROM Orders o
            JOIN Customer c ON o.customer_id = c.id
        """
        params = ()
        if filter_customer:
            query += " WHERE c.id = %s"
            params = (filter_customer,)

        cursor = self.connection.cursor()
        cursor.execute(query, params)
        rows = cursor.fetchall()
        cursor.close()
        self.display_data(rows)

    def filter_data(self):
        cust_id = self.filter_combo.currentData()
        if cust_id is not None:
            self.load_data(filter_customer=cust_id)
        else:
            self.load_data()

    def display_data(self, rows):
        if not rows:
            self.table.setRowCount(0)
            self.status_label.setText("Всего заказов: 0")
            self.current_rows = []
            return

        columns = ['id', 'customer', 'order_date', 'total_amount']
        self.table.setColumnCount(len(columns))
        self.table.setHorizontalHeaderLabels(columns)
        self.table.setRowCount(len(rows))

        for i, row in enumerate(rows):
            self.table.setItem(i, 0, QTableWidgetItem(str(row['id'])))
            self.table.setItem(i, 1, QTableWidgetItem(row['customer']))
            self.table.setItem(i, 2, QTableWidgetItem(str(row['order_date'])))
            self.table.setItem(i, 3, QTableWidgetItem(str(row['total_amount'])))

        self.status_label.setText(f"Всего заказов: {len(rows)}")
        self.current_rows = r
ows

    def sort_data(self):
        col = self.sort_combo.currentText()

if not hasattr(self, 'current_rows') or not self.current_rows:
            return

        col_index = {'customer': 1, 'order_date': 2, 'total_amount': 3}.get(col, 1)

        if col == 'total_amount':
            self.current_rows.sort(key=lambda x: float(x[col_index]) if x[col_index] else 0)
        else:
            self.current_rows.sort(key=lambda x: str(x[col_index]))

        self.display_data(self.current_rows)

    def search_data(self):
        search_text = self.search_edit.text().lower()
        if not search_text or not hasattr(self, 'current_rows'):
            return

        for row in range(self.table.rowCount()):
            found = False
            for col in range(self.table.columnCount()):
                item = self.table.item(row, col)
                if item and search_text in item.text().lower():
                    found = True
                    break

            for col in range(self.table.columnCount()):
                item = self.table.item(row, col)
                if item:
                    item.setBackground(QColor(255, 255, 0) if found else QColor(255, 255, 255))

    def closeEvent(self, event):
        self.connection.close()
        event.accept()

if name == '__main__':
    app = QApplication(sys.argv)
    window = ExamApp()
    window.show()
    sys.exit(app.exec())


"Руководство пользователя модуля анализа заказов".

Титульный лист (название, ФИО, группа).
Назначение программы (анализ заказов: сортировка, фильтрация, поиск, статистика).
Требования к ПО (Python 3, PyQt6, pymysql, MySQL сервер).
Установка и запуск (как запустить exam_app.py).
Описание интерфейса (скриншот главного окна).
Инструкция по сортировке (выбрать поле, нажать "Сортировать").
Инструкция по фильтрации по клиенту (выбрать клиента, нажать "Фильтровать").
Инструкция по поиску с подсветкой (ввести текст, нажать "Найти").
Отображение статистики (количество заказов и общая сумма).
Заключение.
