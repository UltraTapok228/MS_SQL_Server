04.09.2026[cite: 6]

Отель - Компьютер[cite: 6]  
Этажи - Сервера[cite: 6]  
Номера - БДшки[cite: 6]

**Архитектура MS SQL Server**[cite: 6]
* Microsoft SQL Server — это корпоративная реляционная СУБД, построенная по клиент-серверной архитектуре.[cite: 6]
* Клиент отправляет запрос на языке T-SQL, а сервер принимает обращение и выполняет поставленную задачу, возвращая результат.[cite: 6]
* База данных состоит из двух файлов: самого файла данных и журнала транзакций (рабочая область, куда записывается информация до и после выполнения каждой транзакции).[cite: 6]

**Создание БД**[cite: 6]
В состав БД входят:[cite: 6]
* **Таблицы**: хранят собственно данные.[cite: 6]
* **Представления (Views)**: выражения языка SQL, которые возвращают набор данных в виде таблицы.[cite: 6] В обозревателе объектов они хранятся в отдельной папке и используются в том числе для создания статических запросов.[cite: 6]
* **Хранимые процедуры**: выполняют код на языке SQL по отношению к данным к БД (например, получает данные или изменяет их).[cite: 6]
* **BLOB-поля**: данные в формате больших двоичных данных.[cite: 6]
* **Иные метаданные**.[cite: 6]

**Системные БД**[cite: 6]  
Эти базы можно увидеть через SQL Server Management Studio в узле Databases -> System Databases:[cite: 6]
* **master**: главная база данных сервера, хранит все используемые логины пользователей сервера, их роли, различные конфигурационные настройки, имена и информацию о базах данных.[cite: 6]
* **model**: база данных представляет шаблон, на основе которого создаются другие базы данных.[cite: 6]
* **msdb**: хранит информацию о работе планировщика SQL, а также информацию о бекапах баз данных.[cite: 6]
* **tempdb**: используется как хранилище для временных объектов и заново пересоздается при каждом запуске сервера.[cite: 6]

**Какие есть типы данных:**[cite: 6]  
* **Целочисленные типы данных (хранятся числа)**:[cite: 6]
  - Tinyint: от 0 до 255 (занимает 1 байт).[cite: 6]
  - Smallint: от –32 768 до 32 767 (занимает 2 байта).[cite: 6]
  - Int: от –2 147 483 648 до 2 147 483 647 (занимает 4 байта).[cite: 6]
  - Bigint: очень большие числа (занимает 8 байт).[cite: 6]

* **Типы данных для хранения дробных чисел**:[cite: 6]  
  - Real, Float.[cite: 6]
  - Decimal: хранит числа c фиксированной точностью (от 5 до 17 байт в зависимости от количества чисел после запятой).[cite: 6]

* **Битовые (логические) типы данных**:[cite: 6]
  - Bit: хранит значение 0 или 1, фактически аналог булевого типа (занимает 1 байт).[cite: 6]

* **Типы данных даты и времени**:[cite: 6]  
  - Datetime, SmallDatetime.[cite: 6]

* **Денежные типы данных для хранения финансовой информации**:[cite: 6]  
  - Money, Smallmoney.[cite: 6]

* **Символьные (строковые) типы**:[cite: 6]  
  - Char(n): строка фиксированной длины (выделяет по 1 байту на символ, не Unicode).[cite: 6]
  - Varchar(n): строка переменной длины (выделяется 1 байт на символ, не Unicode).[cite: 6]
  - Nchar(n) / Nvarchar(n): хранит строку в кодировке Unicode (выделяется 2 байта на каждый символ).[cite: 6]

* **Специальные типы данных**:[cite: 6]
  - Text, Image (до 2 Гб рисунков), RowGUID (уникальный идентификатор строки), SQL_Variant.[cite: 6]

