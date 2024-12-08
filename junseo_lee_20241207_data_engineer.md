# Data Engineer Assignment

## ToC

- [Common Assumptions](#common-assumptions)
    - [Date Column](#date-column)
    - [Timezone](#timezone)
    - [Currency Exchange Rate](#currency-exchange-rate)
        - [Why these rates?](#why-these-rates)
- [Question 1](#question-1)
- [Question 2](#question-2)
- [Question 3](#question-3)
- [Question 4](#question-4)
- [Question 5](#question-5)

## Common Assumptions

Assumptions made in this section applies to all questions.

### Date Column

Although the schema provided in the `ASSIGNMENT.md` states that the `date` column in the `transactions` table is of
type `DATE`, the `transactions.csv` file contains timestamps with both date and time information. I have assumed that
the `date` column in the `transactions` table is of type `TIMESTAMP`.

### Timezone

As the timestamps in the transactions table are in UTC, I have answered the questions based on the UTC timezone.
For example, when filtering the transactions based on the date, I have used the UTC date range.

### Currency Exchange Rate

After analyzing the transactions table using the below query, I have noticed that the currencies used in the
transactions are KRW, JPY, and USD.

```sql
SELECT DISTINCT total_price_currency
FROM transactions;
```

To answer the questions that require price calculation, I have converted the prices to USD using the exchange rates
provided below. And all the prices in the answers are in USD.

- 1 **KRW** = 0.0007663 **USD** ([Source](https://www.exchange-rates.org/exchange-rate-history/krw-usd-2023))
- 1 **JPY** = 0.007133 **USD** ([Source](https://www.exchange-rates.org/exchange-rate-history/jpy-usd-2023))

Example SQL query to convert the prices to USD:

```sqlite
SELECT CASE
           WHEN total_price_currency = 'KRW' THEN
               total_price * 0.0007663
           WHEN total_price_currency = 'JPY' THEN
               total_price * 0.007133
           ELSE total_price
           END total_price_usd
FROM transactions
LIMIT 10;
```

#### Why these rates?

These rates are the average exchange rates for 2023. The rates are calculated by taking the average of the daily
exchange rates for the year 2023. Since the transactions table only includes the transactions that happened in 2023,
these rates are the most accurate representation of the exchange rates for the transactions in the table. Although using
the daily exchange rates would be more accurate, it would require a more complex calculation, hence I decided to use the
average exchange rates for simplicity.

---

## Question 1

### Answer

hs_code : `215873`

transaction_count : `231`

### SQL Query

```sqlite
WITH num_transactions AS (SELECT hs_code,
                                 COUNT(*) AS transaction_count
                          FROM transactions
                          WHERE date BETWEEN '2023-01-01' AND '2024-01-01'
                          GROUP BY hs_code)
SELECT hs_code,
       transaction_count
FROM num_transactions
WHERE transaction_count = (SELECT MAX(transaction_count)
                           FROM num_transactions);
```

### Explanation(if you need)

#### CTE 1: num_transactions

First, I have created a Common Table Expression (CTE) called `num_transactions` to count the number of transactions for
each `hs_code`. I have used the `COUNT(*)` and `GROUP BY` clauses to count the transactions and group them by the
`hs_code`. In the `WHERE` clause, I have filtered the transactions based on the date range between '2023-01-01' and
'2024-01-01' to only consider the transactions that occurred in 2023.

#### Final SELECT Statement

After creating the `num_transactions` CTE, I have selected the `hs_code` and `transaction_count` from the CTE. To find
the `hs_code` with the maximum transaction count, I have used a subquery in the `WHERE` clause to filter the results
based on the maximum transaction count. This way, I could retrieve the `hs_code` with the highest transaction count.

#### Why MAX() instead of LIMIT 1?

While using `ORDER BY transaction_count DESC LIMIT 1` would return the first row with the maximum transaction count,
using `MAX(transaction_count)` allows us to handle cases where multiple `hs_code` values have the same maximum
transaction count. By using `MAX()`, we ensure that all `hs_code` values with the maximum transaction count are
returned.

---

## Question 2

### Answer

exporter_name : `Company_069`

unique_importer_count : `98`

### SQL Query

```sqlite
WITH unique_exporters AS (SELECT exporter_id,
                                 COUNT(DISTINCT importer_id) AS unique_importer_count
                          FROM transactions
                          WHERE date BETWEEN '2023-01-01' AND '2024-01-01'
                          GROUP BY exporter_id)
SELECT c.name AS exporter_name,
       ue.unique_importer_count
FROM unique_exporters ue
         JOIN companies c ON ue.exporter_id = c.id
WHERE ue.unique_importer_count = (SELECT MAX(unique_importer_count)
                                  FROM unique_exporters);
```

### Explanation(if you need)

#### CTE 1: unique_exporters

First CTE I have created is called `unique_exporters`. In this CTE, I have counted the number of unique importers for
each exporter. I have used the `COUNT(DISTINCT importer_id)` to ensure that each importer is counted only once for each
exporter. I have filtered the transactions based on the date range between '2023-01-01' and '2024-01-01' to only
consider the transactions that occurred in 2023. I have grouped the results by the `exporter_id`.

#### Final SELECT Statement

After creating the `unique_exporters` CTE, I have joined it with the `companies` table to get the exporter's name. This
way, I could reduce the number of rows being `JOIN`ed. Finally, I have selected the exporter's name from the `companies`
table and the unique importer count from the `unique_exporters` CTE. In the `WHERE` clause, I have filtered the results
to only include the exporter with the maximum unique importer count using the `MAX()` function. By using `MAX()`, we
ensure that all exporters with the maximum unique importer count are returned. Although this dataset only contains
one exporter with the maximum unique importer count, using `MAX()` allows us to handle cases where multiple exporters
have the same maximum unique importer count.

---

## Question 3

### Answer

Company Name : `Company_011`

Top 3 Frequently Exported hs_code : `28990`, `148808`, `240838`

### SQL Query

```sqlite
WITH transactions_in_usd AS (SELECT exporter_id,
                                    hs_code,
                                    COUNT(*)                      AS transaction_count,
                                    SUM(CASE
                                            WHEN total_price_currency = 'KRW' THEN
                                                total_price * 0.0007663
                                            WHEN total_price_currency = 'JPY' THEN
                                                total_price * 0.007133
                                            ELSE total_price END) AS total_price_in_usd
                             FROM transactions
                             WHERE date BETWEEN '2023-01-01' AND '2024-01-01'
                             GROUP BY exporter_id, hs_code),
     price_sum_rank AS (SELECT exporter_id,
                               RANK() OVER (ORDER BY SUM(total_price_in_usd) DESC) AS rank
                        FROM transactions_in_usd
                        GROUP BY exporter_id),
     largest_price_sum_exporter AS (SELECT exporter_id
                                    FROM price_sum_rank
                                    WHERE rank = 1),
     hs_code_transaction_count AS (SELECT exporter_id,
                                          hs_code,
                                          transaction_count,
                                          total_price_in_usd,
                                          RANK() OVER (
                                              PARTITION BY exporter_id
                                              ORDER BY
                                                  transaction_count DESC
                                              ) AS transaction_count_rank
                                   FROM transactions_in_usd
                                   WHERE exporter_id IN (SELECT exporter_id
                                                         FROM largest_price_sum_exporter)
                                   GROUP BY exporter_id, hs_code)
SELECT c.name AS exporter_name,
       c.country_code,
       hctc.hs_code,
       hctc.total_price_in_usd,
       hctc.transaction_count
FROM hs_code_transaction_count hctc
         JOIN companies c
              ON hctc.exporter_id = c.id
WHERE transaction_count_rank <= 3;
```

### Explanation(if you need)

#### CTE 1: transactions_in_usd

First, I have created a Common Table Expression (CTE) called `transactions_in_usd` to calculate the total price of each

#### CTE 2: price_sum_rank

#### CTE 3: largest_price_sum_exporter

#### CTE 4: hs_code_transaction_count

#### Final SELECT Statement

---

## Question 4

### Answer

unit price for Orange-related imports in 2023 : `0.1769891386163061`

### SQL Query

```sqlite
WITH orange_transactions_usd AS (SELECT weight,
                                        weight_unit,
                                        CASE
                                            WHEN total_price_currency = 'KRW' THEN
                                                total_price * 0.0007663
                                            WHEN total_price_currency = 'JPY' THEN
                                                total_price * 0.007133
                                            ELSE total_price END AS total_price_usd
                                 FROM transactions
                                 WHERE date BETWEEN '2023-01-01' AND '2024-01-01'
                                   AND LOWER(raw_product_name) LIKE '%orange%'),
     unit_price_usd_row_num AS (SELECT weight_unit,
                                       total_price_usd / weight                 AS unit_price_usd,
                                       ROW_NUMBER() OVER (
                                           PARTITION BY weight_unit
                                           ORDER BY total_price_usd / weight
                                           )                                    AS row_num,
                                       COUNT(*) OVER (PARTITION BY weight_unit) AS total_count
                                FROM orange_transactions_usd),
     quartiles_1 AS (SELECT weight_unit,
                            unit_price_usd AS unit_price_usd
                     FROM unit_price_usd_row_num
                     WHERE row_num = total_count * 0.25),
     quartiles_3 AS (SELECT weight_unit,
                            unit_price_usd AS unit_price_usd
                     FROM unit_price_usd_row_num
                     WHERE row_num = total_count * 0.75),
     iqr_bounds AS (SELECT q1.weight_unit,
                           (q1.unit_price_usd - 1.5 * (q3.unit_price_usd - q1.unit_price_usd)) AS lower_bound,
                           (q3.unit_price_usd + 1.5 * (q3.unit_price_usd - q1.unit_price_usd)) AS upper_bound
                    FROM quartiles_1 q1
                             JOIN
                         quartiles_3 q3 ON q1.weight_unit = q3.weight_unit),
     filtered_unit_prices AS (SELECT up.weight_unit,
                                     up.unit_price_usd
                              FROM unit_price_usd_row_num up
                                       JOIN iqr_bounds ib ON up.weight_unit = ib.weight_unit
                              WHERE up.unit_price_usd BETWEEN ib.lower_bound AND ib.upper_bound)
SELECT weight_unit,
       AVG(unit_price_usd) AS avg_unit_price_usd
FROM filtered_unit_prices
GROUP BY weight_unit;

```

### Explanation(if you need)

---

## Question 5

### Answer

### SQL Query

#### DDL for Dimension Tables

```sqlite
DROP TABLE IF EXISTS dim_company;
CREATE TABLE dim_company
(
    id           INTEGER PRIMARY KEY AUTOINCREMENT,
    company_code TEXT UNIQUE,
    name         TEXT,
    country_code TEXT,
    created_at   TEXT,
    updated_at   TEXT
);

DROP TABLE IF EXISTS dim_product;
CREATE TABLE dim_product
(
    id                      INTEGER PRIMARY KEY AUTOINCREMENT,
    hs_code                 INTEGER,
    raw_product_name        TEXT,
    raw_product_description TEXT,
    created_at              TEXT,
    updated_at              TEXT,
    UNIQUE (hs_code, raw_product_name, raw_product_description)
);

DROP TABLE IF EXISTS dim_date;
CREATE TABLE dim_date
(
    id             INTEGER PRIMARY KEY AUTOINCREMENT,
    date           TEXT,
    year           INTEGER,
    month          INTEGER,
    day_of_month   INTEGER,
    day_of_quarter INTEGER,
    day_of_week    INTEGER,
    created_at     TEXT
);

DROP TABLE IF EXISTS dim_incoterms;
CREATE TABLE dim_incoterms
(
    id         INTEGER PRIMARY KEY AUTOINCREMENT,
    incoterms  TEXT,
    created_at TEXT,
    updated_at TEXT
);

DROP TABLE IF EXISTS dim_transport;
CREATE TABLE dim_transport
(
    id         INTEGER PRIMARY KEY AUTOINCREMENT,
    transport  TEXT,
    created_at TEXT,
    updated_at TEXT
);

DROP TABLE IF EXISTS dim_currency;
CREATE TABLE dim_currency
(
    id            INTEGER PRIMARY KEY AUTOINCREMENT,
    currency_code TEXT,
    created_at    TEXT,
    updated_at    TEXT
);

DROP TABLE IF EXISTS dim_weight_unit;
CREATE TABLE dim_weight_unit
(
    id         INTEGER PRIMARY KEY AUTOINCREMENT,
    unit       TEXT,
    created_at TEXT,
    updated_at TEXT
);

DROP TABLE IF EXISTS dim_time;
CREATE TABLE dim_time
(
    id         INTEGER PRIMARY KEY AUTOINCREMENT,
    hour       INTEGER,
    minute     INTEGER,
    second     INTEGER,
    created_at TEXT
);
```

#### DDL for Fact Table

```sqlite
DROP TABLE IF EXISTS fact_transactions;
CREATE TABLE fact_transactions
(
    transaction_id TEXT PRIMARY KEY,
    date_id        INTEGER NOT NULL,
    time_id        INTEGER NOT NULL,
    exporter_id    INTEGER NOT NULL,
    importer_id    INTEGER NOT NULL,
    product_id     INTEGER NOT NULL,
    incoterms_id   INTEGER NOT NULL,
    transport_id   INTEGER NOT NULL,
    currency_id    INTEGER NOT NULL,
    weight_unit_id INTEGER NOT NULL,
    weight         REAL,
    total_price    REAL,
    created_at     TEXT,
    FOREIGN KEY (date_id) REFERENCES dim_date (id),
    FOREIGN KEY (time_id) REFERENCES dim_time (id),
    FOREIGN KEY (exporter_id) REFERENCES dim_company (id),
    FOREIGN KEY (importer_id) REFERENCES dim_company (id),
    FOREIGN KEY (product_id) REFERENCES dim_product (id),
    FOREIGN KEY (incoterms_id) REFERENCES dim_incoterms (id),
    FOREIGN KEY (transport_id) REFERENCES dim_transport (id),
    FOREIGN KEY (currency_id) REFERENCES dim_currency (id),
    FOREIGN KEY (weight_unit_id) REFERENCES dim_weight_unit (id)
);
```

### Explanation(if you need)

My design for the data mart with large record is based on the star schema. The fact table contains the transaction
statistics, and the dimension tables provide additional information about the companies and products. The schema is
designed to optimize query performance and provide a clear structure for analyzing the data.


---

## Additional Notes

- Refer to the `q_{question_number}.sql` files for the complete SQL queries and DDL statements.

