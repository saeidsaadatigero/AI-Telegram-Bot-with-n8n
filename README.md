# ربات تلگرام هوشمند با n8n و OpenAI

یک نمونه عملی و کامل از ساخت **AI Agent** با پلتفرم اتوماسیون **n8n**؛ رباتی که پیام‌های کاربران در تلگرام را دریافت می‌کند، آن‌ها را با مدل زبانی OpenAI پردازش می‌کند و پاسخ فارسی و کوتاه بازمی‌گرداند — بدون نوشتن حتی یک خط کد سمت سرور.

> این پروژه به عنوان نمونه‌کار آموزشی برای معرفی مفاهیم Workflow، Trigger، Credential و مدیریت خطا در n8n تهیه شده است.

**مخزن گیت‌هاب:** https://github.com/saeidsaadatigero/AI-Telegram-Bot-with-n8n

---

## فهرست مطالب

- [معماری سیستم](#معماری-سیستم)
- [پیش‌نیازها](#پیشنیازها)
- [مراحل راه‌اندازی](#مراحل-راهاندازی)
- [نحوه import کردن Workflow](#نحوه-import-کردن-workflow)
- [تنظیم Credential ها](#تنظیم-credential-ها)
- [توضیح گره‌ها](#توضیح-گرهها)
- [نکات امنیتی](#نکات-امنیتی)
- [رفع اشکال](#رفع-اشکال)
- [ساختار پروژه](#ساختار-پروژه)

---

## معماری سیستم

مسیر یک پیام از لحظه ارسال تا دریافت پاسخ:

```
┌──────────────┐   پیام متنی    ┌───────────────────────────────────────┐
│   کاربر      │ ─────────────► │        Telegram Bot API               │
│  در تلگرام   │ ◄───────────── │  (Webhook به سمت n8n ارسال می‌کند)     │
└──────────────┘    پاسخ AI     └──────────────────┬────────────────────┘
                                                   │ ① Webhook
                                                   ▼
                            ┌──────────────────────────────────────────┐
                            │            n8n (Docker :5678)            │
                            │                                          │
                            │  ① Telegram Trigger                      │
                            │        │ message.text                    │
                            │        ▼                                 │
                            │  ② AI Agent ◄── OpenAI Chat Model        │
                            │        │            (gpt-4o-mini)        │
                            │   ┌────┴────┐                            │
                            │   ▼ موفق    ▼ خطا                        │
                            │  ③ Send   ⑤ Notify User On Failure       │
                            │    Reply                                 │
                            │                                          │
                            │  ④ Error Trigger ──► Alert Admin         │
                            └──────────────────┬───────────────────────┘
                                               │ ② درخواست HTTPS
                                               ▼
                                    ┌────────────────────┐
                                    │    OpenAI API      │
                                    │   (Chat Completion)│
                                    └────────────────────┘
```

خلاصه جریان داده:

`تلگرام → n8n (Telegram Trigger) → OpenAI (AI Agent) → n8n (Send Reply) → تلگرام`

جزئیات بیشتر معماری در فایل [`docs/architecture.md`](docs/architecture.md) آمده است.

---

## پیش‌نیازها

| مورد | توضیح | الزامی؟ |
|---|---|---|
| **Docker Desktop** | برای اجرای n8n روی ویندوز (با بک‌اند WSL2) | بله |
| **حساب OpenAI** | برای دریافت `OPENAI_API_KEY` از [platform.openai.com](https://platform.openai.com/api-keys) با اعتبار فعال | بله |
| **ربات تلگرام** | ساخت ربات از طریق [@BotFather](https://t.me/BotFather) و دریافت Token | بله |
| **Git** | برای کلون کردن مخزن | بله |
| **Node.js 18+** | فقط اگر بخواهید n8n را بدون Docker با `npx n8n` اجرا کنید | اختیاری |

---

## مراحل راه‌اندازی

### ۱) کلون کردن مخزن

```bash
git clone https://github.com/saeidsaadatigero/AI-Telegram-Bot-with-n8n.git
cd AI-Telegram-Bot-with-n8n
```

### ۲) ساخت فایل `.env`

فایل نمونه را کپی کنید و مقادیر واقعی را در آن قرار دهید:

```bash
# ویندوز (PowerShell)
Copy-Item .env.example .env

# لینوکس / مک
cp .env.example .env
```

سپس `.env` را باز کرده و مقادیر زیر را پر کنید:

```ini
OPENAI_API_KEY=sk-proj-...            # کلید واقعی OpenAI
TELEGRAM_BOT_TOKEN=1234567:AA...      # توکن دریافتی از BotFather
TELEGRAM_ADMIN_CHAT_ID=123456789      # شناسه عددی چت شما برای دریافت هشدار خطا
```

> برای پیدا کردن شناسه عددی چت خود، به ربات [@userinfobot](https://t.me/userinfobot) پیام بدهید.

### ۳) اجرای n8n

```bash
docker compose up -d
```

بررسی وضعیت و لاگ‌ها:

```bash
docker compose ps
docker compose logs -f n8n
```

### ۴) باز کردن پنل مدیریت

در مرورگر به آدرس زیر بروید:

```
http://localhost:5678
```

در اولین اجرا، n8n از شما می‌خواهد یک حساب مدیر محلی (ایمیل و رمز عبور) بسازید.

### ۵) import کردن Workflow

فایل `workflow/telegram_ai_bot.json` را طبق [بخش بعدی](#نحوه-import-کردن-workflow) وارد کنید.

### ۶) تنظیم Credential ها

طبق [بخش تنظیم Credential ها](#تنظیم-credential-ها) دو اعتبارنامه `openai_api` و `telegram_bot` را بسازید.

### ۷) فعال‌سازی و تست

۱. کلید **Active** در گوشه بالا سمت راست Workflow را روشن کنید تا Webhook تلگرام ثبت شود.
۲. در تلگرام به ربات خود پیام بدهید، مثلاً: «سلام، n8n چیست؟»
۳. پاسخ باید ظرف چند ثانیه برسد.
۴. برای مشاهده جزئیات اجرا، به تب **Executions** در n8n بروید.

> **نکته مهم درباره Webhook:** تلگرام برای ارسال Webhook باید بتواند از اینترنت به n8n شما دسترسی داشته باشد. روی `localhost` این امکان وجود ندارد. برای تست محلی یکی از این دو راه را انتخاب کنید:
> - **حالت تست:** روی دکمه **Test workflow** بزنید؛ n8n یک Test URL موقت می‌سازد.
> - **تونل:** یک تونل عمومی مثل `ngrok http 5678` بالا بیاورید و آدرس آن را در متغیر `WEBHOOK_URL` در فایل `.env` قرار دهید، سپس کانتینر را ری‌استارت کنید.

---

## نحوه import کردن Workflow

### روش اول: از طریق رابط گرافیکی (پیشنهادی)

۱. وارد `http://localhost:5678` شوید.
۲. از منوی بالا سمت راست روی **⋯ (سه‌نقطه)** یا دکمه **Create Workflow** کلیک کنید.
۳. گزینه **Import from File...** را انتخاب کنید.
۴. فایل `workflow/telegram_ai_bot.json` را از پوشه پروژه انتخاب کنید.
۵. Workflow با هفت گره روی بوم ظاهر می‌شود. آن را با نام `AI Telegram Bot` ذخیره کنید (`Ctrl+S`).

### روش دوم: کپی و Paste

کل محتوای فایل JSON را کپی کنید، وارد یک Workflow خالی شوید و `Ctrl+V` بزنید — n8n ساختار را به‌صورت خودکار می‌سازد.

### روش سوم: خط فرمان (داخل کانتینر)

پوشه `workflow` در `docker-compose.yml` به داخل کانتینر mount شده است:

```bash
docker exec -it n8n-telegram-bot n8n import:workflow --input=/home/node/workflow/telegram_ai_bot.json
docker compose restart n8n
```

---

## تنظیم Credential ها

Workflow به دو اعتبارنامه نیاز دارد. **کلیدها هرگز داخل فایل Workflow ذخیره نمی‌شوند** و باید یک‌بار به‌صورت دستی ساخته شوند.

### ۱) اعتبارنامه OpenAI با نام `openai_api`

۱. در n8n به مسیر **Credentials → Add Credential** بروید.
۲. نوع **OpenAI** را انتخاب کنید.
۳. در فیلد **API Key** مقدار `OPENAI_API_KEY` خود را وارد کنید.
۴. نام اعتبارنامه را دقیقاً `openai_api` بگذارید و ذخیره کنید.
۵. به گره **OpenAI Chat Model** بروید و این اعتبارنامه را از لیست انتخاب کنید.

### ۲) اعتبارنامه تلگرام با نام `telegram_bot`

۱. **Add Credential → Telegram**.
۲. در فیلد **Access Token** مقدار `TELEGRAM_BOT_TOKEN` را وارد کنید.
۳. نام آن را `telegram_bot` بگذارید و ذخیره کنید.
۴. این اعتبارنامه را در هر چهار گره تلگرام انتخاب کنید: `Telegram Trigger`، `Send Reply`، `Notify User On Failure`، `Alert Admin`.

### ۳) فعال‌سازی Error Workflow

به **Workflow Settings** (منوی سه‌نقطه بالا سمت راست) بروید و در فیلد **Error Workflow** همین Workflow را انتخاب کنید تا گره `Error Trigger` هنگام بروز خطای مدیریت‌نشده اجرا شود.

---

## توضیح گره‌ها

| # | نام گره | نوع (Type) | عملکرد | تنظیمات مهم |
|---|---|---|---|---|
| ۱ | **Telegram Trigger** | `n8n-nodes-base.telegramTrigger` | نقطه شروع Workflow؛ با دریافت پیام از تلگرام اجرا می‌شود | `updates: message` — اعتبارنامه `telegram_bot` |
| ۲ | **AI Agent** | `@n8n/n8n-nodes-langchain.agent` | ارسال متن پیام به مدل زبانی و دریافت پاسخ | ورودی: `{{ $json.message.text }}` — پرامپت سیستم فارسی — `onError: continueErrorOutput` |
| ۳ | **OpenAI Chat Model** | `@n8n/n8n-nodes-langchain.lmChatOpenAi` | زیرگره تأمین‌کننده موتور هوش مصنوعی برای AI Agent | مدل `gpt-4o-mini` — `temperature=0.7` — `maxTokens=512` — اعتبارنامه `openai_api` |
| ۴ | **Send Reply** | `n8n-nodes-base.telegram` | ارسال پاسخ مدل به همان چت کاربر | `chatId: {{ $('Telegram Trigger').item.json.message.chat.id }}` — `text: {{ $json.output }}` |
| ۵ | **Notify User On Failure** | `n8n-nodes-base.telegram` | ارسال پیام عذرخواهی به کاربر در صورت خطای AI Agent | متصل به خروجی خطای (شاخه قرمز) گره AI Agent |
| ۶ | **Error Trigger** | `n8n-nodes-base.errorTrigger` | Trigger مستقل برای خطاهای مدیریت‌نشده کل Workflow | نیازمند انتخاب Workflow در تنظیمات Error Workflow |
| ۷ | **Alert Admin** | `n8n-nodes-base.telegram` | ارسال گزارش کامل خطا به ادمین | `chatId: {{ $env.TELEGRAM_ADMIN_CHAT_ID }}` |

> در فیلد `notes` هر گره داخل فایل JSON، توضیح فارسی کامل درباره عملکرد، ورودی، خروجی و نکات پیکربندی نوشته شده است. برای دیدن آن‌ها، در n8n روی گره دابل‌کلیک کرده و به بخش **Notes** در تب **Settings** مراجعه کنید.

### عبارت‌های (Expressions) کلیدی

| هدف | عبارت |
|---|---|
| متن پیام ورودی کاربر | `{{ $json.message.text }}` |
| شناسه چت کاربر | `{{ $json.message.chat.id }}` |
| شناسه چت از گره غیرمجاور | `{{ $('Telegram Trigger').item.json.message.chat.id }}` |
| پاسخ تولیدشده توسط AI Agent | `{{ $json.output }}` |
| پیام خطا در Error Trigger | `{{ $json.execution.error.message }}` |
| خواندن متغیر محیطی | `{{ $env.TELEGRAM_ADMIN_CHAT_ID }}` |

---

## دو نسخه از Workflow

پروژه دو پیاده‌سازی از یک ربات دارد؛ تفاوت آن‌ها فقط در **نحوه دریافت پیام** است.

| | `telegram_ai_bot.json` (Webhook) | `telegram_ai_bot_polling.json` (Polling) |
|---|---|---|
| گره شروع | `Telegram Trigger` | `Schedule Trigger` (هر ۵ ثانیه) |
| دریافت پیام | تلگرام Webhook می‌فرستد | n8n متد `getUpdates` را صدا می‌زند |
| نیاز به آدرس عمومی HTTPS | **بله** | **خیر** |
| تأخیر پاسخ | تقریباً صفر | حداکثر ۵ ثانیه |
| مناسب برای | سرور عملیاتی با دامنه | توسعه و تست روی `localhost` |

### چرا نسخه Polling لازم است؟

گره `Telegram Trigger` هنگام فعال شدن، متد `setWebhook` را صدا می‌زند و آدرس n8n را به تلگرام معرفی می‌کند. اگر n8n روی `localhost` اجرا شود، سرورهای تلگرام هیچ راهی برای رسیدن به آن ندارند و در نتیجه **هیچ پیامی دریافت نمی‌شود** — بدون آنکه خطایی نمایش داده شود.

نسخه Polling این محدودیت را دور می‌زند: به جای اینکه تلگرام به n8n وصل شود، **n8n به تلگرام وصل می‌شود** (ارتباط خروجی) و پیام‌های جدید را می‌گیرد.

### مدیریت وضعیت در نسخه Polling

برای اینکه یک پیام دوبار پردازش نشود، شناسه آخرین پیام در حافظه دائمی Workflow ذخیره می‌شود:

```javascript
const staticData = $getWorkflowStaticData('global');
staticData.telegramOffset = update.update_id + 1;
```

> ⚠️ تابع `$getWorkflowStaticData` فقط در **اجرای Production** (زمانی که Workflow فعال است) داده را ذخیره می‌کند. در اجرای دستی (Execute Workflow) مقدار ذخیره نمی‌شود و پیام‌ها تکراری پردازش می‌شوند.

در نسخه Polling، توکن ربات از متغیر محیطی `{{ $env.TELEGRAM_BOT_TOKEN }}` خوانده می‌شود؛ بنابراین برای گره‌های تلگرام نیازی به Credential نیست و فقط `openai_api` لازم است.

### راه‌اندازی نسخه Webhook روی سرور عملیاتی

۱. n8n را روی سروری با دامنه و گواهی HTTPS معتبر مستقر کنید.
۲. متغیر `WEBHOOK_URL=https://your-domain.com` را در `.env` تنظیم کنید.
۳. کانتینر را ری‌استارت و Workflow را فعال کنید.
۴. ثبت شدن Webhook را بررسی کنید:

```bash
curl "https://api.telegram.org/bot<TOKEN>/getWebhookInfo"
```

اگر مقدار `url` خالی برگردد، Webhook ثبت نشده است.

---

## نکات امنیتی

- 🔒 **هرگز فایل `.env` را commit نکنید.** این فایل در `.gitignore` قرار دارد؛ فقط `.env.example` باید در مخزن باشد.
- 🔑 **کلیدها را در Credential Manager نگه دارید،** نه در پارامترهای گره. n8n اعتبارنامه‌ها را رمزنگاری‌شده در پایگاه‌داده داخلی ذخیره می‌کند و آن‌ها هنگام Export شدن Workflow، در فایل JSON نوشته نمی‌شوند.
- 🧾 **قبل از push کردن، فایل JSON را بازبینی کنید** تا مطمئن شوید هیچ توکن یا کلیدی داخل آن نمانده است.
- 🌐 **پنل n8n را بدون احراز هویت روی اینترنت باز نگذارید.** در محیط عملیاتی از HTTPS و متغیرهای `N8N_BASIC_AUTH_ACTIVE` یا احراز هویت داخلی استفاده کنید.
- ♻️ **کلیدها را دوره‌ای تعویض (Rotate) کنید** و در صورت لو رفتن، بلافاصله از پنل OpenAI و BotFather ابطال کنید.
- 💸 **سقف هزینه تعیین کنید:** در حساب OpenAI یک Usage Limit فعال کنید تا مصرف ناخواسته کنترل شود.

---

## رفع اشکال

| خطا | علت احتمالی | راه‌حل |
|---|---|---|
| `Connection refused` هنگام باز کردن `localhost:5678` | کانتینر بالا نیامده یا پورت اشغال است | `docker compose ps` را بررسی کنید؛ با `docker compose logs n8n` لاگ را ببینید؛ اگر پورت ۵۶۷۸ اشغال است، آن را در `docker-compose.yml` به `5679:5678` تغییر دهید |
| `Invalid API key` / `401 Unauthorized` از OpenAI | کلید اشتباه، منقضی یا دارای فاصله اضافی است | کلید را از پنل OpenAI بازسازی کنید و بدون فاصله در اعتبارنامه `openai_api` بگذارید |
| `429 insufficient_quota` | اعتبار حساب OpenAI تمام شده است | در بخش Billing حساب OpenAI شارژ کنید |
| `Telegram bot not found` / `404 Not Found` | توکن ربات اشتباه است | توکن را با `/mybots` در BotFather بررسی و در اعتبارنامه `telegram_bot` جایگزین کنید |
| **ربات هیچ پاسخی نمی‌دهد و هیچ خطایی هم دیده نمی‌شود** | Webhook ثبت نشده چون n8n روی `localhost` از اینترنت قابل دسترس نیست | با `curl "https://api.telegram.org/bot<TOKEN>/getWebhookInfo"` بررسی کنید؛ اگر `url` خالی بود، از نسخه `telegram_ai_bot_polling.json` استفاده کنید |
| `--tunnel` کار نمی‌کند | این گزینه در n8n نسخه ۲ حذف شده است | از نسخه Polling استفاده کنید یا n8n را پشت یک دامنه عمومی با HTTPS مستقر کنید |
| پیام‌ها در نسخه Polling تکراری پردازش می‌شوند | Workflow فعال نیست و `$getWorkflowStaticData` ذخیره نمی‌کند | Workflow را **Active** کنید تا در حالت Production اجرا شود |
| `Bad Request: chat not found` در گره Alert Admin | مقدار `TELEGRAM_ADMIN_CHAT_ID` نادرست است یا هرگز به ربات پیام نداده‌اید | یک‌بار در تلگرام به ربات `/start` بدهید و شناسه عددی صحیح را در `.env` بگذارید |
| `access to env vars denied` در Expression ها | n8n به‌صورت پیش‌فرض دسترسی به `$env` را می‌بندد | مقدار `N8N_BLOCK_ENV_ACCESS_IN_NODE=false` را در `.env` بگذارید و کانتینر را ری‌استارت کنید، یا شناسه چت را مستقیماً عددی وارد کنید |
| گره AI Agent خطای `No language model` می‌دهد | زیرگره مدل به ورودی Chat Model متصل نیست | خروجی گره `OpenAI Chat Model` را به ورودی **Chat Model** گره `AI Agent` وصل کنید |
| تغییرات پس از ری‌استارت از بین می‌رود | Volume تعریف نشده است | مطمئن شوید بخش `volumes` با `n8n_data:/home/node/.n8n` در `docker-compose.yml` وجود دارد |

---

## ساختار پروژه

```
23-n8n/
├── .env.example               # نمونه متغیرهای محیطی
├── .gitignore                 # نادیده گرفتن .env و فایل‌های موقت
├── README.md                  # همین فایل
├── docker-compose.yml         # راه‌اندازی n8n با Docker
├── workflow/
│   ├── telegram_ai_bot.json           # Workflow نسخه Webhook (عملیاتی)
│   └── telegram_ai_bot_polling.json   # Workflow نسخه Polling (توسعه محلی)
└── docs/
    └── architecture.md        # توضیح معماری و اجزای Workflow
```

---

## مسیرهای توسعه بعدی

- افزودن **Memory** به گره AI Agent برای حفظ تاریخچه گفتگو (`Window Buffer Memory`)
- افزودن **Tool** هایی مثل جست‌وجوی وب یا پایگاه‌داده برای تبدیل ربات به یک Agent واقعی
- پشتیبانی از **پیام صوتی** با گره Whisper
- ذخیره تاریخچه مکالمات در **PostgreSQL** یا **Google Sheets**
- محدودسازی نرخ درخواست (Rate Limit) برای کنترل هزینه

---

## نویسنده

**سعید سعادتی‌گرو** — مهندس هوش مصنوعی و توسعه‌دهنده ارشد Python / Django

مخزن پروژه: https://github.com/saeidsaadatigero/AI-Telegram-Bot-with-n8n
