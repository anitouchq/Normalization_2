# Лабораторная работа №2. Нормализация данных в БД до третьей нормальной формы (3NF)

## 1. Цель работы

Изучить принципы нормализации реляционных баз данных, выявить аномалии обновления, удаления и вставки, привести ненормализованную таблицу к третьей нормальной форме (3NF), выполнить перенос данных и проверить их целостность.

Дополнительно освоить создание резервных копий базы данных (дампов), ведение журнала действий пользователя MySQL (General Log) и его экспорт в отдельный файл.

## 2. Используемые инструменты

- Visual Studio Code
- Git Bash
- Docker CLI (контейнер `my-mysql`)
- MySQL 8.0.46
- SQL

## 3. Создание базы данных и исходной таблицы

В Docker-контейнере `my-mysql` была создана база данных `sales_norm` и исходная ненормализованная таблица `sales_unnormalized`, заполненная 10 записями о продажах.

```sql
CREATE DATABASE IF NOT EXISTS sales_norm;
USE sales_norm;

CREATE TABLE sales_unnormalized (
    sale_id      INT NOT NULL,
    sale_date    DATE NOT NULL,
    client_id    INT NOT NULL,
    client_name  VARCHAR(100) NOT NULL,
    client_city  VARCHAR(100) NOT NULL,
    product_id   INT NOT NULL,
    product_name VARCHAR(100) NOT NULL,
    unit_price   DECIMAL(10,2) NOT NULL,
    quantity     INT NOT NULL,
    PRIMARY KEY (sale_id, product_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

INSERT INTO sales_unnormalized VALUES
(5001, '2026-11-20', 101, 'Иванов Иван',     'Москва',          201, 'Монитор',    15000.00, 2),
(5001, '2026-11-20', 101, 'Иванов Иван',     'Москва',          202, 'Клавиатура',  2000.00, 1),
(5002, '2026-11-20', 102, 'Петрова Мария',   'Санкт-Петербург', 203, 'Мышь',        1000.00, 3),
(5002, '2026-11-20', 102, 'Петрова Мария',   'Санкт-Петербург', 204, 'Наушники',    3500.00, 1),
(5003, '2026-11-20', 101, 'Иванов Иван',     'Москва',          205, 'Колонки',     2500.00, 2),
(5004, '2026-11-20', 103, 'Сидоров Алексей', 'Казань',          202, 'Клавиатура',  2000.00, 5),
(5004, '2026-11-20', 103, 'Сидоров Алексей', 'Казань',          203, 'Мышь',        1000.00, 2),
(5005, '2026-11-20', 102, 'Петрова Мария',   'Санкт-Петербург', 201, 'Монитор',    15000.00, 1),
(5006, '2026-11-20', 103, 'Сидоров Алексей', 'Казань',          204, 'Наушники',    3500.00, 2),
(5006, '2026-11-20', 103, 'Сидоров Алексей', 'Казань',          205, 'Колонки',     2500.00, 1);
```

### Исходные данные

| sale_id | sale_date  | client_id | client_name     | client_city     | product_id | product_name | unit_price | quantity |
|--------:|------------|----------:|-----------------|-----------------|-----------:|--------------|-----------:|---------:|
| 5001    | 2026-11-20 | 101       | Иванов Иван     | Москва          | 201        | Монитор      | 15000.00   | 2        |
| 5001    | 2026-11-20 | 101       | Иванов Иван     | Москва          | 202        | Клавиатура   | 2000.00    | 1        |
| 5002    | 2026-11-20 | 102       | Петрова Мария   | Санкт-Петербург | 203        | Мышь         | 1000.00    | 3        |
| 5002    | 2026-11-20 | 102       | Петрова Мария   | Санкт-Петербург | 204        | Наушники     | 3500.00    | 1        |
| 5003    | 2026-11-20 | 101       | Иванов Иван     | Москва          | 205        | Колонки      | 2500.00    | 2        |
| 5004    | 2026-11-20 | 103       | Сидоров Алексей | Казань          | 202        | Клавиатура   | 2000.00    | 5        |
| 5004    | 2026-11-20 | 103       | Сидоров Алексей | Казань          | 203        | Мышь         | 1000.00    | 2        |
| 5005    | 2026-11-20 | 102       | Петрова Мария   | Санкт-Петербург | 201        | Монитор      | 15000.00   | 1        |
| 5006    | 2026-11-20 | 103       | Сидоров Алексей | Казань          | 204        | Наушники     | 3500.00    | 2        |
| 5006    | 2026-11-20 | 103       | Сидоров Алексей | Казань          | 205        | Колонки      | 2500.00    | 1        |

