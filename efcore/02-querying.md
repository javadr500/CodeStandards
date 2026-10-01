# EF Core Querying Standards

استانداردهای Query نویسی در Entity Framework Core با تمرکز بر خوانایی، Performance و جلوگیری از اجرای Queryهای غیرضروری.

---

# 1. Keep Filtering In Database

تا حد امکان Filtering را در Database انجام دهید، نه بعد از دریافت اطلاعات.

### ❌ Bad

```csharp
var books = await context.Books.ToListAsync();

var result = books
    .Where(x => x.IsPublished)
    .ToList();
```

ابتدا کل اطلاعات دریافت شده و سپس Filter شده است.

### ✅ Good

```csharp
var result = await context.Books
    .Where(x => x.IsPublished)
    .ToListAsync();
```

### Rule

```text
Database Filtering
        ↓
Transfer Minimum Data
        ↓
Application Processing
```

---

# 2. Avoid Premature ToList

`ToList()` و `ToListAsync()` باعث اجرای Query می‌شوند.

### ❌ Bad

```csharp
var books = await context.Books
    .ToListAsync();

var result = books
    .Where(x => x.IsPublished)
    .ToList();
```

### ✅ Good

```csharp
var result = await context.Books
    .Where(x => x.IsPublished)
    .ToListAsync();
```

### Rule

تا زمانی که نیاز واقعی به Materialization ندارید، Query را اجرا نکنید.

---

# 3. Avoid Multiple Enumeration

یک Query را چند بار اجرا نکنید.

### ❌ Bad

```csharp
var query = context.Books
    .Where(x => x.IsPublished);

var count = await query.CountAsync();

var books = await query.ToListAsync();
```

دو Query به Database ارسال می‌شود.

### اگر هر دو واقعاً لازم باشند

این رفتار الزاماً اشتباه نیست؛ مهم است بدانیم که دو Round Trip داریم.

برای مثال در Pagination معمولاً Count و دریافت Page دو Query جدا هستند.

---

# 4. Use Any For Existence Checks

اگر فقط می‌خواهید بدانید رکوردی وجود دارد یا خیر، از `Any` استفاده کنید.

### ❌ Bad

```csharp
var exists = await context.Users
    .CountAsync(x => x.Email == email) > 0;
```

### ✅ Good

```csharp
var exists = await context.Users
    .AnyAsync(x => x.Email == email);
```

### Rule

```text
Existence Check → Any()
```

---

# 5. Select Only Required Columns

اگر فقط چند Property لازم است، Entity کامل را دریافت نکنید.

### ❌ Bad

```csharp
var users = await context.Users
    .ToListAsync();
```

### ✅ Good

```csharp
var users = await context.Users
    .Select(x => new
    {
        x.Id,
        x.Name
    })
    .ToListAsync();
```

### مزایا

- کاهش Data Transfer
- کاهش Memory
- کاهش Materialization
- Query سبک‌تر

---

# 6. Use DTO Projection

برای APIها معمولاً Projection مستقیم به DTO مناسب‌تر از دریافت Entity کامل است.

### Example

```csharp
var result = await context.Books
    .AsNoTracking()
    .Select(x => new BookDto
    {
        Id = x.Id,
        Title = x.Title,
        AuthorName = x.Author.Name
    })
    .ToListAsync();
```

در بسیاری از موارد نیازی به:

```csharp
.Include(x => x.Author)
```

نیست، چون EF Core می‌تواند Navigation مورد نیاز Projection را در SQL لحاظ کند.

---

# 7. Avoid Unnecessary Include

`Include` برای دریافت Related Entityهاست، نه اینکه به‌صورت پیش‌فرض روی Queryها قرار گیرد.

### ❌ Potentially unnecessary

```csharp
var books = await context.Books
    .Include(x => x.Author)
    .ToListAsync();
```

اگر فقط `Author.Name` لازم است.

### ✅ Better

