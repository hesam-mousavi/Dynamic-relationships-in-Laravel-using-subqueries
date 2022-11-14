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
پس اگر تعداد 50 کاربر داشته باشیم، برای اجاری برنامه بالا تعداد 51 کوئری ایجاد می شود.

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

---
[^1]: ([Jonathan
Reinink](https://reinink.ca/articles/dynamic-relationships-in-laravel-using-subqueries))
