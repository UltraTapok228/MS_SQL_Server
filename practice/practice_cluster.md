# Работа с индексами в SQL Server

**Дата:** 01.10.2026

---

## 1. Создание и заполнение базы данных

Мы работаем с базой данных `IndexDemo`.

```sql
CREATE DATABASE IndexDemo;
GO
USE IndexDemo;
GO

CREATE TABLE dbo.Customers (
    CustomerID INT IDENTITY(1,1) NOT NULL,
    Email NVARCHAR(100) NOT NULL,
    LastName NVARCHAR(50) NOT NULL,
    FirstName NVARCHAR(50) NOT NULL,
    City NVARCHAR(50) NOT NULL,
    RegistrationDate DATE NOT NULL,
    Notes NVARCHAR(500) NULL
);
GO
```

### Заполнение тестовыми данными

Заполняем таблицу `Customers` **50 000 строками** тестовых данных:

```sql
INSERT INTO dbo.Customers (Email, LastName, FirstName, City, RegistrationDate, Notes)
SELECT TOP (50000)
    'user' + CAST(ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) AS NVARCHAR(10)) + '@example.com',
    'LastName' + CAST(ABS(CHECKSUM(NEWID())) % 1000 AS NVARCHAR(10)),
    'FirstName' + CAST(ABS(CHECKSUM(NEWID())) % 500 AS NVARCHAR(10)),
    CASE ABS(CHECKSUM(NEWID())) % 5
        WHEN 0 THEN N'Москва'
        WHEN 1 THEN N'Санкт-Петербург'
        WHEN 2 THEN N'Новосибирск'
        WHEN 3 THEN N'Екатеринбург'
        ELSE N'Казань'
    END,
    DATEADD(DAY, ABS(CHECKSUM(NEWID())) % 3650, '2015-01-01'),
    REPLICATE(N'A', 500)
FROM sys.all_objects a
CROSS JOIN sys.all_objects b;
GO
```

### Проверка количества строк

```sql
SELECT COUNT(*) AS TotalRows FROM dbo.Customers;
```

* Создание базы IndexDemo и таблицы Customers с генерацией 50 000 строк тестовых данных создает необходимый массив информации. На маленьких таблицах (сотни строк) выгода от индексов незаметна, но объем в 50 000 записей позволяет наглядно увидеть разницу между прямым поиском и полным сканированием.

---

# 2. Запрос без индекса

### Запрос

```sql
SET STATISTICS IO ON;
SET STATISTICS TIME ON;

SELECT CustomerID, Email, LastName
FROM dbo.Customers
WHERE Email = 'user12345@example.com';
```

### Результат

