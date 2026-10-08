## Подготовка БД
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

-- Заполняем 50 000 строк тестовыми данными
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

-- Проверяем количество строк
SELECT COUNT(*) AS TotalRows FROM dbo.Customers;
GO
```

## Анализ базового запроса без индексов
```sql
USE IndexDemo;
GO
SET STATISTICS IO ON;
SET STATISTICS TIME ON;

SELECT CustomerID, Email, LastName
FROM dbo.Customers
WHERE Email = 'user12345@example.com';
GO
```

**Результат выполнения:**
> <img width="520" height="353" alt="image" src="https://github.com/user-attachments/assets/5419d7ef-03cf-49c8-b348-e172849e78a9" />
> Так как таблица является кучей, оптимизатору приходится использовать оператор Table Scan — полное сканирование таблицы. На вкладке «Сообщения» (Messages) в показателе logical reads (логические чтения) видно большое число, так как сервер прочитал каждую страницу данных, чтобы найти один email.

## Кластеризованный индекс
```sql
USE IndexDemo;
GO
CREATE CLUSTERED INDEX CIX_Customers_CustomerID
ON dbo.Customers (CustomerID);
GO

EXEC sp_helpindex 'dbo.Customers';

-- Точечный поиск
SELECT CustomerID, Email, LastName
FROM dbo.Customers
WHERE CustomerID = 12345;

-- Диапазонный поиск
SELECT CustomerID, Email
FROM dbo.Customers
WHERE CustomerID BETWEEN 10000 AND 10100;
GO
```
**Результат выполнения:**
> План выполнения показывает Clustered Index Seek. Запросы отработали мгновенно, так как сервер спустился по B-дереву напрямую к нужным значениям (или диапазону значений). Логические чтения (logical reads) резко сократились до 2-3 страниц.
> <img width="1050" height="708" alt="image" src="https://github.com/user-attachments/assets/ad96eddc-a2dd-4320-94c9-9a36caf18003" />

## Покрывающий некластеризованный индекс
```sql
USE IndexDemo;
GO
CREATE NONCLUSTERED INDEX NCIX_Customers_Email ON dbo.Customers (Email);
GO

DROP INDEX NCIX_Customers_Email ON dbo.Customers;

CREATE NONCLUSTERED INDEX NCIX_Customers_Email
ON dbo.Customers (Email)
INCLUDE (LastName);
GO

SELECT CustomerID, Email, LastName
FROM dbo.Customers
WHERE Email = 'user12345@example.com';
GO
```
**Результат выполнения:**
> Конструкция INCLUDE (LastName) добавляет поле LastName прямо в листовой уровень некластеризованного индекса (помимо Email и CustomerID, который там есть всегда как ссылка). Теперь индекс полностью "покрывает" наш SELECT CustomerID, Email, LastName. Благодаря этому SQL Server использует Index Seek (NonClustered) и не тратит ресурсы на дорогую операцию Key Lookup для извлечения недостающих полей из основной таблицы.
> <img width="901" height="729" alt="image" src="https://github.com/user-attachments/assets/85adfe72-f2ad-4b22-9b03-abe6efbc3597" />

## Низка селективность и игнорирование индекса
```sql
USE IndexDemo;
GO
CREATE NONCLUSTERED INDEX NCIX_Customers_City ON dbo.Customers (City);
GO

-- Запрос, возвращающий ~20% строк (Москва)
SELECT * FROM dbo.Customers WHERE City = N'Москва';
GO
```
**Результат выполнения:**
> Хотя индекс по городу существует, оптимизатор выбирает Clustered Index Scan. Москвичи составляют около 20% таблицы (10 000 строк). Так как в запросе указано SELECT *, серверу пришлось бы 10 000 раз прыгать из некластеризованного индекса в кластеризованный (Key Lookup), чтобы достать остальные столбцы (Notes, RegistrationDate и т.д.). Математика оптимизатора показывает, что просто отсканировать всю таблицу с начала до конца — дешевле. Это классический пример низкой селективности.
> <img width="872" height="533" alt="image" src="https://github.com/user-attachments/assets/3b3f2b2d-3056-4814-9b86-6969df016b32" />

## Влияние индексов на скорость вставки(DML)
```sql
USE IndexDemo;
GO
SET STATISTICS TIME ON;

