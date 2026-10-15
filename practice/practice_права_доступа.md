# Практическая работа. Система безопасности серверов БД: логины, пользователи, схемы

Создание логинов и пользователей скриптами, системными процедурами и через GUI, создание схемы, выдача прав на чтение.

<!-- TODO: формулировки заданий восстановлены по решению, проверить -->

---

## Задание 1. Управление учётными данными

### Что требовалось
Создать логин с политикой паролей, пользователя базы данных для него, выдать право на чтение таблицы и проверить доступ под этим пользователем.

### Что сделано
В master создан логин SlideUser01 (база по умолчанию College, включены CHECK_EXPIRATION и CHECK_POLICY). В базе College пересоздана тестовая таблица TestTableForTask10, создан пользователь SlideUser01 и выдано право SELECT. Доступ проверен через EXECUTE AS USER.

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

### Результат
<img width="285" height="77" alt="image" src="https://github.com/user-attachments/assets/64238444-ecd1-47c9-8f1c-5982973a5250" />

### Анализ
Логин создаётся на уровне сервера (в master), пользователь — в конкретной базе. Право SELECT выдано только на одну таблицу, поэтому под SlideUser01 она читается, а остальные объекты недоступны.

### Вывод
Для доступа к данным нужны три шага: логин, пользователь базы и выданное право.

---

## Задание 2. Создание учётных записей через скрипт

### Что требовалось
Создать учётную запись SQL Server и учётную запись Windows скриптом.

### Что сделано
Перед созданием удалены старые логины. Создан логин dev1 (слабый пароль, CHECK_POLICY = OFF) и логин Windows [AUSLANDER\Auslander].

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

### Результат
<img width="481" height="102" alt="image" src="https://github.com/user-attachments/assets/9f043c94-3890-40b3-b904-3e60b21cab23" />

Учётная запись Windows уже существует, поэтому SQL Server выдаёт ошибку.

### Анализ
CHECK_POLICY = OFF позволяет создать логин со слабым паролем ('12'), что небезопасно. Повторное создание логина Windows невозможно, если он уже есть на сервере.

### Вывод
В реальной работе CHECK_POLICY должен быть включён.

---

## Задание 3. Создание учётной записи через системные хранимые процедуры

### Что требовалось
Создать логин SQL Server процедурой sp_addlogin и выдать доступ пользователю Windows процедурой sp_grantlogin.

### Что сделано
Старый логин удалён через sp_droplogin, создан логин old_dev2 через sp_addlogin. Выдача доступа Windows-пользователю через sp_grantlogin обёрнута в TRY...CATCH, чтобы пропустить ошибку, если логин уже существует.

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

### Результат
<img width="491" height="109" alt="image" src="https://github.com/user-attachments/assets/db8310e0-17e8-4b15-8b50-1bb71d6fbd26" />

### Анализ
sp_addlogin и sp_grantlogin — устаревшие процедуры, вместо них рекомендуется CREATE LOGIN.

### Вывод
Устаревшие системные процедуры работают, но для новых скриптов лучше использовать CREATE LOGIN.

---

## Задание 4. Создание учётной записи через GUI

### Что требовалось
Создать учётную запись через графический интерфейс SSMS.

### Что сделано
Учётная запись создана через окна SSMS (два шага).

### Результат
Шаг 1:

<img width="375" height="274" alt="image" src="https://github.com/user-attachments/assets/4a1d7bef-b62d-4c90-afe4-9cf8a41b75ef" />

Шаг 2:

<img width="970" height="813" alt="image" src="https://github.com/user-attachments/assets/dfeaa016-e5b3-4c8b-b503-2430868b409f" />

---

## Задание 5. Создание схемы в T-SQL

### Что требовалось
Создать схему dev средствами T-SQL.

### Что сделано
Схема создаётся, только если её ещё нет. CREATE SCHEMA выполняется через EXEC, так как должна быть единственной командой в пакете.

```sql
IF NOT EXISTS (SELECT * FROM sys.schemas WHERE name = 'dev')
    EXEC('CREATE SCHEMA dev AUTHORIZATION dbo');
GO
```

---

## Задание 6. Создание учётной записи и пользователя БД

### Что требовалось
Создать логин Partner.

### Что сделано
Логин Partner создаётся, если его ещё нет.

```sql
IF NOT EXISTS (SELECT * FROM sys.server_principals WHERE name = 'Partner')
    CREATE LOGIN Partner WITH PASSWORD = 'StrongPassword123!';
GO
```

### Результат
<img width="967" height="404" alt="image" src="https://github.com/user-attachments/assets/9d29093e-2a4a-487b-87dd-b67c5b19a52c" />

---

## Задание 7. Создание пользователя старой системной процедурой sp_adduser

### Что требовалось
Подготовить логин для пользователя, которого затем создаёт процедура sp_adduser.

### Что сделано
Создан логин OldUser (если его нет). Если пользователь OldUser уже существует в базе, он удаляется через sp_dropuser.

```sql
IF NOT EXISTS (SELECT * FROM sys.server_principals WHERE name = 'OldUser')
    CREATE LOGIN OldUser WITH PASSWORD = 'StrongPassword123!';
GO

IF EXISTS (SELECT * FROM sys.database_principals WHERE name = 'OldUser')
    EXEC sp_dropuser 'OldUser';
GO
```

### Результат
<img width="969" height="341" alt="image" src="https://github.com/user-attachments/assets/6998935b-4933-448b-a9cd-e0dd6e05d92c" />

---

## Задание 8. Использование sp_adduser

### Что требовалось
Создать пользователя базы данных процедурой sp_adduser и добавить его в роль db_datareader.

### Что сделано
Вызвана sp_adduser для логина OldUser с добавлением в роль db_datareader.

```sql
EXEC sp_adduser @loginame = 'OldUser', @name_in_db = 'OldUser', @grpname = 'db_datareader';
GO

PRINT 'Все объекты по новым слайдам успешно созданы!';
```

### Результат
<!-- TODO: вставить скриншот результата, в исходном файле раздел пустой -->

### Вывод
sp_adduser создаёт пользователя и сразу добавляет его в роль, но, как и sp_addlogin, считается устаревшей: рекомендуется CREATE USER и ALTER ROLE.

---

## Общий вывод

| Задание | Что показал |
|---|---|
| 1 | Полный цикл: логин → пользователь БД → право SELECT → проверка |
| 2 | Создание логинов SQL Server и Windows скриптом |
| 3, 7, 8 | Устаревшие процедуры sp_addlogin, sp_grantlogin, sp_adduser |
| 4 | Создание учётной записи через GUI |
| 5 | Создание схемы в T-SQL |
| 6 | Создание логина Partner |

**Ключевые принципы:**
1. Логин — уровень сервера, пользователь — уровень базы.
2. CHECK_POLICY = ON — обязательное требование безопасности.
3. Устаревшие sp_addlogin и sp_adduser лучше заменять на CREATE LOGIN и CREATE USER.
4. Права проверяются под нужным пользователем через EXECUTE AS и REVERT.

---

## Стек

Microsoft SQL Server 2017, SSMS 22, база College.
