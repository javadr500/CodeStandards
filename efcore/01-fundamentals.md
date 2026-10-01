# EF Core Fundamentals

مرجع سریع مفاهیم پایه و استانداردهای Entity Framework Core.

---

# 1. DbContext

`DbContext` نقطه اصلی ارتباط برنامه با Database در EF Core است.

### Rule

در ASP.NET Core معمولاً `DbContext` باید به‌صورت `Scoped` ثبت شود.

### Example

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString));
```

### نکته

معمولاً هر HTTP Request یک Instance از `DbContext` دریافت می‌کند.

---

# 2. DbSet

`DbSet<TEntity>` نماینده یک جدول یا مجموعه Entityها در EF Core است.

### Example

```csharp
public class AppDbContext : DbContext
{
    public DbSet<Book> Books => Set<Book>();
    public DbSet<Author> Authors => Set<Author>();
}
```

استفاده:

```csharp
var books = await context.Books.ToListAsync();
```

---

# 3. Entity

Entity کلاسی است که EF Core آن را به ساختار Database نگاشت می‌کند.

### Example

```csharp
public class Book
{
    public int Id { get; set; }
    public string Title { get; set; } = null!;
    public int AuthorId { get; set; }
}
```

---

# 4. Primary Key

هر Entity معمولاً یک Primary Key دارد.

EF Core در بسیاری از موارد `Id` یا `{EntityName}Id` را به‌صورت Convention به‌عنوان کلید اصلی تشخیص می‌دهد.

### Example

```csharp
public class Book
{
    public int Id { get; set; }
}
```

یا به‌صورت صریح:

```csharp
modelBuilder.Entity<Book>()
    .HasKey(x => x.Id);
```

---

# 5. Convention

EF Core بسیاری از تنظیمات را بدون Configuration صریح و بر اساس Convention انجام می‌دهد.

### Example

```csharp
public class Book
{
    public int Id { get; set; }

    public string Title { get; set; } = null!;
}
```

EF Core می‌تواند:

```text
Book
 ├── Id
 └── Title
```

را بر اساس Convention به ساختار Database نگاشت کند.

### Rule

تا زمانی که Convention نیاز پروژه را پوشش می‌دهد، Configuration غیرضروری ایجاد نکنید.

---

# 6. Fluent API

برای Configuration دقیق Entityها از Fluent API استفاده می‌شود.

### Example

```csharp
modelBuilder.Entity<Book>()
    .Property(x => x.Title)
    .HasMaxLength(200)
    .IsRequired();
```

مزیت:

- کنترل بیشتر
- Configuration متمرکز
- مناسب برای قوانین پیچیده Database

---

# 7. IEntityTypeConfiguration

برای پروژه‌های بزرگ، Configuration هر Entity را در کلاس جدا قرار دهید.

### Example

```csharp
public class BookConfiguration
    : IEntityTypeConfiguration<Book>
{
    public void Configure(EntityTypeBuilder<Book> builder)
    {
        builder.HasKey(x => x.Id);

        builder.Property(x => x.Title)
            .IsRequired()
            .HasMaxLength(200);
    }
}
```

ثبت Configurationها:

```csharp
modelBuilder.ApplyConfigurationsFromAssembly(
    typeof(AppDbContext).Assembly);
```

### Rule

در پروژه‌های بزرگ، `OnModelCreating` را با صدها خط Configuration شلوغ نکنید.

---

# 8. Data Annotations

می‌توان Configurationهای ساده را با Attribute انجام داد.

### Example

```csharp
public class Book
{
    public int Id { get; set; }

    [Required]
    [MaxLength(200)]
    public string Title { get; set; } = null!;
}
```

### Rule

برای Configurationهای پیچیده‌تر معمولاً Fluent API خواناتر و انعطاف‌پذیرتر است.

---

# 9. SaveChanges

تغییرات Entityها با `SaveChanges` یا `SaveChangesAsync` به Database ارسال می‌شوند.

### Example

```csharp
var book = new Book
{
    Title = "Clean Code"
};

context.Books.Add(book);

await context.SaveChangesAsync();
```

---

# 10. Change Tracking

EF Core تغییرات Entityهایی را که Track می‌کند، دنبال می‌کند.

### Example

```csharp
var book = await context.Books
    .FirstAsync(x => x.Id == id);

book.Title = "New Title";

await context.SaveChangesAsync();
```

EF Core متوجه تغییر `Title` می‌شود و Update مناسب را ایجاد می‌کند.

---

# 11. Entity States

Entity می‌تواند Stateهای مختلفی داشته باشد:

```text
Detached
Unchanged
Added
Modified
Deleted
```

### Example

```csharp
context.Books.Add(book);
```

State:

```text
Added
```

و:

```csharp
context.Books.Remove(book);
```

State:

```text
Deleted
```

---

# 12. AsNoTracking

برای Queryهای فقط خواندنی می‌توان Tracking را غیرفعال کرد.

### Example

```csharp
var books = await context.Books
    .AsNoTracking()
    .ToListAsync();
```

### Rule

برای Read-Only Queryها، در صورت عدم نیاز به Tracking، `AsNoTracking()` را در نظر بگیرید.

---

# 13. IQueryable

`IQueryable` اجازه می‌دهد Query قبل از اجرای واقعی ساخته شود.

### Example

```csharp
IQueryable<Book> query = context.Books;

query = query.Where(x => x.AuthorId == authorId);

