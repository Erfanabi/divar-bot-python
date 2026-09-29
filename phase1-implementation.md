# اجرای فاز ۱ — ربات آگهی خودرو دیوار (Python)

هدف این سند: مسیر گام‌به‌گام ساخت نسخه اولیه (MVP) با کد، شامل **فیلتر + اعلان آگهی جدید + اعلان کاهش قیمت**.

---

## ۱. دامنه فاز ۱

### داخل دامنه
- ساخت، ویرایش، توقف و حذف فیلتر (Mini App تلگرام)
- ارسال اولیه (Backfill): پیدا کردن همه آگهی‌های بازه و نمایش ۱۰ تا ۱۰ تا
- Crawler هر ۱۰ دقیقه و ارسال آگهی‌های جدید
- حذف تکراری قطعی با جدول `deliveries`
- تشخیص تغییر آگهی: اعلان کاهش قیمت (reply) + ویرایش بی‌صدای پیام قبلی + علامت‌گذاری آگهی حذف‌شده
- دستورهای ادمین: `/stats`، `/crawl_now`، `/extend`، `/broadcast`
- استقرار با Docker Compose

### خارج از دامنه (فازهای بعد)
- درگاه پرداخت (در فاز ۱ همه کاربران رایگان یا تمدید دستی با `/extend`)
- `recheck` دوره‌ای، ساعت سکوت، پلن‌های سرعت‌دار، تحلیل قیمت بازار
- مهاجرت از n8n (فقط اگر کاربر فعالی روی آن دارید)

### پیشنهاد اختیاری برای MVP
فیلتر «فروشنده شخصی / نمایشگاه» برای بنگاه‌دارها ارزش بالایی دارد. اگر فیلد آن در پاسخ دیوار در دسترس بود، در مرحله M3 به Matcher اضافه کنید.

---

## ۲. استک

| بخش | ابزار |
|---|---|
| زبان | Python 3.12 |
| ربات | aiogram 3 |
| HTTP | httpx (async) + tenacity |
| زمان‌بندی | APScheduler (AsyncIOScheduler) |
| دیتابیس | PostgreSQL 16 + SQLAlchemy 2 (async) + asyncpg |
| مهاجرت | Alembic |
| تنظیمات | pydantic-settings |
| Mini App | aiohttp (سرو فایل استاتیک) |
| تست | pytest + pytest-asyncio + respx |
| استقرار | Docker + docker compose |

---

## ۳. ساختار پروژه

```
divar-bot-python/
├── app/
│   ├── __main__.py          # bot + scheduler + web server
│   ├── config.py
│   ├── db/        base.py · models.py · repo.py
│   ├── divar/     client.py · parser.py · catalog.py
│   ├── core/      matcher.py · planner.py · formatter.py · diff.py
│   ├── services/  crawler.py · backfill.py · sender.py · changes.py
│   ├── bot/       handlers/ · keyboards.py
│   └── web/       server.py
├── webapp/        filter.html · catalog.json
├── alembic/
├── tests/         fixtures/ · test_*.py
├── Dockerfile
├── docker-compose.yml
├── pyproject.toml
└── .env.example
```

اصل طراحی: **Matcher، Planner و Diff توابع خالص** (بدون I/O)، **DivarClient تنها نقطه تماس با دیوار** و **Sender تنها نقطه ارسال پیام**.

---

## ۴. مراحل اجرا

هر مرحله یک «معیار پذیرش» دارد. تا معیار برآورده نشده، به مرحله بعد نروید.

### M0 — اسکلت و زیرساخت

**کارها**
1. `pyproject.toml` با وابستگی‌ها، `.env.example`، `.gitignore` (شامل `.env`).
2. `config.py` با pydantic-settings.
3. `docker-compose.yml` با دو سرویس `app` و `db` (Postgres).
4. راه‌اندازی Alembic با engine async.

