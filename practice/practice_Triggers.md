# Практическая работа. Триггеры

Создание триггера AFTER INSERT на таблицу студентов в базе College.

---

## Задание 1. Триггер AFTER INSERT


### Что требовалось
Создать триггер, который срабатывает после добавления нового студента, и проверить его работу.

### Что сделано
Создан триггер trg_AfterStudentInsert на таблицу STUDENT. Он читает имя нового студента из псевдотаблицы inserted и выводит сообщение о зачислении. Для проверки добавлен студент.

```sql
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

### Результат
<img width="365" height="92" alt="image" src="https://github.com/user-attachments/assets/d51e90e5-c480-4213-9db4-6430bf120087" />

### Анализ
AFTER-триггер выполняется после успешной вставки. Данные добавленной строки доступны в псевдотаблице inserted. ISNULL защищает от пустого значения имени.

### Вывод
Триггер AFTER INSERT позволяет автоматически реагировать на добавление записей.

---

## Общий вывод

| Задание | Что показал |
|---|---|
| 1 | Триггер AFTER INSERT и псевдотаблица inserted |

---

## Стек

Microsoft SQL Server 2017, SSMS 22, база College.
