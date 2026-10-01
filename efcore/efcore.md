# Entity Framework Core Standards

استانداردها و Best Practiceهای استفاده از **Entity Framework Core** با تمرکز بر خوانایی، کارایی، مقیاس‌پذیری و جلوگیری از مشکلات رایج در دسترسی به داده.

---

# 1. Query Performance

## 1.1 Avoid N+1 Queries

### Rule

از الگوی دسترسی به Navigation Property داخل حلقه که باعث اجرای Query جداگانه برای هر رکورد می‌شود، اجتناب کنید.

### Bad

```csharp
var authors = await context.Authors.ToListAsync();

foreach (var author in authors)
{
    var books = await context.Books
        .Where(b => b.AuthorId == author.Id)
        .ToListAsync();
}
```

اگر 100 نویسنده داشته باشیم، ممکن است:

```text
1 Query → دریافت Authors
100 Queries → دریافت Books
-------------------------
101 Queries
```

این همان مشکل معروف **N+1 Query Problem** است.

---

## 1.2 Prefer Eager Loading When Related Data Is Required

وقتی واقعاً به داده مرتبط نیاز داریم، می‌توان از `Include` و `ThenInclude` استفاده کرد تا EF Core داده‌های مرتبط را در قالب Query مناسب دریافت کند.

### Good

```csharp
var authors = await context.Authors
    .Include(a => a.Books)
    .ToListAsync();
```

برای روابط چندمرحله‌ای:

```csharp
var authors = await context.Authors
    .Include(a => a.Books)
        .ThenInclude(b => b.Category)
    .ToListAsync();
```

### نکته

`Include` ابزار جلوگیری از N+1 است، اما نباید به‌صورت کورکورانه برای همه Navigationها استفاده شود.

برای Queryهایی که فقط چند فیلد مورد نیاز است، معمولاً **Projection** انتخاب مناسب‌تری است.

---

# 2. Prefer Projection Over Loading Entire Entities

اگر فقط بخشی از اطلاعات مورد نیاز است، به جای دریافت Entity کامل، مستقیماً نتیجه مورد نیاز را با `Select` دریافت کنید.

### Bad

```csharp
var authors = await context.Authors
    .Include(a => a.Books)
    .ToListAsync();
```

در حالی که فقط نام نویسنده و تعداد کتاب‌ها لازم است.

### Good

```csharp
var authors = await context.Authors
    .AsNoTracking()
    .Select(a => new AuthorSummaryDto
    {
        Id = a.Id,
        Name = a.Name,
        BookCount = a.Books.Count()
    })
    .ToListAsync();
```

مزایا:

- کاهش حجم داده منتقل‌شده
- کاهش Memory Usage
- کاهش هزینه Materialization
- کاهش Tracking
- SQL هدفمندتر

---

# 3. Use AsNoTracking For Read-Only Queries

اگر Entity قرار نیست Update شود، معمولاً نیازی به Change Tracking نیست.

### Good

```csharp
var author = await context.Authors
    .AsNoTracking()
    .FirstOrDefaultAsync(a => a.Id == id);
```

برای Queryهای Read-Only از `AsNoTracking()` استفاده کنید، مخصوصاً در Queryهای پرتعداد.

### توجه

اگر Entity قرار است در همان `DbContext` تغییر داده شود و Save شود، `AsNoTracking()` ممکن است رفتار مورد نیاز را تغییر دهد.

---

# 4. Compiled Queries

## 4.1 When To Use Compiled Queries

EF Core بخش مهمی از فرآیند پردازش Queryها را cache می‌کند. بنابراین استفاده از Compiled Query نباید به‌عنوان یک Optimization پیش‌فرض برای تمام Queryها در نظر گرفته شود.

`EF.CompileAsyncQuery` می‌تواند در مسیرهای بسیار پرتکرار و Queryهای مشخص و پایدار، سربار پردازش Query را کاهش دهد.

### Good

```csharp
private static readonly Func<
    AppDbContext,
    int,
    Task<AuthorSummaryDto?>
> GetAuthorByIdCompiled =
    EF.CompileAsyncQuery(
        (AppDbContext context, int id) =>
            context.Authors
                .AsNoTracking()
                .Where(a => a.Id == id)
                .Select(a => new AuthorSummaryDto
                {
                    Id = a.Id,
                    Name = a.Name
                })
                .FirstOrDefault());
```

استفاده:

```csharp
public async Task<AuthorSummaryDto?> GetAuthorByIdAsync(int id)
{
    return await GetAuthorByIdCompiled(_context, id);
}
```

### مناسب برای

- Hot Pathها
- Queryهای پرتکرار
- Queryهای ثابت و مشخص
- سیستم‌هایی با حجم بالای Request