```python
# app/config.py
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env")

    bot_token: str
    database_url: str
    admin_ids: list[int] = []
    webapp_base_url: str = ""

    crawl_interval_minutes: int = 10
    crawl_max_pages: int = 3
    backfill_max_pages: int = 30
    summary_list_size: int = 15

    price_drop_min_percent: float = 1.0
    price_drop_min_amount: int = 10_000_000   # تومان
    change_alert_cooldown_hours: int = 6

settings = Settings()
```

**معیار پذیرش:** `docker compose up` بالا می‌آید، `alembic upgrade head` بدون خطا اجرا می‌شود، و ربات به `/start` جواب ساده می‌دهد.

---

### M1 — مدل داده

جدول‌های لازم: `users`، `filters`، `posts`، `deliveries`، `post_changes`.

**نکات کلیدی**
- `users.chat_id` از نوع bigint و یکتا.
- `filters`: `user_id`، `name`، `city_id`، `brands` (آرایه)، بازه سال و کارکرد، `gearbox`، `keywords`، `max_age_days`، `active`، `change_alerts` (`price_drop` | `all` | `none`).
- `posts`: `token` (یکتا)، عنوان، قیمت، سال، کارکرد، شهر، `published_at`، `first_seen_at`، `last_seen_at`، `fingerprint`، `details_fetched`، `status` (`active` | `deleted`).
- `deliveries`: `user_id`، `post_token`، `filter_id`، `status` (`queued` | `sent` | `listed`)، `message_id`، `sent_price`، `last_notified_at`.
- **قید یکتای `(user_id, post_token)` روی `deliveries`** ستون فقرات ضد تکراری است.
- `post_changes`: `post_token`، `field`، `old_value`، `new_value`، `kind`، `detected_at`.

```python
# app/db/models.py (بخش deliveries)
class Delivery(Base):
    __tablename__ = "deliveries"
    __table_args__ = (UniqueConstraint("user_id", "post_token", name="uq_delivery_user_post"),)

    id: Mapped[int] = mapped_column(primary_key=True)
    user_id: Mapped[int] = mapped_column(ForeignKey("users.id"))
    post_token: Mapped[str] = mapped_column(ForeignKey("posts.token"))
    filter_id: Mapped[int] = mapped_column(ForeignKey("filters.id"))
    status: Mapped[str] = mapped_column(default="queued")   # queued | sent | listed
    message_id: Mapped[int | None]
    sent_price: Mapped[int | None]
    last_notified_at: Mapped[datetime | None]
```

الگوی ثبت ارسال (فقط ردیف‌های تازه به Sender می‌روند):

```python
stmt = (
    pg_insert(Delivery)
    .values(rows)
    .on_conflict_do_nothing(constraint="uq_delivery_user_post")
    .returning(Delivery.id, Delivery.user_id, Delivery.post_token)
)
new_rows = (await session.execute(stmt)).all()
```

**معیار پذیرش:** مهاجرت اولیه ساخته می‌شود و یک تست ساده نشان می‌دهد درج دوباره همان `(user, post)` ردیف جدید نمی‌سازد.

---

### M2 — DivarClient و Parser

**کارها**
1. `client.py` با یک `httpx.AsyncClient` مشترک، فاصله بین درخواست‌ها + jitter، **بدون درخواست موازی**، و retry با tenacity.
2. متدها: `search(body, pagination_data=None)` و `get_post(token)`.
3. `parser.py`: تبدیل پاسخ خام به دیتاکلاس `PostData`. فیلد ناموجود نباید کرش کند.
4. `catalog.py`: نگاشت شهر و برند فارسی به کد/کلید API (مثلاً مشهد=`3`، دنا=`Dena`) و تولید `catalog.json` برای Mini App.

endpointها (غیررسمی):

| کار | درخواست |
|---|---|
| جستجو | `POST https://api.divar.ir/v8/postlist/w/search` |
| صفحه بعد | همان درخواست + `pagination_data` از پاسخ قبلی |
| جزئیات | `GET https://api.divar.ir/v8/posts-v2/web/{token}` |
| لینک آگهی | `https://divar.ir/v/{token}` |

