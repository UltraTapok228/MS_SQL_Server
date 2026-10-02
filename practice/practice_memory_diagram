## Создание БД
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

---

## Генерация таблицы и данных
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

## Нагрузка для сбора статистики

Выполнено 4 раза подряд

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

## Сбор аналитики(через скрипты)
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

## Диаграмма
<img width="1443" height="960" alt="image" src="https://github.com/user-attachments/assets/f10faed2-1bef-4488-8784-44b00e008103" />

**Выводы по анализу:**
Инструмент Query Store наглядно показал топ ресурсоемких запросов. Как видно на диаграмме, самым "тяжелым" (Запрос 1) оказался скрипт массовой генерации и вставки тестовых данных (`INSERT INTO dbo.Orders...`). Графический план выполнения в нижней части отчета позволил точно определить узкое место этого запроса: 63% стоимости операции ушло на предварительную сортировку данных перед их физической вставкой в кластеризованный индекс таблицы. 

Query Store доказал свою эффективность, сохранив всю историческую статистику выполнения запросов и их планы непосредственно в базе данных, что позволяет проводить ретроспективный анализ производительности.
