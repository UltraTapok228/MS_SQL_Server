# Практическая работа. Query Store: анализ производительности запросов

Включение Query Store, генерация нагрузки и анализ самых ресурсоёмких запросов в базе QueryStoreDemo.

---

## Задание 1. Создание базы данных и включение Query Store


### Что требовалось
Создать базу данных и включить в ней Query Store с настройками сбора статистики.

### Что сделано
Создана база QueryStoreDemo, включён Query Store и заданы параметры: режим READ_WRITE, хранение данных 30 дней, интервал статистики 60 минут, максимальный размер 1000 МБ, сбор статистики ожиданий.

```sql
CREATE DATABASE QueryStoreDemo;
GO

ALTER DATABASE QueryStoreDemo 
SET QUERY_STORE = ON;
GO

ALTER DATABASE QueryStoreDemo 
SET QUERY_STORE (
    OPERATION_MODE = READ_WRITE,
    CLEANUP_POLICY = (STALE_QUERY_THRESHOLD_DAYS = 30),
    DATA_FLUSH_INTERVAL_SECONDS = 900,
    INTERVAL_LENGTH_MINUTES = 60,
    MAX_STORAGE_SIZE_MB = 1000,
    QUERY_CAPTURE_MODE = AUTO,
    SIZE_BASED_CLEANUP_MODE = AUTO,
    MAX_PLANS_PER_QUERY = 200,
    WAIT_STATS_CAPTURE_MODE = ON
);
GO
```

### Вывод
Query Store включён и готов собирать планы и статистику выполнения запросов.

---

## Задание 2. Генерация таблицы и данных


### Что требовалось
Создать таблицу и заполнить её тестовыми данными для получения нагрузки.

### Что сделано
Создана таблица dbo.Orders с индексом по CustomerID и заполнена 50 000 строк случайных данных.

```sql
USE QueryStoreDemo;
GO

CREATE TABLE dbo.Orders (
    OrderID     INT IDENTITY(1,1) PRIMARY KEY,
    CustomerID  INT NOT NULL,
    OrderDate   DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME(),
    Amount      DECIMAL(18,2) NOT NULL,
    Status      VARCHAR(20) NOT NULL
);
GO

CREATE NONCLUSTERED INDEX IX_Orders_CustomerID ON dbo.Orders(CustomerID);
GO

-- Безопасная заливка: ровно 50 000 строк
INSERT INTO dbo.Orders (CustomerID, Amount, Status)
SELECT TOP (50000)
    ABS(CHECKSUM(NEWID())) % 10000 + 1,
    ABS(CHECKSUM(NEWID())) % 1000 + 10.0,
    CASE ABS(CHECKSUM(NEWID())) % 3 
        WHEN 0 THEN 'New'
        WHEN 1 THEN 'Shipped'
        ELSE 'Closed'
    END
FROM sys.all_objects a
CROSS JOIN sys.all_objects b;
GO
```

### Вывод
Таблица с 50 000 заказов создана, данных достаточно, чтобы запросы дали заметную нагрузку.

---

## Задание 3. Нагрузка для сбора статистики


### Что требовалось
Выполнить набор запросов, чтобы Query Store накопил статистику.

### Что сделано
Набор запросов выполнен 4 раза подряд: лёгкие запросы по индексу, группировка, тяжёлая сортировка и повторный запрос после сброса кэша планов.

```sql
USE QueryStoreDemo;
GO
-- 1. Легкие запросы по индексу
SELECT * FROM dbo.Orders WHERE CustomerID = 105;
SELECT * FROM dbo.Orders WHERE CustomerID BETWEEN 500 AND 1000;

-- 2. Группировка
SELECT Status, COUNT(*) FROM dbo.Orders GROUP BY Status;

-- 3. Тяжелая сортировка (именно она будет самой прожорливой на графиках)
SELECT TOP 100 * FROM dbo.Orders ORDER BY Amount DESC;

-- 4. Сброс кэша планов и повторный вызов
DBCC FREEPROCCACHE;
SELECT * FROM dbo.Orders WHERE CustomerID = 105;
GO
```

