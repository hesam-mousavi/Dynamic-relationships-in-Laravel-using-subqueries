در حال به روز رسانی ...

# Dynamic relationships in Laravel using subqueries [^1]
 بررسی تاثیر عمیق ساب کوئری های لاراول در کارایی 

---
هنگام ساخت برنامه های وب که با پایگاه داده تعامل دارند، همیشه دو هدف مد نظر می باشد:

1. درخواست های پایگاه داده را به حداقل برسانید.
2. مصرف حافظه را به حداقل برسانید.

این اهداف می توانند تأثیر شدیدی بر عملکرد برنامه شما داشته باشند.

توسعه دهندگان معمولا با مشکل n+1 در کوئری ها آشنا هستند و با تکنیکی به نام eager-loading این مشکل را حل می کنند،  با این حال، ما همیشه در هدف دوم بهترین نیستیم، یعنی کاهش مصرف حافظه.<br>در واقع، ما گاهی اوقات بیشتر از اینکه سودی داشته باشیم، تلاش می‌کنیم تا درخواست‌های پایگاه داده را به قیمت استفاده از حافظه کاهش دهیم. (یعنی تعداد کوئری ها را کم میکنیم اما بعضی اوقات همین مساله باعث مصرف حافظه شدید می شود)

اجازه دهید توضیح دهم که چگونه این اتفاق می افتد، و برای برآورده کردن هر دو هدف در برنامه خود چه کاری می توانید انجام دهید.

## چالش:

مثال زیر را در نظر بگیرید. شما یک صفحه برای کاربران در برنامه خود دارید که برخی از اطلاعات مربوط به آنها از جمله آخرین تاریخ ورود به سیستم را نشان می دهد. این صفحه به ظاهر ساده در واقع پیچیدگی جالبی را ارائه می دهد.

| Name      | Email | 	Last Login |
| ----------- | :-----------: | -----------:
| Adam Campbell | adam@hotmeteor.com | Nov 10, 2018 at 12:01pm
| Taylor Otwell | taylor@laravel.com | Never
| Jonathan Reinink | jonathan@reinink.ca | Jun 2, 2018 at 5:30am
| Adam Wathan | adam.wathan@gmail.com | Nov 20, 2018 at 7:49pm

در این برنامه ما ورود کاربران را در جدول login دنبال می‌کنیم، بنابراین می‌توانیم گزارش‌های آماری را روی آن انجام دهیم.<br>در اینجا طرح ساده ای از پایگاه داده را می بینید:

```php
Schema::create('users', function (Blueprint $table) {
    $table->increments('id');
    $table->string('name');
    $table->string('email');
    $table->timestamps();
});

Schema::create('logins', function (Blueprint $table) {
    $table->increments('id');
    $table->integer('user_id');
    $table->string('ip_address');
    $table->timestamp('created_at');
});
```
و در اینجا مدل های مربوط به آن جداول با روابط آنها وجود دارد:

```php
class User extends Model
{
    public function logins()
    {
        return $this->hasMany(Login::class);
    }
}

class Login extends Model
{
    public function user()
    {
        return $this->belongsTo(User::class);
    }
}
```
خب حالا چطور می‌توانیم صفحه کاربران را ایجاد کنیم؟ و به طور خاص، چگونه آخرین تاریخ ورود به سیستم را بدست بیاوریم؟ راه سریع در اینجا ممکن است انجام موارد زیر باشد:

```php
$users = User::all();

@foreach ($users as $user)
    <tr>
        <td>{{ $user->name }}</td>
        <td>{{ $user->email }}</td>
        <td>
            @if ($lastLogin = $user->logins()->latest()->first())
                {{ $lastLogin->created_at->format('M j, Y \a\t g:i a') }}
            @else
                Never
            @endif
        </td>
    </tr>
@endforeach
```
اما، اگر توسعه‌دهنده خوبی باشیم (که هستیم)، در اینجا متوجه مشکلی خواهیم شد.<br>
ما مشکل N+1 معروف را ایجاد کردیم.
یعنی یک کوئری برای دریافت کاربران و تعداد N کوئری برای دریافت اطلاعات لاگین مربوط به هر کاربر!!<br>
پس اگر تعداد 50 کاربر داشته باشیم، برای اجرای برنامه بالا تعداد 51 کوئری ایجاد می شود.

