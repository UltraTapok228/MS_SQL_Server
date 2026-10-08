### AUDIT

## Аудит на уровне сервера
> Этот код настраивает механизм аудита и указывает ему писать логи в папку в корне диска.
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

-- 2. Включаем аудит[cite: 13]
ALTER SERVER AUDIT [MyServerAudit] WITH (STATE = ON);
GO

-- 3. Создаем спецификацию уровня сервера (отслеживаем входы и права)[cite: 13]
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

## Cоздание спецификации для базы данных
> Направление аудита на конкретную таблицу в БД
```sql
USE [view];
GO

-- Создаем спецификацию уровня БД, привязывая ее к серверному аудиту[cite: 13]
CREATE DATABASE AUDIT SPECIFICATION [MyDBAuditSpec]
FOR SERVER AUDIT [MyServerAudit]
-- Отслеживаем все базовые операции именно на нашей таблице Students[cite: 13]
ADD (SELECT, INSERT, UPDATE, DELETE ON dbo.Students BY public), 
ADD (DATABASE_ROLE_MEMBER_CHANGE_GROUP)                               
WITH (STATE = ON);
GO
PRINT 'Спецификация для БД view успешно создана!';
```

## Имитация действий и просмотр
> Запись и чтение логов
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

---

# Результат
> <img width="880" height="339" alt="image" src="https://github.com/user-attachments/assets/7f7b5e0e-4dab-4570-b287-6d36ea2fdae3" />