### Вывод
Query Store зафиксировал планы и показатели выполнения всех запросов.

---

## Задание 4. Сбор аналитики через скрипты


### Что требовалось
Получить из системных представлений Query Store статистику по самым тяжёлым запросам и ожиданиям.

### Что сделано
Выполнены два запроса: топ-20 запросов по затратам CPU за последние 7 дней и топ-20 запросов по суммарному времени ожиданий (wait stats).

```sql
USE QueryStoreDemo;
GO

-- АНАЛИЗ 1: Топ-20 тяжелых запросов по CPU
SELECT TOP 20
    q.query_id,
    qt.query_sql_text,
    SUM(rs.count_executions) AS total_executions,
    AVG(rs.avg_duration) / 1000.0 AS avg_duration_ms,
    SUM(rs.avg_cpu_time * rs.count_executions) / 1000.0 AS total_cpu_ms,
    AVG(rs.avg_logical_io_reads) AS avg_logical_reads
FROM sys.query_store_query q
JOIN sys.query_store_query_text qt ON q.query_text_id = qt.query_text_id
JOIN sys.query_store_plan p ON q.query_id = p.query_id
JOIN sys.query_store_runtime_stats rs ON p.plan_id = rs.plan_id
JOIN sys.query_store_runtime_stats_interval rsi ON rs.runtime_stats_interval_id = rsi.runtime_stats_interval_id
WHERE rsi.start_time >= DATEADD(DAY, -7, GETUTCDATE())
GROUP BY q.query_id, qt.query_sql_text
ORDER BY total_cpu_ms DESC;

-- АНАЛИЗ 2: Статистика ожиданий (Wait Stats)
SELECT TOP 20
    q.query_id,
    qt.query_sql_text,
    ws.wait_category_desc,
    SUM(ws.total_query_wait_time_ms) / 1000.0 AS total_wait_sec
FROM sys.query_store_wait_stats ws
JOIN sys.query_store_plan p ON ws.plan_id = p.plan_id
JOIN sys.query_store_query q ON p.query_id = q.query_id
JOIN sys.query_store_query_text qt ON q.query_text_id = qt.query_text_id
GROUP BY q.query_id, qt.query_sql_text, ws.wait_category_desc
ORDER BY total_wait_sec DESC;
```

### Вывод
Системные представления Query Store позволяют находить ресурсоёмкие запросы и причины ожиданий без внешних инструментов.

---

## Задание 5. Диаграмма и выводы


### Что требовалось
Построить диаграмму ресурсоёмких запросов в Query Store и сделать выводы.

### Что сделано
Открыт отчёт Query Store с диаграммой самых затратных запросов и графическим планом выполнения.

### Результат
<img width="1443" height="960" alt="image" src="https://github.com/user-attachments/assets/f10faed2-1bef-4488-8784-44b00e008103" />

### Анализ
Самым «тяжёлым» (запрос 1) оказался скрипт массовой генерации и вставки тестовых данных (`INSERT INTO dbo.Orders...`). Графический план выполнения в нижней части отчёта позволил точно определить узкое место: 63% стоимости операции ушло на предварительную сортировку данных перед физической вставкой в кластеризованный индекс таблицы.

### Вывод
Query Store наглядно показал топ ресурсоёмких запросов и доказал свою эффективность: он сохраняет всю историческую статистику выполнения запросов и их планы непосредственно в базе данных, что позволяет проводить ретроспективный анализ производительности.

---

## Общий вывод

| Задание | Что показал |
|---|---|
| 1 | Включение и настройка Query Store |
| 2 | Подготовка таблицы и тестовых данных |
| 3 | Создание нагрузки для сбора статистики |
| 4 | Анализ топ-запросов и статистики ожиданий скриптами |
| 5 | Анализ через диаграмму и план выполнения |

**Ключевые принципы:**
1. Query Store хранит планы и статистику запросов в самой базе.
2. Данные доступны через представления sys.query_store_*.
3. Графический отчёт помогает быстро найти узкое место в плане.

---

## Стек

Microsoft SQL Server 2017, SSMS 22, база QueryStoreDemo.
