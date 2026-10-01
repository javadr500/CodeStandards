# EF Core Loading Strategies

روش‌های بارگذاری داده‌های مرتبط در Entity Framework Core.

## 1. Eager Loading

### تعریف
در Eager Loading، داده‌های مرتبط را از ابتدا و همراه Query اصلی درخواست می‌کنیم.

### مثال

```csharp
var authors = await context.Authors
    .Include(a => a.Books)
    .ToListAsync();
```

در این مثال، نویسنده‌ها و کتاب‌های مرتبط آن‌ها دریافت می‌شوند.

### قانون
وقتی از ابتدا به Navigation Propertyها نیاز دارید، Eager Loading را بررسی کنید.

### نکته
`Include` الزاماً به یک `LEFT JOIN` واحد تبدیل نمی‌شود؛ شکل SQL به نوع رابطه، Query و تنظیمات EF Core بستگی دارد.

---

## 2. ThenInclude

### تعریف
برای بارگذاری رابطه‌های چندمرحله‌ای استفاده می‌شود.

### مثال

```csharp
var authors = await context.Authors
    .Include(a => a.Books)
        .ThenInclude(b => b.Category)
    .ToListAsync();
```

ساختار داده:

```text
Author
  └── Books
        └── Category
```

### قانون
از `ThenInclude` برای مشخص کردن مسیر رابطه‌های تو در تو استفاده کنید.

---

## 3. Explicit Loading

### تعریف
در Explicit Loading، ابتدا Entity را دریافت می‌کنیم و سپس در صورت نیاز، اطلاعات مرتبط را صریحاً بارگذاری می‌کنیم.

### مثال

```csharp
var author = await context.Authors
    .FirstAsync(a => a.Id == authorId);

await context.Entry(author)
    .Collection(a => a.Books)
    .LoadAsync();
```

### چه زمانی مناسب است؟
وقتی فقط در بعضی شرایط به داده مرتبط نیاز داریم.

### نکته
اگر این عملیات را برای تعداد زیادی Entity داخل حلقه اجرا کنید، ممکن است دوباره با مشکل N+1 مواجه شوید.

---

## 4. Lazy Loading

### تعریف
در Lazy Loading، داده مرتبط هنگام دسترسی به Navigation Property بارگذاری می‌شود.

### مثال

با فعال بودن Lazy Loading Proxies:

```csharp
public class Author
{
    public int Id { get; set; }

    public string Name { get; set; } = "";

    public virtual ICollection<Book> Books { get; set; }
        = new List<Book>();
}
```

ثبت سرویس:

```csharp
options.UseLazyLoadingProxies()
       .UseSqlServer(connectionString);
```

سپس:

```csharp
var author = await context.Authors
    .FirstAsync(a => a.Id == authorId);

var books = author.Books;
```

دسترسی به `Books` می‌تواند باعث اجرای Query جداگانه شود.

### هشدار
Lazy Loading ممکن است Queryهای پنهان و مشکل N+1 ایجاد کند؛ به‌خصوص هنگام پیمایش مجموعه‌ای از Entityها.

---

## 5. Eager vs Explicit vs Lazy Loading

| روش | زمان بارگذاری | کاربرد |
|---|---|---|
| Eager | همراه Query اصلی | داده مرتبط از ابتدا لازم است |
| Explicit | با درخواست صریح بعدی | داده مرتبط فقط در شرایط خاص لازم است |
| Lazy | هنگام دسترسی به Navigation | بارگذاری خودکار و تنبل |

### قانون انتخاب
روش بارگذاری را براساس نیاز واقعی داده و تعداد Queryها انتخاب کنید؛ هیچ روشی برای همه شرایط بهترین نیست.

---

## 6. Filtered Include

### تعریف
با Filtered Include می‌توان فقط بخشی از داده‌های مرتبط را بارگذاری کرد.

### مثال

```csharp
var authors = await context.Authors
    .Include(a => a.Books
        .Where(b => b.IsPublished))
    .ToListAsync();
```

در این مثال، فقط کتاب‌های منتشرشده بارگذاری می‌شوند.

### قانون
وقتی به تمام داده‌های مرتبط نیاز ندارید، Filtered Include را بررسی کنید.

### نکته
در Queryهای Tracking، Navigation Fix-up ممکن است باعث شود Entityهای قبلاً Trackشده که شرط فیلتر را ندارند نیز در Navigation ظاهر شوند. برای نتایج قابل پیش‌بینی، در صورت مناسب بودن از `AsNoTracking()` یا یک `DbContext` تازه استفاده کنید.

---

## 7. Multiple Collection Includes

### مشکل
بارگذاری چند Collection با `Include` ممکن است تعداد ردیف‌های حاصل از JOIN را به‌شدت افزایش دهد.

### مثال

```csharp
var authors = await context.Authors
    .Include(a => a.Books)
    .Include(a => a.Awards)
    .ToListAsync();
```

اگر یک نویسنده ۱۰ کتاب و ۵ جایزه داشته باشد، JOIN هم‌زمان این دو Collection ممکن است برای آن نویسنده ۵۰ ترکیب ردیفی تولید کند.

### قانون
در Queryهای دارای چند Collection، حجم داده و شکل SQL را بررسی کنید.

---

## 8. AsSplitQuery

