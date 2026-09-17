ДЗ на 17.09

* Создам БД:
> <img width="963" height="304" alt="image" src="https://github.com/user-attachments/assets/c19a57f0-fe9f-4e3a-8684-5984011b9989" />

* Сделаю 5 таблиц, 1 из которых не будет с ними связана:
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
* Заполню их данными:
```sql
-- заполение данными
INSERT INTO ServerRules (RuleText, Pentaly) VALUES ('Запрещено дюпать шмот', 'Бан на долго!')
INSERT INTO Players (Nickname) VALUES ('maxcombo_201'), ('Edgar_77772');
INSERT INTO Shops (OwnerID, LocationsCoords, ShopName) VALUES (1, 'X:42 Y:67 Z:52', 'Человек у Шлепок');
INSERT INTO Items (ItemName, BasePrice) VALUES ('Алмаз', 50.00), ('Железный слиток', 10.00);
INSERT INTO Trades (ShopID, ItemID, Quantity) VALUES (1, 1, 5);
GO
```
* Вывод одной строки их каждой таблицы
```sql
SELECT TOP 1 * FROM ServerRules;
SELECT TOP 1 * FROM Players;
SELECT TOP 1 * FROM Shops;
SELECT TOP 1 * FROM Items;
SELECT TOP 1 * FROM Trades;
GO
```

* Результат выполнения запроса:
<img width="448" height="421" alt="image" src="https://github.com/user-attachments/assets/11924e70-96b1-4960-ad9c-3758b61a34ac" />

* Диаграма:
<img width="825" height="752" alt="Снимок экрана 2026-09-17 214528" src="https://github.com/user-attachments/assets/9d5a74f7-ba1c-45c6-8f79-6a30c1184bb8" />

* Сама БД:
[minecraftServerDB.sql](https://github.com/user-attachments/files/32349498/minecraftServerDB.sql)
