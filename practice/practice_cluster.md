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

скорить поиск и получение данных, однако их наличие также может увеличивать стоимость операций изменения данных, например `INSERT`, `UPDATE` и `DELETE`. Поэтому при проектировании базы данных важно учитывать не только скорость чтения, но и характер нагрузки на таблицу.
