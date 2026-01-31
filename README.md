

```markdown
#  Trino + PostgreSQL + ClickHouse: Федеративная аналитика в Docker

Этот проект демонстрирует, как с помощью Trino объединить данные из PostgreSQL и ClickHouse в одном SQL-запросе.  
Trino выступает как федеративный SQL-координатор, позволяя выполнять аналитику "на лету" без ETL.

---

##  Структура проекта

```

![Структура проекта](https://github.com/user-attachments/assets/524bdc33-bc7b-4119-814b-976575f92f50)

---

##  Запуск инфраструктуры

```bash
docker-compose up -d
```

Проверь статус контейнеров:

```bash
docker-compose logs -f
```

Ожидаемые запущенные сервисы: `trino`, `postgres`, `clickhouse`.

---

##  Заполнение данных

### 1. PostgreSQL: создание таблицы и вставка данных

Подключись к PostgreSQL:

```bash
docker exec -it postgres psql -U trino -d demo
```

Выполни SQL:

```sql
CREATE TABLE public.orders (
    order_id SERIAL PRIMARY KEY,
    customer_name VARCHAR(100),
    amount DECIMAL
);

INSERT INTO public.orders (customer_name, amount) VALUES
('Alice', 120.50),
('Bob', 300.00),
('Charlie', 50.75);
```

Выйди из сессии:

```sql
\q
```

---

### 2. ClickHouse: создание таблицы и вставка данных

Подключись к ClickHouse:

```bash
docker exec -it clickhouse clickhouse-client -u trino --password trino
```

Выполни:

```sql
CREATE TABLE default.payments (
    payment_id UInt32,
    order_id UInt32,
    status String
) ENGINE = MergeTree()
ORDER BY payment_id;

INSERT INTO default.payments VALUES
(1, 1, 'Paid'),
(2, 2, 'Pending'),
(3, 3, 'Paid');
```

Проверь:

```sql
SELECT * FROM default.payments;
```

Выйди:

```sql
exit
```

---

##  Подключение к Trino

```bash
docker exec -it trino trino
```

---

##  Основные команды в Trino

### Показать все подключённые каталоги

```sql
SHOW CATALOGS;
```

### Показать схемы

```sql
SHOW SCHEMAS FROM postgres;
SHOW SCHEMAS FROM clickhouse;
```

### Показать таблицы

```sql
SHOW TABLES FROM postgres.public;
SHOW TABLES FROM clickhouse.default;
```

---

##  Федеративный запрос: объединение PostgreSQL и ClickHouse

>  ClickHouse хранит строки как `String`, но Trino может интерпретировать их как `VARBINARY`.  
> Используй `from_utf8()` для корректного отображения.

```sql
SELECT 
    o.order_id,
    o.customer_name,
    from_utf8(p.status) AS status
FROM postgres.public.orders o
JOIN clickhouse.default.payments p
    ON o.order_id = p.order_id;
```

**Ожидаемый результат:**

| order_id | customer_name | status  |
|----------|---------------|---------|
| 1        | Alice         | Paid    |
| 2        | Bob           | Pending |
| 3        | Charlie       | Paid    |

---

##  Примеры сложных запросов

### Объединение данных из двух источников (`UNION ALL`)

```sql
SELECT 'pg' AS source, order_id, customer_name, NULL AS status
FROM postgres.public.orders

UNION ALL

SELECT 'ch' AS source, order_id, NULL AS customer_name, from_utf8(status)
FROM clickhouse.default.payments;
```

### Агрегация по источникам

```sql
SELECT 
    'pg' AS source,
    COUNT(*) AS cnt
FROM postgres.public.orders

UNION ALL

SELECT 
    'ch' AS source,
    COUNT(*) AS cnt
FROM clickhouse.default.payments;
```

---

##  Производительность: что нужно знать

Trino — федеративная СУБД, позволяющая объединять данные из разных источников в одном SQL-запросе.

###  Преимущества
- Нет необходимости копировать данные.
- Гибкость: анализ "на лету".
- Поддержка множества коннекторов (S3, Kafka, MySQL, Hive и др.).

###  Ограничения при использовании >2–5 источников
- **Сетевой оверхед** — данные тянутся по сети.
- **Pushdown не всегда работает** — фильтры могут не передаваться в источник.
- **Конверсия типов** — особенно с ClickHouse (`String` → `VARBINARY`).
- **Нагрузка на координатор** — может стать узким местом.

>  **Рекомендация**: используй 2–4 источника в одном запросе.  
> Для сложных сценариев — построй ETL-пайплайны (Airflow, dbt) или материализуй данные.

---

##  Дополнительные улучшения

### 1. Явное указание движка в ClickHouse

```sql
ENGINE = MergeTree ORDER BY payment_id
```

### 2. Индексы в ClickHouse
Используй `ORDER BY`, `PRIMARY KEY` для оптимизации производительности.

### 3. Проверка pushdown

```sql
EXPLAIN
SELECT * FROM postgres.public.orders WHERE amount > 100;
```

Если в плане есть `remote` фильтр — значит, `WHERE` был передан в PostgreSQL (**predicate pushdown**).

---

##  Итог

С помощью этого проекта ты можешь:
- Выполнять SQL-запросы к разным БД одновременно.
- Проводить аналитику без предварительного копирования данных.
- Исследовать возможности федеративных запросов в Trino.



