## 1 Триггер. AFTER INSERT

```
USE College;
GO

CREATE TRIGGER trg_AfterStudentInsert
ON STUDENT
AFTER INSERT
AS
BEGIN
    DECLARE @NewStudentName NVARCHAR(100);
    
    SELECT @NewStudentName = NAME FROM inserted;
    
    PRINT 'В базу успешно зачислен новый студент — ' + ISNULL(@NewStudentName, 'Неизвестно');
END;
GO

INSERT INTO STUDENT (IDGROUP, NAME, BDAY) VALUES (1, 'Гена Букин', '2000-08-08');
```

* Результат выполнения:
  <img width="365" height="92" alt="image" src="https://github.com/user-attachments/assets/d51e90e5-c480-4213-9db4-6430bf120087" />  


## 24.09.26
### Транзакции

**BEGIN TRANSACTION**
```
BEGIN TRANSACTION;

SELECT COUNT(*) FROM AUDITORIUM;

INSERT INTO AUDITORIUM VALUES ('123-1', 'ЛК', 60, '123-1');

SELECT COUNT(*) FROM AUDITORIUM;

COMMIT;

SELECT COUNT(*) FROM AUDITORIUM;
```  

**ROLLBACK**
```
BEGIN TRANSACTION;

SELECT COUNT(*) FROM AUDITORIUM;

INSERT INTO AUDITORIUM VALUES ('124-1', 'ЛК', 60, '124-1');

SELECT COUNT(*) FROM AUDITORIUM;

ROLLBACK;

SELECT COUNT(*) FROM AUDITORIUM;
```

> Результат:
<img width="718" height="326" alt="image" src="https://github.com/user-attachments/assets/06f14d54-9852-4e86-a090-ee5b79076997" />  

**SAVE TRANSACTION**
```
BEGIN TRANSACTION;

INSERT INTO AUDITORIUM VALUES ('999-1', 'ЛК', 60, '999-1');
SAVE TRANSACTION A;

INSERT INTO AUDITORIUM VALUES ('999-2', 'ЛК', 60, '999-2');
SAVE TRANSACTION B;

INSERT INTO AUDITORIUM VALUES ('999-3', 'ЛК', 60, '999-3');
ROLLBACK TRANSACTION B;

INSERT INTO AUDITORIUM VALUES ('999-4', 'ЛК', 60, '999-4');
ROLLBACK TRANSACTION A;

COMMIT TRANSACTION;

SELECT * FROM AUDITORIUM
WHERE AUDITORIUM IN('999-1', '999-2', '999-3', '999-4');
```

> Результат:
<img width="558" height="69" alt="image" src="https://github.com/user-attachments/assets/53af0769-6ece-43e6-905e-6e96169bdbcb" />  

**TRANCOUNT**
```
-- Шаг 1: Транзакций нет (@@TRANCOUNT = 0)
SELECT COUNT(*) '1', @@TRANCOUNT 'TRANCOUNT' FROM AUDITORIUM;
BEGIN TRANSACTION A;

-- Шаг 2: Открыта первая транзакция (@@TRANCOUNT = 1)
INSERT INTO AUDITORIUM VALUES ('128-1', 'ЛК', 60, '128-1');
SELECT COUNT(*) '2', @@TRANCOUNT 'TRANCOUNT' FROM AUDITORIUM;

BEGIN TRANSACTION B;

-- Шаг 3: Открыта вложенная транзакция (@@TRANCOUNT = 2)
DELETE AUDITORIUM WHERE AUDITORIUM = '128-1';
SELECT COUNT(*) '3', @@TRANCOUNT 'TRANCOUNT' FROM AUDITORIUM;

COMMIT TRANSACTION B;

-- Шаг 4: Вложенная транзакция закрыта, но внешняя еще активна (@@TRANCOUNT = 1)
SELECT COUNT(*) '4', @@TRANCOUNT 'TRANCOUNT' FROM AUDITORIUM;

-- Шаг 5: Снова вставляем запись
INSERT INTO AUDITORIUM VALUES ('128-1', 'ЛК', 60, '128-1');
SELECT COUNT(*) '5', @@TRANCOUNT 'TRANCOUNT' FROM AUDITORIUM;

COMMIT TRANSACTION A;

-- Шаг 6: Все транзакции закрыты, данные зафиксированы (@@TRANCOUNT = 0)
SELECT COUNT(*) '6', @@TRANCOUNT 'TRANCOUNT' FROM AUDITORIUM;
```
> Результат:
<img width="707" height="405" alt="image" src="https://github.com/user-attachments/assets/4fb6bc23-1dbe-47b5-b649-c043142040c2" />  

**Блокировки**
```
SELECT resource_type,
DB_NAME(resource_database_id) as database_name,
request_session_id, request_mode,
request_status
FROM sys.dm_tran_locks;
```
> Результат:
<img width="560" height="270" alt="image" src="https://github.com/user-attachments/assets/0ae62503-69a6-4394-a6ce-df90652f861b" />

**Взаимоблокировки**
#1
```
USE College;
GO

BEGIN TRANSACTION;
-- Блокируем первую аудиторию
UPDATE AUDITORIUM
	SET AUDITORIUM_CAPACITY = 70
	WHERE AUDITORIUM = '999-1';

-- ожидание
WAITFOR DELAY '00:00:10';

-- Пытаемся заблокировать вторую аудиторию
UPDATE AUDITORIUM
	SET AUDITORIUM_CAPACITY = 80
	WHERE AUDITORIUM = '999-2';
COMMIT;
```
#2
```
USE College;
GO

BEGIN TRANSACTION;
-- Блокируем вторую аудиторию
UPDATE AUDITORIUM
	SET AUDITORIUM_CAPACITY = 70
	WHERE AUDITORIUM = '999-2';

-- ожидание
WAITFOR DELAY '00:00:10';

-- Пытаемся заблокировать первую аудиторию
UPDATE AUDITORIUM
	SET AUDITORIUM_CAPACITY = 80
	WHERE AUDITORIUM = '999-1';
COMMIT;
```

> Результат:
<img width="376" height="133" alt="image" src="https://github.com/user-attachments/assets/422f885e-af2b-4749-b469-8ea687783921" />


