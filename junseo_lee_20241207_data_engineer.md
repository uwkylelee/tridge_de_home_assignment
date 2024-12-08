# Data Engineer Assignment

## ToC

<details>
    <summary>Table of Contents</summary>

- [Common Assumptions](#common-assumptions)
    - [Date Column](#date-column)
    - [Timezone](#timezone)
    - [Currency Exchange Rate](#currency-exchange-rate)
        - [Why these rates?](#why-these-rates)
- [Question 1](#question-1)
    - [Answer](#answer)
    - [SQL Query](#sql-query)
    - [Explanation(if you need)](#explanationif-you-need)
        - [CTE 1: num_transactions](#cte-1-num_transactions)
        - [Final SELECT Statement](#final-select-statement)
        - [Why MAX() instead of LIMIT 1?](#why-max-instead-of-limit-1)
- [Question 2](#question-2)
    - [Answer](#answer-1)
    - [SQL Query](#sql-query-1)
    - [Explanation(if you need)](#explanationif-you-need-1)
        - [CTE 1: unique_exporters](#cte-1-unique_exporters)
        - [Final SELECT Statement](#final-select-statement-1)
- [Question 3](#question-3)
    - [Answer](#answer-2)
    - [SQL Query](#sql-query-2)
    - [Explanation(if you need)](#explanationif-you-need-2)
        - [What this reveals about market concentration](#what-this-reveals-about-market-concentration)
        - [CTE 1: transactions_in_usd](#cte-1-transactions_in_usd)
        - [CTE 2: price_sum_rank](#cte-2-price_sum_rank)
        - [CTE 3: largest_price_sum_exporter](#cte-3-largest_price_sum_exporter)
        - [CTE 4: hs_code_transaction_count](#cte-4-hs_code_transaction_count)
        - [Final SELECT Statement](#final-select-statement-2)
- [Question 4](#question-4)
    - [Answer](#answer-3)
    - [SQL Query](#sql-query-3)
    - [Explanation(if you need)](#explanationif-you-need-3)
        - [Methodology for Determining "Reasonable" Unit Price](#methodology-for-determining-reasonable-unit-price)
        - [CTE 1: orange_transactions_usd](#cte-1-orange_transactions_usd)
        - [CTE 2: unit_price_usd_row_num](#cte-2-unit_price_usd_row_num)
        - [CTE 3: quartiles_1](#cte-3-quartiles_1)
        - [CTE 4: quartiles_3](#cte-4-quartiles_3)
        - [CTE 5: iqr_bounds](#cte-5-iqr_bounds)
        - [Final SELECT Statement](#final-select-statement-3)
- [Question 5](#question-5)
    - [Answer](#answer-4)
    - [SQL Query](#sql-query-4)
    - [Explanation(if you need)](#explanationif-you-need-4)
- [Additional Notes](#additional-notes)

</details> 

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

### Explanation

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

### Explanation

#### What this reveals about market concentration

By calculating the number of unique importers for each exporter, I could gain insights into market concentration. A
higher unique importer count indicates that the exporter has relationships with a more diverse set of importing
companies. This diversity can reflect the exporter's market reach, competitiveness, and ability to engage with a broad
range of importers. There are several implications of having relationships with a higher number of importers:

**From an Exporter Perspective:**

1. Exporters with a large number of unique importers, like Company_069, can leverage their dominant position to
   negotiate better terms and maintain a competitive edge in the market.

2. Being able to attract a wide range of importers indicates a strong reputation.

3. A higher count of unique importers reduces dependency on a few clients, which mitigates the risk of revenue loss if
   one or more importers cease trading.

**From a Market Perspective:**

1. A diverse set of importers suggests a competitive market environment where exporters strive to attract and retain
   clients through various strategies, such as pricing, quality, and service offerings.

2. Importers dealing with dominant exporters may face challenges in negotiating favorable terms due to the exporter’s
   strong market position.

3. A high concentration of trade with a single exporter may lead to reliance on that supplier, posing risks if supply
   disruptions occur.

#### CTE 1: unique_exporters

The first CTE, named `unique_exporters`, calculates the number of distinct importers for each exporter. This is achieved
by using `COUNT(DISTINCT importer_id)` to ensure each importer is counted only once per exporter. The transactions are
filtered to include only those occurring in 2023 using the `WHERE` clause. The `GROUP BY exporter_id` clause groups the
results by the exporter's ID.

#### Final SELECT Statement

Once the `unique_exporters` CTE is created, it is joined with the `companies` table to retrieve the exporter's name.
This approach reduces the number of rows involved in the `JOIN` operation. The final selection includes the exporter's
name from the `companies` table and the count of unique importers from the `unique_exporters` CTE.

In the `WHERE` clause, the results are filtered to include only the exporters with the highest unique importer count
by applying the `MAX()` function. This ensures that all exporters sharing the maximum unique importer count are
returned. Although the dataset given contains only one exporter with the maximum count, using `MAX()` ensures
allows us to handle cases where multiple exporters have the same maximum unique importer count.

---

## Question 3

### Answer

Company Name : `Company_011`

Country Code : BR

Top 3 Frequently Exported hs_code : `28990`, `148808`, `240838`

Total Price in USD rounded to 2 decimal places : `149720.44`, `53450.84`, `42815.77`

Transaction Count : `13`, `13`, `13`

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
WHERE transaction_count_rank <= 3
ORDER BY hctc.transaction_count DESC
LIMIT 3;
```

### Explanation

#### CTE 1: transactions_in_usd

This CTE converts transaction amounts into USD and aggregates the data by exporter_id and hs_code. The conversion is
done based on the currency of the transaction using the provided exchange rates in the assumptions section.
Additionally, I aggregated the data by exporter_id and hs_code to calculate the total transaction count and the sum of
transaction prices in USD for each combination of exporter and hs_code. By grouping the data in this way, we can
reduce the number of rows involved in the subsequent analysis.

#### CTE 2: price_sum_rank

This CTE ranks exporters based on their total transaction value in USD. After grouping the data
by `exporter_id`, I used the `RANK()` function to assign ranks in descending order of the sum of `total_price_in_usd`.
Exporters with the highest total transaction value receive a rank of 1. In this way, we can ensure that if multiple
exporters have the same total transaction value, they will share the same rank. Using `LIMIT 1` would return only one
exporter, but using `RANK()` allows us to handle cases where multiple exporters have the same total transaction value.

#### CTE 3: largest_price_sum_exporter

In this CTE, I filtered the results from the `price_sum_rank` CTE to select only the exporters with a rank of 1.

#### CTE 4: hs_code_transaction_count

This CTE ranks `hs_code` values for the top-performing exporters (notably, only one exporter in the given dataset)
identified in the previous step. The data is partitioned by `exporter_id` and sorted by transaction count in descending
order, with ranks assigned using the `RANK()` function. This approach helps highlight the product categories with the
most transactions for the leading exporter. Again, the `RANK()` function is used to handle cases where
multiple `hs_code` values have the same transaction count. The question asks for the top 3 frequently exported `hs_code`
values, so we limit the results to the top 3 ranks in the final `SELECT` statement.

#### Final SELECT Statement

In the final `SELECT` statement, I joined the `hs_code_transaction_count` CTE with the `companies` table to retrieve the
exporter's name and country code. Performing the `JOIN` operation at this stage ensures that we only join the rows that
meet the specified criteria, reducing the number of rows involved in the operation. I filtered the results to show only
the top 3 frequently exported `hs_code` using the `transaction_count_rank <= 3` condition. This condition might return
more than 3 rows if multiple `hs_code` values share the same transaction count rank (e.g., 4 `hs_code` values with the
same transaction count rank of 1). Hence, I finalized the query by limiting the results to the top 3 ranks
using `ORDER BY hctc.transaction_count DESC LIMIT 3`.

---

## Question 4

### Answer

unit price for Orange-related imports in 2023 rounded to 5 decimal places : `0.17699` **USD**

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
                             JOIN quartiles_3 q3
                                  ON q1.weight_unit = q3.weight_unit)
SELECT up.weight_unit,
       AVG(up.unit_price_usd) AS avg_unit_price_usd
FROM unit_price_usd_row_num up
         JOIN iqr_bounds ib ON up.weight_unit = ib.weight_unit
WHERE up.unit_price_usd BETWEEN ib.lower_bound AND ib.upper_bound
GROUP BY up.weight_unit;
```

### Explanation

The SQL query is written to calculate a reasonable unit price for Orange-related imports in 2023. Although the dataset
only contains one weight unit (KG), the query is designed to be scalable for multiple weight units.

#### Methodology for Determining "Reasonable" Unit Price

To establish a reasonable unit price for Orange-related imports in 2023, I applied the Interquartile Range (IQR) method
to eliminate outliers before averaging unit prices. The IQR method is a reliable statistical approach for detecting and
filtering extreme values. By removing unit prices outside the range of Q1 - 1.5 × IQR to Q3 + 1.5 × IQR, only reasonable
data points are considered.

#### CTE 1: orange_transactions_usd

This CTE filters the transactions to include only those related to oranges by checking if the `raw_product_name` column
contains the word 'orange'. It also converts the total price to USD based on the currency of the transaction using the
provided exchange rates.

#### CTE 2: unit_price_usd_row_num

This CTE calculates the unit price in USD for each transaction by dividing the total price in USD by the weight. It also
assigns a row number to each row within the partition of `weight_unit` based on the unit price in USD.
The `ROW_NUMBER()` function is used to assign a unique row number to each row within the partition, and
the `COUNT(*) OVER (PARTITION BY weight_unit)` function calculates the total count of rows within the partition.

#### CTE 3: quartiles_1

This CTE calculates the first quartile (Q1) of the unit prices in USD for each `weight_unit`. The first quartile
represents the 25th percentile of the unit prices. By filtering the results based on the row number equal to
`total_count * 0.25`, we can identify the unit price at the first quartile. In this case, since the total count is an
even number, the first quartile can be simply calculated as `total_count * 0.25`.

#### CTE 4: quartiles_3

This CTE calculates the third quartile (Q3) of the unit prices in USD for each `weight_unit`. The third quartile
represents the 75th percentile of the unit prices. By filtering the results based on the row number equal to
`total_count * 0.75`, we can identify the unit price at the third quartile. Similar to the first quartile calculation,
the third quartile can be calculated as `total_count * 0.75`.

#### CTE 5: iqr_bounds

This CTE calculates the lower and upper bounds for the interquartile range (IQR) for each `weight_unit`. The IQR is
calculated as the difference between the Q3 and the Q1. The lower bound is calculated as `Q1 - 1.5 * IQR`, and the upper
bound is calculated as `Q3 + 1.5 * IQR`. These bounds are used to identify the outliers in the data.

#### Final SELECT Statement

In this CTE, the unit prices in USD are filtered based on the IQR bounds calculated in the previous step. By joining the
`unit_price_usd_row_num` CTE with the `iqr_bounds` CTE, we can filter out the outliers and focus on the unit prices
within the IQR. Then, for each `weight_unit`, the average unit price in USD is calculated using the `AVG()` function.

---

## Question 5

### Answer

To design a data mart for company transaction statistics with approximately 5 billion records, I would propose the
following schema based on the star schema model.

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

#### Indexes

```sqlite
-- Indexes for Fact Table
CREATE UNIQUE INDEX idx_fact_transactions_pk ON fact_transactions (transaction_id);
CREATE INDEX idx_fact_date_id ON fact_transactions (date_id);
CREATE INDEX idx_fact_time_id ON fact_transactions (time_id);
CREATE INDEX idx_fact_exporter_id ON fact_transactions (exporter_id);
CREATE INDEX idx_fact_importer_id ON fact_transactions (importer_id);
CREATE INDEX idx_fact_product_id ON fact_transactions (product_id);
CREATE INDEX idx_fact_incoterms_id ON fact_transactions (incoterms_id);
CREATE INDEX idx_fact_transport_id ON fact_transactions (transport_id);
CREATE INDEX idx_fact_currency_id ON fact_transactions (currency_id);
CREATE INDEX idx_fact_weight_unit_id ON fact_transactions (weight_unit_id);

-- Indexes for Dimension Tables
-- Company
CREATE UNIQUE INDEX idx_dim_company_pk ON dim_company (id);
CREATE INDEX idx_dim_company_code ON dim_company (company_code);
CREATE INDEX idx_dim_company_country_code ON dim_company (country_code);

-- Product
CREATE UNIQUE INDEX idx_dim_product_pk ON dim_product (id);
CREATE INDEX idx_dim_product_unique ON dim_product (hs_code, raw_product_name, raw_product_description);

-- Incoterms
CREATE UNIQUE INDEX idx_dim_incoterms_pk ON dim_incoterms (id);
CREATE INDEX idx_dim_incoterms_incoterms ON dim_incoterms (incoterms);

-- Transport
CREATE UNIQUE INDEX idx_dim_transport_pk ON dim_transport (id);
CREATE INDEX idx_dim_transport_transport ON dim_transport (transport);

-- Currency
CREATE UNIQUE INDEX idx_dim_currency_pk ON dim_currency (id);
CREATE INDEX idx_dim_currency_code ON dim_currency (currency_code);

-- Weight Unit
CREATE UNIQUE INDEX idx_dim_weight_unit_pk ON dim_weight_unit (id);
CREATE INDEX idx_dim_weight_unit_unit ON dim_weight_unit (unit);

-- Date
CREATE UNIQUE INDEX idx_dim_date_pk ON dim_date (id);
CREATE INDEX idx_dim_date_date ON dim_date (date);

-- Time
CREATE UNIQUE INDEX idx_dim_time_pk ON dim_time (id);


```

### Explanation

#### Star Schema Design

The star schema is composed of:

1. **Fact Table**:
    - `fact_transactions`: This table is exactly the same as the `transactions` table in the original schema but with
      foreign keys to dimension tables.

2. **Dimension Tables**:
    - `dim_company`: Stores exporter and importer details, such as company name and country code.
    - `dim_product`: Contains product information, including HS codes and descriptions.
    - `dim_date`: Provides date-related attributes (year, month, day) for time-based analysis.
    - `dim_time`: Captures time-specific details (hour, minute, second) to support granular analysis.
    - `dim_incoterms`: Holds standardized trade terms (e.g., CIF, FOB) to analyze trade agreements.
    - `dim_transport`: Details transport modes (e.g., sea, air) for logistics analysis.
    - `dim_currency`: Contains currency codes to support multi-currency transactions.
    - `dim_weight_unit`: Standardizes weight units for consistency in analysis.

#### Design Considerations

1. **Scalability**:
    - The use of surrogate keys in the fact table ensures efficient indexing and lookup.
    - Dimension tables are normalized to avoid redundancy and maintain consistency.

2. **Performance Optimization**:
    - The fact table includes only numerical values and foreign keys to minimize data size and optimize storage.
    - Indexing on foreign keys and commonly queried columns will speed up join and filter operations.

3. **Flexibility**:
    - The star schema allows for easy expansion by adding new dimensions or facts without affecting existing data.

---

## References

1. https://www.khanacademy.org/math/cc-sixth-grade-math/cc-6th-data-statistics/cc-6th/a/interquartile-range-review
2. https://www.databricks.com/glossary/star-schema
3. https://www.sqlite.org/
