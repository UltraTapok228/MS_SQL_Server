# Практическая работа. Функции, процедуры, APPLY, группировки и оконные функции

Дата: 18.09.26

Хранимые процедуры и функции, оператор APPLY, методы группировки и оконные функции в базе College.

---

## Задание 1. Процедура подсчёта студентов

<!-- TODO: формулировка восстановлена по решению, проверить -->
### Что требовалось
Создать процедуру, которая считает строки в таблице со студентами, и вызвать её.

### Что сделано
Создана процедура GetStudentcsCount, возвращающая количество студентов, и выполнен её вызов.

```sql
USE College;
GO

CREATE PROCEDURE GetStudentcsCount
AS
BEGIN
	SELECT COUNT(*) AS [Кол-во студентов]
	FROM STUDENT;
END;
GO
```

```sql
EXEC GetStudentcsCount;
```

### Результат
<img width="320" height="238" alt="image" src="https://github.com/user-attachments/assets/d039ff1f-3c79-4837-a11b-7b051018c28c" />

### Вывод
Хранимая процедура инкапсулирует запрос и вызывается командой EXEC.

---

## Задача 1. Средний балл студента

### Что требовалось
Создать функцию GetStudentAvgGrade(student_id INT), которая возвращает средний балл студента по всем дисциплинам. Если оценок нет — возвращает 0.

### Что сделано
Функция создана (код и результат на скриншоте).

### Результат
<img width="712" height="587" alt="image" src="https://github.com/user-attachments/assets/83451c5e-6755-49cf-85af-6c596899336a" />

---

## Задача 3. Добавление нового студента

### Что требовалось
Создать процедуру AddStudent, которая принимает имя, фамилию, email, group_id и год поступления и добавляет студента. Проверить, что группа существует.

### Что сделано
Процедура создана, перед добавлением проверяется существование группы (код и результат на скриншоте).

### Результат
<img width="638" height="643" alt="image" src="https://github.com/user-attachments/assets/1d51c97e-05d3-4ae8-aa82-cb2cbfb3afea" />

---

## Задание 2. Оператор APPLY (2 вида)

<!-- TODO: формулировка восстановлена по решению, проверить -->
### Что требовалось
Показать два вида оператора APPLY: CROSS APPLY и OUTER APPLY.

### Что сделано
Выполнены запросы с обоими вариантами.

### Результат
**CROSS APPLY** — связывает таблицы, возвращая данные только если подзапрос что-то нашёл.

<img width="424" height="511" alt="image" src="https://github.com/user-attachments/assets/839f9ae4-1c34-4a10-8c8c-3f1068c03fdc" />

**OUTER APPLY** — работает как LEFT JOIN, возвращает группу, даже если в ней ещё нет ни одного студента.

<img width="522" height="496" alt="image" src="https://github.com/user-attachments/assets/d0f94a90-42ff-4c53-9287-eb11fe2cb804" />

### Вывод
CROSS APPLY отсекает строки без результата подзапроса, OUTER APPLY сохраняет их.

---

## Задание 3. Классические методы группировки (5 штук)

<!-- TODO: формулировка восстановлена по решению, проверить -->
### Что требовалось
Показать пять классических методов группировки данных.

### Что сделано
Выполнены запросы с пятью видами группировки.

### Результат
**1. Классический GROUP BY с агрегатной функцией**

<img width="451" height="314" alt="image" src="https://github.com/user-attachments/assets/95361a38-b8cc-4d28-a323-2edbec9bfb60" />

**2. GROUP BY с условием HAVING**

<img width="447" height="301" alt="image" src="https://github.com/user-attachments/assets/cec2e339-6995-48a0-bddb-2119388c2648" />

**3. GROUP BY ROLLUP**

<img width="475" height="424" alt="image" src="https://github.com/user-attachments/assets/0f6b26d6-e3ba-470d-87c8-933a82e9ef18" />

**4. GROUPING SETS**

<img width="491" height="448" alt="image" src="https://github.com/user-attachments/assets/e8d62313-6132-4762-acee-a03b4b3425b7" />

**5. Классическая группировка в подзапросе**

<img width="600" height="479" alt="image" src="https://github.com/user-attachments/assets/60ad6f29-8979-4130-9bb9-c607ac75b2f8" />

---

## Задание 4. Оконные функции (3 штуки)

<!-- TODO: формулировка восстановлена по решению, проверить -->
### Что требовалось
Показать три вида оконных функций.

### Что сделано
Выполнены запросы с тремя видами окон.

### Результат
**1. Окно с базовой агрегацией — OVER (PARTITION BY)**

<img width="590" height="354" alt="image" src="https://github.com/user-attachments/assets/41f54b28-265d-47d8-a988-4ed8138edcc7" />

**2. Окно с функцией ранжирования — ROW_NUMBER**

<img width="705" height="355" alt="image" src="https://github.com/user-attachments/assets/2b711478-f51d-4446-a470-c993d977f345" />

**3. Окно со смещением — LAG / LEAD**

<img width="540" height="363" alt="image" src="https://github.com/user-attachments/assets/1c0d826b-2d41-4413-95e8-09dc9a3ea1e8" />

---

## Общий вывод

| Задание | Что показал |
|---|---|
| 1 | Хранимая процедура и её вызов |
| Задачи 1, 3 | Пользовательская функция и процедура с проверкой |
| 2 | CROSS APPLY и OUTER APPLY |
| 3 | GROUP BY, HAVING, ROLLUP, GROUPING SETS, группировка в подзапросе |
| 4 | Оконные функции: PARTITION BY, ROW_NUMBER, LAG/LEAD |

---

## Стек

Microsoft SQL Server 2017, SSMS 22, база College.
