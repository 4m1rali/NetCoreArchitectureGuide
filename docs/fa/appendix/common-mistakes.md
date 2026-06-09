# پیوست د — ۳۰ اشتباه رایج

> از این خطاها در استقرارهای Production Multi-WAN اجتناب کنید.

---

| # | اشتباه | پیامد | رفع |
|---|--------|-------|-----|
| 1 | ECMP + masquerade NAT | Sessionها تصادفی می‌شکنند | از PCC استفاده کنید |
| 2 | بدون check-gateway روی Routeها | WAN مرده فعال می‌ماند | `check-gateway=ping` اضافه کنید |
| 3 | FastTrack با PCC | Mangle Bypass می‌شود | FastTrack را غیرفعال کنید |
| 4 | passthrough=yes گم شده | فقط اولین بسته Mark می‌شود | همیشه passthrough=yes تنظیم کنید |
| 5 | قوانین PCC زیر Policy rules | VoIP از PCC عبور می‌کند | Policy rules اول |
| 6 | Masquerade سراسری واحد | تداخل NAT در Return | Masquerade به‌ازای هر Interface |
| 7 | LTE در Classifier PCC | قبض داده عظیم | LTE فقط Failover |
| 8 | بدون MSS clamp روی PPPoE | بسته‌های بزرگ Fail می‌شوند | MSS clamp 1440 |
| 9 | Interfaceهای WAN در Bridge | Multi-WAN Route نمی‌شود | Router mode روی WAN |
| 10 | بدون قانون Anti-loop | Loop مسیریابی در Failover | Drop forward WAN-to-WAN |
| 11 | بدون Backup قبل از تغییر | Config غیرقابل بازیابی | Export قبل از هر تغییر |
| 12 | تست در Production | Outage در ساعات کاری | تست Lab اول |
| 13 | Distance یکسان روی Routeهای Failover | اولویت غیرقابل پیش‌بینی | distance=1,2,3 |
| 14 | بدون Recursive route برای PPPoE | Gateway unreachable | Interface به‌عنوان Gateway |
| 15 | DNS مستقیم به ISP | Resolution ناسازگار | Force DNS از طریق روتر |
| 16 | add-default-route=yes پیش‌فرض روی DHCP | DHCP Routeهای شما را Override می‌کند | add-default-route=no |
| 17 | Classifier N/M اشتباه | WAN در توزیع گم شده | N=تعداد WAN، M=0 تا N-1 |
| 18 | بدون Netwatch روی Routeهای PCC | PCC به WAN مرده می‌فرستد | Script disable Netwatch |
| 19 | Firewall established را Drop می‌کند | همه Sessionها می‌شکنند | Accept established اول |
| 20 | Winbox باز روی WAN | نقض امنیت | محدود به LAN/VPN |
| 21 | NTP پیکربندی نشده | Logها برای Debug بی‌فایده | NTP را فعال کنید |
| 22 | Connection tracking خیلی کوچک | Drop در ساعات Peak | max-entries را افزایش دهید |
| 23 | VRRP بدون preempt | Backup پس از Recovery Master می‌ماند | preempt=yes |
| 24 | BGP بدون Route filter | Full table روتر را می‌کشد | Filter فقط default |
| 25 | QoS قبل از Mangle PCC | ترتیب Classification اشتباه | PCC اول، QoS دوم |
| 26 | VPN بدون Policy route | ترافیک Tunnel PCC-balanced | Force routing-mark |
| 27 | بدون Monitoring | شکست‌ها توسط کاربران کشف می‌شود | SNMP + Netwatch |
| 28 | Upgrade بدون Backup | Rollback ممکن نیست | Export + binary backup |
| 29 | نادیده گرفتن تفاوت MTU | شکست‌های جزئی مرموز | MTU + MSS به‌ازای هر Interface |
| 30 | لایسنس Level 3 برای BGP | BGP در دسترس نیست | ارتقا به Level 4+ |

---

## طبقه‌بندی شدت

| شدت | اشتباهات |
|-----|----------|
| **CRITICAL** (Outage فوری) | #1, #2, #4, #6, #9, #16, #19 |
| **HIGH** (خدمت تخریب‌شده) | #3, #5, #7, #8, #14, #18, #22, #26 |
| **MEDIUM** (امنیت/پایداری) | #10, #15, #20, #24, #27, #29 |
| **LOW** (عملیاتی) | #11, #12, #13, #17, #21, #23, #25, #28, #30 |

---

**[← FAQ](faq.md)** | **[Cheat Sheet →](cheat-sheet.md)**
