### Система безопасности серверов БД

---

## Управление учетными данными
```sql
USE master;
GO

-- 1. Сбрасываем контекст (на всякий случай)
REVERT;
GO

-- 2. Удаляем старый логин, если он остался
IF EXISTS (SELECT * FROM sys.server_principals WHERE name = 'SlideUser01')
    DROP LOGIN SlideUser01;
GO

-- 3. Создаем логин по презентации (выполняется строго в master от имени admin)
CREATE LOGIN SlideUser01
WITH 
    PASSWORD = 'SuperPassword123!',
    DEFAULT_DATABASE = College,
    CHECK_EXPIRATION = ON,
    CHECK_POLICY = ON;
GO

-- 4. Переходим в базу College
USE College;
GO

-- 5. Пересоздаем тестовую таблицу
IF OBJECT_ID('dbo.TestTableForTask10', 'U') IS NOT NULL 
    DROP TABLE dbo.TestTableForTask10;

CREATE TABLE dbo.TestTableForTask10 (ID INT, SomeValue NVARCHAR(50));
INSERT INTO dbo.TestTableForTask10 VALUES (1, 'Тестовая строка из презентации');
GO

-- 6. Создаем пользователя в БД для этого логина
IF EXISTS (SELECT * FROM sys.database_principals WHERE name = 'SlideUser01')
    DROP USER SlideUser01;
GO

CREATE USER SlideUser01 FOR LOGIN SlideUser01;
GO

-- 7. Выдаем права на чтение
GRANT SELECT ON dbo.TestTableForTask10 TO SlideUser01;
GO

-- 8. Проверяем работу под созданным пользователем
EXECUTE AS USER = 'SlideUser01';

-- Должна успешно вернуться строка из таблицы
SELECT * FROM dbo.TestTableForTask10;

-- Возвращаемся под админа
REVERT;
GO

PRINT 'Всё выполнено идеально!';
```
**Результат выполнения: **
<img width="285" height="77" alt="image" src="https://github.com/user-attachments/assets/64238444-ecd1-47c9-8f1c-5982973a5250" />

---

## Создание учётных записей через скрипт
```sql
USE master;
GO

-- 0. Очищаем старые логины, если они оставались
IF EXISTS (SELECT * FROM sys.server_principals WHERE name = 'dev1') DROP LOGIN dev1;
IF EXISTS (SELECT * FROM sys.server_principals WHERE name = 'AUSLANDER\spfusc') DROP LOGIN [AUSLANDER\spfusc]; -- адаптированный пример для Windows
GO

-- 1. Пример создания учетной записи SQL Server (со слайда 1)
CREATE LOGIN dev1
WITH PASSWORD = '12',
     DEFAULT_DATABASE = College,
     DEFAULT_LANGUAGE = [us_english],
     CHECK_EXPIRATION = OFF,
     CHECK_POLICY = OFF;
GO

-- 2. Пример создания учетной записи Windows
CREATE LOGIN [AUSLANDER\Auslander] 
FROM WINDOWS 
WITH DEFAULT_DATABASE = College, 
     DEFAULT_LANGUAGE = us_english;
GO

PRINT 'Успешно';
```

**Результат выполнения:**
> <img width="481" height="102" alt="image" src="https://github.com/user-attachments/assets/9f043c94-3890-40b3-b904-3e60b21cab23" />
> Учетная запись уже существует, поэтому ругается

---

## Создание учетной записи через системные хранимые процедуры
```sql
USE master;
GO

-- Очиcтка мусора
IF EXISTS (SELECT * FROM sys.server_principals WHERE name = 'old_dev2') 
    EXEC master.dbo.sp_droplogin 'old_dev2';
GO

-- 1. Создание логина SQL Server через старую процедуру sp_addlogin
EXEC master.dbo.sp_addlogin 
    @loginame = 'old_dev2', 
    @passwd = '12', 
    @defdb = 'College', 
    @deflanguage = 'us_english';
GO

-- 2. Выдача доступа Windows-пользователю через sp_grantlogin
BEGIN TRY
    EXEC master.dbo.sp_grantlogin @loginame = [AUSLANDER\Auslander];
END TRY
BEGIN CATCH
    PRINT 'Логин Windows уже существует или не найден, проигнорировано.';
END CATCH
GO

PRINT 'Успешно';
```
**Результат выполнения:**
<img width="491" height="109" alt="image" src="https://github.com/user-attachments/assets/db8310e0-17e8-4b15-8b50-1bb71d6fbd26" />

---

## Создание через GUI
* Шаг 1:
<img width="375" height="274" alt="image" src="https://github.com/user-attachments/assets/4a1d7bef-b62d-4c90-afe4-9cf8a41b75ef" />

* Шаг 2:
<img width="970" height="813" alt="image" src="https://github.com/user-attachments/assets/dfeaa016-e5b3-4c8b-b503-2430868b409f" />

---

## Создание схемы в T-SQL
```sql
IF NOT EXISTS (SELECT * FROM sys.schemas WHERE name = 'dev')
    EXEC('CREATE SCHEMA dev AUTHORIZATION dbo');
GO
```

---

## Создание учётной записи и пользователя БД
```sql
IF NOT EXISTS (SELECT * FROM sys.server_principals WHERE name = 'Partner')
    CREATE LOGIN Partner WITH PASSWORD = 'StrongPassword123!';
GO
```

**Результат выполнения:**
<img width="967" height="404" alt="image" src="https://github.com/user-attachments/assets/9d29093e-2a4a-487b-87dd-b67c5b19a52c" />

---

## Создание пользователя старой системной процедурой sp_adduser
```sql
IF NOT EXISTS (SELECT * FROM sys.server_principals WHERE name = 'OldUser')
    CREATE LOGIN OldUser WITH PASSWORD = 'StrongPassword123!';
GO

IF EXISTS (SELECT * FROM sys.database_principals WHERE name = 'OldUser')
    EXEC sp_dropuser 'OldUser';
GO
```

**Результат выполнения:**
<img width="969" height="341" alt="image" src="https://github.com/user-attachments/assets/6998935b-4933-448b-a9cd-e0dd6e05d92c" />

---

## Использование sp_adduser
```sql
EXEC sp_adduser @loginame = 'OldUser', @name_in_db = 'OldUser', @grpname = 'db_datareader';
GO

PRINT 'Все объекты по новым слайдам успешно созданы!';
```

**Результаты выполнения:**