### تعریف
`AsSplitQuery()` به EF Core می‌گوید Collectionهای Includeشده را به‌جای یک Query ترکیبی، با چند Query جداگانه بارگذاری کند.

### مثال

```csharp
var authors = await context.Authors
    .Include(a => a.Books)
    .Include(a => a.Awards)
    .AsSplitQuery()
    .ToListAsync();
```

### مزایا
- کاهش تکرار داده‌های والد در نتیجه JOIN
- کاهش احتمال انفجار تعداد ردیف‌ها در چند Collection

### معایب و ملاحظات
- اجرای چند Query به‌جای یک Query
- احتمال مشاهده داده‌های متفاوت میان Queryها در صورت تغییر هم‌زمان داده‌ها
- امکان نیاز به Transaction با Isolation مناسب، اگر سازگاری یکپارچه نتایج ضروری باشد

### قانون
`AsSplitQuery()` را براساس اندازه داده، تعداد Collectionها و نیازهای سازگاری انتخاب کنید.

---

## 9. AsSingleQuery

### تعریف
`AsSingleQuery()` بارگذاری را در قالب یک Query واحد انجام می‌دهد.

### مثال

```csharp
var authors = await context.Authors
    .Include(a => a.Books)
    .Include(a => a.Awards)
    .AsSingleQuery()
    .ToListAsync();
```

### قانون
Single Query می‌تواند برای بعضی Queryها مناسب‌تر باشد؛ اما در بارگذاری چند Collection باید هزینه JOIN و تکرار داده‌ها را بررسی کنید.

---

## 10. Projection Instead of Include

### تعریف
اگر فقط چند فیلد از داده مرتبط نیاز دارید، به‌جای بارگذاری Entityهای کامل، نتیجه را با `Select` بسازید.

### مثال

```csharp
var authors = await context.Authors
    .AsNoTracking()
    .Select(a => new AuthorDto
    {
        Id = a.Id,
        Name = a.Name,
        BookCount = a.Books.Count()
    })
    .ToListAsync();
```

### مزایا
- دریافت فقط اطلاعات موردنیاز
- کاهش حجم داده منتقل‌شده
- کاهش هزینه Materialization

### قانون
برای DTOها و خروجی API، ابتدا بررسی کنید که Projection از `Include` مناسب‌تر است یا خیر.

---

## 11. Avoid N+1 During Loading

### کد نامناسب

```csharp
var authors = await context.Authors
    .ToListAsync();

foreach (var author in authors)
{
    var books = await context.Books
        .Where(b => b.AuthorId == author.Id)
        .ToListAsync();
}
```

اگر ۱۰۰ نویسنده داشته باشیم، این کد می‌تواند ۱۰۱ Query اجرا کند.

### راهکار با Eager Loading

```csharp
var authors = await context.Authors
    .Include(a => a.Books)
    .ToListAsync();
```

### راهکار با Projection

```csharp
var authors = await context.Authors
    .Select(a => new AuthorDto
    {
        Id = a.Id,
        Name = a.Name,
        BookCount = a.Books.Count()
    })
    .ToListAsync();
```

### قانون
صرفاً برای حذف N+1 از `Include` استفاده نکنید؛ اگر داده مرتبط برای خروجی لازم نیست، آن را دریافت نکنید.

---

## 12. Avoid Loading Unnecessary Navigation Properties

### نامناسب

```csharp
var books = await context.Books
    .Include(b => b.Author)
    .Include(b => b.Reviews)
    .Include(b => b.Category)
    .ToListAsync();
```

اگر فقط عنوان کتاب لازم است، بارگذاری این رابطه‌ها غیرضروری است.

### مناسب‌تر

```csharp
var titles = await context.Books
    .Select(b => b.Title)
    .ToListAsync();
```

### قانون
هر `Include` باید دلیل مشخصی داشته باشد.

---

## 13. Loading Checklist

- [ ] آیا داده مرتبط واقعاً لازم است؟
- [ ] آیا Eager Loading مناسب است؟
- [ ] آیا Explicit Loading به تعداد Queryهای زیاد منجر می‌شود؟
- [ ] آیا Lazy Loading باعث N+1 می‌شود؟
- [ ] آیا `ThenInclude` فقط برای مسیرهای لازم استفاده شده؟
- [ ] آیا Filtered Include مفید است؟
- [ ] آیا چند Collection هم‌زمان بارگذاری می‌شوند؟
- [ ] آیا `AsSplitQuery()` باید بررسی شود؟
- [ ] آیا Projection انتخاب مناسب‌تری است؟
- [ ] آیا SQL و تعداد Queryها بررسی شده‌اند؟

---

## Golden Rules

1. Eager Loading برای داده‌های مرتبطی که از ابتدا لازم‌اند.
2. Explicit Loading برای بارگذاری شرطی و صریح.
3. Lazy Loading با آگاهی از Queryهای پنهان.
4. استفاده از `ThenInclude` برای مسیرهای چندمرحله‌ای.
5. بررسی Filtered Include برای محدودکردن داده‌های مرتبط.
6. بررسی `AsSplitQuery()` هنگام بارگذاری چند Collection.
7. ترجیح Projection برای DTOها و خروجی‌های محدود.
8. جلوگیری از N+1 و بارگذاری غیرضروری.
9. بررسی SQL تولیدشده و تعداد Round Tripها.


