## Задание 1. Создание SQL-логина с политикой паролей
```sql
USE master;
GO

CREATE LOGIN TestUser01 
WITH PASSWORD = 'P@ssw0rd123!', 
CHECK_POLICY = ON;
GO
```

<img width="653" height="905" alt="image" src="https://github.com/user-attachments/assets/90ab5728-d803-46b9-98ad-de5447bd381a" />
<img width="541" height="497" alt="image" src="https://github.com/user-attachments/assets/b0a227dd-eada-4328-8fe2-2f3755c6f71b" />

---

## Задание 2. Создание логина Windows и сопоставление с пользователем БД
```sql
USE master;
GO

-- 1. Обращаемся к реальной учетке твоей Windows
CREATE LOGIN [AUSLANDER\Auslander] FROM WINDOWS;
GO

USE College;
GO

-- 2. Создаем пользователя БД с нужным по заданию именем и привязываем к логину
CREATE USER JohnDoe FOR LOGIN [AUSLANDER\Auslander];
GO
```
<img width="241" height="100" alt="image" src="https://github.com/user-attachments/assets/30919d30-9423-437d-b27d-5e433fe1257e" />

---

## Задание 3. Создание пользователя без логина
```sql
-- Создадим БД, если ее нет
IF DB_ID('TestDB') IS NULL CREATE DATABASE TestDB;
GO

USE TestDB;
GO

CREATE USER AppServiceUser WITHOUT LOGIN;
GO
```
<img width="503" height="125" alt="image" src="https://github.com/user-attachments/assets/73144e38-673c-4cbd-86aa-233dcae50303" />

**Что произошло?**
> Пользователь WITHOUT LOGIN физически не может подключиться к серверу баз данных извне (ни через SSMS, ни через еще что тл), так как у него нет учетных данных (логина/пароля).

---

## Задание 4. Создание логина с истёкшим сроком действия пароля
```sql
USE master;
GO

CREATE LOGIN TempUser 
WITH PASSWORD = 'TempPassword123!' MUST_CHANGE, 
CHECK_EXPIRATION = ON, 
CHECK_POLICY = ON;
GO
```
<img width="656" height="895" alt="image" src="https://github.com/user-attachments/assets/9cbb3d7a-1571-4d4e-a119-1529328eb222" />

> Система попросила придумать новый пароль:
> <img width="626" height="346" alt="image" src="https://github.com/user-attachments/assets/51090103-48bd-4a68-94eb-2467f3266503" />

---

## Задание 5. Просмотр списка всех логинов и пользователей
```sql
-- 1. Список логинов сервера (выполняется в любой БД, обращается к sys.server_principals)
SELECT 
    name AS LoginName, 
    create_date AS CreationDate, 
    is_disabled AS IsDisabled
FROM sys.server_principals
WHERE type IN ('S', 'U', 'G') -- S = SQL Login, U = Windows User, G = Windows Group
ORDER BY name;

-- 2. Список пользователей текущей базы данных
SELECT 
    name AS UserName, 
    create_date AS CreationDate, 
    type_desc AS UserType
FROM sys.database_principals
WHERE type IN ('S', 'U', 'G')
ORDER BY name;
```
[!ВСТАВИТЬ СКРИН С РАБОЕЧЕГО СТОЛА!]
---

## Задание 6. Добавление логина в фиксированную серверну
```sql
USE master;
GO

IF NOT EXISTS (SELECT * FROM sys.server_principals WHERE name = 'TestUser02')
    CREATE LOGIN TestUser02 WITH PASSWORD = 'StrongPassword123!';
GO

ALTER SERVER ROLE securityadmin ADD MEMBER TestUser02;
GO

SELECT IS_SRVROLEMEMBER('securityadmin', 'TestUser02') AS IsSecurityAdmin;
```
<img width="241" height="98" alt="image" src="https://github.com/user-attachments/assets/81c58cd4-0d29-411b-90b9-4a40f6bc70af" />

---

## Задание 7. Создание пользовательской серверной роли
```sql
USE master;
GO

-- 1. Создаем роль и выдаем права
CREATE SERVER ROLE CustomServerRole;
GRANT ALTER ANY LOGIN TO CustomServerRole;

-- 2. Добавляем пользователя
ALTER SERVER ROLE CustomServerRole ADD MEMBER TestUser01;
GO

-- 3. Проверка: переключаем контекст на TestUser01
EXECUTE AS LOGIN = 'TestUser01';

-- Пробуем создать новый логин (должно пройти успешно благодаря роли)
CREATE LOGIN TestUser02 WITH PASSWORD = 'Password123!';
PRINT 'Логин успешно создан!';

-- Возвращаемся под свою учетную запись (sysadmin)
REVERT;

-- Убираем мусор
DROP LOGIN TestUser02;
```

<img width="421" height="81" alt="image" src="https://github.com/user-attachments/assets/b9323d21-c484-4c27-b66c-64bd30893c49" />

---

## Задание 8. Проверка членства в серверных ролях
```sql
SELECT 
    p.name AS RoleName, 
    m.name AS LoginName
FROM sys.server_role_members srm
JOIN sys.server_principals p ON srm.role_principal_id = p.principal_id
JOIN sys.server_principals m ON srm.member_principal_id = m.principal_id
WHERE m.name = 'TestUser01';
```
<img width="296" height="112" alt="image" src="https://github.com/user-attachments/assets/86bb4f6f-46ef-4b26-84cb-948f75f596ff" />

---

## Задание 9. Удаление логина из роли и удаление пользовательский роли
```sql
USE master;
GO

-- 1. удаление пользователя из стандартной и пользовательской роли
ALTER SERVER ROLE securityadmin DROP MEMBER TestUser01;
ALTER SERVER ROLE CustomServerRole DROP MEMBER TestUser01;

-- 2. удаление пустую серверную роль
DROP SERVER ROLE CustomServerRole;
```
<img width="455" height="101" alt="image" src="https://github.com/user-attachments/assets/1901d901-7ec5-4142-9775-53672e1f433c" />

---

## Задание 10. Добавление пользователя в фиксированную роль БД