```csharp
var books = await context.Books
    .Select(x => new BookDto
    {
        Id = x.Id,
        Title = x.Title,
        AuthorName = x.Author.Name
    })
    .ToListAsync();
```

---

# 8. Avoid N+1 Queries

دسترسی به Database داخل Loop می‌تواند باعث N+1 شود.

### ❌ Bad

```csharp
var authors = await context.Authors
    .ToListAsync();

foreach (var author in authors)
{
    var books = await context.Books
        .Where(x => x.AuthorId == author.Id)
        .ToListAsync();
}
```

### ✅ Better

```csharp
var authors = await context.Authors
    .Include(x => x.Books)
    .ToListAsync();
```

یا در صورت نیاز فقط به اطلاعات مشخص:

```csharp
var result = await context.Authors
    .Select(x => new AuthorDto
    {
        Id = x.Id,
        Name = x.Name,
        BookCount = x.Books.Count()
    })
    .ToListAsync();
```

---

# 9. IQueryable vs IEnumerable

این تفاوت بسیار مهم است.

### IQueryable

Query معمولاً در سمت Database اجرا می‌شود:

```csharp
IQueryable<Book> query = context.Books;

query = query.Where(x => x.IsPublished);

var result = await query.ToListAsync();
```

### IEnumerable

پردازش روی داده‌های موجود در Memory انجام می‌شود.

```csharp
var books = await context.Books
    .ToListAsync();

IEnumerable<Book> result = books
    .Where(x => x.IsPublished);
```

### Rule

قبل از Materialization، تا حد امکان Query را در قالب `IQueryable` نگه دارید.

---

# 10. Be Careful With AsEnumerable

`AsEnumerable()` می‌تواند ادامه پردازش را از Database به Memory منتقل کند.

### Example

```csharp
var result = context.Books
    .Where(x => x.IsPublished)
    .AsEnumerable()
    .Where(x => SomeCustomMethod(x.Title))
    .ToList();
```

قسمت بعد از `AsEnumerable()` روی Memory اجرا می‌شود.

### Rule

`AsEnumerable()` را آگاهانه استفاده کنید.

---

# 11. Avoid Client-Side Processing

تا جایی که امکان دارد عملیات قابل ترجمه به SQL را در Database انجام دهید.

### ❌ Bad

```csharp
var books = await context.Books
    .ToListAsync();

var result = books
    .Where(x => x.Title.StartsWith("C"))
    .ToList();
```

### ✅ Good

```csharp
var result = await context.Books
    .Where(x => x.Title.StartsWith("C"))
    .ToListAsync();
```

---

# 12. Use FirstOrDefault Correctly

اگر فقط اولین رکورد مورد نیاز است:

```csharp
var book = await context.Books
    .FirstOrDefaultAsync(x => x.Id == id);
```

به جای دریافت همه رکوردها:

```csharp
var books = await context.Books
    .Where(x => x.Id == id)
    .ToListAsync();

var book = books.FirstOrDefault();
```

---

# 13. Single vs First

این دو مفهوم متفاوتی دارند.

### First

می‌گوید:

> یک رکورد پیدا کن؛ اگر چند رکورد بود، اولین مورد را بده.

```csharp
var user = await context.Users
    .FirstOrDefaultAsync(x => x.Email == email);
```

### Single

می‌گوید:

> دقیقاً حداکثر یک رکورد باید وجود داشته باشد.

```csharp
var user = await context.Users
    .SingleOrDefaultAsync(x => x.Email == email);
```

اگر بیش از یک رکورد وجود داشته باشد، `SingleOrDefault` Exception ایجاد می‌کند.

### Rule

اگر Business Rule می‌گوید مقدار Unique است، `Single` می‌تواند Intent کد را بهتر نشان دهد.

---

# 14. Use Find For Primary Key Lookup

برای جستجوی مستقیم بر اساس Primary Key:

```csharp
var book = await context.Books
    .FindAsync(id);
```

`Find` می‌تواند ابتدا Change Tracker را بررسی کند.

