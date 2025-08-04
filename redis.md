# 📘 راهنمای جامع Redis (سطح مقدماتی تا متوسط)

> نوشته‌شده با کمک ChatGPT - مناسب برای استفاده در GitHub

---

## 🧠 Redis چیست؟

Redis یک دیتابیس در حافظه (In-Memory) و ساختار داده NoSQL است که برای سرعت بالا و عملکرد بلادرنگ طراحی شده. Redis از ساختارهای داده متنوعی پشتیبانی می‌کند و در کاربردهایی مثل کش، صف، شمارنده، پیام‌رسانی و ... بسیار محبوب است.

---

## 📂 مدیریت کلی دیتابیس

### 🔸 FLUSHDB

```redis
FLUSHDB
```

تمام کلیدهای دیتابیس فعلی را حذف می‌کند.

### 🔸 FLUSHALL

```redis
FLUSHALL
```

تمام کلیدهای همه دیتابیس‌های Redis را حذف می‌کند.

### 🔸 SELECT

```redis
SELECT index
```

انتخاب دیتابیس بر اساس شماره (۰ تا ۱۵ به‌صورت پیش‌فرض).

### 🔸 DBSIZE

```redis
DBSIZE
```

تعداد کل کلیدهای دیتابیس فعلی را برمی‌گرداند.

### 🔸 INFO

```redis
INFO
```

اطلاعات کامل از وضعیت Redis شامل memory, clients, stats و ...

### 🔸 PING

```redis
PING
```

بررسی اتصال با Redis. پاسخ پیش‌فرض: `PONG`

---

## 🧾 String

### 🔸 SET

```redis
SET key value
```

ذخیره یک مقدار متنی در کلید.

### 🔸 GET

```redis
GET key
```

خواندن مقدار کلید متنی.

### 🔸 INCR / DECR

```redis
INCR key
DECR key
```

افزایش یا کاهش عددی مقدار ذخیره‌شده.

### 🔸 APPEND

```redis
APPEND key value
```

اضافه‌کردن متن به انتهای مقدار موجود.

### 🔸 STRLEN

```redis
STRLEN key
```

طول مقدار رشته‌ای کلید را برمی‌گرداند.

---

## 📋 List

### 🔸 LPUSH / RPUSH

```redis
LPUSH key value [value ...]
RPUSH key value [value ...]
```

افزودن مقدار به ابتدای (LPUSH) یا انتهای (RPUSH) لیست.

### 🔸 LPOP / RPOP

```redis
LPOP key
RPOP key
```

حذف و دریافت اولین یا آخرین عنصر لیست.

### 🔸 LRANGE

```redis
LRANGE key start stop
```

بازیابی بخش یا تمام آیتم‌های لیست بر اساس بازه‌ای از اندیس.

### 🔸 LLEN

```redis
LLEN key
```

تعداد آیتم‌های لیست.

---

## 🧮 Set

### 🔸 SADD

```redis
SADD key member [member ...]
```

افزودن عضو/اعضا به مجموعه (بدون تکرار).

### 🔸 SMEMBERS

```redis
SMEMBERS key
```

دریافت تمام اعضای مجموعه.

### 🔸 SISMEMBER

```redis
SISMEMBER key member
```

بررسی وجود عضو در مجموعه (برمی‌گرداند ۰ یا ۱).

### 🔸 SREM

```redis
SREM key member [member ...]
```

حذف اعضا از مجموعه.

### 🔸 SCARD

```redis
SCARD key
```

تعداد اعضای مجموعه.

---

## 🏆 Sorted Set

### 🔸 ZADD

```redis
ZADD key score member [score member ...]
```

افزودن عضو با امتیاز (score) به مجموعه مرتب‌شده.

### 🔸 ZRANGE / ZREVRANGE

```redis
ZRANGE key start stop [WITHSCORES]
ZREVRANGE key start stop [WITHSCORES]
```

بازیابی اعضا با ترتیب صعودی/نزولی.

### 🔸 ZREM

```redis
ZREM key member [member ...]
```

حذف اعضا.

### 🔸 ZCARD

```redis
ZCARD key
```

تعداد اعضای Sorted Set.

---

## 🧰 Hash

### 🔸 HSET / HMSET

```redis
HSET key field value
HMSET key field1 value1 field2 value2 ...
```

تنظیم مقدار یک یا چند فیلد در Hash.

### 🔸 HGET / HGETALL

```redis
HGET key field
HGETALL key
```

خواندن مقدار یک فیلد یا همه فیلدها.

### 🔸 HDEL

```redis
HDEL key field [field ...]
```

حذف فیلدها از Hash.

### 🔸 HLEN

```redis
HLEN key
```

تعداد فیلدها در Hash.

---

## ⏱ مدیریت زمان و کلیدها

### 🔸 EXPIRE

```redis
EXPIRE key seconds
```

تنظیم زمان انقضا برای یک کلید.

### 🔸 TTL

```redis
TTL key
```

مانده‌زمان انقضای یک کلید را نشان می‌دهد.

### 🔸 KEYS

```redis
KEYS pattern
```

لیست کلیدهایی که با الگو مطابقت دارند (در محیط production پیشنهاد نمی‌شود).

### 🔸 SCAN

```redis
SCAN cursor [MATCH pattern] [COUNT count]
```

جایگزین بهتر برای KEYS جهت پیمایش کلیدها بدون فشار بالا.

---

## 🎯 Bitmaps (مقدماتی)

### 🔸 SETBIT / GETBIT

```redis
SETBIT key offset value
GETBIT key offset
```

تنظیم یا خواندن بیت در موقعیت خاص.

### 🔸 BITCOUNT

```redis
BITCOUNT key
```

شمارش بیت‌های ۱ در یک کلید.

---

## 🔢 HyperLogLog (مقدماتی)

### 🔸 PFADD

```redis
PFADD key element [element ...]
```

افزودن مقدار برای شمارش تقریبی اعضای یکتا.

### 🔸 PFCOUNT

```redis
PFCOUNT key
```

تعداد تقریبی اعضای یکتا.

---

## 📊 Streams (مقدمه)

### 🔸 XADD

```redis
XADD mystream * field1 value1 [field2 value2 ...]
```

افزودن رویداد به استریم.

### 🔸 XRANGE

```redis
XRANGE mystream - +
```

خواندن پیام‌های استریم.

---

## 📣 Pub/Sub (مختصر)

### 🔸 PUBLISH / SUBSCRIBE

```redis
PUBLISH channel message
SUBSCRIBE channel
```

ارسال و دریافت پیام از طریق کانال.

---

> 📌 برای اطلاعات بیشتر: redis.io/commands

---