![Результат запроса без индекса](https://github.com/user-attachments/assets/60622746-9ed1-4e24-9551-6e68e9af0f9c)

### Что произошло?

* Поиск по конкретному Email без созданного индекса заставляет SQL Server использовать самый ресурсоемкий метод — полное сканирование таблицы (Table Scan).   Серверу приходится последовательно проверять каждую строку, чтобы найти нужное совпадение. С ростом объема данных этот процесс будет занимать все больше времени и оперативной памяти.
  
---

# 3. Кластеризованный индекс

## Создание кластеризованного индекса

```sql
CREATE CLUSTERED INDEX CIX_Customers_CustomerID
ON dbo.Customers (CustomerID);
GO

EXEC sp_helpindex 'dbo.Customers';
```

После этого выполняются следующие запросы:

```sql
SELECT CustomerID, Email, LastName
FROM dbo.Customers
WHERE CustomerID = 12345;
```

```sql
SELECT CustomerID, Email
FROM dbo.Customers
WHERE CustomerID BETWEEN 10000 AND 10100;
```

### Результат

![Результат работы кластеризованного индекса](https://github.com/user-attachments/assets/33c9c965-7992-43de-b0ab-3eb0ab7e2ab1)

![Результат диапазонного запроса](https://github.com/user-attachments/assets/c1a7d8db-cb5d-4233-b4c1-79f334229830)

### Что произошло?

Создание кластеризованного индекса по CustomerID физически упорядочивает строки на диске по этому столбцу.   Точечный поиск (CustomerID = 12345) и выборка диапазона (BETWEEN 10000 AND 10100) становятся максимально эффективными. Сервер точно знает, в каком месте диска лежат эти данные, и может забрать диапазон за одно последовательное чтение.

---

# 4. Некластеризованный индекс

## Создание индекса по Email

```sql
CREATE NONCLUSTERED INDEX NCIX_Customers_Email
ON dbo.Customers (Email);
GO
```

После этого индекс удаляется:

```sql
DROP INDEX NCIX_Customers_Email ON dbo.Customers;
```

И создаётся заново с включённым столбцом `LastName`:

```sql
CREATE NONCLUSTERED INDEX NCIX_Customers_Email
ON dbo.Customers (Email)
INCLUDE (LastName);
GO
```

### Запрос

```sql
SELECT CustomerID, Email, LastName
FROM dbo.Customers
WHERE Email = 'user12345@example.com';
```

---

## Индекс по City

```sql
CREATE NONCLUSTERED INDEX NCIX_Customers_City
ON dbo.Customers (City);
GO
```

### Запрос, возвращающий ~20% строк (Москва)

```sql
SELECT *
FROM dbo.Customers
WHERE City = N'Москва';
```

### Результат

![Результат поиска по Email](https://github.com/user-attachments/assets/a4610332-6a00-4630-a0cf-8d55e4d9e7d8)

![Результат поиска по City](https://github.com/user-attachments/assets/b6810332-0369-4605-9034-cd8a3dad0e32)

### Что произошло?

Индекс по Email с INCLUDE: Некластеризованный индекс создает отдельную структуру для быстрого поиска уникальных значений. Добавление конструкции INCLUDE (LastName) делает индекс «покрывающим». Это значит, что SQL Server может отдать запрошенные данные (Email и LastName) напрямую из структуры индекса, не тратя ресурсы на чтение основной таблицы.   Индекс по City (Низкая селективность): В таблице генерируется всего 5 вариантов городов, из-за чего город «Москва» встречается примерно у 20% строк. В таких случаях некластеризованный индекс теряет свою эффективность. Оптимизатору SQL Server часто дешевле просканировать всю таблицу целиком, чем обращаться к индексу для каждой пятой записи.
---

# 5. Работа с запросами

## Первая вставка данных

Выполняем:

```sql
SET STATISTICS TIME ON;

INSERT INTO dbo.Customers (Email, LastName, FirstName, City, RegistrationDate, Notes)
SELECT TOP (1000)
    'newuser' + CAST(ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) AS NVARCHAR(10)) + '@test.com',
    'NewLast' + CAST(ABS(CHECKSUM(NEWID())) % 100 AS NVARCHAR(10)),
    'NewFirst' + CAST(ABS(CHECKSUM(NEWID())) % 100 AS NVARCHAR(10)),
    N'Москва',
    GETDATE(),
    REPLICATE(N'B', 500)
FROM sys.all_objects;
GO
```

---

## Удаление некластеризованных индексов

```sql
DROP INDEX NCIX_Customers_Email ON dbo.Customers;
DROP INDEX NCIX_Customers_City ON dbo.Customers;
```

**Кластеризованный индекс оставляем для сравнения.**

---

## Вторая вставка данных

Вставьте ещё 1000 строк с теми же данными:

```sql
INSERT INTO dbo.Customers (Email, LastName, FirstName, City, RegistrationDate, Notes)
SELECT TOP (1000)
    'newuser2_' + CAST(ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) AS NVARCHAR(10)) + '@test.com',
    'NewLast' + CAST(ABS(CHECKSUM(NEWID())) % 100 AS NVARCHAR(10)),
    'NewFirst' + CAST(ABS(CHECKSUM(NEWID())) % 100 AS NVARCHAR(10)),
    N'Москва',
    GETDATE(),
    REPLICATE(N'B', 500)
FROM sys.all_objects;
GO
```

### Результат

![Результат вставки данных](https://github.com/user-attachments/assets/ebd441c3-d59d-419c-8608-0138fcf16586)

* Документ демонстрирует вставку 1000 строк при наличии некластеризованных индексов и вставку еще 1000 строк после их удаления.

* Этот этап наглядно показывает главный компромисс при проектировании БД: индексы критически ускоряют поиск и чтение данных, но увеличивают стоимость операций изменения (INSERT, UPDATE, DELETE). При добавлении новой строки серверу приходится не только записывать данные в таблицу, но и обновлять структуру каждого связанного индекса.

---

# 6. Статистика активных запросов

### Статистика активных запросов

![Статистика активных запросов](https://github.com/user-attachments/assets/2a5d10fb-dd92-4048-8e13-a84c4d8afb14)

![Дополнительная статистика активных запросов](https://github.com/user-attachments/assets/834a79ed-859e-4ab4-8b26-5327efe06759)

---

**Дата:** 02.10.2026

---

## Задача 1. Создание кластеризованного индекса на head-таблице
```sql
USE IndexPractice;
GO

SET STATISTICS IO ON;

-- 1. Смотрим план и logical reads ДО создания индекса (будет Table Scan)
SELECT * FROM dbo.Products WHERE ProductID = 50000;
GO

-- 2. Создаем кластеризованный индекс
CREATE CLUSTERED INDEX CIX_Products_ProductID ON dbo.Products (ProductID);
GO

-- 3. Смотрим план и logical reads ПОСЛЕ (будет Clustered Index Seek, чтений станет 2-3)
SELECT * FROM dbo.Products WHERE ProductID = 50000;
GO
```

Результат:
<img width="339" height="163" alt="image" src="https://github.com/user-attachments/assets/a3eda7c5-7123-40a1-994b-35edb74ed8c9" />

<img width="892" height="591" alt="image" src="https://github.com/user-attachments/assets/4331978d-64b7-41e2-83a5-d547bf3fafc4" />

---

## Задача 2. Некластеризованный индекс для точечного поиска
```sql
USE IndexPractice;
GO

-- 1. Создаем некластеризованный индекс по имени
CREATE NONCLUSTERED INDEX NCIX_Products_Name ON dbo.Products (Name);
GO

-- 2. Выполняем поиск конкретного товара
SELECT ProductID, Name, Price 
FROM dbo.Products 
WHERE Name = N'Товар-777';
GO
```

Результат:
<img width="898" height="453" alt="image" src="https://github.com/user-attachments/assets/db5c5c5d-7b55-4ca6-9879-0799eeaf6cc8" />

<img width="912" height="631" alt="image" src="https://github.com/user-attachments/assets/5e14afe0-83ad-403c-a3e5-cfd48cec7bef" />

---

## Задача 3. Разница между Seek и Scan
```sql
USE IndexPractice;
GO

-- Запрос 1: Точечный поиск
SELECT * FROM dbo.Orders WHERE OrderID = 100;

-- Запрос 2: Диапазонный поиск
SELECT * FROM dbo.Orders WHERE OrderID > 100 AND OrderID < 200;

-- Запрос 3: Выборка всех строк
SELECT * FROM dbo.Orders WHERE OrderID > 0;
```

Результат(план выполнения):
<img width="698" height="711" alt="image" src="https://github.com/user-attachments/assets/b08cc8f4-cc53-4e50-9380-7ba663ceacb3" />

Пояснение:
> **Запрос 1 (OrderID = 100)**: Оператор Clustered Index Seek. Сервер мгновенно спускается по B-дереву кластеризованного индекса прямо к строке с номером 100. Это самая эффективная и дешевая операция для точечного поиска.

> **Запрос 2 (OrderID > 100 AND OrderID < 200)**: Оператор Clustered Index Seek. Несмотря на то что это поиск нескольких строк, сервер использует Seek, чтобы быстро найти начальную точку (101), а затем читает данные по порядку до 199. Технически это называется Range Scan (сканирование диапазона), но в графическом плане SQL Server помечает это иконкой Seek.

> **Запрос 3 (OrderID > 0)**: Оператор Clustered Index Scan. Оптимизатор понимает, что под условие > 0 попадают все 500 000 строк таблицы. Вместо того чтобы спускаться по B-дереву 500 тысяч раз (что очень долго), ему гораздо выгоднее просто прочитать весь листовой уровень индекса (саму таблицу) от первой до последней страницы. Поэтому выбирается Scan.

---

## Задача 4. Влияние selectivity на выбор индекса
```sql
-- Создание некластеризованных индексов
CREATE NONCLUSTERED INDEX IX_Users_Gender ON dbo.Users (Gender);
CREATE NONCLUSTERED INDEX IX_Users_Email ON dbo.Users (Email);

-- Проверка влияния селективности на план выполнения
SELECT * FROM dbo.Users WHERE Gender = 'M';
SELECT * FROM dbo.Users WHERE Email = 'user150000@test.com';
```

<img width="799" height="622" alt="image" src="https://github.com/user-attachments/assets/e5c9407d-b9cb-41ff-807a-079b14af29c4" />

---

## Задача 5. Удаление и пересоздание индекса
```sql
USE IndexPractice;
GO


SELECT index_type_desc, page_count, record_count
FROM sys.dm_db_index_physical_stats(DB_ID(), OBJECT_ID('dbo.Customers'), INDEXPROPERTY(OBJECT_ID('dbo.Customers'), 'NCIX_Email', 'IndexID'), NULL, 'DETAILED');

-- 1. Удаляем индекс
DROP INDEX NCIX_Email ON dbo.Customers;

-- 2. Проверяем, что он исчез (выведет только кластерный PK)
SELECT name, type_desc FROM sys.indexes WHERE object_id = OBJECT_ID('dbo.Customers');

-- 3. Пересоздаем индекс, добавив LastName как ключевой столбец
CREATE NONCLUSTERED INDEX NCIX_Email ON dbo.Customers (Email, LastName);

-- 4. Смотрим размер ПОСЛЕ (page_count станет больше, так как индекс "потяжелел")
SELECT index_type_desc, page_count, record_count
FROM sys.dm_db_index_physical_stats(DB_ID(), OBJECT_ID('dbo.Customers'), INDEXPROPERTY(OBJECT_ID('dbo.Customers'), 'NCIX_Email', 'IndexID'), NULL, 'DETAILED');
```

Результат:
<img width="869" height="549" alt="image" src="https://github.com/user-attachments/assets/910dabad-f952-4c22-a590-7c45bccf4f05" />

<img width="892" height="793" alt="image" src="https://github.com/user-attachments/assets/91e414d2-7cfc-4832-bdee-bb5ac098804f" />

---

## Задача 6
```sql
-- 1. Подготовка
CREATE TABLE dbo.OrdersTask6 (
    OrderID INT IDENTITY(1,1) PRIMARY KEY,
    CustomerID INT NOT NULL,
    OrderDate DATE NOT NULL,
    TotalAmount DECIMAL(18,2) NOT NULL,
    Filler CHAR(50) DEFAULT 'A' -- чтобы раздуть таблицу
);

INSERT INTO dbo.OrdersTask6 (CustomerID, OrderDate, TotalAmount)
SELECT TOP (50000)
    ABS(CHECKSUM(NEWID())) % 1000,
    DATEADD(DAY, -(ABS(CHECKSUM(NEWID())) % 1000), GETDATE()),
    ABS(CHECKSUM(NEWID())) % 10000 / 100.0
FROM sys.all_objects a CROSS JOIN sys.all_objects b;
GO

SET STATISTICS IO ON;

-- 2. Индекс БЕЗ INCLUDE (будет Key Lookup)
CREATE NONCLUSTERED INDEX IX_Orders_CustDate ON dbo.OrdersTask6 (CustomerID, OrderDate);
GO

SELECT OrderID, CustomerID, OrderDate, TotalAmount
FROM dbo.OrdersTask6 
WHERE CustomerID = 50 AND OrderDate >= '2022-01-01';
GO

-- 3. Индекс С INCLUDE (Покрывающий)
DROP INDEX IX_Orders_CustDate ON dbo.OrdersTask6;
CREATE NONCLUSTERED INDEX IX_Orders_CustDate_Inc 
ON dbo.OrdersTask6 (CustomerID, OrderDate) INCLUDE (TotalAmount);
GO

SELECT OrderID, CustomerID, OrderDate, TotalAmount
FROM dbo.OrdersTask6 
WHERE CustomerID = 50 AND OrderDate >= '2022-01-01';
GO
```
<img width="361" height="434" alt="image" src="https://github.com/user-attachments/assets/c2993bda-e2ba-4f5f-b146-ca11c6148ac5" />

---

## Задача 7
```sql
CREATE TABLE dbo.Sales (
    SaleID INT IDENTITY(1,1) PRIMARY KEY,
    RegionID INT NOT NULL,
    SaleDate DATE NOT NULL,
    Amount DECIMAL(18,2) NOT NULL
);

INSERT INTO dbo.Sales (RegionID, SaleDate, Amount)
SELECT TOP (50000)
    ABS(CHECKSUM(NEWID())) % 50,
    DATEADD(DAY, -(ABS(CHECKSUM(NEWID())) % 365), '2024-12-31'),
    100.0
FROM sys.all_objects a CROSS JOIN sys.all_objects b;
GO

-- Создаем два индекса с разным порядком ключей
CREATE NONCLUSTERED INDEX IX_1 ON dbo.Sales (RegionID, SaleDate);
CREATE NONCLUSTERED INDEX IX_2 ON dbo.Sales (SaleDate, RegionID);
GO

-- Запрос 1: Ищем по обоим полям
SELECT * FROM dbo.Sales WHERE RegionID = 5 AND SaleDate = '2024-01-15';

-- Запрос 2: Ищем только по второму полю из IX_1
SELECT * FROM dbo.Sales WHERE SaleDate = '2024-01-15';
GO
```
<img width="296" height="494" alt="image" src="https://github.com/user-attachments/assets/5eab7a40-f33f-4728-9607-3ff017bfc714" />

---

## Задача 8
```sql
-- Создаем Heap (кучу) - без PRIMARY KEY
CREATE TABLE dbo.LogHeap (
    LogID INT IDENTITY(1,1) NOT NULL, 
    LogDate DATE NOT NULL,
    LogMessage VARCHAR(100) DEFAULT 'System ok'
);

-- Создаем таблицу под кластер
CREATE TABLE dbo.LogClustered (
    LogID INT IDENTITY(1,1) NOT NULL,
    LogDate DATE NOT NULL,
    LogMessage VARCHAR(100) DEFAULT 'System ok'
);

-- Заполняем Кучу
INSERT INTO dbo.LogHeap (LogDate)
SELECT TOP (50000) DATEADD(DAY, -(ABS(CHECKSUM(NEWID())) % 365), '2024-12-31')
FROM sys.all_objects a CROSS JOIN sys.all_objects b;

-- Копируем те же данные в кластерную таблицу
INSERT INTO dbo.LogClustered (LogDate) SELECT LogDate FROM dbo.LogHeap;

-- Создаем кластеризованный индекс по дате
CREATE CLUSTERED INDEX CIX_LogClustered_LogDate ON dbo.LogClustered (LogDate);
GO

SET STATISTICS IO ON;
SET STATISTICS TIME ON;

-- Выполняем идентичные диапазонные запросы
SELECT * FROM dbo.LogHeap WHERE LogDate BETWEEN '2024-01-01' AND '2024-01-31';

SELECT * FROM dbo.LogClustered WHERE LogDate BETWEEN '2024-01-01' AND '2024-01-31';
GO
```
<img width="263" height="495" alt="image" src="https://github.com/user-attachments/assets/6743a8d9-324f-4712-b23a-84577b656392" />
