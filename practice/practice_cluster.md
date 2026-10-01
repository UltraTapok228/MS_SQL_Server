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

Так как для столбца `Email` ещё не создан индекс, SQL Server не может быстро найти нужную строку по значению `Email`.

В результате выполняется **полный просмотр таблицы (Table Scan)** — сервер последовательно проверяет большое количество строк, пока не найдёт подходящую.

При небольшом количестве данных это может быть незаметно, однако при увеличении таблицы такой подход становится менее эффективным.

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

Кластеризованный индекс определяет физический порядок хранения строк таблицы по значению индексируемого столбца.

В данном случае индекс создан по:

```text
CustomerID
```

Поэтому поиск конкретного `CustomerID` и выборка диапазона значений выполняются эффективно.

Особенно хорошо это заметно на диапазонном запросе:

```sql
WHERE CustomerID BETWEEN 10000 AND 10100
```

SQL Server может быстро перейти к нужному диапазону индекса и последовательно получить необходимые строки.

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

Некластеризованный индекс хранит отдельную структуру для поиска значений индексируемого столбца.

В случае с `Email` индекс позволяет быстро найти нужную запись, так как значение `Email` является достаточно уникальным.

Конструкция:

```sql
INCLUDE (LastName)
```

добавляет `LastName` в индекс как дополнительный столбец. Это позволяет SQL Server получить часть необходимых данных непосредственно из индекса.

Для `City` ситуация отличается. В таблице всего пять городов, поэтому каждое значение встречается примерно у **20% строк**.

Например:

```sql
WHERE City = N'Москва'
```

возвращает большое количество записей.

При таком запросе использование индекса не всегда оказывается выгоднее полного просмотра таблицы. SQL Server самостоятельно выбирает наиболее подходящий план выполнения запроса.

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

---

# 6. Статистика активных запросов

### Статистика активных запросов

![Статистика активных запросов](https://github.com/user-attachments/assets/2a5d10fb-dd92-4048-8e13-a84c4d8afb14)

![Дополнительная статистика активных запросов](https://github.com/user-attachments/assets/834a79ed-859e-4ab4-8b26-5327efe06759)

---

скорить поиск и получение данных, однако их наличие также может увеличивать стоимость операций изменения данных, например `INSERT`, `UPDATE` и `DELETE`. Поэтому при проектировании базы данных важно учитывать не только скорость чтения, но и характер нагрузки на таблицу.
