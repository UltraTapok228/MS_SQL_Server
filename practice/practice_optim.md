# Практическая работа. Оптимизация запросов с помощью индексов

Сравнение поиска без индексов, с кластеризованным, покрывающим и некластеризованным индексами, а также влияние индексов на скорость вставки. База IndexDemo.

---

## Задание 1. Подготовка базы данных


### Что требовалось
Создать базу данных и таблицу Customers, заполнить её 50 000 строк тестовых данных.

### Что сделано
Создана база IndexDemo и таблица dbo.Customers без индексов (heap). Таблица заполнена 50 000 строк случайных данных, затем проверено количество строк.

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

### Вывод
Подготовлена таблица, на которой разница между индексами будет заметна.

---

## Задание 2. Анализ базового запроса без индексов


### Что требовалось
Выполнить поиск по Email в таблице без индексов и посмотреть план и логические чтения.

### Что сделано
Включена статистика ввода-вывода и времени, выполнен поиск по конкретному Email.

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

### Результат
<img width="520" height="353" alt="image" src="https://github.com/user-attachments/assets/5419d7ef-03cf-49c8-b348-e172849e78a9" />

### Анализ
Так как таблица является кучей, оптимизатору приходится использовать оператор Table Scan — полное сканирование таблицы. На вкладке «Сообщения» в показателе logical reads видно большое число, так как сервер прочитал каждую страницу данных, чтобы найти один email.

### Вывод
Без индекса поиск одной записи требует чтения всей таблицы.

---

## Задание 3. Кластеризованный индекс


### Что требовалось
Создать кластеризованный индекс и сравнить точечный и диапазонный поиск.

### Что сделано
Создан кластеризованный индекс по CustomerID, выведен список индексов и выполнены точечный и диапазонный запросы.

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

### Результат
<img width="1050" height="708" alt="image" src="https://github.com/user-attachments/assets/ad96eddc-a2dd-4320-94c9-9a36caf18003" />

### Анализ
План выполнения показывает Clustered Index Seek. Запросы отработали мгновенно, так как сервер спустился по B-дереву напрямую к нужным значениям (или диапазону значений). Логические чтения сократились до 2–3 страниц.

### Вывод
Кластеризованный индекс эффективен и для точечного, и для диапазонного поиска по ключу.

---

## Задание 4. Покрывающий некластеризованный индекс


### Что требовалось
Создать некластеризованный индекс по Email, затем сделать его покрывающим через INCLUDE.

### Что сделано
Создан индекс по Email, затем удалён и пересоздан с включённым столбцом LastName. Повторён поиск по Email.

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

### Результат
<img width="901" height="729" alt="image" src="https://github.com/user-attachments/assets/85adfe72-f2ad-4b22-9b03-abe6efbc3597" />

### Анализ
Конструкция INCLUDE (LastName) добавляет поле LastName прямо в листовой уровень некластеризованного индекса (помимо Email и CustomerID, который там есть всегда как ссылка). Теперь индекс полностью «покрывает» запрос SELECT CustomerID, Email, LastName. Благодаря этому SQL Server использует Index Seek (NonClustered) и не тратит ресурсы на дорогую операцию Key Lookup.

### Вывод
Покрывающий индекс устраняет Key Lookup.

---

## Задание 5. Низкая селективность и игнорирование индекса


### Что требовалось
Показать, что индекс по столбцу с низкой селективностью может не использоваться.

### Что сделано
Создан индекс по City и выполнен запрос, возвращающий около 20% строк (Москва).

```sql
USE IndexDemo;
GO
CREATE NONCLUSTERED INDEX NCIX_Customers_City ON dbo.Customers (City);
GO

-- Запрос, возвращающий ~20% строк (Москва)
SELECT * FROM dbo.Customers WHERE City = N'Москва';
GO
```

### Результат
<img width="872" height="533" alt="image" src="https://github.com/user-attachments/assets/3b3f2b2d-3056-4814-9b86-6969df016b32" />

### Анализ
Хотя индекс по городу существует, оптимизатор выбирает Clustered Index Scan. Москвичи составляют около 20% таблицы (10 000 строк). Так как в запросе SELECT *, серверу пришлось бы 10 000 раз прыгать из некластеризованного индекса в кластеризованный (Key Lookup), чтобы достать остальные столбцы. Просто отсканировать всю таблицу дешевле. Это классический пример низкой селективности.

### Вывод
Индекс эффективен только при высокой селективности условия.

---

## Задание 6. Влияние индексов на скорость вставки (DML)


### Что требовалось
Сравнить скорость вставки 1000 строк при наличии некластеризованных индексов и без них.

### Что сделано
Сначала вставлено 1000 строк при существующих индексах. Затем некластеризованные индексы удалены, и вставлено ещё 1000 строк (кластеризованный индекс оставлен для сравнения).

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

### Результат
<img width="1577" height="665" alt="image" src="https://github.com/user-attachments/assets/af9b9b31-0950-4bcd-b492-80099965a593" />
<img width="1562" height="715" alt="image" src="https://github.com/user-attachments/assets/e673ed32-ff07-496f-9bff-5dcce5fdb915" />

### Анализ
На вкладке «Сообщения» в показателе времени выполнения (elapsed time) видно, что вторая вставка (без некластеризованных индексов) отработала быстрее. Индексы ускоряют чтение, но замедляют INSERT, UPDATE и DELETE, так как серверу приходится синхронно обновлять все B-деревья.

### Вывод
Каждый дополнительный индекс — плата за скорость изменения данных.

---

## Задание 7. Системные функции исследования индексов


### Что требовалось
Исследовать свойства индексов системными функциями и восстановить удалённые индексы.

### Что сделано
Выведен список индексов (sp_helpindex), получены свойства кластеризованного индекса функцией INDEXPROPERTY (IsClustered, IndexDepth, IsUnique). Удалённые ранее некластеризованные индексы созданы заново.

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

### Результат
<img width="376" height="79" alt="image" src="https://github.com/user-attachments/assets/52a6a9b0-0b6c-42f1-a8bf-09e624a72b9f" />
<img width="1577" height="969" alt="image" src="https://github.com/user-attachments/assets/993a09b7-f77a-4aa4-ab2d-911e17d77ed8" />

### Вывод
Системные функции (sp_helpindex, INDEXPROPERTY) позволяют узнать тип, глубину и уникальность индексов без обращения к плану выполнения.

---

## Общий вывод

| Задание | Что показал |
|---|---|
| 1–2 | Table Scan на таблице без индексов |
| 3 | Clustered Index Seek для точечного и диапазонного поиска |
| 4 | Покрывающий индекс (INCLUDE) устраняет Key Lookup |
| 5 | Низкая селективность: индекс игнорируется |
| 6 | Индексы замедляют вставку |
| 7 | Системные функции исследования индексов |

**Ключевые принципы:**
1. Кучу без индексов приходится сканировать целиком.
2. Кластеризованный индекс упорядочивает данные и ускоряет поиск по диапазону.
3. INCLUDE делает индекс покрывающим.
4. Индекс на столбце с низкой селективностью оптимизатор может не использовать.
5. Чем больше индексов, тем дороже INSERT, UPDATE и DELETE.

---

## Стек

Microsoft SQL Server 2017, SSMS 22, база IndexDemo.
