# Практическая работа. Индексы в SQL Server

Работа с кластеризованными и некластеризованными индексами, покрывающими индексами, порядком столбцов в составном индексе, сравнением heap и кластеризованной таблицы.

---

# Часть 1 (01.10.2026). Работа с индексами в базе IndexDemo

## Задание 1. Создание и заполнение базы данных


### Что требовалось
Создать базу данных IndexDemo и таблицу Customers, заполнить её 50 000 строк тестовых данных.

### Что сделано
Создана база IndexDemo и таблица dbo.Customers. Таблица заполнена 50 000 строк случайных данных (Email, фамилия, имя, город, дата регистрации, примечания), затем проверено количество строк.

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

```sql
SELECT COUNT(*) AS TotalRows FROM dbo.Customers;
```

### Анализ
На маленьких таблицах (сотни строк) выгода от индексов незаметна, но объём в 50 000 записей позволяет наглядно увидеть разницу между прямым поиском и полным сканированием.

### Вывод
Подготовлен массив данных, достаточный для сравнения методов поиска.

---

## Задание 2. Запрос без индекса


### Что требовалось
Выполнить поиск по Email в таблице без индексов и посмотреть статистику.

### Что сделано
Включена статистика ввода-вывода и времени, выполнен поиск по конкретному Email.

```sql
SET STATISTICS IO ON;
SET STATISTICS TIME ON;

SELECT CustomerID, Email, LastName
FROM dbo.Customers
WHERE Email = 'user12345@example.com';
```

