

# EF Core — Change Tracking

## 1. What Is Change Tracking?

EF Core تغییرات Entityهایی را که تحت نظارت `DbContext` هستند دنبال می‌کند و هنگام اجرای `SaveChangesAsync` آن‌ها را در دیتابیس اعمال می‌کند.

```csharp
var author = await context.Authors
    .FirstAsync(a => a.Id == id);

author.Name = "Updated Name";

await context.SaveChangesAsync();
```

در این مثال، EF Core تغییر `Name` را تشخیص می‌دهد و دستور SQL مناسب را اجرا می‌کند.

**قاعده:** برای Entityهای tracked معمولاً نیازی به فراخوانی `Update` بعد از تغییر Property نیست.

---

## 2. Entity States

هر Entity تحت Tracking یکی از وضعیت‌های اصلی زیر را دارد:

| State | Meaning |
|---|---|
| `Detached` | توسط این Context ردیابی نمی‌شود. |
| `Unchanged` | ردیابی می‌شود اما تغییر شناسایی‌شده‌ای ندارد. |
| `Added` | برای درج در دیتابیس آماده است. |
| `Modified` | برای به‌روزرسانی آماده است. |
| `Deleted` | برای حذف آماده است. |

مشاهده وضعیت:

```csharp
var author = new Author
{
    Name = "Ali"
};

context.Authors.Add(author);

Console.WriteLine(
    context.Entry(author).State); // Added
```

---

## 3. Add vs Attach vs Update vs Remove

### Add

Entity را برای درج علامت‌گذاری می‌کند. رفتار روی Entityهای مرتبط نیز به وضعیت و کلیدهای آن‌ها وابسته است.

```csharp
var author = new Author
{
    Name = "Ali"
};

context.Authors.Add(author);

await context.SaveChangesAsync();
```

### Attach

معمولاً Entity موجود را با وضعیت `Unchanged` به Context متصل می‌کند.

```csharp
var author = new Author
{
    Id = 10,
    Name = "Ali"
};

context.Authors.Attach(author);
```

**نکته:** اگر Entity یا Entityهای مرتبط کلید تولیدشونده‌ی مقداردهی‌نشده داشته باشند، رفتار می‌تواند متفاوت باشد.

### Update

Entity را برای به‌روزرسانی علامت‌گذاری می‌کند. در حالت متداول Entity جداشده با کلید موجود، Propertyهای آن به‌عنوان تغییرکرده علامت‌گذاری می‌شوند؛ رفتار دقیق به وضعیت کلید و گراف Entityها بستگی دارد.

```csharp
var author = new Author
{
    Id = 10,
    Name = "Updated Name"
};

context.Authors.Update(author);

await context.SaveChangesAsync();
```

**هشدار:** استفاده‌ی مستقیم از `Update` روی یک DTO ناقص یا Entity جداشده ممکن است باعث بازنویسی Propertyها با مقادیر پیش‌فرض یا `null` شود.

### Remove

Entity را برای حذف علامت‌گذاری می‌کند.

```csharp
var author = await context.Authors
    .FindAsync(id);

if (author is not null)
{
    context.Authors.Remove(author);
    await context.SaveChangesAsync();
}
```

---

## 4. Tracked Query vs AsNoTracking

### Tracked Query

به‌صورت پیش‌فرض، Entityهایی که از Queryهای معمول EF Core دریافت می‌شوند، ردیابی می‌شوند.

```csharp
var author = await context.Authors
    .FirstAsync(a => a.Id == id);

author.Name = "New Name";

await context.SaveChangesAsync();
```

### AsNoTracking

برای خواندن اطلاعات بدون نیاز به Tracking:

```csharp
var authors = await context.Authors
    .AsNoTracking()
    .Select(a => new
    {
        a.Id,
        a.Name
    })
    .ToListAsync();
```

**قاعده:** برای Queryهای صرفاً خواندنی، `AsNoTracking` می‌تواند سربار Tracking را کاهش دهد؛ اما تأثیر واقعی به نوع Query و حجم داده بستگی دارد.

---

## 5. AsNoTrackingWithIdentityResolution

گاهی یک Entity در نتیجه‌ی Query چند بار ظاهر می‌شود؛ مثلاً در Queryهای دارای روابط.

```csharp
var authors = await context.Authors
    .AsNoTrackingWithIdentityResolution()
    .Include(a => a.Books)
    .ToListAsync();
```

این روش در طول ساخت نتیجه، Instanceهای تکراری مربوط به یک کلید را یکسان‌سازی می‌کند، بدون آنکه Entityها را در Change Tracker اصلی Context ثبت کند.

**تفاوت مهم:**
- `AsNoTracking`: بدون Tracking و بدون تضمین Identity Resolution.
- `AsNoTrackingWithIdentityResolution`: بدون Tracking معمول Context، همراه با Identity Resolution موقت.

---

## 6. Updating a Tracked Entity

وقتی Entity را از دیتابیس دریافت کرده‌اید، فقط Property موردنظر را تغییر دهید.

```csharp
var author = await context.Authors
    .FirstOrDefaultAsync(a => a.Id == id);

if (author is null)
{
    return;
}

author.Name = "Updated Name";

await context.SaveChangesAsync();
```

