# Лабораторная работа. Создание базы данных и связанных таблиц (MinecraftServer)

ДЗ на 17.09. Создание базы данных, таблиц со связями, наполнение данными и построение диаграммы.

---

## Задание 1. Создание базы данных

<!-- TODO: формулировка восстановлена по решению, проверить -->
### Что требовалось
Создать базу данных.

### Что сделано
Создана база данных MinecraftServer.

### Результат
<img width="963" height="304" alt="image" src="https://github.com/user-attachments/assets/c19a57f0-fe9f-4e3a-8684-5984011b9989" />

### Вывод
База данных создана и готова к добавлению таблиц.

---

## Задание 2. Создание таблиц

<!-- TODO: формулировка восстановлена по решению, проверить -->
### Что требовалось
Создать 5 таблиц, одна из которых не связана с остальными.

### Что сделано
Созданы пять таблиц по тематике Minecraft-сервера:

- ServerRules — правила сервера (таблица без связей).
- Players — игроки.
- Shops — магазины, связаны с Players через OwnerID.
- Items — предметы.
- Trades — сделки, связаны с Shops и Items.

```sql
USE MinecraftServer;
GO

-- Таблица без связей
CREATE TABLE ServerRules (
	RuleID INT IDENTITY(1,1) PRIMARY KEY,
	RuleText NVARCHAR(255) NOT NULL,
	Pentaly NVARCHAR(100)
);

-- таблица с игроками
CREATE TABLE Players (
	PlayerID INT IDENTITY(1,1) PRIMARY KEY,
	Nickname NVARCHAR(50) NOT NULL,
	JoinDate DATE DEFAULT GETDATE()
);

-- таблица магазинов(она связанна с players)
CREATE TABLE Shops (
	ShopID INT IDENTITY(1,1) PRIMARY KEY,
	OwnerID INT FOREIGN KEY REFERENCES Players(PlayerID),
	LocationsCoords NVARCHAR(50),
	ShopName NVARCHAR(100)
);

-- таблица предметов
CREATE TABLE Items (
    ItemID INT IDENTITY(1,1) PRIMARY KEY,
    ItemName NVARCHAR(50) NOT NULL,
    BasePrice MONEY
);

-- сделки
CREATE TABLE Trades (
    TradeID INT IDENTITY(1,1) PRIMARY KEY,
    ShopID INT FOREIGN KEY REFERENCES Shops(ShopID),
    ItemID INT FOREIGN KEY REFERENCES Items(ItemID),
    Quantity INT,
    TradeDate DATETIME DEFAULT GETDATE()
);
```

### Анализ
Связи между таблицами: Players → Shops (один игрок — несколько магазинов), Shops → Trades и Items → Trades (таблица Trades связывает магазины и предметы). ServerRules остаётся независимой.

### Вывод
Созданы пять таблиц с первичными и внешними ключами, одна из них изолирована от остальных.

---

## Задание 3. Наполнение данными и проверка

<!-- TODO: формулировка восстановлена по решению, проверить -->
### Что требовалось
Заполнить таблицы данными и вывести по одной строке из каждой таблицы.

### Что сделано
Во все таблицы добавлены тестовые записи, затем из каждой выведена одна строка через SELECT TOP 1.

```sql
-- заполение данными
INSERT INTO ServerRules (RuleText, Pentaly) VALUES ('Запрещено дюпать шмот', 'Бан на долго!')
INSERT INTO Players (Nickname) VALUES ('maxcombo_201'), ('Edgar_77772');
INSERT INTO Shops (OwnerID, LocationsCoords, ShopName) VALUES (1, 'X:42 Y:67 Z:52', 'Человек у Шлепок');
INSERT INTO Items (ItemName, BasePrice) VALUES ('Алмаз', 50.00), ('Железный слиток', 10.00);
INSERT INTO Trades (ShopID, ItemID, Quantity) VALUES (1, 1, 5);
GO
```

```sql
SELECT TOP 1 * FROM ServerRules;
SELECT TOP 1 * FROM Players;
SELECT TOP 1 * FROM Shops;
SELECT TOP 1 * FROM Items;
SELECT TOP 1 * FROM Trades;
GO
```

### Результат
<img width="448" height="421" alt="image" src="https://github.com/user-attachments/assets/11924e70-96b1-4960-ad9c-3758b61a34ac" />

### Вывод
Данные добавлены, связи между таблицами работают: ключи в Shops и Trades ссылаются на существующие записи.

---

## Задание 4. Диаграмма и файл базы

<!-- TODO: формулировка восстановлена по решению, проверить -->
### Что требовалось
Построить диаграмму базы данных и приложить саму базу.

### Что сделано
Построена диаграмма связей таблиц, а скрипт базы данных выгружен в файл.

### Результат
Диаграмма:

<img width="825" height="752" alt="Снимок экрана 2026-09-17 214528" src="https://github.com/user-attachments/assets/9d5a74f7-ba1c-45c6-8f79-6a30c1184bb8" />

Сама БД: [minecraftServerDB.sql](https://github.com/user-attachments/files/32349498/minecraftServerDB.sql)

### Вывод
Диаграмма наглядно показывает структуру базы и связи между таблицами.

---

## Общий вывод

| Задание | Что показал |
|---|---|
| 1 | Создание базы данных |
| 2 | Таблицы с первичными и внешними ключами |
| 3 | Наполнение данными и проверка через SELECT TOP 1 |
| 4 | Диаграмма базы и выгрузка скрипта |

**Ключевые принципы:**
1. PRIMARY KEY с IDENTITY — автоматическая нумерация записей.
2. FOREIGN KEY обеспечивает целостность связей между таблицами.
3. Таблица-связка (Trades) соединяет несколько сущностей.
4. Значения по умолчанию (DEFAULT GETDATE()) заполняют даты автоматически.

---

## Стек

Microsoft SQL Server 2017, SSMS 22, база MinecraftServer.