### Rule

برای Primary Key lookup، `FindAsync` را در نظر بگیرید.

---

# 15. Pagination

برای داده‌های زیاد، کل Dataset را دریافت نکنید.

### Offset Pagination

```csharp
var books = await context.Books
    .OrderBy(x => x.Id)
    .Skip((page - 1) * pageSize)
    .Take(pageSize)
    .ToListAsync();
```

مثلاً:

```text
page = 3
pageSize = 20

Skip(40)
Take(20)
```

---

# 16. Always Order Before Skip/Take

Pagination بدون Order مشخص می‌تواند نتیجه قابل اتکایی نداشته باشد.

### ❌ Bad

```csharp
var books = await context.Books
    .Skip(20)
    .Take(20)
    .ToListAsync();
```

### ✅ Good

```csharp
var books = await context.Books
    .OrderBy(x => x.Id)
    .Skip(20)
    .Take(20)
    .ToListAsync();
```

---

# 17. Keyset Pagination

برای Datasetهای بسیار بزرگ، Keyset Pagination می‌تواند جایگزین مناسبی برای Offset Pagination باشد.

### Example

```csharp
var books = await context.Books
    .Where(x => x.Id > lastId)
    .OrderBy(x => x.Id)
    .Take(20)
    .ToListAsync();
```

به جای:

```csharp
.Skip(100000)
```

از آخرین شناسه دریافت‌شده استفاده می‌شود.

---

# 18. Use Split Queries Carefully

وقتی چند Collection با `Include` دریافت می‌کنید، ممکن است Joinهای متعدد باعث افزایش شدید تعداد Rowهای نتیجه شوند.

### Example

```csharp
var authors = await context.Authors
    .Include(x => x.Books)
    .Include(x => x.Awards)
    .AsSplitQuery()
    .ToListAsync();
```

`AsSplitQuery()` می‌تواند Query را به چند SQL Query تقسیم کند.

### Rule

Split Query یک Optimization عمومی نیست؛ Execution Plan و حجم داده را بررسی کنید.

---

# 19. Filtered Include

در صورت نیاز می‌توان Related Data را فیلتر کرد.

### Example

```csharp
var authors = await context.Authors
    .Include(x => x.Books
        .Where(b => b.IsPublished))
    .ToListAsync();
```

به جای دریافت تمام کتاب‌ها، فقط کتاب‌های Published دریافت می‌شوند.

---

# 20. ExecuteUpdate

اگر قصد Update تعداد زیادی رکورد را دارید، لازم نیست همه Entityها را Load کنید.

### ❌ معمولاً نامناسب برای Bulk Update

```csharp
var books = await context.Books
    .Where(x => !x.IsPublished)
    .ToListAsync();

foreach (var book in books)
{
    book.IsPublished = true;
}

await context.SaveChangesAsync();
```

### ✅

```csharp
await context.Books
    .Where(x => !x.IsPublished)
    .ExecuteUpdateAsync(setters =>
        setters.SetProperty(x => x.IsPublished, true));
```

این روش می‌تواند مستقیماً عملیات Update را در Database انجام دهد.

### Important

`ExecuteUpdate` از Change Tracker عبور می‌کند.

بنابراین Entityهایی که قبلاً در همان `DbContext` Track شده‌اند ممکن است State قدیمی داشته باشند.

---

# 21. ExecuteDelete

برای حذف تعداد زیادی رکورد بدون Load کردن Entityها:

```csharp
await context.Books
    .Where(x => x.IsDeleted)
    .ExecuteDeleteAsync();
```

به جای:

```csharp
var books = await context.Books
    .Where(x => x.IsDeleted)
    .ToListAsync();

context.Books.RemoveRange(books);

await context.SaveChangesAsync();
```

---

# 22. Query Tags

برای پیدا کردن Queryهای خاص در Logging و Performance Analysis می‌توان از `TagWith` استفاده کرد.

### Example