### Результат
![Результат запроса без индекса](https://github.com/user-attachments/assets/60622746-9ed1-4e24-9551-6e68e9af0f9c)

### Анализ
Поиск по конкретному Email без индекса заставляет SQL Server использовать самый ресурсоёмкий метод — полное сканирование таблицы (Table Scan). Серверу приходится последовательно проверять каждую строку, чтобы найти нужное совпадение. С ростом объёма данных этот процесс будет занимать всё больше времени и оперативной памяти.

### Вывод
Без индекса время поиска растёт вместе с размером таблицы.

---

## Задание 3. Кластеризованный индекс


### Что требовалось
Создать кластеризованный индекс и проверить точечный и диапазонный поиск.

### Что сделано
Создан кластеризованный индекс по CustomerID, выведен список индексов, затем выполнены точечный запрос и запрос по диапазону.

```sql
CREATE CLUSTERED INDEX CIX_Customers_CustomerID
ON dbo.Customers (CustomerID);
GO

EXEC sp_helpindex 'dbo.Customers';
```

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

### Анализ
Кластеризованный индекс по CustomerID физически упорядочивает строки на диске по этому столбцу. Точечный поиск и выборка диапазона становятся максимально эффективными: сервер точно знает, где лежат данные, и может забрать диапазон за одно последовательное чтение.

### Вывод
Кластеризованный индекс особенно полезен для поиска по ключу и по диапазону ключа.

---

## Задание 4. Некластеризованный индекс


### Что требовалось
Создать некластеризованные индексы по Email (с INCLUDE) и по City и сравнить их эффективность.

### Что сделано
Создан индекс по Email, затем удалён и пересоздан с INCLUDE (LastName). Выполнен поиск по Email. Также создан индекс по City и выполнен запрос, возвращающий около 20% строк (Москва).

```sql
CREATE NONCLUSTERED INDEX NCIX_Customers_Email
ON dbo.Customers (Email);
GO
```

```sql
DROP INDEX NCIX_Customers_Email ON dbo.Customers;
```

```sql
CREATE NONCLUSTERED INDEX NCIX_Customers_Email
ON dbo.Customers (Email)
INCLUDE (LastName);
GO
```

```sql
SELECT CustomerID, Email, LastName
FROM dbo.Customers
WHERE Email = 'user12345@example.com';
```

```sql
CREATE NONCLUSTERED INDEX NCIX_Customers_City
ON dbo.Customers (City);
GO
```

```sql
-- Запрос, возвращающий ~20% строк (Москва)
SELECT *
FROM dbo.Customers
WHERE City = N'Москва';
```

### Результат
![Результат поиска по Email](https://github.com/user-attachments/assets/a4610332-6a00-4630-a0cf-8d55e4d9e7d8)

![Результат поиска по City](https://github.com/user-attachments/assets/b6810332-0369-4605-9034-cd8a3dad0e32)

### Анализ
**Индекс по Email с INCLUDE.** Некластеризованный индекс создаёт отдельную структуру для быстрого поиска уникальных значений. INCLUDE (LastName) делает индекс «покрывающим»: SQL Server может отдать запрошенные данные (Email и LastName) напрямую из индекса, не читая основную таблицу.

**Индекс по City (низкая селективность).** В таблице всего 5 вариантов городов, поэтому «Москва» встречается примерно у 20% строк. В таких случаях некластеризованный индекс теряет эффективность: оптимизатору часто дешевле просканировать всю таблицу, чем обращаться к индексу для каждой пятой записи.

### Вывод
Индекс полезен на столбцах с высокой селективностью, а покрывающий индекс экономит обращение к основной таблице.

---

## Задание 5. Влияние индексов на вставку данных


### Что требовалось
Сравнить скорость вставки 1000 строк при наличии некластеризованных индексов и после их удаления.

### Что сделано
Сначала вставлено 1000 строк при существующих индексах. Затем некластеризованные индексы удалены (кластеризованный оставлен для сравнения) и вставлено ещё 1000 строк с теми же данными.

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

```sql
DROP INDEX NCIX_Customers_Email ON dbo.Customers;
DROP INDEX NCIX_Customers_City ON dbo.Customers;
```

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

### Анализ
Этап показывает главный компромисс при проектировании БД: индексы ускоряют поиск и чтение данных, но увеличивают стоимость операций изменения (INSERT, UPDATE, DELETE). При добавлении строки серверу приходится не только записать данные в таблицу, но и обновить структуру каждого связанного индекса.

### Вывод
Лишние индексы замедляют запись данных.

---

## Задание 6. Статистика активных запросов

<!-- TODO: формулировка восстановлена по заголовку, проверить -->
### Что требовалось
Посмотреть статистику активных запросов.

### Что сделано
Открыт отчёт со статистикой активных запросов в SSMS.

### Результат
![Статистика активных запросов](https://github.com/user-attachments/assets/2a5d10fb-dd92-4048-8e13-a84c4d8afb14)

![Дополнительная статистика активных запросов](https://github.com/user-attachments/assets/834a79ed-859e-4ab4-8b26-5327efe06759)

---

# Часть 2 (02.10.2026). Индексы: практические задачи (база IndexPractice)

## Задача 1. Создание кластеризованного индекса на heap-таблице


### Что требовалось
Сравнить план и logical reads для поиска по ProductID в таблице-куче и после создания кластеризованного индекса.

### Что сделано
Включена статистика IO. Выполнен поиск по ProductID до создания индекса (Table Scan), затем создан кластеризованный индекс по ProductID и поиск выполнен повторно (Clustered Index Seek, чтений становится 2–3).

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

### Результат
<img width="339" height="163" alt="image" src="https://github.com/user-attachments/assets/a3eda7c5-7123-40a1-994b-35edb74ed8c9" />

<img width="892" height="591" alt="image" src="https://github.com/user-attachments/assets/4331978d-64b7-41e2-83a5-d547bf3fafc4" />

### Вывод
Кластеризованный индекс заменяет полное сканирование таблицы поиском по B-дереву.

---

## Задача 2. Некластеризованный индекс для точечного поиска


### Что требовалось
Создать некластеризованный индекс по имени товара и выполнить точечный поиск.

### Что сделано
Создан индекс по столбцу Name и выполнен поиск конкретного товара.

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

### Результат
<img width="898" height="453" alt="image" src="https://github.com/user-attachments/assets/db5c5c5d-7b55-4ca6-9879-0799eeaf6cc8" />

<img width="912" height="631" alt="image" src="https://github.com/user-attachments/assets/5e14afe0-83ad-403c-a3e5-cfd48cec7bef" />

### Вывод
Некластеризованный индекс по часто используемому столбцу ускоряет точечный поиск.

---

## Задача 3. Разница между Seek и Scan


### Что требовалось
Сравнить планы выполнения для точечного, диапазонного поиска и выборки всех строк.

### Что сделано
Выполнены три запроса к таблице Orders.

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

### Результат (план выполнения)
<img width="698" height="711" alt="image" src="https://github.com/user-attachments/assets/b08cc8f4-cc53-4e50-9380-7ba663ceacb3" />

### Анализ
**Запрос 1 (OrderID = 100):** Clustered Index Seek. Сервер мгновенно спускается по B-дереву кластеризованного индекса прямо к строке 100. Это самая эффективная операция для точечного поиска.

**Запрос 2 (OrderID > 100 AND OrderID < 200):** Clustered Index Seek. Несмотря на выборку нескольких строк, сервер использует Seek, чтобы быстро найти начальную точку (101), а затем читает данные по порядку до 199. Технически это Range Scan, но в графическом плане он помечен иконкой Seek.

**Запрос 3 (OrderID > 0):** Clustered Index Scan. Под условие попадают все 500 000 строк. Вместо 500 тысяч спусков по B-дереву выгоднее прочитать весь листовой уровень индекса от первой до последней страницы.

### Вывод
Seek используется при выборке небольшой части данных, Scan — когда нужна почти вся таблица.

---

## Задача 4. Влияние selectivity на выбор индекса


### Что требовалось
Показать, как селективность столбца влияет на выбор индекса оптимизатором.

### Что сделано
Созданы два некластеризованных индекса: по Gender (мало разных значений) и по Email (уникальные значения). Выполнены запросы по обоим столбцам.

```sql
-- Создание некластеризованных индексов
CREATE NONCLUSTERED INDEX IX_Users_Gender ON dbo.Users (Gender);
CREATE NONCLUSTERED INDEX IX_Users_Email ON dbo.Users (Email);

-- Проверка влияния селективности на план выполнения
SELECT * FROM dbo.Users WHERE Gender = 'M';
SELECT * FROM dbo.Users WHERE Email = 'user150000@test.com';
```

### Результат
<img width="799" height="622" alt="image" src="https://github.com/user-attachments/assets/e5c9407d-b9cb-41ff-807a-079b14af29c4" />

### Вывод
Индекс по высокоселективному столбцу (Email) эффективен, а по низкоселективному (Gender) оптимизатор, как правило, не использует.

---

## Задача 5. Удаление и пересоздание индекса


### Что требовалось
Удалить индекс, пересоздать его с дополнительным столбцом и сравнить размер индекса.

### Что сделано
Проверен размер индекса NCIX_Email (page_count, record_count), индекс удалён, список индексов проверен через sys.indexes (остался только кластерный PK). Индекс пересоздан с LastName как ключевым столбцом, размер проверен повторно (page_count стал больше, так как индекс «потяжелел»).

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

### Результат
<img width="869" height="549" alt="image" src="https://github.com/user-attachments/assets/910dabad-f952-4c22-a590-7c45bccf4f05" />

<img width="892" height="793" alt="image" src="https://github.com/user-attachments/assets/91e414d2-7cfc-4832-bdee-bb5ac098804f" />

### Вывод
Добавление столбца в ключ индекса увеличивает его размер (число страниц).

---

## Задача 6. Covering index через INCLUDE

### Что требовалось
Создать составной некластеризованный индекс на (CustomerID, OrderDate), убедиться, что без INCLUDE возникает Key Lookup, затем добавить TotalAmount через INCLUDE и сравнить logical reads.

### Что сделано
Создана таблица dbo.OrdersTask6 на 50 000 строк. Сначала создан индекс без INCLUDE (будет Key Lookup), затем он удалён и создан индекс с INCLUDE (TotalAmount), после чего запрос выполнен повторно.

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

### Результат
<img width="361" height="434" alt="image" src="https://github.com/user-attachments/assets/c2993bda-e2ba-4f5f-b146-ca11c6148ac5" />

### Анализ
Без INCLUDE столбца TotalAmount нет в индексе, поэтому SQL Server делает Key Lookup в кластеризованный индекс за каждой найденной строкой. С INCLUDE значение хранится в листьях некластеризованного индекса, Key Lookup не нужен, количество logical reads снижается.

### Вывод
Ключевые столбцы поиска ставятся в ON (...), нужные для SELECT — в INCLUDE (...).

---

## Задача 7. Порядок столбцов в составном индексе

### Что требовалось
Создать IX_1 на (RegionID, SaleDate) и IX_2 на (SaleDate, RegionID). Выполнить запросы по обоим полям и только по SaleDate. Определить, какой индекс эффективнее, и объяснить роль leftmost column.

### Что сделано
Создана таблица dbo.Sales на 50 000 строк и два индекса с разным порядком ключей. Выполнены два запроса: по обоим полям и только по второму полю IX_1 (SaleDate).

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

### Результат
<img width="296" height="494" alt="image" src="https://github.com/user-attachments/assets/5eab7a40-f33f-4728-9607-3ff017bfc714" />

### Анализ
Составной индекс (A, B) — это B-дерево, отсортированное сначала по A, потом по B. Индекс эффективно используется только при наличии левого префикса A в условии WHERE:

- WHERE A = … — используется;
- WHERE A = … AND B = … — используется;
- WHERE B = … (без A) — не используется эффективно.

Поэтому запрос по обоим полям оптимально обслуживает IX_1, а запрос только по SaleDate — IX_2, где SaleDate стоит первым.

### Вывод
Порядок столбцов в составном индексе критичен: если запросы бывают только по второму столбцу, нужен отдельный индекс, начинающийся с него.

---

## Задача 8. Кластеризованный индекс и heap для диапазонных запросов

### Что требовалось
Создать две одинаковые таблицы: LogHeap (куча) и LogClustered (кластеризованный индекс по LogDate). Выполнить диапазонный запрос по LogDate, сравнить logical reads и время, объяснить разницу.

### Что сделано
Созданы две таблицы по 50 000 строк с одинаковыми данными. Для LogClustered создан кластеризованный индекс по LogDate. Выполнены одинаковые диапазонные запросы на обеих таблицах со статистикой IO и TIME.

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

### Результат
<img width="263" height="495" alt="image" src="https://github.com/user-attachments/assets/6743a8d9-324f-4712-b23a-84577b656392" />

### Анализ
Куча физически не упорядочена, поэтому SQL Server не может найти диапазон без полного сканирования (Table Scan). Кластеризованный индекс упорядочивает строки по LogDate: сервер находит границы диапазона и читает только страницы между ними (Clustered Index Seek), что даёт меньше logical reads и меньшее время.

### Вывод
Для диапазонных запросов (BETWEEN, >, <) кластеризованный индекс на столбце диапазона даёт радикальное ускорение по сравнению с кучей.

---

## Общий вывод

| Задание | Что показал |
|---|---|
| Часть 1, 1–2 | Table Scan на таблице без индексов |
| Часть 1, 3 | Кластеризованный индекс: точечный и диапазонный поиск |
| Часть 1, 4 | Покрывающий индекс (INCLUDE) и низкая селективность |
| Часть 1, 5 | Индексы замедляют вставку |
| Часть 2, 1–2 | Кластеризованный и некластеризованный индексы |
| Часть 2, 3 | Seek против Scan |
| Часть 2, 4 | Селективность и выбор индекса |
| Часть 2, 5 | Размер индекса при изменении ключа |
| Часть 2, 6 | Key Lookup и covering index |
| Часть 2, 7 | Порядок столбцов (leftmost prefix) |
| Часть 2, 8 | Heap против кластеризованной таблицы для диапазонов |

**Ключевые принципы:**
1. Без индекса — полное сканирование таблицы.
2. Кластеризованный индекс упорядочивает данные, идеален для диапазонов.
3. INCLUDE делает индекс покрывающим и убирает Key Lookup.
4. Индекс на низкоселективном столбце оптимизатор может игнорировать.
5. Порядок столбцов в составном индексе критичен.
6. Каждый индекс замедляет INSERT, UPDATE и DELETE.

---

## Стек

Microsoft SQL Server 2017, SSMS 22, базы IndexDemo, IndexPractice.
