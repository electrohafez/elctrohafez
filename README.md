# نیرو الکتریک ⚡

فروشگاه اینترنتی تجهیزات برق با **بک‌اند Node.js، دیتابیس PostgreSQL، پنل مدیریت** و اتصال امن به **n8n**؛ آمادهٔ GitHub و Railway.

- فروشگاه فارسی و راست‌چین، سبد خرید، ثبت سفارش (پرداخت در محل / پس از تماس)، پیگیری سفارش
- پنل مدیریت در `/admin`: سفارش‌ها، تغییر وضعیت، انبار، قیمت، افزودن کالا، رویدادهای n8n
- قیمت، موجودی و هزینهٔ ارسال فقط روی سرور حساب می‌شود؛ جلوگیری از فروش بیش از موجودی با قفل تراکنشی
- درخواست تکراری (دوبار کلیک، تلاش مجدد شبکه، وب‌هوک تکراری) سفارش یا کسر موجودی دوباره ایجاد نمی‌کند
- فقط یک وابستگی در زمان اجرا: `pg`

## معماری

```
مرورگر (public/)  ──REST JSON──►  Node.js (src/)  ──►  PostgreSQL
                                      │  outbox (همان تراکنش سفارش)
                                      ▼
                           worker با retry و dead-letter  ──HMAC──►  n8n
                                      ▲
                  POST /api/webhooks/n8n  ◄──HMAC──  n8n (پیک، پیامک، CRM)
```

| پوشه | محتوا |
|---|---|
| `src/server.js` | راه‌اندازی، worker، خاموشی امن (SIGTERM) |
| `src/app.js` | مسیریابی، هدرهای امنیتی، rate limit، request id، خطاهای ساختاریافته |
| `src/services/` | کاتالوگ، سبد، ثبت سفارش، وضعیت سفارش، انبار، outbox، رویدادهای ورودی n8n |
| `src/lib/` | اعتبارسنجی، امضای HMAC، نشست مدیر، محاسبهٔ قیمت، ماشین وضعیت سفارش |
| `migrations/` | اسکیمای دیتابیس (SQL) |
| `scripts/` | migrate، seed، smoke test، تست n8n |
| `.railway/railway.ts` | پیکربندی اختیاری Railway به‌صورت کد |
| `public/` | فروشگاه و پنل مدیریت (بدون build) |
| `test/` | تست واحد و تست یکپارچه با Postgres واقعی |

### مدل داده
`products` ← `variants` (SKU، قیمت) ← `inventory` (available / reserved / sold) · `carts`, `cart_items` · `orders` + `order_items` (عکس لحظهٔ خرید؛ تغییر قیمت بعدی سفارش قدیمی را عوض نمی‌کند) · `order_events` (تاریخچهٔ کامل با منبع) · `outbox` · `inbound_events` · `inventory_adjustments`

### چرخهٔ موجودی
ثبت سفارش: `available → reserved` · تأیید: `reserved → sold` · لغو: برگشت به `available` · ارسال و تحویل: بدون تغییر موجودی.

## API

| متد | مسیر | توضیح |
|---|---|---|
| GET | `/health` | زنده بودن سرویس |
| GET | `/ready` | اتصال دیتابیس + اجرای همهٔ migrationها |
| GET | `/api/categories` | دسته‌ها |
| GET | `/api/products?category=&q=&limit=&offset=` | کالاها با تنوع و موجودی |
| GET | `/api/products/:idOrSlug` | یک کالا |
| POST | `/api/cart` | `{ items: [{ variantId, qty }] }` |
| GET / PATCH | `/api/cart/:id` | خواندن / جایگزینی اقلام؛ جمع‌ها از سرور |
| POST | `/api/checkout` | هدر `Idempotency-Key` الزامی؛ `{ cartId, customer: { name, phone, city, address, postalCode?, note? }, paymentMethod: "cod" \| "call" }` → `201` و کد پیگیری |
| GET | `/api/orders/:code?phone=` | وضعیت عمومی سفارش (بدون نشانی) |
| POST | `/api/webhooks/n8n` | رویداد امضاشده از n8n |
| — | `/api/admin/*` | ورود، آمار، سفارش‌ها، تغییر وضعیت، انبار، قیمت، کالا، outbox |

خطاها همیشه این شکل را دارند: `{ "error": { "code", "message", "details", "requestId" } }`. قرارداد کامل n8n در [`docs/n8n.md`](docs/n8n.md).

## اجرای محلی

پیش‌نیاز: Node.js 20.10+ و Docker (یا هر PostgreSQL 13+).

```bash
npm install                 # package-lock.json ساخته می‌شود؛ آن را commit کنید
docker compose up -d        # PostgreSQL محلی
cp .env.example .env        # ADMIN_PASSWORD و SESSION_SECRET را پر کنید
npm run migrate
npm run seed                # ۱۲ کالای نمونه
npm run dev                 # http://localhost:3000  و  http://localhost:3000/admin
```

ساخت رمز تصادفی: `node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"`

تست‌ها:
```bash
npm run lint && npm run typecheck && npm run test:unit
DATABASE_URL=postgres://app:app@localhost:5432/niroo npm run test:integration
```

## انتشار روی GitHub

```bash
git init && git add . && git commit -m "Niroo Electric store"
git branch -M main
git remote add origin https://github.com/<USER>/niroo-electric.git
git push -u origin main
```
فایل `.env` در `.gitignore` است و هیچ رازی وارد مخزن نمی‌شود. GitHub Actions (`.github/workflows/ci.yml`) روی هر push و PR: lint، بررسی syntax، تست واحد، migrate و seed روی Postgres واقعی، تست یکپارچه و smoke test را اجرا می‌کند.

