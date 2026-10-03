
# Anti-Join و بررسی عدم وجود رکورد

برای یافتن رکوردهایی که **رکورد متناظر در جدول دیگر ندارند**، الگوهای زیر رایج هستند:

- `NOT EXISTS`
- `LEFT JOIN ... WHERE ... IS NULL`
- `NOT IN`

## 1. الگوی پیشنهادی: `NOT EXISTS`

در سناریوهای بررسی عدم وجود رکورد، `NOT EXISTS` معمولاً انتخاب مناسبی است.

```sql
SELECT c.Id, c.Name
FROM Customers AS c
WHERE NOT EXISTS
(
    SELECT 1
    FROM Orders AS o
    WHERE o.CustomerId = c.Id
);
```

معنای کوئری:

> مشتریانی را برگردان که هیچ سفارش متناظری ندارند.

### مزایا

- منطق کوئری بسیار واضح است.
- در حضور `NULL` مانند `NOT IN` دچار مشکل منطقی نمی‌شود.
- برای Anti-Join بسیار مناسب است.
- وقتی فقط وجود یا عدم وجود رکورد اهمیت دارد، `SELECT 1` کافی است.
- SQL Server می‌تواند این الگو را به Anti Semi Join در Execution Plan تبدیل کند.

---

## 2. `LEFT JOIN ... IS NULL`

همان منطق را می‌توان با `LEFT JOIN` پیاده‌سازی کرد:

```sql
SELECT c.Id, c.Name
FROM Customers AS c
LEFT JOIN Orders AS o
    ON o.CustomerId = c.Id
WHERE o.Id IS NULL;
```

منطق:

```text
Customers
    |
    | LEFT JOIN
    v
Orders
    |
    +-- Match Found
    |
    +-- No Match
          |
          v
       NULL
          |
          v
      WHERE o.Id IS NULL
```

### نکته مهم

شرط `IS NULL` بهتر است روی ستونی قرار بگیرد که در رکورد واقعی `NULL` نمی‌شود؛ معمولاً Primary Key:

```sql
WHERE o.Id IS NULL
```

نه لزوماً:

```sql
WHERE o.CustomerId IS NULL
```

زیرا `CustomerId` ممکن است در خود جدول `Orders` قابلیت `NULL` داشته باشد.

---

## 3. مخاطرات `NOT IN`

نمونه:

```sql
SELECT c.Id, c.Name
FROM Customers AS c
WHERE c.Id NOT IN
(
    SELECT o.CustomerId
    FROM Orders AS o
);
```

این کوئری در صورتی که Subquery مقدار `NULL` برگرداند، می‌تواند نتیجه غیرمنتظره ایجاد کند.

فرض کنیم نتیجه Subquery:

```text
2
5
NULL
```

باشد.

شرط:

```sql
c.Id NOT IN (2, 5, NULL)
```

در SQL Server تحت منطق سه‌ارزشی SQL قرار می‌گیرد.

برای مثال:

```text
10 <> 2
AND
10 <> 5
AND
10 <> NULL
```

عبارت:

```sql
10 <> NULL
```

برابر `UNKNOWN` است.

بنابراین کل شرط ممکن است `UNKNOWN` شود و رکورد توسط `WHERE` انتخاب نشود.

در نتیجه وجود حتی یک `NULL` در مجموعه Subquery می‌تواند باعث شود نتیجه `NOT IN` برخلاف انتظار باشد.

---

## 4. مقایسه

| الگو | رفتار با NULL | مناسب برای Anti-Join | خوانایی |
|---|---|---|---|
| `NOT EXISTS` | امن از این نظر | بله | بسیار خوب |
| `LEFT JOIN ... IS NULL` | امن از این نظر | بله | خوب |
| `NOT IN` | دارای ریسک `NULL` | فقط با کنترل NULL | ساده ولی حساس |

---

## 5. استاندارد پیشنهادی پروژه

برای بررسی عدم وجود رکورد:

### ترجیح اول

```sql
WHERE NOT EXISTS
(
    SELECT 1
    FROM ChildTable AS c
    WHERE c.ParentId = p.Id
)
```

### گزینه مناسب دیگر

```sql
LEFT JOIN ChildTable AS c
    ON c.ParentId = p.Id
WHERE c.Id IS NULL
```

### استفاده محتاطانه از `NOT IN`

از:

```sql
WHERE Id NOT IN
(
    SELECT ForeignKey
    FROM OtherTable
)
```

فقط زمانی استفاده شود که `NULL` بودن ستون Subquery به‌صورت قطعی کنترل شده باشد.

برای مثال:

```sql
WHERE Id NOT IN
(
    SELECT ForeignKey
    FROM OtherTable
    WHERE ForeignKey IS NOT NULL
)
```

با این حال، اگر هدف صرفاً بررسی عدم وجود رکورد است، `NOT EXISTS` معمولاً انتخاب شفاف‌تری است.

---

## 6. نکته Performance

نباید این قانون را به‌صورت مطلق در نظر گرفت که:

```text
NOT EXISTS همیشه سریع‌تر از LEFT JOIN است.
```

SQL Server Query Optimizer ممکن است هر دو الگو را به Execution Plan مشابه، از جمله Anti Semi Join، تبدیل کند.

بنابراین انتخاب بین این دو باید بر اساس:

1. Correctness
2. رفتار `NULL`
3. خوانایی
4. Execution Plan
5. Indexها
6. حجم داده
7. Cardinality

انجام شود.

در صورت وجود Performance Problem، Execution Plan واقعی بررسی شود و صرفاً بر اساس شکل Syntax درباره Performance تصمیم‌گیری نشود.

---

## 7. Rule

> **برای بررسی «رکورد متناظر وجود ندارد»، به‌صورت پیش‌فرض `NOT EXISTS` را ترجیح بده؛ `LEFT JOIN ... IS NULL` نیز الگوی معتبر است. `NOT IN` فقط زمانی استفاده شود که رفتار `NULL` کاملاً کنترل شده باشد.**

### خلاصه

```text
Need "Does a related row exist?"
            |
            v
       NOT EXISTS
            |
            +---- Preferred
            |
            +---- NULL-safe

LEFT JOIN + IS NULL
            |
            +---- Valid Anti-Join
            |
            +---- Check nullable columns carefully

NOT IN
            |
            +---- Beware NULL
            |
            +---- Use only with controlled NULL semantics
```