-- Вставка 1000 строк С ИНДЕКСАМИ
INSERT INTO dbo.Customers (Email, LastName, FirstName, City, RegistrationDate, Notes)
SELECT TOP (1000)
    'newuser' + CAST(ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) AS NVARCHAR(10)) + '@test.com',
    'NewLast' + CAST(ABS(CHECKSUM(NEWID())) % 100 AS NVARCHAR(10)),
    'NewFirst' + CAST(ABS(CHECKSUM(NEWID())) % 100 AS NVARCHAR(10)),
    N'Москва', GETDATE(), REPLICATE(N'B', 500)
FROM sys.all_objects;
GO

-- Удаляем некластеризованные индексы
DROP INDEX NCIX_Customers_Email ON dbo.Customers;
DROP INDEX NCIX_Customers_City ON dbo.Customers;
GO

-- Вставка 1000 строк БЕЗ НИХ (Кластеризованный индекс оставляем для сравнения)
INSERT INTO dbo.Customers (Email, LastName, FirstName, City, RegistrationDate, Notes)
SELECT TOP (1000)
    'newuser2_' + CAST(ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) AS NVARCHAR(10)) + '@test.com',
    'NewLast' + CAST(ABS(CHECKSUM(NEWID())) % 100 AS NVARCHAR(10)),
    'NewFirst' + CAST(ABS(CHECKSUM(NEWID())) % 100 AS NVARCHAR(10)),
    N'Москва', GETDATE(), REPLICATE(N'B', 500)
FROM sys.all_objects;
GO
```
**Результат выполнения**
> На вкладке «Сообщения» (Messages) в показателе Время выполнения SQL Server (elapsed time) видно, что вторая вставка (без некластеризованных индексов) отработала быстрее. Индексы ускоряют операции чтения (SELECT), но замедляют операции изменения данных (INSERT, UPDATE, DELETE), так как серверу приходится тратить время и ресурсы на синхронное обновление всех B-деревьев.
> <img width="1577" height="665" alt="image" src="https://github.com/user-attachments/assets/af9b9b31-0950-4bcd-b492-80099965a593" />
> <img width="1562" height="715" alt="image" src="https://github.com/user-attachments/assets/e673ed32-ff07-496f-9bff-5dcce5fdb915" />

## Системные функции исследования индексов
```sql
USE IndexDemo;
GO
EXEC sp_helpindex 'dbo.Customers';

SELECT 
    INDEXPROPERTY(OBJECT_ID('dbo.Customers'), 'CIX_Customers_CustomerID', 'IsClustered') AS IsClustered,
    INDEXPROPERTY(OBJECT_ID('dbo.Customers'), 'CIX_Customers_CustomerID', 'IndexDepth') AS IndexDepth,
    INDEXPROPERTY(OBJECT_ID('dbo.Customers'), 'CIX_Customers_CustomerID', 'IsUnique') AS IsUnique;

-- Восстановление индексов
CREATE NONCLUSTERED INDEX NCIX_Customers_Email ON dbo.Customers (Email) INCLUDE (LastName);
CREATE NONCLUSTERED INDEX NCIX_Customers_City ON dbo.Customers (City);
GO
```
**Результат выполнения:**
> <img width="376" height="79" alt="image" src="https://github.com/user-attachments/assets/52a6a9b0-0b6c-42f1-a8bf-09e624a72b9f" />
> <img width="1577" height="969" alt="image" src="https://github.com/user-attachments/assets/993a09b7-f77a-4aa4-ab2d-911e17d77ed8" />
