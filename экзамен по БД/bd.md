**Общая подготовка:**
- Экземпляр SQL Server 2025 (Developer Edition).
- Скрипт `00_setup.sql`, который создаёт БД `ShopDB` со схемами `app`, `rpt`, `sec`, `audit` и таблицами:
  - `app.Customers(CustomerID, TenantID, Region, Name, Email, CreatedAt)`
  - `app.Orders(OrderID, CustomerID, OrderDate, Status, Total)`
  - `app.OrderDetails(OrderDetailID, OrderID, ProductID, Qty, Price)`
  - `app.Products(ProductID, CategoryID, Name, Price)`
  - `app.Categories(CategoryID, ParentID, Name)`
  - `app.Payments(PaymentID, OrderID, PaidAt, Amount)`
  - `audit.OrderLog(LogID, OrderID, Action, ChangedAt, ChangedBy)`
- Наполнение: ~50 000 заказов, ~200 000 позиций, ~5 000 клиентов, ~500 товаров, дерево категорий глубиной 5.
- Каждый билет работает со своей копией БД: `ShopDB_T01` … `ShopDB_T15`.

**Формат билета:** 5 заданий, 90 минут, отчёт — скрипты + вывод + пояснения.

---

## Билет 2. «Аудит после ADO.NET»

**Подготовка:** восстановить `ShopDB_T02`, включить `READ_COMMITTED_SNAPSHOT ON`.

**Задание 1.**
- Найти в `app.Orders` колонки с `DATETIME` вместо `DATETIME2`, `VARCHAR` вместо `NVARCHAR`, `NULL` там, где ожидается `NOT NULL`.
- Найти отсутствующие FK между `app.OrderDetails` и `app.Products`.
- Заменить курсор по заказам на CTE + `MERGE` в `app.OrderDetails`.

**Задание 2.**
- `RANK()` и `LAG()` для сравнения суммы заказов по месяцам у каждого клиента.
- `OUTER APPLY` для последнего платежа по заказу.
- Создать `rpt.vw_AppCustomer` как стабильный контракт.
- Создать `app.usp_UpsertCustomer` с `OUTPUT @Result` и error contract.
- Создать inline TVF `app.fn_OrdersByStatus(@Status)` и сравнить с scalar по плану.

**Задание 3.**
- `INSTEAD OF UPDATE` триггер на `rpt.vw_AppCustomer` с multi-row обработкой.
- Транзакция с `SAVE TRANSACTION` и откатом к точке сохранения.
- Воспроизвести фантомное чтение при `READ COMMITTED SNAPSHOT`.
- Найти транзакцию, блокирующую `ALTER TABLE app.Orders`.
- Снять граф deadlock через Extended Events (`system_health`), исправить порядок доступа.

**Задание 4.**
- Сравнить estimated и actual plan, оценить cardinality estimate.
- Сравнить clustered и heap на `app.OrderDetails`.
- Создать filtered index `WHERE Status='Active'`, покрыть запрос.
- В Query Store найти регрессию после `UPDATE STATISTICS`.
- Выполнить `MERGE` большой таблицы пакетами, следить за tempdb.

**Задание 5.**
- Создать схему `app`, роль `app_executor`, grant execute на процедуры.
- RLS по `Region` на `app.Customers`.
- Audit на `UPDATE` к `app.Customers`, DDM на `Email`.
- Full + log backup, восстановление на точку `STOPAT`.

---

## Регламент билета (90 минут, 5 заданий)

| Задание | Темы |
|---------|------|
| 1 | Стенд, аудит схемы, CTE, set-based |
| 2 | Оконные функции, APPLY, представления, процедуры, функции |
| 3 | Триггеры, транзакции, изоляция, locks, deadlock |
| 4 | Execution Plan, индексы, SARGability, Query Store, tempdb |
| 5 | Безопасность, least privilege, RLS, аудит, backup/recovery |

**Критерии оценки:** работающий стенд, корректные T-SQL-скрипты, воспроизведённые сценарии блокировок и deadlock, разобранные планы, настроенная безопасность, рабочий backup chain, письменный отчёт с пояснениями.
