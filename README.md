آموزش کامل نصب و راه‌اندازی VeilRoute Beta 1.2
نصب سرور Ubuntu، اتصال کلاینت ویندوز و راه‌اندازی SOCKS5
VeilRoute یک ابزار پراکسی مبتنی بر TLS است که اتصال برنامه‌های پشتیبانی‌کننده از SOCKS5 را از طریق یک سرور VPS عبور می‌دهد.
در این آموزش از نسخه عمومی و بدون مشخصات شخصی VeilRoute Beta 1.2 استفاده می‌کنیم.
۱. پیش‌نیازها
سرور
یک VPS با سیستم‌عامل Ubuntu
دسترسی SSH به سرور
یک IPv4 عمومی یا دامنه
دسترسی به تنظیمات Firewall یا Security Group
پورت TCP آزاد؛ در این آموزش از 8443 استفاده می‌کنیم.
کامپیوتر ویندوز
Windows 10 یا 11
Python نسخه 3.10 یا بالاتر
فایل فشرده VeilRoute-Beta-1.2-Clean-Fixed.zip
کلید SSH با پسوند PEM
۲. آماده‌سازی فایل‌ها
ابتدا فایل ZIP را از حالت فشرده خارج کنید.
ساختار پوشه باید شامل مواردی مانند این باشد:
VeilRoute-Beta-1.2/
│
├── README.md
├── pyproject.toml
├── veilroute/
├── server/
├── tunnel_engine/
├── tests/
├── docs/
└── wintun-sdk/
مسیر پوشه را در ویندوز پیدا کنید. برای مثال:
C:\VeilRoute-Beta-1.2
۳. انتقال فایل سرور به VPS
PowerShell را باز کنید و به مسیر پروژه بروید:
cd C:\VeilRoute-Beta-1.2
دستور زیر را با اطلاعات سرور خود تکمیل کنید:
scp -i "$env:USERPROFILE\Downloads\YOUR-KEY.pem" -r .\server ubuntu@YOUR_SERVER_IP:/home/ubuntu/VeilRoute-Beta-1.2/
موارد زیر را جایگزین کنید:
YOUR-KEY.pem: نام فایل کلید SSH
YOUR_SERVER_IP: آی‌پی سرور
ubuntu: نام کاربری SSH سرور؛ در Ubuntu AWS معمولاً ubuntu است.
حالا به سرور متصل شوید:
ssh -i "$env:USERPROFILE\Downloads\YOUR-KEY.pem" ubuntu@YOUR_SERVER_IP
۴. نصب سرور روی Ubuntu
پس از ورود به VPS، دستورات زیر را اجرا کنید:
sudo apt-get update
sudo apt-get install -y openssl
سپس پوشه پروژه را آماده کنید:
sudo mkdir -p /home/ubuntu/VeilRoute-Beta-1.2
sudo chown -R ubuntu:ubuntu /home/ubuntu/VeilRoute-Beta-1.2
اکنون نصب‌کننده را اجرا کنید:
sudo VEILROUTE_PORT=8443 bash /home/ubuntu/VeilRoute-Beta-1.2/server/install.sh
نصب‌کننده به‌صورت خودکار:
فایل سرور را در مسیر /opt/veilroute/ نصب می‌کند.
گواهی TLS و کلید خصوصی ایجاد می‌کند.
سرویس systemd را ایجاد و فعال می‌کند.
اگر UFW فعال باشد، پورت انتخاب‌شده را در آن باز می‌کند.
۵. تنظیم فایروال VPS
اگر از AWS استفاده می‌کنید، وارد بخش EC2 و سپس Security Groups شوید.
در قسمت Inbound Rules یک قانون جدید بسازید:
Type: Custom TCP
Port: 8443
Source: آدرس IP مجاز شما
برای امنیت بیشتر، بهتر است دسترسی را به IP کلاینت‌های مورد اعتماد محدود کنید.
در صورت استفاده از UFW نیز وضعیت را بررسی کنید:
sudo ufw status
اگر فعال است و نصب‌کننده پورت را باز نکرده، اجرا کنید:
sudo ufw allow 8443/tcp
توجه: پورت SSH یعنی TCP 22 را نبندید.
۶. بررسی وضعیت سرور
برای بررسی اجرای سرویس:
sudo systemctl status veilroute --no-pager
برای بررسی پورت:
sudo ss -lntp | grep 8443
برای مشاهده گزارش‌ها:
sudo journalctl -u veilroute -n 30 --no-pager
اگر همه‌چیز درست باشد، سرویس باید در وضعیت Active (running) باشد و روی پورت 8443 گوش کند.
۷. دریافت اثر انگشت TLS
برای دریافت SHA-256 Fingerprint گواهی سرور، دستور زیر را در VPS اجرا کنید:
sudo openssl x509 -in /etc/veilroute/server.crt -noout -fingerprint -sha256
خروجی شبیه این خواهد بود:
sha256 Fingerprint=AA:BB:CC:DD:...
مقدار کامل بعد از علامت مساوی را کپی کنید.
این اثر انگشت باید در برنامه کلاینت وارد شود. اثر انگشت هر سرور متفاوت است.
مهم: اثر انگشت را از همان سروری بگیرید که قصد اتصال به آن را دارید. بررسی TLS را غیرفعال نکنید.
۸. دریافت کد یک‌بارمصرف Pairing
برای مشاهده کد جفت‌سازی:
sudo journalctl -u veilroute -n 100 --no-pager | grep 'ONE-TIME PAIRING CODE'
این کد فقط برای جفت‌سازی اولیه دستگاه است.
کد را محرمانه نگه دارید.
آن را در گروه‌ها، تصاویر یا GitHub منتشر نکنید.
کد فقط یک بار قابل استفاده است.
اگر کد مصرف یا افشا شد، باید وضعیت جفت‌سازی به‌صورت کنترل‌شده بازنشانی شود.
نکته: صرفاً Restart کردن سرویس، کد قبلی را عوض نمی‌کند.
۹. نصب کلاینت روی ویندوز
به کامپیوتر ویندوز برگردید.
PowerShell را باز کنید:
cd C:\VeilRoute-Beta-1.2
محیط مجازی Python را ایجاد کنید:
py -3 -m venv .venv
فعال‌سازی محیط:
.\.venv\Scripts\Activate.ps1
نصب پروژه:
python -m pip install -e .
اجرای برنامه:
python -m veilroute
اگر PowerShell اجازه فعال‌سازی محیط مجازی را نداد:
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
سپس مجدداً دستور فعال‌سازی را اجرا کنید.
اگر نصب Editable به‌دلیل دانلود نشدن ابزارهای ساخت با خطا مواجه شد، برنامه را مستقیماً اجرا کنید:
.\.venv\Scripts\python.exe -m veilroute
۱۰. جفت‌سازی دستگاه با سرور
پس از اجرای برنامه، منوی اصلی نمایش داده می‌شود.
گزینه زیر را انتخاب کنید:
2 - Pair this device with one-time code
سپس اطلاعات سرور خود را وارد کنید:
Server IP or Domain: YOUR_SERVER_IP
Port: 8443
SHA256 Fingerprint: YOUR_SERVER_FINGERPRINT
Pairing Code: YOUR_ONE_TIME_CODE
در صورت موفقیت باید پیام زیر نمایش داده شود:
Pairing successful; device token saved locally.
یعنی دستگاه با موفقیت جفت شده و توکن آن به‌صورت محلی ذخیره شده است.
توکن دستگاه را محرمانه نگه دارید. فایل تنظیمات کلاینت در این مسیر ذخیره می‌شود:
%USERPROFILE%\.veilroute\client.json
این فایل را در اختیار دیگران قرار ندهید.
۱۱. راه‌اندازی SOCKS5
پس از Pairing موفق، در منوی برنامه گزینه زیر را انتخاب کنید:
1 - Start local SOCKS5 proxy
مقادیر پیش‌فرض:
Host: 127.0.0.1
Port: 1080
Protocol: SOCKS5
اکنون می‌توانید مرورگر یا برنامه‌ای را که از SOCKS5 پشتیبانی می‌کند، با مشخصات زیر تنظیم کنید:
Proxy Type: SOCKS5
Proxy Host: 127.0.0.1
Proxy Port: 1080
پنجره برنامه VeilRoute باید هنگام استفاده از پراکسی باز بماند.
۱۲. تست اتصال
در منوی برنامه گزینه 8 برای آزمایش ترافیک HTTP از طریق رله وجود دارد.
این گزینه می‌تواند IP خروجی مشاهده‌شده را گزارش کند.
همچنین برای بررسی دسترسی TCP به سرور از PowerShell استفاده کنید:
Test-NetConnection YOUR_SERVER_IP -Port 8443
اگر خروجی شامل این مقدار باشد:
TcpTestSucceeded : True
یعنی اتصال TCP به پورت برقرار شده است.
این تست به‌تنهایی موفقیت TLS یا عملکرد SOCKS5 را تأیید نمی‌کند.
۱۳. دستورات مدیریت سرور
مشاهده وضعیت:
sudo systemctl status veilroute --no-pager
راه‌اندازی مجدد:
sudo systemctl restart veilroute
توقف سرویس:
sudo systemctl stop veilroute
شروع سرویس:
sudo systemctl start veilroute
مشاهده زنده لاگ‌ها:
sudo journalctl -u veilroute -f
بررسی پورت:
sudo ss -lntp | grep 8443
۱۴. رفع خطاهای متداول
خطای Pairing Timeout
ابتدا وضعیت سرویس را بررسی کنید:
sudo systemctl status veilroute --no-pager
سپس:
sudo ss -lntp | grep 8443
از ویندوز نیز اجرا کنید:
Test-NetConnection YOUR_SERVER_IP -Port 8443
موارد زیر را بررسی کنید:
پورت در AWS Security Group باز باشد.
پورت در UFW باز باشد.
IP یا دامنه صحیح باشد.
اثر انگشت مربوط به همان سرور باشد.
کد Pairing معتبر و مصرف‌نشده باشد.
نسخه کلاینت و سرور با یکدیگر مطابقت داشته باشند.
خطای Fingerprint Mismatch
اثر انگشت را مجدداً روی VPS دریافت کنید:
sudo openssl x509 -in /etc/veilroute/server.crt -noout -fingerprint -sha256
تمام کاراکترها را با مقدار کلاینت مقایسه کنید.
پورت باز است ولی Pairing انجام نمی‌شود
این دستور:
Test-NetConnection YOUR_SERVER_IP -Port 8443
فقط دسترسی TCP را آزمایش می‌کند و موفقیت TLS یا پروتکل VeilRoute را تضمین نمی‌کند.
لاگ سرور را بررسی کنید:
sudo journalctl -u veilroute -n 50 --no-pager
SOCKS5 کار نمی‌کند
ابتدا Pairing را انجام دهید.
گزینه 1 را انتخاب کنید.
برنامه را هنگام استفاده باز نگه دارید.
در مرورگر، SOCKS5 را انتخاب کنید، نه HTTP Proxy.
آدرس و پورت را دقیق وارد کنید: 127.0.0.1:1080
۱۵. نکات امنیتی مهم
فقط از سرورهایی استفاده کنید که مالک آن هستید یا مجوز استفاده از آن‌ها را دارید.
کلید SSH و کلید خصوصی TLS را منتشر نکنید.
کد Pairing و توکن دستگاه را محرمانه نگه دارید.
فایل client.json را در GitHub قرار ندهید.
پراکسی محلی را روی 127.0.0.1 نگه دارید و آن را برای شبکه عمومی باز نکنید.
دسترسی پورت VPS را تا حد امکان به IPهای مورد اعتماد محدود کنید.
این نسخه یک پراکسی TCP مبتنی بر SOCKS5 است، نه VPN کامل سیستمی.
UDP، DNS سراسری و تمام ترافیک دستگاه از مسیر این رله عبور نمی‌کنند.
اجزای Wintun و TUN در این نسخه آزمایشی هستند و به مسیر عملیاتی رله متصل نشده‌اند.
این نسخه هنوز ممیزی امنیتی مستقل نشده و برای کاربردهای حساس و تولیدی بدون بررسی امنیتی توصیه نمی‌شود.
پایان آموزش
اگر مراحل بالا را به‌ترتیب انجام دهید، ابتدا سرور VeilRoute روی Ubuntu اجرا می‌شود، سپس کلاینت ویندوز با استفاده از IP یا دامنه، پورت، اثر انگشت TLS و کد یک‌بارمصرف جفت می‌شود و در نهایت پراکسی محلی SOCKS5 روی 127.0.0.1:1080 در دسترس قرار می‌گیرد.
پروژه: VeilRoute Beta 1.2
نوع اتصال: TLS over TCP
پروکسی محلی: SOCKS5
پورت پیش‌فرض سرور: 8443
پورت پیش‌فرض محلی: 1080