**Основные команды T-SQL и работа в SSMS**[cite: 6]
* **Создание базы данных**: выполняется командой `CREATE DATABASE название_бд`.[cite: 6] В графическом интерфейсе это делается через правый клик по узлу Databases -> New Database.[cite: 6]
* **Удаление базы данных**: применяется команда `DROP DATABASE название_бд`.[cite: 6]
* **Прикрепление существующей БД (.mdf файла)**: выполняется командой `CREATE DATABASE имя_бд ON PRIMARY(FILENAME='путь_к_файлу_mdf') FOR ATTACH;`.[cite: 6]
* **Создание таблиц**: команда `CREATE TABLE название_таблицы (...)` с перечислением столбцов и их типов.[cite: 6]
* **Дизайнер таблиц в SSMS**: при создании через интерфейс содержит три колонки: Column Name (имя), Data Type (тип) и Allow Nulls (может ли быть пустым).[cite: 6]
* **Установка первичного ключа**: правый клик по столбцу -> Set Primary Key (появляется иконка "золотой ключ").[cite: 6]
* **Создание автосчетчика (identity)**: в свойствах столбца нужно развернуть пункт "Спецификация идентификатора", установить значение "Да" и задать начальное значение и шаг приращения (обычно 1).[cite: 6]
* **Выполнение запросов**: для создания нового запроса нужно нажать New Query, а для выполнения написанного кода — кнопку Execute или клавишу F5.[cite: 6] Для извлечения данных из таблиц используется команда `SELECT`.[cite: 6]

## 10.09.2026 ##[cite: 6]

**Продвинутое создание БД (через код T-SQL)**[cite: 6]

База данных может создаваться с тонкой настройкой физических файлов.[cite: 6] В скрипте можно определить:[cite: 6]
* **.mdf** — основной файл данных (Primary).[cite: 6]
* **.ndf** — вторичные файлы данных (дополнительные).[cite: 6]
* **.ldf** — файл журнала транзакций (Log).[cite: 6]
* **FILEGROUP** — файловые группы (позволяют логически объединять файлы, например, `FG1`).[cite: 6]

Пример скрипта с настройкой файлов:[cite: 6]
```sql
CREATE DATABASE BSTU 
ON PRIMARY (
    NAME = N'BSTU_1', 
    FILENAME = N'D:\BD\BSTU_1.mdf', 
    SIZE = 10240Kb, 
    MAXSIZE = UNLIMITED, 
    FILEGROWTH = 1024Kb
), 
(
    NAME = N'BSTU_2', 
    FILENAME = N'D:\BD\BSTU_2.ndf', 
    SIZE = 10240KB, 
    MAXSIZE = 1Gb, 
    FILEGROWTH = 25%
), 
FILEGROUP FG1 (
    NAME = N'BSTU_fg1_1', 
    FILENAME = N'D:\BD\BSTU_3.ndf', 
    SIZE = 10240Kb, 
    MAXSIZE = 1Gb, 
    FILEGROWTH = 25%
), 
(
    NAME = N'BSTU_fg1_2', 
    FILENAME = N'D:\BD\BSTU_4.ndf', 
    SIZE = 10240Kb, 
    MAXSIZE = 1Gb, 
    FILEGROWTH = 25%
) 
LOG ON (
    NAME = N'BSTU_log', 
    FILENAME = N'D:\BD\BSTU_log.ldf', 
    SIZE = 10240Kb, 
    MAXSIZE = 2048Gb, 
    FILEGROWTH = 10%
);
```

**Разбор параметров файла:**[cite: 6]
* `NAME` — логическое имя файла внутри сервера.[cite: 6]
* `FILENAME` — физический путь к файлу на жестком диске.[cite: 6]
* `SIZE` — начальный выделенный размер.[cite: 6]
* `MAXSIZE` — максимальный размер (можно ограничить в Gb/Kb или указать `UNLIMITED`).[cite: 6]
* `FILEGROWTH` — шаг автоматического расширения файла (в мегабайтах/килобайтах или процентах).[cite: 6]

---

**Категории команд языка T-SQL**[cite: 6]

Все команды T-SQL логически делятся на несколько групп:[cite: 6]

* **DDL (Data Definition Language)** — язык определения данных.[cite: 6] Отвечает за структуру:[cite: 6]
  - `CREATE` — создание объектов (БД, таблиц).[cite: 6]
  - `ALTER` — изменение структуры объектов.[cite: 6]
  - `DROP` — удаление объектов.[cite: 6]

* **DML (Data Manipulation Language)** — язык манипулирования данными.[cite: 6] Отвечает за сами записи:[cite: 6]
  - `SELECT` — выборка (чтение) данных из таблиц.[cite: 6]
  - `INSERT` — вставка новых строк.[cite: 6]
  - `UPDATE` — обновление (изменение) существующих данных.[cite: 6]
  - `DELETE` — удаление строк.[cite: 6]

