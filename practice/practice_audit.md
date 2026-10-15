# Практическая работа. Аудит (SQL Server Audit)

Настройка аудита на уровне сервера и базы данных, имитация действий и чтение журнала аудита.

---

## Задание 1. Аудит на уровне сервера


### Что требовалось
Настроить механизм аудита сервера, который пишет логи в файл, и создать спецификацию аудита уровня сервера.

### Что сделано
Создан серверный аудит MyServerAudit с записью в папку `C:\SQLAuditLogs\` (максимум 10 МБ, до 5 файлов ротации). Аудит включён, затем создана спецификация MyServerAuditSpec, которая отслеживает неудачные и успешные входы, а также изменения прав на уровне сервера и базы.

```sql
USE master;
GO

-- 1. Создаем сам аудит сервера
CREATE SERVER AUDIT [MyServerAudit]
TO FILE 
( 
    FILEPATH = N'C:\SQLAuditLogs\', -- Убедись, что папка создана!
    MAXSIZE = 10 MB,               
    MAX_ROLLOVER_FILES = 5,        
    RESERVE_DISK_SPACE = OFF         
)
WITH 
(
    QUEUE_DELAY = 1000,             
    ON_FAILURE = CONTINUE           
);
GO

-- 2. Включаем аудит
ALTER SERVER AUDIT [MyServerAudit] WITH (STATE = ON);
GO

-- 3. Создаем спецификацию уровня сервера (отслеживаем входы и права)
CREATE SERVER AUDIT SPECIFICATION [MyServerAuditSpec]
FOR SERVER AUDIT [MyServerAudit]
ADD (FAILED_LOGIN_GROUP),              
ADD (SUCCESSFUL_LOGIN_GROUP),          
ADD (SERVER_PERMISSION_CHANGE_GROUP),  
ADD (DATABASE_PERMISSION_CHANGE_GROUP) 
WITH (STATE = ON);                     
GO
PRINT 'Аудит сервера успешно создан и запущен!';
```

### Анализ
| Группа | Что отслеживает |
|---|---|
| FAILED_LOGIN_GROUP | Неудачные попытки входа |
| SUCCESSFUL_LOGIN_GROUP | Успешные входы |
| SERVER_PERMISSION_CHANGE_GROUP | Изменения прав на уровне сервера |
| DATABASE_PERMISSION_CHANGE_GROUP | Изменения прав на уровне баз данных |

`ON_FAILURE = CONTINUE` — сервер продолжает работать, даже если запись в аудит не удалась. Папка для логов должна существовать заранее.

### Вывод
Серверный аудит состоит из двух частей: сам аудит (куда писать) и спецификация (что писать).

---

## Задание 2. Спецификация аудита для базы данных


### Что требовалось
Направить аудит на конкретную таблицу в базе данных.

### Что сделано
В базе view создана спецификация MyDBAuditSpec, привязанная к серверному аудиту. Она отслеживает операции SELECT, INSERT, UPDATE и DELETE над таблицей dbo.Students для роли public, а также изменения членства в ролях базы.

```sql
USE [view];
GO

-- Создаем спецификацию уровня БД, привязывая ее к серверному аудиту
CREATE DATABASE AUDIT SPECIFICATION [MyDBAuditSpec]
FOR SERVER AUDIT [MyServerAudit]
-- Отслеживаем все базовые операции именно на нашей таблице Students
ADD (SELECT, INSERT, UPDATE, DELETE ON dbo.Students BY public), 
ADD (DATABASE_ROLE_MEMBER_CHANGE_GROUP)                               
WITH (STATE = ON);
GO
PRINT 'Спецификация для БД view успешно создана!';
```

### Анализ
`BY public` означает, что аудит фиксирует действия любых пользователей, так как все они входят в роль public.

### Вывод
Спецификация уровня базы позволяет вести аудит точечно — на конкретную таблицу и конкретные операции.

---

## Задание 3. Имитация действий и просмотр журнала


### Что требовалось
Выполнить действия над таблицей, попадающие под аудит, и прочитать записи из файла аудита.

### Что сделано
Выполнены SELECT и UPDATE над dbo.Students. Затем сделана пауза в 2 секунды, чтобы буфер аудита успел сбросить данные в файл, и журнал прочитан функцией sys.fn_get_audit_file с фильтром по таблице Students.

```sql
USE [view];
GO

-- 1. Выполняем действия, которые попадут в аудит
SELECT * FROM dbo.Students;
UPDATE dbo.Students SET FullName = FullName WHERE StudentID = 1;

-- 2. Делаем паузу в 2 секунды, чтобы буфер аудита успел сбросить данные в файл на диске
WAITFOR DELAY '00:00:02';

-- 3. Читаем логи из файла аудита 
SELECT 
    event_time AS [Время],                    
    action_id AS [Код действия],                     
    succeeded AS [Успешно],                     
    session_server_principal_name AS [Пользователь], 
    database_name AS [База данных],                 
    object_name AS [Таблица],                   
    statement AS [Текст SQL-запроса]                      
FROM sys.fn_get_audit_file 
(
    'C:\SQLAuditLogs\*.sqlaudit',  -- Читаем все файлы аудита из нашей папки 
    DEFAULT, 
    DEFAULT
)
WHERE object_name = 'Students' -- Фильтруем, чтобы показать только действия с нашей таблицей
ORDER BY event_time DESC; 
```

### Результат
<img width="880" height="339" alt="image" src="https://github.com/user-attachments/assets/7f7b5e0e-4dab-4570-b287-6d36ea2fdae3" />

### Анализ
В журнале видны время события, код действия, пользователь, база, таблица и текст запроса. Так можно отследить, кто и что делал с данными.

### Вывод
Аудит фиксирует действия пользователей в файл, который читается стандартной функцией sys.fn_get_audit_file.

---

## Общий вывод

| Задание | Что показал |
|---|---|
| 1 | Серверный аудит и спецификация уровня сервера |
| 2 | Спецификация аудита уровня базы на таблицу |
| 3 | Чтение журнала аудита |

**Ключевые принципы:**
1. Аудит (куда писать) и спецификация (что писать) — разные объекты.
2. Спецификации бывают серверные и уровня базы данных.
3. Журнал читается функцией sys.fn_get_audit_file.

---

## Стек

Microsoft SQL Server 2017, SSMS 22, база view.