## 4. Выявление аномалий

В таблице `sales_unnormalized` присутствуют частичные и транзитивные зависимости, что приводит к трём основным аномалиям. Для их демонстрации использовались транзакции с последующим откатом (`ROLLBACK`), чтобы сохранить исходные данные.

1. **Аномалия обновления.** Данные клиента дублируются в нескольких строках: изменение города в одной записи приводит к противоречию в данных.
   ```sql
   START TRANSACTION;
   UPDATE sales_unnormalized SET client_city = 'Сочи' WHERE sale_id = 5001 AND product_id = 201;
   SELECT sale_id, client_id, client_name, client_city FROM sales_unnormalized WHERE client_id = 101;
   ROLLBACK;
   ```

2. **Аномалия вставки.** Невозможно добавить нового клиента без указания продажи и товара из-за ограничений `NOT NULL` и первичного ключа `(sale_id, product_id)`.
   ```sql
   INSERT INTO sales_unnormalized (client_id, client_name, client_city)
   VALUES (104, 'Козлов Дмитрий', 'Екатеринбург');
   -- Ошибка: обязательные поля продажи и товара не заполнены
   ```

3. **Аномалия удаления.** При удалении всех продаж клиента полностью теряется информация о самом клиенте.
   ```sql
   START TRANSACTION;
   DELETE FROM sales_unnormalized WHERE sale_id IN (5004, 5006);
   SELECT DISTINCT client_id, client_name, client_city FROM sales_unnormalized;
   ROLLBACK;
   ```

## 5. Нормализация до 3NF и перенос данных

Исходная таблица разделена на четыре связанные таблицы с первичными (`PRIMARY KEY`) и внешними (`FOREIGN KEY`) ключами:

| Таблица | Назначение | Ключи |
|---|---|---|
| `clients` | Справочник клиентов | PK: `client_id` |
| `products` | Справочник товаров | PK: `product_id` |
| `sales` | Заголовки продаж | PK: `sale_id`, FK: `client_id → clients(client_id)` |
| `sale_items` | Позиции продаж | PK: `(sale_id, product_id)`, FK: `sale_id`, `product_id` |

### Создание нормализованных таблиц

```sql
CREATE TABLE clients (
    client_id   INT NOT NULL,
    client_name VARCHAR(100) NOT NULL,
    client_city VARCHAR(100) NOT NULL,
    PRIMARY KEY (client_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

CREATE TABLE products (
    product_id   INT NOT NULL,
    product_name VARCHAR(100) NOT NULL,
    unit_price   DECIMAL(10,2) NOT NULL,
    PRIMARY KEY (product_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

CREATE TABLE sales (
    sale_id   INT NOT NULL,
    sale_date DATE NOT NULL,
    client_id INT NOT NULL,
    PRIMARY KEY (sale_id),
    KEY client_id (client_id),
    CONSTRAINT sales_ibfk_1 FOREIGN KEY (client_id) REFERENCES clients (client_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

CREATE TABLE sale_items (
    sale_id    INT NOT NULL,
    product_id INT NOT NULL,
    quantity   INT NOT NULL,
    PRIMARY KEY (sale_id, product_id),
    KEY product_id (product_id),
    CONSTRAINT sale_items_ibfk_1 FOREIGN KEY (sale_id) REFERENCES sales (sale_id),
    CONSTRAINT sale_items_ibfk_2 FOREIGN KEY (product_id) REFERENCES products (product_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
```

### Что означает перенос данных в четыре таблицы

Перенос данных означает, что записи из исходной таблицы `sales_unnormalized` распределяются по четырём новым связанным таблицам. Это не создание пустых таблиц: SQL-команды `INSERT INTO ... SELECT` копируют в них соответствующие данные из исходной таблицы.

- `clients` — сведения о клиентах. Каждый клиент хранится один раз: 3 записи.
- `products` — сведения о товарах. Каждый товар хранится один раз: 5 записей.
- `sales` — сведения о самих продажах: дата и клиент, оформивший продажу. Всего 6 записей.
- `sale_items` — состав продаж: какой товар и в каком количестве входит в каждую продажу. Всего 10 записей.

Например, если одна продажа содержит несколько товаров, информация о самой продаже хранится в `sales` один раз, а каждый товар этой продажи — отдельной строкой в `sale_items`. Связи между таблицами обеспечиваются внешними ключами. Таким образом, данные не теряются, а распределяются по таблицам по своему назначению, что уменьшает дублирование.