## لانچ روی Railway

1. **New Project → Deploy from GitHub repo** و مخزن را انتخاب کنید (شاخهٔ `main`). Railway خودش Node.js را تشخیص می‌دهد و `npm start` را اجرا می‌کند.
2. در همان پروژه **+ New → Database → PostgreSQL** اضافه کنید.
3. در سرویس اپ، تب **Variables** (رازها را **Seal** کنید):

   | متغیر | مقدار |
   |---|---|
   | `DATABASE_URL` | `${{Postgres.DATABASE_URL}}` (Reference Variable) |
   | `NODE_ENV` | `production` |
   | `TRUST_PROXY` | `true` |
   | `ADMIN_PASSWORD` | رمز قوی، حداقل ۱۲ نویسه |
   | `SESSION_SECRET` | ۶۴ نویسهٔ تصادفی |
   | `N8N_WEBHOOK_URL` | آدرس https وب‌هوک n8n (تا آماده شدن n8n خالی بگذارید؛ سفارش‌ها در صف می‌مانند) |
   | `N8N_WEBHOOK_SECRET` | راز تصادفی، همان مقدار در n8n |
   | `N8N_INBOUND_SECRET` | راز تصادفی دوم، برای n8n ← فروشگاه |
   | `SHIPPING_FEE`, `FREE_SHIPPING_OVER` | اختیاری، به تومان |

4. تب **Settings → Deploy**:
   - **Pre-deploy Command:** `npm run migrate` (اگر migration شکست بخورد، نسخهٔ جدید بالا نمی‌آید)
   - **Healthcheck Path:** `/ready`
   - **Restart Policy:** On Failure
5. در **Settings → Source** گزینهٔ **Wait for CI** را روشن کنید تا تا وقتی GitHub Actions سبز نشده، deploy انجام نشود.
6. **Settings → Networking → Generate Domain** (یا دامنهٔ خودتان).
7. کالاهای نمونه (اختیاری): `railway run npm run seed` یا از پنل مدیریت کالا اضافه کنید.

> **مدیریت با کد (اختیاری):** Railway فایل `railway.json` را منسوخ کرده و از ۱ دسامبر ۲۰۲۶ نمی‌خواند. به‌جایش `.railway/railway.ts` در مخزن هست: `npm i -D railway`، مقدار `REPO` را عوض کنید، بعد `railway login && railway link && railway config plan && railway config apply`. رازها با `preserve()` در کد نمی‌آیند و همان مقادیر داشبورد حفظ می‌شوند.

### بعد از deploy (قبل از قبول سفارش واقعی)
```bash
BASE_URL=https://<your-app>.up.railway.app npm run smoke
N8N_WEBHOOK_URL=… N8N_WEBHOOK_SECRET=… npm run n8n:test
```
سپس یک سفارش آزمایشی کامل بزنید، در `/admin` تأیید و لغو کنید و مطمئن شوید رویدادها در تب «رویدادهای n8n» در وضعیت «ارسال‌شده» هستند.

## چک‌لیست تحویل
- [ ] مرور کالا و انتخاب تنوع کار می‌کند
- [ ] سبد و checkout جمع‌ها را روی سرور حساب می‌کنند
- [ ] موجودی اتمیک رزرو می‌شود و فروش بیش از موجودی ممکن نیست (تست هم‌زمانی در CI)
- [ ] checkout و وب‌هوک تکراری بی‌اثرند
- [ ] وضعیت سفارش قابل پیگیری است
- [ ] n8n رویداد مستند را می‌گیرد و تلاش‌های مجدد در پنل دیده می‌شوند
- [ ] هیچ رازی در مخزن و فایل‌های مرورگر نیست
- [ ] CI سبز است و Railway از شاخهٔ `main` deploy می‌کند
- [ ] `/health` و `/ready` در production سالم‌اند

## بازگشت (Rollback) و بازیابی
- **کد:** در Railway → Deployments روی deploy سالم قبلی **Redeploy** بزنید. migrationها فقط افزایشی‌اند؛ migration جدید را طوری بنویسید که با نسخهٔ قبلی کد سازگار باشد.
- **دیتابیس:** Backups سرویس PostgreSQL در Railway را فعال کنید و قبل از migration بزرگ یک backup دستی بگیرید.
- **n8n از دسترس خارج شد:** سفارش‌ها همچنان ثبت می‌شوند؛ رویدادها در outbox صف می‌شوند و پس از برگشت n8n خودکار ارسال می‌شوند. رویدادهای `dead` را از پنل «ارسال دوباره» کنید.
- **عیب‌یابی:** هر پاسخ هدر `X-Request-Id` دارد و همان شناسه در لاگ‌های JSON سرور (Railway → Logs) هست.

## نکات مقیاس و گام بعدی
- rate limit در حافظه است؛ اگر بیش از یک replica اجرا کردید آن را به Redis منتقل کنید. worker outbox با `FOR UPDATE SKIP LOCKED` روی چند replica امن است.
- پرداخت آنلاین (مثلاً زرین‌پال) را به‌صورت مرحلهٔ بعد اضافه کنید: callback درگاه را مثل وب‌هوک n8n با idempotency و ثبت در `order_events` پیاده کنید.
- فونت Vazirmatn از Google Fonts لود می‌شود؛ برای سرعت بهتر در ایران فایل woff2 را در `public/fonts` بگذارید و `font-src 'self'` کافی است.
