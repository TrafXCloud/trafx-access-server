# TrafX Access Server

**ساخت [TrafX Cloud](https://trafx.cloud)** · تلگرام: [@trafxcloud](https://t.me/trafxcloud) · [English](#english)

سرور دسترسی (پراکسی و VPN) که ورود کاربرها را از **پنل RADIUS** شما می‌گیرد: کاربر با همان نام کاربری و رمزی که
در پنل دارد وصل می‌شود، و ورود، ثبت حجم مصرفی، محدودیت سرعت، زمان و قطع کاربر همه از پنل کنترل می‌شود.
روی **میکروتیک (کانتینر RouterOS 7)** یا **سرور لینوکس (Docker)** نصب می‌شود. **رایگان و بدون لایسنس.**

> **با هر پنلی که RADIUS دارد کار می‌کند.** این سرور فقط از RADIUS استاندارد (RFC 2865، 2866، 5176) استفاده
> می‌کند و به پنل خاصی وابسته نیست: TrafX Radius، پنل‌های بر پایه‌ی FreeRADIUS و هر پنل دیگری که بتوانید در آن یک
> NAS (کلاینت RADIUS) تعریف کنید. جزئیات در [سازگاری با پنل‌ها](#compat).

## فهرست

- [این سرور چه می‌دهد](#features)
- [قبل از شروع: چه چیزهایی لازم است](#requirements)
- [قدم ۱: تعریف NAS در پنل](#step1)
- [قدم ۲ (الف): نصب روی میکروتیک](#mikrotik)
- [قدم ۲ (ب): نصب روی سرور لینوکس](#linux)
- [قدم ۳: تست و اتصال کاربر](#step3)
- [سازگاری با پنل‌ها](#compat)
- [مشکلات رایج](#troubleshooting)
- [ایمیج‌ها و شرایط استفاده](#images)

<a id="features"></a>

## این سرور چه می‌دهد

| سرویس | پورت | به چه درد می‌خورد |
|---|---|---|
| SOCKS5 | 1080 TCP + UDP | تلگرام (حتی تماس صوتی و تصویری)، مرورگر، هر برنامه‌ای که SOCKS دارد |
| HTTP پراکسی | همان 1080 | پراکسی ویندوز، مک، iOS، اندروید و مرورگرها (اختیاری) |
| HTTPS پراکسی | همان 1080 | مثل HTTP ولی رمزنگاری‌شده؛ رمز کاربر در مسیر دیده نمی‌شود (اختیاری، دامنه لازم است) |
| OpenConnect / AnyConnect | 443 TCP + UDP | VPN کامل (همه‌ی ترافیک گوشی یا کامپیوتر)، چند لوکیشن روی یک دامنه (اختیاری) |

- یک کاربر با چند برنامه و چند اتصال از یک IP = **یک نشست** در پنل؛ پس محدودیت «تعداد کاربر همزمان» پنل درست کار می‌کند.
- حجم دقیق، محدودیت سرعت (**Mikrotik-Rate-Limit**، با تغییر لحظه‌ای از پنل)، **Session-Timeout** و **قطع کاربر از پنل**.
- کاربر به شبکه‌ی داخلی سرور و پورت 25 (ارسال ایمیل اسپم) دسترسی ندارد؛ رمز اشتباه و تلاش زیاد محدود می‌شود.
- ترافیک کاربرها می‌تواند از یک تونل (GRE، WireGuard، L2TP، ...) بیرون برود یا مستقیم؛ در OpenConnect هر لوکیشن از تونل خودش.

<a id="requirements"></a>

## قبل از شروع: چه چیزهایی لازم است

1. **یک پنل RADIUS** (هر پنلی، [سازگاری](#compat)) و **آی‌پی آن**. سرور شما باید بتواند به پورت‌های
   UDP 1812 و 1813 پنل وصل شود.
2. **یکی از این دو:**
   - میکروتیک با RouterOS 7 (CHR یا روتر با پردازنده‌ی x86_64، arm64 یا arm)، یا
   - یک سرور لینوکس (Ubuntu پیشنهاد می‌شود) با دسترسی root.
3. **آی‌پی عمومی** همان روتر یا سرور.
4. فقط برای OpenConnect و HTTPS: **یک دامنه یا ساب‌دامنه** (مثلاً `oc.example.com`) که رکورد A آن به آی‌پی روتر یا
   سرور اشاره کند. در Cloudflare حتماً ابر نارنجی (Proxy) را خاموش کنید.

در مثال‌های این راهنما این مقدارها را فرض کرده‌ایم؛ مقدارهای خودتان را جایگزین کنید:

| چیز | مقدار مثال |
|---|---|
| آی‌پی پنل | `203.0.113.10` |
| آی‌پی عمومی روتر یا سرور | `198.51.100.20` |
| secret (رمز مشترک پنل و سرور) | `Xk29fhQe8LmvP0wz3Tn7` |
| دامنه‌ی OpenConnect | `oc.example.com` |
| یک کاربر پنل | نام کاربری `ali`، رمز `1234abcd` |

<a id="step1"></a>

## قدم ۱: تعریف NAS در پنل

در پنل یک **NAS** (در بعضی پنل‌ها «سرور»، «روتر» یا «RADIUS Client») بسازید:

| فیلد | مقدار | مثال |
|---|---|---|
| IP | آی‌پی عمومی روتر یا سرور | `198.51.100.20` |
| Secret | یک رمز بلند و تصادفی (حداقل ۸ کاراکتر، بهتر ۲۰ کاراکتر، بدون فاصله و علامت `"`) | `Xk29fhQe8LmvP0wz3Tn7` |
| نوع | در TrafX Radius: `other`؛ در پنل‌های دیگر: `other` یا `mikrotik` یا هر گزینه‌ی عمومی | |

- اگر در پنل سرویس کاربرها به NAS خاص محدود است، این NAS را هم به سرویس اضافه کنید، وگرنه کاربر رد می‌شود.
- اگر پنل برای «قطع کاربر» پورت می‌خواهد، **3799** بدهید.
- یک secret ساده برای ساختن (روی لینوکس یا مک): `openssl rand -hex 16`

<a id="mikrotik"></a>

## قدم ۲ (الف): نصب روی میکروتیک

<a id="router-prep"></a>

### آماده کردن روتر (یک بار)

**۱. پکیج container:** از [mikrotik.com/download](https://mikrotik.com/download) فایل *Extra packages* هم‌نسخه‌ی
روتر را بگیرید، `container-*.npk` را در Winbox به بخش **Files** بکشید و روتر را ریبوت کنید. بعد بررسی کنید:

```
/system package print where name=container
```

**۲. اجازه‌ی container در device-mode:**

```
/system device-mode update container=yes
```

بعد از این دستور، روتر فیزیکی منتظر می‌ماند تا **دکمه‌ی reset را یک بار بزنید** یا **برق را قطع و وصل کنید** (حداکثر
۵ دقیقه). روی CHR (سرور مجازی) کافی است از پنل سرور مجازی یک بار ری‌استارت سخت (power off / on) کنید.

**۳. DNS و اینترنت روتر** باید کار کند (ایمیج از `ghcr.io` دانلود می‌شود که از ایران باز است):

```
/ping ghcr.io count=3
```

### نصب

در **Terminal** میکروتیک (در Winbox: دکمه‌ی New Terminal) این‌ها را یکی‌یکی بزنید. فقط دو مقدار اول را با
مقدار خودتان عوض کنید:

```
:global tsPanel "203.0.113.10"
:global tsSecret "Xk29fhQe8LmvP0wz3Tn7"
/tool fetch url="https://raw.githubusercontent.com/TrafXCloud/trafx-access-server/main/mikrotik/install.rsc" dst-path=trafx-socks-install.rsc
/import trafx-socks-install.rsc
```

همین. بعد از چند دقیقه (دانلود ایمیج) SOCKS5 روی پورت 1080 آماده است. برو به [قدم ۳](#step3).

متغیرهای اختیاری (قبل از `/tool fetch` بزنید):

| متغیر | کارش | مثال |
|---|---|---|
| `tsTunnel` | ترافیک کاربرها از این routing table (تونل) بیرون برود؛ اگر نگذارید، مستقیم بیرون می‌رود | `:global tsTunnel "to-uk"` |
| `tsPort` | پورت پراکسی (پیش‌فرض 1080) | `:global tsPort "8443"` |
| `tsPublicIP` | اگر آی‌پی عمومی خودکار درست تشخیص داده نشد | `:global tsPublicIP "198.51.100.20"` |
| `tsDisk` | دیسکی که ایمیج روی آن ذخیره شود (خودکار انتخاب می‌شود) | `:global tsDisk "disk1"` |
| `tsHttp` / `tsHttps` | HTTP / HTTPS پراکسی ([پایین‌تر](#http)) | `:global tsHttp "yes"` |
| `tsOcserv` و بقیه‌ی `tsOc...` | OpenConnect ([پایین‌تر](#ocserv)) | `:global tsOcserv "yes"` |

اسم routing tableهای روتر را با `/routing table print` ببینید.

اسکریپت یک bridge جدا (`br-trafx-socks`، شبکه‌ی `172.31.250.0/24`) می‌سازد و با شبکه‌های دیگر روتر تداخل ندارد.
همه‌ی چیزهایی که می‌سازد توضیح (comment) `trafx-socks` دارند. بعد از نصب، متغیرها از حافظه‌ی روتر پاک می‌شوند.

<a id="http"></a>

### HTTP و HTTPS پراکسی

قبل از `/tool fetch` اضافه کنید:

```
:global tsHttp "yes"
:global tsHttps "yes"
```

- `tsHttp`: HTTP پراکسی روی همان پورت 1080، با همان کاربرها.
- `tsHttps`: HTTPS پراکسی؛ از گواهی Let's Encrypt بخش OpenConnect استفاده می‌کند، پس `tsOcserv`، `tsOcHost` و
  `tsOcEmail` (پایین) هم لازم است. کلاینت باید با **دامنه** وصل شود، نه آی‌پی.
- HTTP و HTTPS پراکسی UDP ندارند: تماس تلگرام فقط با SOCKS5 کار می‌کند (خود تلگرام HTTP پراکسی را قبول نمی‌کند).
- Chrome و Edge پراکسی HTTPS را فقط با فایل PAC قبول می‌کنند؛ Firefox و curl مستقیم.

<a id="ocserv"></a>

### OpenConnect / AnyConnect (ocserv)

همان NAS و همان secret پنل کافی است. مثال کامل: نصب SOCKS + OpenConnect با دامنه و دو لوکیشن خارج:

```
:global tsPanel "203.0.113.10"
:global tsSecret "Xk29fhQe8LmvP0wz3Tn7"
:global tsOcserv "yes"
:global tsOcHost "oc.example.com"
:global tsOcEmail "you@example.com"
:global tsOcLocations "to-uk=UK;to-de=Germany"
:global tsOcDefaultName "Iran"
/tool fetch url="https://raw.githubusercontent.com/TrafXCloud/trafx-access-server/main/mikrotik/install.rsc" dst-path=trafx-socks-install.rsc
/import trafx-socks-install.rsc
```

- **پورت:** پیش‌فرض 443 (`tsOcPort` برای عوض کردن). اگر سرویسی از خود روتر (مثل `www-ssl`، `reverse-proxy`،
  SSTP) یا قانون NAT دیگری این پورت را گرفته باشد، اسکریپت قبل از هر تغییری متوقف می‌شود و می‌گوید چه کنید.
- **گواهی (`tsOcHost` و `tsOcEmail`):** گواهی Let's Encrypt که خودکار تمدید می‌شود. رکورد A دامنه باید به IP روتر
  اشاره کند و پورت 80 روتر آزاد باشد. اگر وب‌سرویس روتر روی 80 است، جابه‌جایش کنید:
  `/ip service set www port=8080`. بدون این دو، گواهی self-signed است و کاربر هشدار امنیتی می‌بیند.
- **لوکیشن‌ها (`tsOcLocations`، اختیاری):** به شکل `خروجی=اسم`، جداشده با `;` (حداکثر ۱۰). «خروجی» یا اسم یک
  **routing table** است یا اسم یک **اینترفیس تونل نقطه‌به‌نقطه** (WireGuard، GRE، IPIP، ...) که اسکریپت برایش
  جدول و route می‌سازد. ترتیب منو همان ترتیب لیست است. کاربر در AnyConnect از منوی **Group** لوکیشن را انتخاب
  می‌کند (بعد از اتصال اول، لوکیشن‌ها در لیست اتصال‌های اپ هم اضافه می‌شوند)؛ پرچم کشور خودکار اضافه می‌شود.
  `tsOcProfile "no"`: فقط منوی Group، بدون اضافه شدن به لیست اپ.
- **`tsOcDefaultName`:** اسم ورودی بدون لوکیشن (خروج از `tsTunnel` یا مستقیم). `"none"` یعنی این ورودی نباشد و
  اولین لوکیشن پیش‌فرض شود.

### نصب دوباره، آپدیت، حذف

**نصب دوباره** (مثلاً برای اضافه کردن لوکیشن یا روشن کردن HTTP): فقط متغیرهای جدید را بزنید و دو خط
`/tool fetch` و `/import` نصب را تکرار کنید. پنل، secret، تونل، دامنه و لوکیشن‌های قبلی از تنظیمات روتر برداشته
می‌شوند. مثلاً فقط برای روشن کردن HTTP:

```
:global tsHttp "yes"
/tool fetch url="https://raw.githubusercontent.com/TrafXCloud/trafx-access-server/main/mikrotik/install.rsc" dst-path=trafx-socks-install.rsc
/import trafx-socks-install.rsc
```

**آپدیت** به آخرین نسخه:

```
/tool fetch url="https://raw.githubusercontent.com/TrafXCloud/trafx-access-server/main/mikrotik/update.rsc" dst-path=trafx-socks-update.rsc
/import trafx-socks-update.rsc
```

**حذف کامل** (فقط چیزهایی که خودش ساخته پاک می‌شود؛ گواهی در پوشه‌ی `trafx-socks-data` روی دیسک می‌ماند):

```
/tool fetch url="https://raw.githubusercontent.com/TrafXCloud/trafx-access-server/main/mikrotik/uninstall.rsc" dst-path=trafx-socks-uninstall.rsc
/import trafx-socks-uninstall.rsc
```

### بررسی روی میکروتیک

```
/log print where message~"trafx"
```

باید این خط‌ها دیده شود:

- `accounting-on acknowledged by panel` → ارتباط با پنل درست است.
- `trafx-socks started` → پراکسی بالا آمده.
- با OpenConnect: `ocserv started` و (با دامنه) `certificate ok`.

<a id="linux"></a>

## قدم ۲ (ب): نصب روی سرور لینوکس (Docker)

**۱. نصب Docker** (اگر ندارید). روی Ubuntu:

```bash
apt update && apt install -y docker.io docker-compose-v2
```

(روی سرورهای ایران `download.docker.com` و Docker Hub بسته است؛ همین پکیج Ubuntu کار می‌کند. ایمیج‌های ما روی
`ghcr.io` هستند که از ایران باز است.)

**۲. دانلود فایل‌ها:**

```bash
mkdir -p /opt/trafx && cd /opt/trafx
curl -fsSLO https://raw.githubusercontent.com/TrafXCloud/trafx-access-server/main/linux/docker-compose.yml
curl -fsSL -o .env https://raw.githubusercontent.com/TrafXCloud/trafx-access-server/main/linux/env.example
```

**۳. تنظیم `.env`:** با `nano .env` باز کنید. حداقل این سه خط را پر کنید:

```ini
RADIUS_SERVER=203.0.113.10
RADIUS_SECRET=Xk29fhQe8LmvP0wz3Tn7
NAS_IP=198.51.100.20
```

- **فقط SOCKS (بدون OpenConnect و دامنه):** `OCSERV_ENABLED=1` را به `OCSERV_ENABLED=0` تغییر دهید.
- **با OpenConnect:** `OCSERV_HOST=oc.example.com` و `ACME_EMAIL=you@example.com` را با دامنه و ایمیل خودتان پر کنید.
- **HTTP / HTTPS پراکسی:** `#` اول خط‌های `HTTP_ENABLED=1` و `HTTPS_ENABLED=1` را بردارید (HTTPS به OpenConnect و دامنه نیاز دارد).
- توضیح بقیه‌ی گزینه‌ها داخل همان فایل است.

ذخیره در nano: `Ctrl+O` و Enter، خروج: `Ctrl+X`.

**۴. اجرا:**

```bash
sysctl -w net.ipv4.ip_forward=1 && echo net.ipv4.ip_forward=1 > /etc/sysctl.d/90-trafx.conf
docker compose up -d
docker compose logs -f
```

در لاگ باید `accounting-on acknowledged by panel` و `trafx-socks started` (و با OpenConnect `ocserv started`)
ببینید. برای خروج از لاگ `Ctrl+C` (سرور روشن می‌ماند).

**۵. فایروال:** اگر فایروال دارید (مثلاً `ufw`) این پورت‌ها را باز کنید:

```bash
ufw allow 1080
ufw allow 443
ufw allow 80/tcp
ufw allow from 203.0.113.10 to any port 3799 proto udp
```

(1080 پراکسی، 443 OpenConnect، 80 فقط برای گرفتن گواهی، 3799 فقط از آی‌پی پنل برای قطع کاربر و تغییر سرعت.)

**آپدیت:**

```bash
cd /opt/trafx && docker compose pull && docker compose up -d
```

**لوکیشن جدا برای OpenConnect:** فیلد چهارم `OCSERV_LOCATIONS` اسم اینترفیس تونلی است که روی سرور ساخته‌اید،
مثلاً `uk:UK:10.21.0.0/16:wg-uk`، یا برای تونل لایه‌ی ۲ `fi:Finland:10.21.0.0/16:l2tp-fi@172.31.98.2`.

<a id="step3"></a>

## قدم ۳: تست و اتصال کاربر

اول یک کاربر تستی در پنل بسازید (مثال: `ali` / `1234abcd`) که سرویس فعال و حجم دارد.

### تست سریع از یک کامپیوتر دیگر

```bash
curl -x socks5h://ali:1234abcd@198.51.100.20:1080 https://ifconfig.me
```

باید آی‌پی خروجی سرور (یا تونل) را چاپ کند، و در پنل یک نشست آنلاین برای `ali` دیده شود.

### تلگرام (SOCKS5، با تماس)

**Settings → Data and Storage → Proxy Settings → Add Proxy → SOCKS5**

| فیلد | مقدار |
|---|---|
| Server | `198.51.100.20` |
| Port | `1080` |
| Username | `ali` |
| Password | `1234abcd` |

برای تماس صوتی و تصویری گزینه‌ی **Use proxy for calls** را هم روشن کنید.

### مرورگر Firefox (SOCKS5)

**Settings → Network Settings → Manual proxy configuration**: در SOCKS Host آی‌پی سرور و پورت 1080، گزینه‌ی
**SOCKS v5** و **Proxy DNS when using SOCKS v5**. هنگام باز کردن اولین سایت نام کاربری و رمز پرسیده می‌شود.

### ویندوز، مک، اندروید و آیفون (HTTP پراکسی)

نیاز به `tsHttp "yes"` یا `HTTP_ENABLED=1` دارد. در تنظیمات پراکسی دستگاه (Wi-Fi → Proxy → Manual) سرور
`198.51.100.20` و پورت `1080` را بدهید؛ نام کاربری و رمز را دستگاه یا مرورگر می‌پرسد. نشانی کامل برای برنامه‌هایی
که خودشان پراکسی می‌گیرند: `http://ali:1234abcd@198.51.100.20:1080` (با HTTPS:
`https://ali:1234abcd@oc.example.com:1080`).

### AnyConnect / OpenConnect (VPN کامل)

اپ **Cisco Secure Client (AnyConnect)** یا **OpenConnect** را نصب کنید، اتصال جدید بسازید و آدرس را
`oc.example.com` بدهید (اگر پورت را عوض کرده‌اید: `oc.example.com:4443`). نام کاربری و رمز همان پنل است. اگر
لوکیشن تعریف کرده‌اید، از منوی **Group** انتخاب کنید.

<a id="compat"></a>

## سازگاری با پنل‌ها

TrafX Access Server برای پنل شما یک **NAS (کلاینت RADIUS) معمولی** است، درست مثل یک میکروتیک یا سرور VPN.
هر پنل یا سرور RADIUS که این‌ها را دارد پشتیبانی می‌شود:

| کار | استاندارد | توضیح |
|---|---|---|
| ورود کاربر | RFC 2865، PAP | پورت 1812 UDP |
| ثبت مصرف (Start / Interim / Stop، Accounting-On/Off) | RFC 2866 | پورت 1813 UDP |
| قطع کاربر و تغییر سرعت (Disconnect / CoA) | RFC 5176 | پورت 3799 UDP، فقط از آی‌پی پنل، با Message-Authenticator |
| محدودیت سرعت | `Mikrotik-Rate-Limit` (VSA میکروتیک) | همان صفتی که پنل‌ها برای میکروتیک می‌فرستند، مثلاً `10M/20M` |
| زمان و فاصله‌ی ثبت مصرف | `Session-Timeout`، `Acct-Interim-Interval` | اگر پنل Interim نفرستد، ۵ دقیقه |

- اگر پنل شما برای میکروتیک سرعت می‌گذارد، برای این سرور هم بدون تغییر کار می‌کند.
- پورت‌های غیرپیش‌فرض پنل در لینوکس با `RADIUS_AUTH_PORT` و `RADIUS_ACCT_PORT` در `.env` تنظیم می‌شوند.
- چند سرور دسترسی روی یک پنل: برای هر کدام یک NAS با آی‌پی خودش بسازید.

<a id="troubleshooting"></a>

## مشکلات رایج

| مشکل | علت و راه‌حل |
|---|---|
| در لاگ `accounting-on` جواب نمی‌گیرد | NAS در پنل با آی‌پی **عمومی** همین سرور ساخته نشده، secret دو طرف یکی نیست، یا فایروال بین سرور و پنل پورت 1812/1813 UDP را بسته است. |
| کاربر رد می‌شود ولی رمز درست است | سرویس کاربر به NAS محدود است و این NAS در آن نیست؛ یا حجم/زمان کاربر تمام شده. پیام رد در لاگ (`auth: rejected`) آمده است. |
| `install and enable the container package first` | پکیج container نصب نیست ([آماده کردن روتر](#router-prep)). |
| کانتینر ساخته نمی‌شود یا خطای device-mode | `device-mode` را فعال و دکمه‌ی reset یا قطع برق را انجام نداده‌اید. |
| `routing table ... not found` | اسم `tsTunnel` اشتباه است؛ با `/routing table print` اسم درست را ببینید. |
| گواهی گرفته نمی‌شود | رکورد A دامنه به آی‌پی سرور نیست، Cloudflare Proxy روشن است، یا پورت 80 بسته یا گرفته شده است. |
| قطع کاربر از پنل کار نمی‌کند | پورت 3799 UDP از آی‌پی پنل به سرور باز نیست، یا پنل از آی‌پی دیگری درخواست می‌فرستد. |
| تلگرام وصل است ولی تماس برقرار نمی‌شود | از SOCKS5 استفاده کنید (نه HTTP) و پورت 1080 **UDP** هم باز باشد. |

اگر حل نشد، خروجی لاگ (`/log print where message~"trafx"` یا `docker compose logs --tail 100`) را در
[تلگرام](https://t.me/trafxcloud) بفرستید.

<a id="images"></a>

## ایمیج‌ها

| ایمیج | محتوا |
|---|---|
| `ghcr.io/trafxcloud/trafx-access-server:latest` | SOCKS5 + HTTP/HTTPS (حدود ۵ مگ) |
| `ghcr.io/trafxcloud/trafx-access-server:ocserv` | همان + OpenConnect (ocserv) |

amd64، arm64 و arm/v7. تگ نسخه‌دار هم هست (مثلاً `0.5.0` و `0.5.0-ocserv`).

## شرایط استفاده

استفاده رایگان است، بدون هیچ تضمینی («as is»). سورس برنامه عمومی نیست؛ این مخزن فقط اسکریپت‌های نصب و
راهنماست. بخش OpenConnect بر پایه‌ی ocserv (GPLv2، بدون تغییر) و کار
[mraghili1985](https://github.com/mraghili1985) است.

پشتیبانی و پنل TrafX Radius: [trafx.cloud](https://trafx.cloud) · [t.me/trafxcloud](https://t.me/trafxcloud)

---

<a id="english"></a>

## English

**TrafX Access Server** by [TrafX Cloud](https://trafx.cloud) ([Telegram](https://t.me/trafxcloud)) is a free
proxy / VPN access server that authenticates against **any RADIUS panel**. Users log in with their panel
username and password; login, accounting, rate limits, session time and disconnects all come from the panel.

**Works with any RADIUS server, not only TrafX Radius.** It is a plain NAS (RADIUS client) using standard
RADIUS only: PAP authentication (RFC 2865, UDP 1812), accounting Start / Interim / Stop and Accounting-On/Off
(RFC 2866, UDP 1813), Disconnect and CoA (RFC 5176, UDP 3799, from the panel's IP only, with
Message-Authenticator), the `Mikrotik-Rate-Limit` VSA for speed (the same attribute panels send to MikroTik),
`Session-Timeout` and `Acct-Interim-Interval`.

- **SOCKS5** (TCP + UDP, incl. Telegram calls), **HTTP** and **HTTPS** proxy on one port (1080),
  and optional **OpenConnect / AnyConnect** (ocserv) on 443 with Let's Encrypt and several exit locations on
  one host name.
- One session per user and client address; exact byte counts, live rate-limit changes by CoA,
  Session-Timeout, panel Disconnect.
- Runs as a container on **MikroTik RouterOS 7** (x86_64, arm64, arm) or on **Linux with Docker**.

### Quick start

1. **Panel:** add a NAS with this server's public IP and a long random secret (type `other` in TrafX Radius,
   any generic type elsewhere). Give port 3799 if the panel asks for a disconnect port.
2. **MikroTik** (RouterOS 7, `container` package installed, `/system device-mode update container=yes` confirmed):

   ```
   :global tsPanel "203.0.113.10"
   :global tsSecret "LONG_RANDOM_SECRET"
   /tool fetch url="https://raw.githubusercontent.com/TrafXCloud/trafx-access-server/main/mikrotik/install.rsc" dst-path=trafx-socks-install.rsc
   /import trafx-socks-install.rsc
   ```

   Optional globals: `tsTunnel` (routing table for user traffic), `tsPort`, `tsHttp "yes"`, `tsHttps "yes"`,
   `tsOcserv "yes"` with `tsOcHost` / `tsOcEmail` / `tsOcLocations` / `tsOcDefaultName`. Update and remove with
   `mikrotik/update.rsc` and `mikrotik/uninstall.rsc` the same way. Check with `/log print where message~"trafx"`.
3. **Linux:**

   ```bash
   mkdir -p /opt/trafx && cd /opt/trafx
   curl -fsSLO https://raw.githubusercontent.com/TrafXCloud/trafx-access-server/main/linux/docker-compose.yml
   curl -fsSL -o .env https://raw.githubusercontent.com/TrafXCloud/trafx-access-server/main/linux/env.example
   nano .env    # RADIUS_SERVER, RADIUS_SECRET, NAS_IP; OCSERV_ENABLED=0 for proxy only
   sysctl -w net.ipv4.ip_forward=1 && echo net.ipv4.ip_forward=1 > /etc/sysctl.d/90-trafx.conf
   docker compose up -d && docker compose logs -f
   ```

   Open 1080 TCP/UDP, 443 TCP/UDP, 80 TCP (certificate), and 3799 UDP from the panel only.
4. **Test:** `curl -x socks5h://USER:PASS@SERVER_IP:1080 https://ifconfig.me`, then look for the online
   session in the panel. Logs should show `accounting-on acknowledged by panel` and `trafx-socks started`.

Images: `ghcr.io/trafxcloud/trafx-access-server:latest` (proxy only) and `:ocserv` (with OpenConnect),
amd64 / arm64 / arm/v7.

Free to use, provided as is, no warranty. The source code is not public; this repository holds the install
scripts and documentation only.