**قبل از نوشتن parser:** چند پاسخ واقعی را در `tests/fixtures/` ذخیره کنید. parser را روی همین‌ها بنویسید و تست کنید.

**معیار پذیرش:** یک اسکریپت دستی برای «دنا، مشهد» ۲۵ آگهی معتبر با قیمت و سال برمی‌گرداند و تست parser با fixtureها سبز است.

---

### M3 — Matcher و Planner (توابع خالص)

**Matcher دو مرحله‌ای**
- `pre_match(post_list_data, filter)`: فقط با داده‌های لیست (برند، شهر، قیمت، کارکرد تقریبی). ارزان است و برای شمارش اولیه استفاده می‌شود.
- `full_match(post_full, filter)`: با جزئیات (گیربکس، سال دقیق، کلمات توضیحات، تاریخ انتشار).

```python
# app/core/matcher.py
def pre_match(post: PostListItem, f: Filter) -> bool: ...
def full_match(post: PostFull, f: Filter) -> bool: ...
```

**Planner:** فیلترهای فعال را گروه‌بندی می‌کند تا برای چند فیلتر مشابه (مثلاً یک شهر و یک برند) فقط یک جستجو به دیوار زده شود، و فیلتر کاربران منقضی یا بلاک‌شده را حذف می‌کند.

**تست‌های واحد (اولویت بالا)**
- هر شرط Matcher جداگانه (برند، سال، کارکرد، کلمات، گیربکس)
- مقدار خالی یا `None` یعنی «بدون محدودیت»
- Planner: گروه‌بندی درست، اجتماع بازه‌ها، حذف کاربر منقضی

**معیار پذیرش:** پوشش تست Matcher و Planner کامل و سبز.

---

### M4 — Sender

تنها نقطه ارسال پیام. یک `asyncio.Queue` و یک worker.

**مسئولیت‌ها**
- rate limit: حدود ۱ پیام در ثانیه برای هر چت و سقف کلی تلگرام
- مدیریت `429` (`retry_after`) و `TelegramForbiddenError` (کاربر ربات را بلاک کرده ← `users.is_blocked = true` و توقف ارسال)
- سه عملیات: `send`، `reply`، `edit`
- ذخیره `message_id` در `deliveries` بعد از ارسال موفق

**Formatter:** کارت آگهی (عنوان، قیمت، سال، کارکرد، زمان، دکمه لینک). حتماً `html.escape` روی همه متن‌های دیوار، و رعایت سقف طول caption.

**معیار پذیرش:** ارسال ۵۰ پیام پشت‌سرهم بدون خطای 429؛ بلاک کردن ربات باعث `is_blocked=true` و بدون خطای تکراری در لاگ می‌شود.

---

### M5 — هندلرهای ربات و Mini App

**هندلرها**
- `/start`: ساخت کاربر، اشتراک آزمایشی (یا نامحدود در فاز ۱)، کیبورد اصلی
- «➕ فیلتر جدید»: باز کردن Mini App با `WebAppInfo`
- دریافت `web_app_data`: اعتبارسنجی ← درج `filters` ← پیام «✅ فیلتر ذخیره شد» + خلاصه ← شروع Backfill با `asyncio.create_task`
- «📋 فیلترهای من»: لیست با نشان 🟢 / ⏸ و دستورهای `/edit_ID`، `/pause_ID`، `/resume_ID`، `/del_ID`
- گزینه «اعلان تغییر آگهی‌ها» (`change_alerts`) در فرم Mini App

**Mini App:** `webapp/filter.html` (فارسی، RTL) که `catalog.json` را می‌خواند و با `tg.sendData()` نتیجه را می‌فرستد. با aiohttp سرو می‌شود. توجه: Mini App برای باز شدن در تلگرام به **HTTPS** نیاز دارد (در توسعه می‌توانید از یک tunnel استفاده کنید).