var books = await query.ToListAsync();
```

تا قبل از `ToListAsync()` معمولاً Query اجرا نشده است.

---

# 14. Deferred Execution

بسیاری از Queryهای EF Core تا زمانی که یک عملیات Materialization انجام نشود، اجرا نمی‌شوند.

### Example

```csharp
var query = context.Books
    .Where(x => x.AuthorId == authorId);

// Database Query اجرا نشده

var books = await query.ToListAsync();

// اینجا Query اجرا می‌شود
```

### نکته

عملیات‌هایی مانند:

```csharp
ToList()
ToListAsync()
First()
FirstAsync()
Count()
CountAsync()
Any()
AnyAsync()
```

می‌توانند باعث اجرای Query شوند.

---

# 15. Projection

به جای دریافت Entity کامل، فقط اطلاعات مورد نیاز را دریافت کنید.

### Bad

```csharp
var books = await context.Books
    .ToListAsync();
```

اگر فقط عنوان لازم باشد:

### Good

```csharp
var titles = await context.Books
    .Select(x => x.Title)
    .ToListAsync();
```

یا DTO:

```csharp
var result = await context.Books
    .Select(x => new BookDto
    {
        Id = x.Id,
        Title = x.Title
    })
    .ToListAsync();
```

---

# 16. First vs Single

این دو متد رفتار متفاوتی دارند.

### First

اولین رکورد را برمی‌گرداند.

```csharp
var book = await context.Books
    .FirstOrDefaultAsync(x => x.Id == id);
```

### Single

انتظار دارد حداکثر یک رکورد وجود داشته باشد و اگر چند رکورد پیدا شود Exception ایجاد می‌کند.

```csharp
var user = await context.Users
    .SingleAsync(x => x.Email == email);
```

### Rule

اگر Business Rule می‌گوید مقدار باید Unique باشد، `Single` می‌تواند معنای دقیق‌تری داشته باشد.

---

# 17. Any Instead of Count

اگر فقط می‌خواهیم بدانیم رکوردی وجود دارد یا نه، `Any` مناسب‌تر است.

### Bad

```csharp
var exists = await context.Books
    .CountAsync(x => x.AuthorId == authorId) > 0;
```

### Good

```csharp
var exists = await context.Books
    .AnyAsync(x => x.AuthorId == authorId);
```

### Rule

برای بررسی وجود رکورد:

```text
Any() > Count() > ToList()
```

از نظر هدف Query، `Any` انتخاب طبیعی‌تری است.

---

# 18. FindAsync

اگر Entity با Primary Key جستجو می‌شود، `FindAsync` گزینه مناسبی است.

### Example

```csharp
var book = await context.Books
    .FindAsync(id);
```

`FindAsync` می‌تواند ابتدا Entity موردنظر را در Change Tracker بررسی کند و در صورت نیاز به Database مراجعه کند.

---

# 19. Pagination

هیچ‌گاه در داده‌های بزرگ بدون نیاز، کل جدول را دریافت نکنید.

### Bad

```csharp
var books = await context.Books
    .ToListAsync();
```

### Good

```csharp
var books = await context.Books
    .OrderBy(x => x.Id)
    .Skip((page - 1) * pageSize)
    .Take(pageSize)
    .ToListAsync();
```

### نکته

برای Datasetهای بسیار بزرگ، Keyset/Cursor Pagination می‌تواند گزینه مناسب‌تری نسبت به Offset Pagination باشد.

---

# 20. CancellationToken

در عملیات Async طولانی، بهتر است امکان Cancellation وجود داشته باشد.

### Example

```csharp
public async Task<List<Book>> GetBooksAsync(
    CancellationToken cancellationToken)
{
    return await context.Books
        .AsNoTracking()
        .ToListAsync(cancellationToken);
}
```

---

# 21. Never Use DbContext as Singleton

`DbContext` برای استفاده همزمان توسط چند Thread طراحی نشده است.

### Bad

```csharp
services.AddSingleton<AppDbContext>();
```

### Good

```csharp
services.AddDbContext<AppDbContext>();
```

در ASP.NET Core، این ثبت به‌طور معمول Scoped است.

---

# 22. Avoid Premature Optimization

هر قابلیت Performance را فقط به دلیل اینکه «سریع‌تر به نظر می‌رسد» اضافه نکنید.

مثلاً:

```text
Compiled Query
Caching
Raw SQL
Bulk Operations
```

قبل از استفاده باید مشخص شود که واقعاً Bottleneck وجود دارد.

### Rule

```text
Measure
   ↓
Find Bottleneck
   ↓
Optimize
   ↓
Measure Again
```

---

# 23. EF Core Performance Golden Rules

```text
1. Avoid N+1
2. Use Projection when possible
3. Use AsNoTracking for Read-Only queries
4. Do not load unnecessary columns
5. Use Pagination for large datasets
6. Use Any instead of Count when checking existence
7. Check generated SQL
8. Create appropriate indexes
9. Avoid unnecessary Include
10. Measure before optimizing
```

---

# Quick Checklist

- [ ] DbContext Lifetime صحیح است
- [ ] DbContext Singleton نیست
- [ ] Queryهای Read-Only بررسی شده‌اند
- [ ] `AsNoTracking` در موارد مناسب استفاده شده
- [ ] Projection بررسی شده
- [ ] N+1 وجود ندارد
- [ ] Pagination برای داده‌های حجیم وجود دارد
- [ ] `Any` برای بررسی وجود استفاده شده
- [ ] Queryهای غیرضروری اجرا نمی‌شوند
- [ ] SQL تولیدشده بررسی شده
- [ ] Indexهای لازم وجود دارند
- [ ] قبل از Performance Optimization اندازه‌گیری انجام شده است