```sql
select * from "users";
select * from "logins" where "logins"."user_id" = 1 and "logins"."user_id" is not null order by "created_at" desc limit 1;
select * from "logins" where "logins"."user_id" = 2 and "logins"."user_id" is not null order by "created_at" desc limit 1;
// ...
select * from "logins" where "logins"."user_id" = 49 and "logins"."user_id" is not null order by "created_at" desc limit 1;
select * from "logins" where "logins"."user_id" = 50 and "logins"."user_id" is not null order by "created_at" desc limit 1;
```
ما یک راه حل بهتر سراغ داریم و آن استفاده از eager-load می باشد:

```php
$users = User::with('logins')->get();

@foreach ($users as $user)
    <tr>
        <td>{{ $user->name }}</td>
        <td>{{ $user->email }}</td>
        <td>
            @if ($user->logins->isNotEmpty())
                {{ $user->logins->sortByDesc('created_at')->first()->created_at->format('M j, Y \a\t g:i a') }}
            @else
                Never
            @endif
        </td>
    </tr>
@endforeach
```
این راه حل فقط به دو کوئری پایگاه داده نیاز دارد. یکی برای کاربران و دیگری برای سوابق ورود مربوطه. ظاهرا موفقیت امیز بود.

خوب، نه دقیقا. اینجاست که بحث حافظه به یک مشکل تبدیل می شود. مطمئناً، ما از مشکل N+1 اجتناب کرده‌ایم، اما در واقع یک مشکل حافظه بسیار بزرگ‌تر ایجاد کرده‌ایم:

| کاربر در هر صفحه | 50 کاربر |
| :----------- | -----------: | 
| متوسط تعداد لاگین برای هر کاربر | 250 مرتبه |
|کل رکوردهای لاگین کاربران | 12500 رکورد|

ما اکنون در حال بارگیری 12500 رکورد login به سیستم هستیم تا بتوانیم آخرین ورود به سیستم را برای هر کاربر نشان دهیم.<br>
 این نه تنها حافظه را مصرف می کند، بلکه به محاسبات اضافی نیز نیاز دارد، زیرا هر رکورد باید به عنوان یک مدل Eloquent مقداردهی اولیه شود. و این یک مثال کاملا محافظه کارانه است. شما به راحتی می توانید با موقعیت های مشابهی مواجه شوید که منجر به بارگیری میلیون ها رکورد می شود.

 ## سیستم کش:
 ممکن است در این مرحله فکر کنید، "مشکل خاصی نیست، من فقط last_login_id را در جدول کاربران کش می کنم". مثلا:

```php
Schema::create('users', function (Blueprint $table) {
   $table->integer('last_login_id');
});
```
اکنون هنگامی که کاربر وارد می شود، رکورد ورود جدید را ایجاد می کنیم و سپس کلید خارجی last_login_id را در کاربر به روز می کنیم. سپس یک رابطه lastLogin در مدل کاربری خود ایجاد می کنیم و به صورت eager_load، آن رابطه را بارگذاری می کنیم.

```php
$users = User::with('lastLogin')->get();
```

و البته این یک راه حل کاملا معتبر است. اما توجه داشته باشید، caching اغلب به این سادگی نیست. بله،درست هست که در بعضی شرایط استفاده از denormalization  مناسب می باشد اما ما می توانیم بهتر عمل کنیم.

## معرفی subqueries:

راه دیگری هم برای حل این مشکل وجود دارد و آن هم با پرس و جوی فرعی (subqueries) است. پرسش‌های فرعی به ما امکان می‌دهند ستون‌های اضافی (ویژگی‌ها) را درست در database query داده خود انتخاب کنیم (users query در مثال ما). بیایید ببینیم چگونه می توانیم این کار را انجام دهیم.

```php
$users = User::query()
    ->addSelect(['last_login_at' => Login::select('created_at')
        ->whereColumn('user_id', 'users.id')
        ->latest()
        ->take(1)
    ])
    ->withCasts(['last_login_at' => 'datetime'])
    ->get();

@foreach ($users as $user)
    <tr>
        <td>{{ $user->name }}</td>
        <td>{{ $user->email }}</td>
        <td>
            @if ($user->last_login_at)
                {{ $user->last_login_at->format('M j, Y \a\t g:i a') }}
            @else
                Never
            @endif
        </td>
    </tr>
@endforeach
```

---
[^1]: ([Jonathan
Reinink](https://reinink.ca/articles/dynamic-relationships-in-laravel-using-subqueries))
