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