### Перенос данных

```sql
INSERT INTO clients (client_id, client_name, client_city)
SELECT DISTINCT client_id, client_name, client_city FROM sales_unnormalized;

INSERT INTO products (product_id, product_name, unit_price)
SELECT DISTINCT product_id, product_name, unit_price FROM sales_unnormalized;

INSERT INTO sales (sale_id, sale_date, client_id)
SELECT DISTINCT sale_id, sale_date, client_id FROM sales_unnormalized;

INSERT INTO sale_items (sale_id, product_id, quantity)
SELECT sale_id, product_id, quantity FROM sales_unnormalized;
```

## 6. Проверка целостности данных

После переноса выполнена сверка количества записей, общего числа единиц товара и суммарной стоимости продаж до и после нормализации.

```sql
-- Контроль до нормализации
SELECT COUNT(*) AS rows_before, SUM(quantity) AS qty_before, SUM(unit_price * quantity) AS total_before
FROM sales_unnormalized;

-- Контроль после нормализации
SELECT COUNT(*) AS rows_after, SUM(si.quantity) AS qty_after, SUM(p.unit_price * si.quantity) AS total_after
FROM sale_items si
JOIN products p ON si.product_id = p.product_id;
```

| Показатель | До нормализации | После нормализации |
|---|---:|---:|
| Позиции продаж | 10 | 10 |
| Общее количество товаров | 20 | 20 |
| Общая стоимость продаж | 80 000.00 | 80 000.00 |

Количество уникальных записей в новых таблицах:
- `clients` — 3 записи;
- `products` — 5 записей;
- `sales` — 6 записей;
- `sale_items` — 10 записей.

После успешной проверки исходная таблица `sales_unnormalized` была удалена (`DROP TABLE sales_unnormalized;`). Повторная проверка операций добавления, обновления и удаления подтвердила отсутствие аномалий.

## 7. Резервное копирование и журнал действий (General Log)

Для фиксации состояния базы данных созданы два SQL-дампа, а также экспортирован журнал действий пользователя `general.log` (с параметрами `general_log = ON`, `log_output = FILE`, `general_log_file = /var/lib/mysql/general.log`).

### Файлы лабораторной работы

- [Дамп базы данных до нормализации (`sales_norm_before.sql`)](./sales_norm_before.sql)
- [Дамп базы данных после нормализации (`sales_norm_after.sql`)](./sales_norm_after.sql)
- [Журнал действий пользователя MySQL (`general.log`)](./general.log)

## 8. Вывод

В ходе лабораторной работы были изучены проблемы ненормализованных таблиц и способы их устранения. Исходная таблица `sales_unnormalized` приведена к третьей нормальной форме (3NF) путём декомпозиции на четыре связанные таблицы (`clients`, `products`, `sales`, `sale_items`). Аномалии обновления, вставки и удаления устранены, а контрольные проверки подтвердили полную сохранность данных. Освоены создание дампов MySQL и ведение журнала General Log.

## 9. Скриншоты выполнения работы

*Рисунок 1 — Выполнение шага 1:*
![Скриншот 1](./2IMG/1.png)

*Рисунок 2 — Выполнение шага 2:*
![Скриншот 2](./2IMG/2.png)

*Рисунок 3 — Выполнение шага 3:*
![Скриншот 3](./2IMG/3.png)

*Рисунок 4 — Выполнение шага 4:*
![Скриншот 4](./2IMG/4.png)

*Рисунок 5 — Выполнение шага 5:*
![Скриншот 5](./2IMG/5.png)

*Рисунок 6 — Выполнение шага 6:*
![Скриншот 6](./2IMG/6.png)

*Рисунок 7 — Выполнение шага 7:*
![Скриншот 7](./2IMG/7.png)

*Рисунок 8 — Выполнение шага 8:*
![Скриншот 8](./2IMG/8.png)

*Рисунок 9 — Выполнение шага 9:*
![Скриншот 9](./2IMG/9.png)

*Рисунок 10 — Выполнение шага 10:*
![Скриншот 10](./2IMG/10.png)

*Рисунок 11 — Выполнение шага 11:*
![Скриншот 11](./2IMG/11.png)

*Рисунок 12 — Выполнение шага 12:*
![Скриншот 12](./2IMG/12.png)

*Рисунок 13 — Выполнение шага 13:*
![Скриншот 13](./2IMG/13.png)

*Рисунок 14 — Выполнение шага 14:*
![Скриншот 14](./2IMG/14.png)