* **DCL (Data Control Language)** — язык управления доступом:[cite: 6]
  - `GRANT` — выдача разрешений.[cite: 6]
  - `REVOKE` — отзыв выданных разрешений.[cite: 6]
  - `DENY` — явный запрет на действия.[cite: 6]

11.09.26[cite: 6]
 
**T-sql**[cite: 6]
* Запрос с 14 слайда, CAST:[cite: 6]
* <img width="1633" height="845" alt="image" src="https://github.com/user-attachments/assets/b78e93cf-7299-49f6-b8b5-6ee4d7b565ca" />

* Запрос с 8 слайда, оператор PRINT[cite: 6]
* <img width="684" height="685" alt="image" src="https://github.com/user-attachments/assets/699ca62a-d8df-47ad-b7ca-c92aad7321d2" />

**База SQL-сервер**[cite: 6]
* 54 слайд[cite: 6]
* <img width="570" height="633" alt="image" src="https://github.com/user-attachments/assets/96fa9bca-6c2f-499c-9925-08b885fc6e05" />

* 52 слайд[cite: 6]
* <img width="1103" height="704" alt="image" src="https://github.com/user-attachments/assets/7c90fd94-3ed2-45bc-8e93-2944e240d318" />

 **Презентация T-sql**[cite: 6]

 * **28 Слайд**[cite: 6]
 * <img width="1812" height="841" alt="image" src="https://github.com/user-attachments/assets/da290362-4453-41ae-a0d5-b705cc3dc226" />

 * **29 слайд**[cite: 6]
 * <img width="1173" height="458" alt="image" src="https://github.com/user-attachments/assets/4345b8d4-5812-42fa-8a24-e897b98712b9" />
 > Отработано успешно, с результатом: `общее количество мест от 100 до 200`[cite: 6]

* **30 слайд**[cite: 6]
* Первый запрос:[cite: 6]
* <img width="1036" height="482" alt="image" src="https://github.com/user-attachments/assets/45754455-750d-42d0-be0e-403f3767d375" />
---
* Второй запрос:[cite: 6]
* <img width="1386" height="480" alt="image" src="https://github.com/user-attachments/assets/221582a7-fc44-4cc2-a168-40e6e10fa717" />

* **31 слайд**[cite: 6]
* <img width="1414" height="466" alt="image" src="https://github.com/user-attachments/assets/aa07c9ef-6c03-4070-bb59-f57aad353b5c" />

* **32 слайд**[cite: 6]
* Первый запрос:[cite: 6]
* <img width="1817" height="473" alt="image" src="https://github.com/user-attachments/assets/0fc9272e-1c72-4027-9f71-eb72bfefc741" />
---
* Второй запрос:[cite: 6]
* <img width="1396" height="459" alt="image" src="https://github.com/user-attachments/assets/fd494fa9-3912-4718-8f26-dfff779c599c" />


* **33 слайд**[cite: 6]
* <img width="1394" height="208" alt="image" src="https://github.com/user-attachments/assets/4463f6c3-74b5-436a-bcd4-07600cfffda0" />

* **34 слайд**[cite: 6]
* <img width="1161" height="478" alt="image" src="https://github.com/user-attachments/assets/d823c636-88cb-4396-b8d4-98dc2a962bc3" />

* **35 слайд**[cite: 6]
  Первый пример:[cite: 6]
* <img width="569" height="336" alt="image" src="https://github.com/user-attachments/assets/65d44653-c356-4c6e-8c3d-aa3f93eca8ef" />
---
  Второй пример:[cite: 6]
* <img width="563" height="391" alt="image" src="https://github.com/user-attachments/assets/b0f2d80a-d5b7-44b1-97b7-fe49939e62b8" />

16.09.26[cite: 6]
* **6 слайд**[cite: 6]
<img width="559" height="401" alt="image" src="https://github.com/user-attachments/assets/06a2a1a6-789a-4c8f-a9de-db516840e78d" />

* **Запрос с пустой колонкой**[cite: 6]
<img width="542" height="400" alt="image" src="https://github.com/user-attachments/assets/711b7586-1d5f-4e57-8e05-00288c11222d" />

* **Запрос с несколькими окнами**[cite: 6]
<img width="590" height="408" alt="image" src="https://github.com/user-attachments/assets/c5d9546c-93cf-4799-b698-1f7d2fb18c3a" />

