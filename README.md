# TrafX Access Server

**ساخت [TrafX Cloud](https://trafx.cloud)** · تلگرام: [@trafxcloud](https://t.me/trafxcloud) · [English](#english)

سرور دسترسی برای پنل **TrafX Radius**: کاربرها با همان نام کاربری و رمز پنل وصل می‌شوند؛ ورود، ثبت مصرف،
محدودیت سرعت، زمان و قطع کاربر همه از پنل (RADIUS) می‌آید. روی **میکروتیک (کانتینر RouterOS 7)** یا
**سرور لینوکس (Docker)** نصب می‌شود. **رایگان، بدون لایسنس.**

| سرویس | پورت | توضیح |
|---|---|---|
| SOCKS5 | 1080 TCP + UDP | تلگرام (با تماس صوتی و تصویری)، مرورگر، هر برنامه‌ای که SOCKS دارد |
| HTTP پراکسی | همان 1080 | تنظیم پراکسی ویندوز، مک، iOS، اندروید و مرورگرها (اختیاری) |
| HTTPS پراکسی | همان 1080 | پراکسی با TLS، رمز کاربر رمزنگاری می‌شود (اختیاری، نیاز به دامنه) |
| OpenConnect / AnyConnect | 443 TCP + UDP | VPN کامل با ocserv، چند لوکیشن روی یک دامنه، گواهی Let's Encrypt (اختیاری) |

- یک کاربر با چند برنامه و اتصال از یک IP = **یک نشست** در پنل (سقف کاربر همزمان درست کار می‌کند).
- حجم دقیق، **Mikrotik-Rate-Limit** پنل (با تغییر لحظه‌ای)، **Session-Timeout**، و **Disconnect** از پنل.
- کاربر به شبکه‌ی داخلی سرور و پورت 25 دسترسی ندارد؛ رمز اشتباه و تلاش زیاد محدود می‌شود.
- خروج ترافیک کاربرها از تونل (GRE، WireGuard، L2TP، ...) یا مستقیم؛ برای OpenConnect هر لوکیشن از تونل خودش.

## سمت پنل

یک NAS بسازید: **IP** = آی‌پی عمومی روتر یا سرور، **نوع** = `other`، **secret** = یک رمز بلند و تصادفی.
اگر سرویس کاربر به NAS محدود است، این NAS را هم به سرویس اضافه کنید.

## نصب روی میکروتیک

پیش‌نیاز: RouterOS 7 با پکیج `container` و `device-mode` فعال برای container؛ معماری x86_64 (CHR)،
arm64 یا arm (ARMv7).

در ترمینال میکروتیک (مقدارها را عوض کنید):

```
:global tsPanel "PANEL_IP"
:global tsSecret "NAS_SECRET"
:global tsTunnel "to-uk"
/tool fetch url="https://raw.githubusercontent.com/TrafXCloud/trafx-access-server/main/mikrotik/install.rsc" dst-path=trafx-socks-install.rsc
/import trafx-socks-install.rsc
```

- `tsTunnel` اختیاری است: نام routing table تونلی که ترافیک کاربرها از آن بیرون برود. اگر نگذارید، مستقیم بیرون می‌رود.
- اختیاری: `tsPublicIP` (اگر تشخیص خودکار IP عمومی درست نبود)، `tsDisk`، `tsPort` (پیش‌فرض 1080).
- بعد از نصب، این متغیرها از حافظه‌ی روتر پاک می‌شوند.

اسکریپت یک bridge جدا (`br-trafx-socks`، شبکه‌ی `172.31.250.0/24`) می‌سازد و با شبکه‌های دیگر روتر تداخل ندارد.
همه‌ی چیزهایی که می‌سازد توضیح `trafx-socks` دارند.

### HTTP و HTTPS پراکسی

قبل از `/tool fetch` اضافه کنید:

```
:global tsHttp "yes"
:global tsHttps "yes"
```

- `tsHttp`: HTTP پراکسی روی همان پورت (`http://USER:PASS@ROUTER_IP:1080`).
- `tsHttps`: HTTPS پراکسی؛ از گواهی Let's Encrypt بخش OpenConnect استفاده می‌کند، پس `tsOcserv`، `tsOcHost` و
  `tsOcEmail` (پایین) لازم است. کلاینت با دامنه وصل می‌شود: `https://USER:PASS@oc.example.com:1080`.
- UDP ندارند (تماس تلگرام فقط با SOCKS؛ خود تلگرام HTTP پراکسی را قبول نمی‌کند).
- Chrome / Edge پراکسی HTTPS را فقط با فایل PAC قبول می‌کنند؛ Firefox و curl مستقیم.

### OpenConnect / AnyConnect (ocserv)

همان NAS و همان secret؛ قبل از `/tool fetch` اضافه کنید:

```
:global tsOcserv "yes"
:global tsOcHost "oc.example.com"
:global tsOcEmail "you@example.com"
:global tsOcLocations "to-uk=UK;to-de=Germany"
:global tsOcDefaultName "Iran"
```

- پورت پیش‌فرض 443 (`tsOcPort` برای عوض کردن). اگر سرویسی از خود روتر (مثل `www-ssl`، `reverse-proxy`،
  SSTP) یا قانون NAT دیگری این پورت را گرفته باشد، اسکریپت قبل از هر تغییری متوقف می‌شود و می‌گوید چه کنید.
- `tsOcHost` و `tsOcEmail`: گواهی Let's Encrypt که خودکار تمدید می‌شود. رکورد A دامنه باید به IP روتر اشاره کند
  (در Cloudflare پراکسی خاموش) و پورت 80 روتر آزاد باشد (مثلاً `/ip service set www port=8080`). بدون این دو،
  گواهی self-signed است.
- `tsOcLocations` (اختیاری): لوکیشن‌ها روی همان دامنه، به شکل `خروجی=اسم` (حداکثر ۱۰). خروجی یک
  **جدول مسیریابی** است یا یک **اینترفیس تونل نقطه‌به‌نقطه** (WireGuard، GRE، IPIP، ...) که اسکریپت برایش جدول و
  route می‌سازد. ترتیب منو همان ترتیب لیست است. کاربر در AnyConnect از منوی Group (و بعد از اتصال اول، از لیست
  اتصال‌های اپ) انتخاب می‌کند؛ پرچم کشور خودکار اضافه می‌شود. `tsOcProfile "no"`: فقط منوی Group.
- `tsOcDefaultName`: اسم ورودی بدون لوکیشن (خروج از `tsTunnel` یا مستقیم)؛ `"none"` = این ورودی نباشد و
  اولین لوکیشن پیش‌فرض شود.

نصب دوباره (مثلاً برای عوض کردن لوکیشن‌ها یا روشن کردن HTTP) فقط متغیرهای جدید را لازم دارد؛ پنل، secret،
تونل، دامنه و لوکیشن‌های قبلی از تنظیمات روتر برداشته می‌شوند. گواهی در پوشه‌ی `trafx-socks-data` روی دیسک می‌ماند.

### آپدیت

```
/tool fetch url="https://raw.githubusercontent.com/TrafXCloud/trafx-access-server/main/mikrotik/update.rsc" dst-path=trafx-socks-update.rsc
/import trafx-socks-update.rsc
```

### حذف

```
/tool fetch url="https://raw.githubusercontent.com/TrafXCloud/trafx-access-server/main/mikrotik/uninstall.rsc" dst-path=trafx-socks-uninstall.rsc
/import trafx-socks-uninstall.rsc
```

### بررسی

```
/log print where message~"trafx"
```

باید `accounting-on acknowledged by panel` و `trafx-socks started` دیده شود؛ با ocserv همچنین `ocserv started` و
(با دامنه) `certificate ok`. اگر `accounting-on` جواب نگرفت: NAS در پنل، secret، یا فایروال بین روتر و پنل.

## نصب روی سرور لینوکس (Docker)

```bash
mkdir -p /opt/trafx && cd /opt/trafx
curl -fsSLO https://raw.githubusercontent.com/TrafXCloud/trafx-access-server/main/linux/docker-compose.yml
curl -fsSL -o .env https://raw.githubusercontent.com/TrafXCloud/trafx-access-server/main/linux/env.example
nano .env
sysctl -w net.ipv4.ip_forward=1 && echo net.ipv4.ip_forward=1 > /etc/sysctl.d/90-trafx.conf
docker compose up -d && docker compose logs -f
```

- در `.env`: `RADIUS_SERVER` (آی‌پی پنل)، `RADIUS_SECRET`، `NAS_IP` (آی‌پی عمومی همین سرور)، و برای OpenConnect
  `OCSERV_HOST` + `ACME_EMAIL`. توضیح همه‌ی گزینه‌ها داخل همان فایل است.
- HTTP / HTTPS پراکسی: `HTTP_ENABLED=1`، `HTTPS_ENABLED=1` (HTTPS از گواهی OpenConnect استفاده می‌کند).
- فقط SOCKS / HTTP بدون OpenConnect: `OCSERV_ENABLED=0`.
- پورت‌ها در فایروال: 1080 TCP/UDP، 443 TCP/UDP، 80 TCP (گواهی)، 3799 UDP فقط برای آی‌پی پنل.
- لوکیشن جدا برای OpenConnect: فیلد چهارم `OCSERV_LOCATIONS` اینترفیس تونلی است که روی سرور ساخته‌اید
  (`uk:UK:10.21.0.0/16:wg-uk`، یا برای تونل لایه‌ی ۲ `fi:Finland:10.21.0.0/16:l2tp-fi@172.31.98.2`).
- آپدیت: `docker compose pull && docker compose up -d`.

Docker Hub از ایران بسته است: Docker را از پکیج‌های Ubuntu نصب کنید (`apt install docker.io docker-compose-v2`).
ایمیج‌ها روی `ghcr.io` هستند و از ایران در دسترس‌اند.

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

## English

**TrafX Access Server** by [TrafX Cloud](https://trafx.cloud) ([Telegram](https://t.me/trafxcloud)) is a free
access server for the **TrafX Radius** panel. Users log in with their panel username and password; login,
accounting, rate limits, session time and disconnects all come from the panel over RADIUS.

- **SOCKS5** (TCP + UDP, incl. Telegram calls), **HTTP** and **HTTPS** proxy on one port (1080),
  and optional **OpenConnect / AnyConnect** (ocserv) on 443 with Let's Encrypt and several exit locations on
  one host name.
- One session per user and client address; exact byte counts, Mikrotik-Rate-Limit (changeable by CoA),
  Session-Timeout, panel Disconnect.
- Runs as a container on **MikroTik RouterOS 7** (x86_64, arm64, arm) or on **Linux with Docker**.

Panel: add a NAS with this server's public IP, type `other`, and a long secret.

MikroTik:

```
:global tsPanel "PANEL_IP"
:global tsSecret "NAS_SECRET"
/tool fetch url="https://raw.githubusercontent.com/TrafXCloud/trafx-access-server/main/mikrotik/install.rsc" dst-path=trafx-socks-install.rsc
/import trafx-socks-install.rsc
```

Optional globals: `tsTunnel` (routing table for user traffic), `tsHttp "yes"`, `tsHttps "yes"`,
`tsOcserv "yes"` with `tsOcHost` / `tsOcEmail` / `tsOcLocations`. Update and remove with `mikrotik/update.rsc`
and `mikrotik/uninstall.rsc` the same way.

Linux: `linux/docker-compose.yml` + `linux/env.example` (copy to `.env`, fill in, `docker compose up -d`).

Images: `ghcr.io/trafxcloud/trafx-access-server:latest` (proxy only) and `:ocserv` (with OpenConnect),
amd64 / arm64 / arm/v7.

Free to use, provided as is, no warranty. The source code is not public; this repository holds the install
scripts and documentation only.