**اعتبارسنجی سمت سرور را جدی بگیرید:** هر داده‌ای که از Mini App می‌آید نامطمئن است (کد شهر و برند فقط از `catalog`، بازه‌ها عددی و منطقی).

**معیار پذیرش:** از `/start` تا ذخیره فیلتر و ویرایش/توقف/حذف آن، بدون خطا کار می‌کند.

---

### M6 — Backfill

```python
async def run_backfill(filter_id: int) -> None: ...
async def show_next_page(user_id: int, filter_id: int) -> None: ...
async def show_summary_list(user_id: int, filter_id: int) -> None: ...
```

**جریان `run_backfill`**
1. ارسال پیام «🔎 در حال جستجو…» و ویرایش آن با پیشرفت صفحات.
2. صفحه‌زنی تا رسیدن به تاریخ مرز (`now − max_age_days`) یا `backfill_max_pages`.
3. `pre_match` روی همه آگهی‌ها و upsert در `posts`.
4. درج همه کاندیدها در `deliveries` با `status='queued'` (مرتب از جدیدترین).
5. پیام جمع‌بندی «حدود N آگهی» + ۱۰ کارت اول (جزئیات فقط برای این ۱۰ تا) + دکمه‌های «۱۰ آگهی بعدی» و «فهرست خلاصه».

**جزئیات مهم**
- جزئیات «تنبل» گرفته می‌شود: فقط وقتی نوبت نمایش می‌رسد. آگهی‌ای که `full_match` را رد کند حذف و از صف جایگزین می‌شود تا صفحه ۱۰ تایی پر شود.
- کارت همیشه از آخرین نسخه `posts` ساخته می‌شود.
- یک `asyncio.Lock` برای هر `user_id` تا دو Backfill همزمان اجرا نشود.
- آگهی‌های `queued` جزو «دیده‌شده» حساب می‌شوند.

**معیار پذیرش:** برای فیلتری با حدود ۱۰۰ آگهی، دکمه «۱۰ آگهی بعدی» تا پایان صف کار می‌کند و در آخر پیام «✅ همه آگهی‌ها نمایش داده شد» می‌آید.

---

### M7 — Crawler، Diff و اعلان کاهش قیمت

**Crawler (`run_crawl`)**

```
1. groups = planner.plan(فیلترهای فعال)
2. برای هر group (با try/except جدا):
     a. صفحه اول جستجو؛ اگر همه آگهی‌های صفحه «جدید» بودند و صفحه بعد هست ← صفحه بعد (تا crawl_max_pages)
     b. upsert در posts (last_seen_at)
     c. آگهی قبلاً دیده‌شده که قیمت/عنوان لیستش فرق کرده ← get_post ← diff ← post_changes ← changes.handle_changes()
     d. pre_match برای هر آگهی و هر فیلتر گروه
     e. آگهی‌های کاندید با details_fetched=false ← get_post
     f. full_match ← لیست (user, post, filter)
3. برای هر کاربر هر آگهی فقط یک بار (حتی اگر چند فیلتر جور شد)
4. INSERT INTO deliveries ... ON CONFLICT DO NOTHING RETURNING ← فقط ردیف‌های تازه به sender
```

- job با `max_instances=1` و `coalesce=True`.
- **Warm-up:** اگر جدول `posts` خالی است، اجرای اول فقط ثبت می‌کند و چیزی نمی‌فرستد.

**Diff (`core/diff.py`)**

```python
def fingerprint(post) -> str:
    # هش از: قیمت، کارکرد، عنوان، توضیحات نرمال‌شده، تعداد عکس
    ...

def diff(old, new) -> list[Change]:
    # kind: price_drop | price_up | content | deleted
    ...
```

اگر fingerprint عوض نشده باشد (مثلاً نردبان بدون تغییر)، کاری انجام نمی‌شود. اگر `get_post` پاسخ 404 یا «حذف‌شده» بدهد، `status='deleted'`.