### قبل از استفاده

ابتدا Performance را اندازه‌گیری کنید.

```text
Measure
   ↓
Identify Bottleneck
   ↓
Optimize
   ↓
Benchmark
   ↓
Keep / Revert
```

Compiled Query نباید جایگزین طراحی صحیح Query، Index و Projection شود.

---

# 5. Database Indexing

## 5.1 Indexes Are Critical For Query Performance

بهینه بودن LINQ به‌تنهایی تضمین‌کننده Performance نیست.

اگر Database مجبور به انجام:

```text
Full Table Scan
```

باشد، حتی Query LINQ بسیار تمیز نیز ممکن است عملکرد مناسبی نداشته باشد.

ستون‌هایی که به‌طور مکرر در موارد زیر استفاده می‌شوند باید از نظر Index بررسی شوند:

- `WHERE`
- `JOIN`
- `ORDER BY`
- `GROUP BY`
- Foreign Keys
- جستجوهای پرتکرار

---

# 6. Configure Indexes With EF Core

ایندکس‌ها را می‌توان با Fluent API تعریف کرد.

### Foreign Key Index

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Book>()
        .HasIndex(b => b.AuthorId)
        .HasDatabaseName("IX_Books_AuthorId");
}
```

این Index می‌تواند در Queryهایی مانند موارد زیر مفید باشد:

```csharp
var books = await context.Books
    .Where(b => b.AuthorId == authorId)
    .ToListAsync();
```

و همچنین در بسیاری از Queryهای مرتبط با `JOIN`.

---

# 7. Composite Indexes

برای Queryهایی که به‌طور مکرر چند ستون را با هم فیلتر می‌کنند، می‌توان از Composite Index استفاده کرد.

### Example

```csharp
modelBuilder.Entity<Book>()
    .HasIndex(b => new
    {
        b.IsPublished,
        b.PublishedDate
    })
    .HasDatabaseName("IX_Books_Published_Filter");
```

برای Query:

```csharp
var books = await context.Books
    .Where(b =>
        b.IsPublished &&
        b.PublishedDate >= startDate)
    .ToListAsync();
```

می‌تواند مفید باشد.

### Important

ترتیب ستون‌های Composite Index مهم است.

این تصمیم باید بر اساس مواردی مانند:

- Query Pattern
- Selectivity
- Sort Order
- Cardinality
- حجم داده
- Execution Plan

انجام شود.

بنابراین نباید صرفاً هر ستونی که در `WHERE` وجود دارد را به یک Index تبدیل کرد.

---

# 8. Index Frequently Searched Columns

مثلاً اگر جستجوی `Author.Name` بسیار پرتکرار است:

```csharp
modelBuilder.Entity<Author>()
    .HasIndex(a => a.Name)
    .HasDatabaseName("IX_Authors_Name");
```

اما قبل از ایجاد Index باید بررسی شود:

- Query واقعاً پرتکرار است؟
- Selectivity مناسب است؟
- Database Engine چگونه Query را اجرا می‌کند؟
- هزینه Index در عملیات `INSERT` و `UPDATE` قابل قبول است؟

Index بیش از حد نیز می‌تواند Performance عملیات نوشتن را کاهش دهد.

---

# 9. Query Optimization Checklist

قبل از Optimization یک Query، موارد زیر را بررسی کنید:

- [ ] آیا N+1 Query وجود دارد؟
- [ ] آیا `Include` واقعاً لازم است؟
- [ ] آیا Projection با `Select` مناسب‌تر است؟
- [ ] آیا `AsNoTracking()` قابل استفاده است؟
- [ ] آیا Query داده اضافی دریافت می‌کند؟
- [ ] آیا Pagination وجود دارد؟
- [ ] آیا Index مناسب وجود دارد؟
- [ ] آیا Composite Index لازم است؟
- [ ] آیا Execution Plan بررسی شده است؟
- [ ] آیا Query واقعاً Bottleneck است؟
- [ ] آیا قبل و بعد از تغییر Benchmark انجام شده است؟

---

# 10. General Rule

### ❌ این فرض اشتباه است:

```text
LINQ خوب
    ↓
SQL خوب
    ↓
Performance خوب
```

### ✅ Performance واقعی نتیجه تعامل چند بخش است:

```text
LINQ / EF Core
       ↓
Generated SQL
       ↓
Database Query Plan
       ↓
Indexes
       ↓
Data Volume
       ↓
Network
       ↓
Materialization
       ↓
Application Memory
```

بنابراین برای Performance همیشه فقط به کد C# نگاه نکنید؛ **SQL تولیدشده و Execution Plan دیتابیس نیز باید بررسی شوند.**