```csharp
var books = await context.Books
    .TagWith("Get Published Books")
    .Where(x => x.IsPublished)
    .ToListAsync();
```

این Tag در SQL تولیدشده قابل مشاهده است.

---

# 23. Inspect Generated SQL

برای بررسی Query تولیدشده:

```csharp
var query = context.Books
    .Where(x => x.IsPublished);

var sql = query.ToQueryString();
```

### Rule

در Performance Troubleshooting فقط به LINQ نگاه نکنید.

```text
LINQ
 ↓
Generated SQL
 ↓
Execution Plan
 ↓
Indexes
 ↓
Performance
```

---

# 24. Avoid Raw SQL Unless Justified

اولویت معمول:

```text
LINQ
 ↓
EF Core
```

در صورت نیاز:

```text
Raw SQL
```

مثلاً:

```csharp
var books = await context.Books
    .FromSql($"SELECT * FROM Books WHERE IsPublished = 1")
    .ToListAsync();
```

پارامترها باید به شکل امن و Parameterized استفاده شوند.

### ❌ خطرناک

```csharp
var sql = $"SELECT * FROM Books WHERE Title = '{title}'";
```

---

# 25. CancellationToken

در APIها و عملیات طولانی، CancellationToken را به Query منتقل کنید.

```csharp
var books = await context.Books
    .AsNoTracking()
    .ToListAsync(cancellationToken);
```

این امکان را می‌دهد عملیات Database در صورت Cancellation درخواست، قابل لغو باشد.

---

# 26. Keep Query Composition Readable

Query پیچیده را در یک خط ننویسید.

### ❌ Bad

```csharp
var result = await context.Books.Where(x => x.IsPublished && x.Author.IsActive).OrderByDescending(x => x.PublishedDate).Select(x => new BookDto { Id = x.Id, Title = x.Title }).ToListAsync();
```

### ✅ Good

```csharp
var result = await context.Books
    .AsNoTracking()
    .Where(x => x.IsPublished)
    .Where(x => x.Author.IsActive)
    .OrderByDescending(x => x.PublishedDate)
    .Select(x => new BookDto
    {
        Id = x.Id,
        Title = x.Title
    })
    .ToListAsync();
```

خوانایی Query خودش بخشی از Maintainability است.

---

# 27. Query Checklist

قبل از Commit کردن یک Query جدید:

- [ ] آیا Filter در Database انجام می‌شود؟
- [ ] آیا `ToList()` زودتر از زمان لازم اجرا نشده؟
- [ ] آیا N+1 وجود ندارد؟
- [ ] آیا `Include` واقعاً لازم است؟
- [ ] آیا Projection امکان‌پذیر است؟
- [ ] آیا فقط Columns مورد نیاز دریافت می‌شوند؟
- [ ] آیا `AsNoTracking` مناسب است؟
- [ ] آیا Pagination لازم است؟
- [ ] آیا `OrderBy` قبل از `Skip/Take` وجود دارد؟
- [ ] آیا Query چند بار اجرا نمی‌شود؟
- [ ] آیا `Any` به جای `Count > 0` استفاده شده؟
- [ ] آیا SQL تولیدشده بررسی شده؟
- [ ] آیا Index مناسب وجود دارد؟
- [ ] آیا CancellationToken منتقل شده؟
- [ ] آیا Raw SQL واقعاً ضروری است؟

---

# Golden Rules

```text
1. Filter in Database
2. Select only what you need
3. Avoid unnecessary Include
4. Avoid N+1
5. Avoid premature ToList
6. Use AsNoTracking for Read-Only queries
7. Use Any for existence checks
8. Use Find for Primary Key lookup
9. Always order paginated queries
10. Consider Keyset Pagination for large datasets
11. Use ExecuteUpdate/Delete for suitable bulk operations
12. Inspect generated SQL when performance matters
13. Use Raw SQL only when justified
14. Pass CancellationToken to async queries
15. Measure before optimizing
```