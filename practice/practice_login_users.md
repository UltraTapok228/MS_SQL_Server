# Практическая работа. Логины, пользователи и роли

Создание логинов и пользователей, работа с фиксированными и пользовательскими серверными ролями, выдача прав.



---

## Задание 1. Создание SQL-логина с политикой паролей

### Что требовалось
Создать SQL-логин с включённой проверкой политики паролей.

### Что сделано
Создан логин TestUser01 с паролем, соответствующим политике, и CHECK_POLICY = ON.

```sql
USE master;
GO

CREATE LOGIN TestUser01 
WITH PASSWORD = 'P@ssw0rd123!', 
CHECK_POLICY = ON;
GO
```

### Результат
<img width="653" height="905" alt="image" src="https://github.com/user-attachments/assets/90ab5728-d803-46b9-98ad-de5447bd381a" />
<img width="541" height="497" alt="image" src="https://github.com/user-attachments/assets/b0a227dd-eada-4328-8fe2-2f3755c6f71b" />

### Вывод
При CHECK_POLICY = ON пароль обязан соответствовать политике Windows (длина, сложность).

---

## Задание 2. Создание логина Windows и сопоставление с пользователем БД

### Что требовалось
Создать логин для учётной записи Windows и сопоставить его с пользователем базы данных.

### Что сделано
Создан логин на основе реальной учётной записи Windows, затем в базе College создан пользователь JohnDoe, привязанный к этому логину.

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

### Результат
<img width="241" height="100" alt="image" src="https://github.com/user-attachments/assets/30919d30-9423-437d-b27d-5e433fe1257e" />

### Вывод
Логин существует на уровне сервера, пользователь — на уровне базы, и они связываются через FOR LOGIN.

---

## Задание 3. Создание пользователя без логина

### Что требовалось
Создать пользователя базы данных без логина.

### Что сделано
Создана база TestDB (если её нет) и пользователь AppServiceUser WITHOUT LOGIN.

```sql
-- Создадим БД, если ее нет
IF DB_ID('TestDB') IS NULL CREATE DATABASE TestDB;
GO

USE TestDB;
GO

CREATE USER AppServiceUser WITHOUT LOGIN;
GO
```

### Результат
<img width="503" height="125" alt="image" src="https://github.com/user-attachments/assets/73144e38-673c-4cbd-86aa-233dcae50303" />

### Анализ
Пользователь WITHOUT LOGIN физически не может подключиться к серверу извне (ни через SSMS, ни другим способом), так как у него нет учётных данных (логина и пароля).

### Вывод
Такие пользователи нужны для внутренних задач: выдачи прав, подмены контекста (EXECUTE AS), подписи модулей.

---

## Задание 4. Создание логина с истёкшим сроком действия пароля

### Что требовалось
Создать логин, у которого пароль нужно сменить при первом входе.

### Что сделано
Создан логин TempUser с опцией MUST_CHANGE и включёнными CHECK_EXPIRATION и CHECK_POLICY.

```sql
USE master;
GO

CREATE LOGIN TempUser 
WITH PASSWORD = 'TempPassword123!' MUST_CHANGE, 
CHECK_EXPIRATION = ON, 
CHECK_POLICY = ON;
GO
```

### Результат
<img width="656" height="895" alt="image" src="https://github.com/user-attachments/assets/9cbb3d7a-1571-4d4e-a119-1529328eb222" />

Система попросила придумать новый пароль:

<img width="626" height="346" alt="image" src="https://github.com/user-attachments/assets/51090103-48bd-4a68-94eb-2467f3266503" />

### Вывод
MUST_CHANGE заставляет пользователя сменить временный пароль при первом входе.

---

## Задание 5. Просмотр списка всех логинов и пользователей

### Что требовалось
Вывести список логинов сервера и пользователей текущей базы данных.

### Что сделано
Выполнены два запроса: к sys.server_principals (логины) и к sys.database_principals (пользователи базы).

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

### Результат
<!-- TODO: вставить скриншот с рабочего стола -->

### Вывод
Логины хранятся в sys.server_principals, пользователи базы — в sys.database_principals.

---

## Задание 6. Добавление логина в фиксированную серверную роль

### Что требовалось
Добавить логин в фиксированную серверную роль и проверить членство.

### Что сделано
Создан логин TestUser02 (если его нет), добавлен в роль securityadmin, членство проверено функцией IS_SRVROLEMEMBER.

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

### Результат
<img width="241" height="98" alt="image" src="https://github.com/user-attachments/assets/81c58cd4-0d29-411b-90b9-4a40f6bc70af" />

### Вывод
IS_SRVROLEMEMBER возвращает 1, если логин входит в указанную роль.

---

## Задание 7. Создание пользовательской серверной роли

### Что требовалось
Создать пользовательскую серверную роль с правом, добавить в неё логин и проверить работу права.

### Что сделано
Создана роль CustomServerRole с правом ALTER ANY LOGIN, в неё добавлен TestUser01. Затем выполнена проверка под этим логином (EXECUTE AS LOGIN): создание нового логина прошло успешно. После проверки контекст возвращён (REVERT), тестовый логин удалён.

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