EF Core تغییرات را تشخیص می‌دهد و معمولاً فقط Propertyهای تغییرکرده را در دستور `UPDATE` قرار می‌دهد.

---

## 7. Updating a Specific Property

اگر Entity جداشده است و می‌خواهید فقط یک Property به‌روزرسانی شود:

```csharp
var author = new Author
{
    Id = id,
    Name = "Updated Name"
};

context.Attach(author);

context.Entry(author)
    .Property(a => a.Name)
    .IsModified = true;

await context.SaveChangesAsync();
```

این الگو از علامت‌گذاری همه‌ی Propertyها به‌عنوان تغییرکرده جلوگیری می‌کند.

**نکته:** فقط Propertyهایی را تغییر دهید که کاربر اجازه‌ی ویرایش آن‌ها را دارد؛ این موضوع در جلوگیری از Overposting مهم است.

---

## 8. Inspecting the Change Tracker

برای بررسی Entityهای ردیابی‌شده:

```csharp
var entries = context.ChangeTracker.Entries();

foreach (var entry in entries)
{
    Console.WriteLine(
        $"{entry.Entity.GetType().Name}: {entry.State}");
}
```

این قابلیت برای Debugging و بررسی تغییرات پیش از ذخیره مفید است.

---

## 9. DetectChanges

EF Core معمولاً در زمان‌های لازم تغییرات Propertyها را شناسایی می‌کند.

```csharp
context.ChangeTracker.DetectChanges();
```

در اغلب کاربردهای عادی، نیازی نیست این متد را دستی فراخوانی کنید.

**قاعده:** تنظیماتی مانند غیرفعال‌کردن `AutoDetectChangesEnabled` را فقط در صورت وجود مشکل کارایی اندازه‌گیری‌شده و با درک کامل پیامدها تغییر دهید.

---

## 10. Disconnected Entities

در برنامه‌های وب، Entity معمولاً در یک درخواست خوانده می‌شود و اطلاعات ویرایش‌شده در درخواست دیگری ارسال می‌شود. بنابراین Entity ارسالی اغلب توسط Context فعلی ردیابی نمی‌شود.

دو الگوی رایج:

**الگوی اول — دریافت Entity و اعمال تغییرات مجاز:**

```csharp
var author = await context.Authors
    .FirstOrDefaultAsync(a => a.Id == request.Id);

if (author is null)
{
    return;
}

author.Name = request.Name;

await context.SaveChangesAsync();
```

**الگوی دوم — اتصال Entity و علامت‌گذاری Property مشخص:**

```csharp
var author = new Author
{
    Id = request.Id,
    Name = request.Name
};

context.Attach(author);

context.Entry(author)
    .Property(a => a.Name)
    .IsModified = true;

await context.SaveChangesAsync();
```

الگوی اول برای بسیاری از عملیات معمول وب خواناتر است و امکان کنترل دسترسی و اعتبارسنجی مقادیر را فراهم می‌کند. الگوی دوم در شرایط مشخص می‌تواند از خواندن اولیه جلوگیری کند.

---

## 11. Change Tracking and ExecuteUpdate

دستورهای `ExecuteUpdateAsync` و `ExecuteDeleteAsync` مستقیماً روی دیتابیس اجرا می‌شوند و تغییرات را از مسیر معمول Change Tracker اعمال نمی‌کنند.

```csharp
await context.Authors
    .Where(a => a.Id == id)
    .ExecuteUpdateAsync(setters =>
        setters.SetProperty(
            a => a.Name,
            "Updated Name"));
```

اگر Entity متناظر از قبل در Context ردیابی شده باشد، وضعیت آن لزوماً با مقدار جدید دیتابیس هماهنگ نمی‌شود.

برای جلوگیری از خواندن داده‌ی قدیمی، از Context مناسب استفاده کنید یا Entity ردیابی‌شده را مجدداً بارگذاری کنید.

---

## 12. Checklist

- [ ] برای تغییر Entity ردیابی‌شده، بی‌دلیل `Update` فراخوانی نکن.
- [ ] برای Queryهای صرفاً خواندنی، `AsNoTracking` را بررسی کن.
- [ ] روی DTO ناقص از `Update` بدون بررسی پیامدها استفاده نکن.
- [ ] فقط Propertyهای مجاز را تغییر بده.
- [ ] وضعیت Entityها را هنگام Debugging بررسی کن.
- [ ] تغییرات `ExecuteUpdate` و `ExecuteDelete` را از Tracking معمول جدا بدان.
- [ ] در عملیات حساس، مدیریت هم‌زمانی را نیز در نظر بگیر.

## Golden Rules

1. `SaveChangesAsync` تغییرات Entityهای tracked را ذخیره می‌کند.
2. برای Entity tracked، تغییر Property و سپس `SaveChangesAsync` معمولاً کافی است.
3. `Update` جایگزین اعتبارسنجی و کنترل دسترسی نیست.
4. `AsNoTracking` برای خواندن مفید است، نه برای Entityای که انتظار دارید تغییراتش به‌طور معمول Track شود.
5. برای سناریوهای هم‌زمانی و تعارض ویرایش، فایل `concurrency.md` را ببینید.




