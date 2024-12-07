# Data Engineer Assignment

## ToC

- [Initial Analysis & Common Assumptions](#initial-analysis--common-assumptions)
    - [Date Range](#date-range)
        - [SQL Queries](#sql-queries)
    - [Currency Exchange Rate](#currency-exchange-rate)
        - [Why these rates?](#why-these-rates)
- [Question 1](#question-1)
- [Question 2](#question-2)
- [Question 3](#question-3)
- [Question 4](#question-4)
- [Question 5](#question-5)

---

## Initial Analysis & Common Assumptions

Assumptions made in this section applies to all questions. Each question may have additional assumptions.

### Date Range

After the initial analysis of the transactions table, I have noticed that all transactions happened in 2023. Therefore,
I have not applied any date filtering in the queries. If the transactions table contained transactions from multiple
years, I would have applied a date filter to only include the transactions that happened in 2023.

#### SQL Queries

Query to check the date range of the transactions:

```sql
SELECT MIN(date) AS min_date,
       MAX(date) AS max_date
FROM transactions;
```

`WHERE` clause that would have been added to the queries if the transactions table contained transactions from multiple
years:

```sql
WHERE date BETWEEN '2023-01-01' AND '2024-01-01'
```

Above `WHERE` clause uses the date index effectively and improves the query performance.

### Currency Exchange Rate

After analyzing the transactions table using the below query, I have noticed that the currencies used in the
transactions are KRW, JPY, and USD.

```sql
SELECT DISTINCT total_price_currency
FROM transactions;
```

To answer the questions that require price calculation, I have converted the prices to USD using the exchange rates
provided below.

- 1 KRW = 0.0007663 USD ([Source](https://www.exchange-rates.org/exchange-rate-history/krw-usd-2023))
- 1 JPY = 0.007133 USD ([Source](https://www.exchange-rates.org/exchange-rate-history/jpy-usd-2023))

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

```sql
WITH NUM_TRANSACTION_CTE AS (SELECT hs_code,
                                    COUNT(*) AS  transaction_count,
                                    ROW_NUMBER() OVER (ORDER BY COUNT(*) DESC) AS rank
                             FROM transactions
                             GROUP BY hs_code)

SELECT hs_code,
       transaction_count
FROM NUM_TRANSACTION_CTE
WHERE rank = 1;
```

### Explanation(if you need)

#### ROW_NUMBER() Function Instead of LIMIT

---

## Question 2

### Answer

exporter_name : `Company_069`

unique_importer_count : `98`

### SQL Query

```sql
WITH unique_importers AS (SELECT exporter_id,
                                 COUNT(DISTINCT importer_id)                                   AS unique_importer_count,
                                 ROW_NUMBER() OVER (ORDER BY COUNT(DISTINCT importer_id) DESC) AS rank
                          FROM transactions
                          WHERE date BETWEEN '2023-01-01' AND '2024-01-01'
                          GROUP BY exporter_id)
SELECT c.name AS exporter_name,
       unique_importer_count
FROM unique_importers AS ui
         JOIN companies AS c
              ON ui.exporter_id = c.id
WHERE rank = 1
ORDER BY unique_importer_count DESC;
```

### Explanation(if you need)

---

## Question 3

### Answer

Company Name : `Company_011`

Top 3 Frequently Exported hs_code : `28990`, `148808`, `240838`


### SQL Query

```sql
WITH NUM_TRANSACTION_CTE AS (SELECT hs_code,
                                    COUNT(*) AS  transaction_count,
                                    ROW_NUMBER() OVER (ORDER BY COUNT(*) DESC) AS rank
                             FROM transactions
                             GROUP BY hs_code)

SELECT hs_code,
       transaction_count
FROM NUM_TRANSACTION_CTE
WHERE rank = 1;
```

### Explanation(if you need)

---

## Question 4

### Answer

hs_code : 

### SQL Query

```sql

```

### Explanation(if you need)

---

## Question 5

### Answer

hs_code : 

### SQL Query

```sql

```

### Explanation(if you need)


---

## Additional Notes