### Результат
<img width="421" height="81" alt="image" src="https://github.com/user-attachments/assets/b9323d21-c484-4c27-b66c-64bd30893c49" />

### Вывод
Пользовательская серверная роль позволяет выдать точечное право, не давая широких фиксированных ролей.

---

## Задание 8. Проверка членства в серверных ролях

### Что требовалось
Вывести серверные роли, в которых состоит логин.

### Что сделано
Выполнен запрос к sys.server_role_members с соединением по sys.server_principals для логина TestUser01.

```sql
SELECT 
    p.name AS RoleName, 
    m.name AS LoginName
FROM sys.server_role_members srm
JOIN sys.server_principals p ON srm.role_principal_id = p.principal_id
JOIN sys.server_principals m ON srm.member_principal_id = m.principal_id
WHERE m.name = 'TestUser01';
```

### Результат
<img width="296" height="112" alt="image" src="https://github.com/user-attachments/assets/86bb4f6f-46ef-4b26-84cb-948f75f596ff" />

---

## Задание 9. Удаление логина из роли и удаление пользовательской роли

### Что требовалось
Убрать логин из стандартной и пользовательской ролей и удалить пользовательскую роль.

### Что сделано
TestUser01 удалён из ролей securityadmin и CustomServerRole, после чего пустая роль CustomServerRole удалена.

```sql
USE master;
GO

-- 1. удаление пользователя из стандартной и пользовательской роли
ALTER SERVER ROLE securityadmin DROP MEMBER TestUser01;
ALTER SERVER ROLE CustomServerRole DROP MEMBER TestUser01;

-- 2. удаление пустую серверную роль
DROP SERVER ROLE CustomServerRole;
```

### Результат
<img width="455" height="101" alt="image" src="https://github.com/user-attachments/assets/1901d901-7ec5-4142-9775-53672e1f433c" />

### Вывод
Удалить серверную роль можно только после того, как из неё убраны все члены.

---

## Задание 11. Роль с правами на таблицу

### Что требовалось
Создать роль SalesRole с правами SELECT, INSERT, UPDATE на таблицу Sales.SalesOrderHeader и добавить в неё пользователя JohnDoe.

### Что сделано
```sql
CREATE ROLE SalesRole;

GRANT SELECT, INSERT, UPDATE ON Sales.SalesOrderHeader TO SalesRole;

ALTER ROLE SalesRole ADD MEMBER JohnDoe;
GO
```

---

## Задание 12. Проверка доступа к объектам схемы

### Что требовалось
Проверить доступ пользователя RestrictedUser к объекту схемы без прямого запрета и к объекту с явным DENY.

### Что сделано
Под контекстом RestrictedUser выполнены два запроса: к Sales.SalesOrderHeader и к Sales.CreditCard (на неё установлен явный DENY).

```sql
USE College;
GO
EXECUTE AS USER = 'RestrictedUser';

-- 1. Доступ к объекту схемы без прямого запрета
SELECT * FROM Sales.SalesOrderHeader;

-- 2. Доступ к объекту с явным DENY (запрещено)
SELECT * FROM Sales.CreditCard;

REVERT;
GO
```

---

## Задание 13. Права на вызов процедур

### Что требовалось
Проверить, что пользователь AppServiceUser может вызвать разрешённую процедуру и не может вызвать неразрешённую.

### Что сделано
Создана процедура uspOtherProc. Под контекстом AppServiceUser вызваны разрешённая процедура uspGetEmployeeManagers и неразрешённая uspOtherProc.

```sql
USE College;
GO

IF OBJECT_ID('dbo.uspOtherProc', 'P') IS NOT NULL DROP PROCEDURE dbo.uspOtherProc;
GO
CREATE PROCEDURE dbo.uspOtherProc AS SELECT 'Секретно' AS Msg;
GO

EXECUTE AS USER = 'AppServiceUser';

-- 1 Вызов разрешенной процедуры
EXEC dbo.uspGetEmployeeManagers;

-- 2 Вызов неразрешенной процедуры
EXEC dbo.uspOtherProc;

REVERT;
GO
```

---

## Общий вывод

| Задание | Что показал |
|---|---|
| 1, 4 | SQL-логины: политика паролей, MUST_CHANGE |
| 2, 3 | Windows-логин, пользователь БД, пользователь без логина |
| 5, 8 | Просмотр логинов, пользователей и членства в ролях |
| 6, 7, 9 | Фиксированные и пользовательские серверные роли |
| 11–13 | Права на таблицы и процедуры, проверка через EXECUTE AS |

**Ключевые принципы:**
1. Логин — уровень сервера, пользователь — уровень базы.
2. CHECK_POLICY = ON — обязательное требование безопасности.
3. Права выдаются ролям, а не напрямую (принцип минимальных привилегий).
4. EXECUTE AS ... / REVERT позволяет проверить права, не заходя под другим пользователем.

---

## Стек

Microsoft SQL Server 2017, SSMS 22, базы College, TestDB.