**رفتار هر تغییر (`services/changes.py`)**

| تغییر | رفتار |
|---|---|
| کاهش قیمت بیش از آستانه | reply جدید «📉 کاهش قیمت» + ویرایش بی‌صدای پیام قبلی |
| کاهش قیمت کمتر از آستانه | فقط ویرایش بی‌صدا |
| افزایش قیمت / تغییر توضیحات یا کارکرد | فقط ویرایش بی‌صدا |
| آگهی حذف یا فروخته شد | ویرایش بی‌صدا: «❌ دیگر موجود نیست» |
| آگهی بعد از تغییر وارد فیلتر شد | مثل آگهی جدید، با برچسب «قیمت کاهش یافت و وارد فیلتر شما شد» |
| نردبان بدون تغییر | هیچ کاری |

**ضد اسپم**

```python
def price_drop_significant(sent_price: int, new_price: int) -> bool:
    drop = sent_price - new_price
    if drop <= 0:
        return False
    return (
        drop / sent_price * 100 >= settings.price_drop_min_percent
        or drop >= settings.price_drop_min_amount
    )
```

- مبنای مقایسه `sent_price` است (قیمتی که کاربر دیده)، نه قیمت قبلی.
- حداکثر یک اعلان برای هر آگهی و هر کاربر در `change_alert_cooldown_hours`.
- بعد از هر اعلان، `sent_price` و `last_notified_at` به‌روز می‌شوند.
- برای ردیف‌های `listed` (بدون `message_id`) اعلان کاهش قیمت مستقل و بدون reply می‌رود.

**معیار پذیرش:** تست‌های `test_crawler.py` و `test_diff.py` سبز؛ در تست دستی، بالا بردن قیمت در دیتابیس و اجرای `/crawl_now` ← reply «کاهش قیمت»؛ تکرار در کمتر از ۶ ساعت ← بدون اعلان جدید.

---

### M8 — ادمین و مشاهده‌پذیری

- دستورها (فقط `ADMIN_IDS`): `/stats`، `/user <id>`، `/extend <id> <days>`، `/broadcast <متن>`، `/crawl_now`
- هر اجرای Crawler یک خط لاگ خلاصه می‌نویسد:
  `groups=8 pages=11 new_posts=5 details=5 matched=7 sent=7 changed=3 price_drops=1 duration=38s`
- هشدار به ادمین اگر دو اجرای پشت‌سرهم Crawler شکست خورد، یا پارسر برای بیش از ۵۰٪ آگهی‌ها قیمت/سال پیدا نکرد (نشانه تغییر API دیوار).
- اختیاری: endpoint `/health` برای UptimeRobot.

---

### M9 — استقرار

1. سرور با Docker و دامنه‌ای با HTTPS (برای Mini App). Webhook یا Polling؛ در توسعه Polling.
2. `docker compose up -d` با volume برای Postgres.
3. پشتیبان‌گیری روزانه با `pg_dump`.
4. `BOT_TOKEN` و `.env` هرگز در git نباشند. اگر توکن لو رفت، در BotFather `/revoke`.
5. اجرای سناریوی تست دستی (بخش ۶) قبل از دعوت کاربران.

---

## ۵. ترتیب پیشنهادی و زمان‌بندی تقریبی

این ارقام تخمین کلی برای یک نفر است و به تجربه شما با aiogram و async بستگی دارد.

| هفته | مراحل |
|---|---|
| ۱ | M0، M1، M2 |
| ۲ | M3، M4، M5 |
| ۳ | M6، M7 |
| ۴ | M8، M9، تست با کاربر واقعی |

---

## ۶. تست

**تست واحد** (`pytest -q`، بدون اینترنت)