* **Обычная сортировка**[cite: 6]
<img width="595" height="426" alt="image" src="https://github.com/user-attachments/assets/17d02c10-67e6-4ae4-80b2-cb01d4140db5" />

* **Ранжирование внутри групп(с использованием PARTITION BY)**[cite: 6]
<img width="590" height="399" alt="image" src="https://github.com/user-attachments/assets/aff16327-b062-4ee5-852e-9187c1cba013" />

18.09.26 Функции и процедуры[cite: 6]

* Код процедуры которая посчитает строки в таблице со Студентами:[cite: 6]
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

* Вызов процедуры[cite: 6]
```sql
EXEC GetStudentcsCount;
```

* Результат выполнения:[cite: 6]
  > <img width="320" height="238" alt="image" src="https://github.com/user-attachments/assets/d039ff1f-3c79-4837-a11b-7b051018c28c" />


* Задача 1. Средний балл студента[cite: 6]

> Задание: Создайте функцию GetStudentAvgGrade(student_id INT), которая возвращает средний балл студента по всем дисциплинам.[cite: 6] Если оценок нет — возвращает 0.[cite: 6]

  > <img width="712" height="587" alt="image" src="https://github.com/user-attachments/assets/83451c5e-6755-49cf-85af-6c596899336a" />


* Задача 3. Добавление нового студента[cite: 6]

> Задание: Создайте процедуру AddStudent, которая принимает имя, фамилию, email, group_id и год поступления, и добавляет студента.[cite: 6] Проверьте, что группа существует.[cite: 6]

 > <img width="638" height="643" alt="image" src="https://github.com/user-attachments/assets/1d51c97e-05d3-4ae8-aa82-cb2cbfb3afea" />


 * **APPLY(2 штуки)**[cite: 6]
  1. CROSS APPLY - связывает таблицы, возвращаю данные только если подзапрос что то нашел[cite: 6]
     <img width="424" height="511" alt="image" src="https://github.com/user-attachments/assets/839f9ae4-1c34-4a10-8c8c-3f1068c03fdc" />

  2. OUTER APPLY - работает как LEFT JOIN, возвращает группу, даже если в ней еще нет ни одного студента[cite: 6]
    <img width="522" height="496" alt="image" src="https://github.com/user-attachments/assets/d0f94a90-42ff-4c53-9287-eb11fe2cb804" />

* **Классические методы группировки(5 штук)**[cite: 6]
  1. Классический GROUP BY с агрегатной функцией[cite: 6]
    <img width="451" height="314" alt="image" src="https://github.com/user-attachments/assets/95361a38-b8cc-4d28-a323-2edbec9bfb60" />

  2. GROUP BY с условием HAVING[cite: 6]
    <img width="447" height="301" alt="image" src="https://github.com/user-attachments/assets/cec2e339-6995-48a0-bddb-2119388c2648" />

  3. GROUP BY ROLLUP[cite: 6]
    <img width="475" height="424" alt="image" src="https://github.com/user-attachments/assets/0f6b26d6-e3ba-470d-87c8-933a82e9ef18" />

  4. GROUPING SETS[cite: 6]
    <img width="491" height="448" alt="image" src="https://github.com/user-attachments/assets/e8d62313-6132-4762-acee-a03b4b3425b7" />

  5. Классическая группировка в подзапросе[cite: 6]
    <img width="600" height="479" alt="image" src="https://github.com/user-attachments/assets/60ad6f29-8979-4130-9bb9-c607ac75b2f8" />

* **Оконные функции(3 штуки)**[cite: 6]
  1. Окно с базовой агрегацией/OVER PRTITION BY[cite: 6]
    <img width="590" height="354" alt="image" src="https://github.com/user-attachments/assets/41f54b28-265d-47d8-a988-4ed8138edcc7" />

  2. Окно с функцией ранжирования/ROW_NUMBER[cite: 6]
    <img width="705" height="355" alt="image" src="https://github.com/user-attachments/assets/2b711478-f51d-4446-a470-c993d977f345" />


  3. Окно со смещением/LAG/LEAD[cite: 6]
    <img width="540" height="363" alt="image" src="https://github.com/user-attachments/assets/1c0d826b-2d41-4413-95e8-09dc9a3ea1e8" />