| فایل | چه چیزی را می‌سنجد |
|---|---|
| `test_matcher.py` | شرط‌های `pre_match` و `full_match` |
| `test_planner.py` | گروه‌بندی، حذف کاربر منقضی و بلاک |
| `test_parser.py` | fixtureهای واقعی؛ فیلد ناموجود کرش نکند |
| `test_diff.py` | fingerprint ثابت برای نردبان؛ تشخیص `price_drop` / `price_up` / `content` / `deleted` |
| `test_backfill.py` | صفحه‌زنی تا تاریخ مرز؛ صف `queued`؛ پر شدن صفحه ۱۰ تایی |
| `test_crawler.py` | یک آگهی دو بار دیده شود ← یک بار ارسال؛ چند فیلتر یک کاربر ← یک پیام؛ warm-up؛ آگهی `queued` دوباره ارسال نشود؛ کاهش قیمت ← reply؛ cooldown |
| `test_formatter.py` | escape شدن HTML، طول caption |

پاسخ‌های دیوار با **respx** mock می‌شوند.

**سناریوی تست دستی**

1. `/start` ← پیام خوش‌آمد و کیبورد
2. «➕ فیلتر جدید» ← Mini App باز می‌شود
3. ذخیره فیلتر (مثلاً دنا، مشهد، اتوماتیک، ۱۶ روز) ← تأیید + خلاصه
4. Backfill ← پیام پیشرفت، جمع‌بندی، ۱۰ کارت
5. «۱۰ آگهی بعدی» تا پایان صف
6. `/crawl_now` ← آگهی‌های Backfill دوباره ارسال نشوند
7. شبیه‌سازی کاهش قیمت در دیتابیس ← reply «📉 کاهش قیمت»؛ تکرار در کمتر از ۶ ساعت ← بدون اعلان
8. `/pause_…` و `/resume_…`، `/edit_…`، `/del_…`
9. بلاک کردن ربات و `/crawl_now` ← `is_blocked=true` بدون خطای تکراری
10. ری‌استارت کانتینر ← هیچ آگهی تکراری ارسال نشود

---

## ۷. ریسک‌ها

| ریسک | مقابله |
|---|---|
| API غیررسمی دیوار تغییر می‌کند یا IP محدود می‌شود | همه تماس‌ها فقط در `divar/`؛ هشدار پارسر؛ فاصله و jitter؛ کش جزئیات؛ بررسی پلتفرم رسمی Kenar برای نسخه تجاری |
| کاهش قیمت فقط برای آگهی‌هایی که دوباره در نتایج ظاهر می‌شوند دیده می‌شود | محدودیت شناخته‌شده فاز ۱؛ `recheck` در فاز ۲ |
| ارسال زیاد و خطای 429 تلگرام | صف Sender با rate limit |
| Mini App بدون HTTPS باز نمی‌شود | دامنه و گواهی از همان ابتدای M5 |
| داده نامعتبر از Mini App | اعتبارسنجی سمت سرور با `catalog` |

---

## ۸. تعریف «تمام شدن فاز ۱»

- [ ] کاربر می‌تواند فیلتر بسازد، ویرایش، متوقف و حذف کند
- [ ] Backfill همه آگهی‌های بازه را پیدا می‌کند و صفحه‌به‌صفحه نشان می‌دهد
- [ ] هر ۱۰ دقیقه آگهی‌های جدید بدون تکرار ارسال می‌شوند (حتی بعد از ری‌استارت)
- [ ] کاهش قیمت با reply اعلان می‌شود و ضد اسپم (آستانه + cooldown) کار می‌کند
- [ ] آگهی حذف‌شده یا تغییرکرده، پیام قبلی را بی‌صدا به‌روز می‌کند
- [ ] دستورهای ادمین و لاگ خلاصه Crawler کار می‌کنند
- [ ] تست‌های واحد سبز و سناریوی دستی کامل پاس شده
- [ ] روی سرور با HTTPS مستقر است و پشتیبان‌گیری فعال است
- [ ] حداقل ۱۰ تا ۲۰ کاربر واقعی (ترجیحاً بنگاه‌دار) دو هفته از آن استفاده کرده‌اند و بازخوردشان ثبت شده است
