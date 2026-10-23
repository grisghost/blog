---
title: "حل مشكلة المواقع الحكومية السعودية و App Store مع AdGuard Home"
description: "الدليل التقني الشامل لحل توقف المواقع الحكومية السعودية وتعطل App Store عند استخدام AdGuard Home على سيرفر خارجي."
date: 2026-10-23 12:00:00 +0200
categories: [Self-Hosting, Networking]
tags: [adguard-home, dns, saudi-arabia, app-store, docker, linux, self-hosting]
dir: rtl
---

يُعد استخدام AdGuard Home على سيرفر سحابي خارجي خياراً ممتازاً لحجب الإعلانات وحماية الخصوصية على كافة أجهزتك. ولكن عند استخدام هذا الإعداد داخل المملكة العربية السعودية مع وجود السيرفر في دولة أخرى كأوروبا أو أمريكا، تظهر مشكلتان شائعتان:

1. **توقف المواقع الحكومية والتعليمية السعودية** — مثل أبشر، ونفاذ، حيث ترفض الفتح تماماً أو لا تظهر طلباتها في Query Log.
2. **تعطل تحميل التطبيقات في App Store** — عند محاولة إصلاح مشكلة المواقع السعودية يتوقف المتجر عن التنزيل وتظل دائرة التحميل تدور بلا نهاية.

---

## السبب التقني للمشكلة

تحدث هذه المشكلة نتيجة تصادم ثلاث آليات شبكية في وقت واحد.

### 1. الحظر والتوجيه الجغرافي (Geo-DNS & Geo-Blocking)

المنصات الحكومية السعودية تستخدم جدران حماية تشترط أن يكون المستخدم داخل السعودية. عندما يكون سيرفر AdGuard Home في أوروبا ويطلب عنوان موقع معين، يرى الخادم الحكومي الطلب قادماً من أوروبا فيرفض الاستجابة.

### 2. عزل الشبكة في Docker

عند تثبيت AdGuard Home داخل Docker بالوضع الافتراضي (Bridge Mode)، يقوم Docker بعمل NAT داخلي. النتيجة: AdGuard Home لا يرى الـ IP السعودي الحقيقي لجوالك، بل يرى كل الطلبات قادمة من عنوان وهمي مثل `172.17.0.1`.

### 3. معضلة ECS مع خوادم أبل

لحل مشكلة المواقع السعودية نُفعّل ميزة **EDNS Client Subnet (ECS)** التي ترفق جزءاً من الـ IP السعودي مع طلب DNS. لكن شبكات توزيع المحتوى الخاصة بأبل (`mzstatic.com`، `cdn-apple.com`) تعتمد على Anycast و ISP Peering. عندما يصلها طلب من سيرفر أوروبي يحمل إشارة ECS لشبكة سعودية، يحدث تضارب في التوجيه (Routing Mismatch) يؤدي لتعليق التحميل في App Store.

---

## فلسفة الحل

الحل يعتمد على **Split/Conditional Upstream DNS**:

- **إلغاء عزل الشبكة** — الانتقال من Docker إلى التثبيت المباشر على النظام ليرى AdGuard Home الـ IP الحقيقي.
- **Cloudflare كخادم افتراضي** — لا يمرر ECS افتراضياً، مما يعني أن 99% من المواقع وخوادم App Store ستعمل بسرعة دون أي تداخل جغرافي.
- **Google DNS للمواقع السعودية فقط** — يدعم ECS بالكامل ويُستخدم حصراً عند الطلب على نطاقات `.sa`.

---

## خطوات التطبيق

### الخطوة 1: تثبيت AdGuard Home مباشرة على السيرفر

أوقف حاوية Docker القديمة وثبّت AdGuard Home مباشرة على Linux:

```bash
# إيقاف وحذف الحاوية القديمة
sudo docker stop adguardhome && sudo docker rm adguardhome

# تحرير البورت 53 من systemd-resolved
sudo mkdir -p /etc/systemd/resolved.conf.d
echo -e "[Resolve]\nDNS=1.1.1.1\nDNSStubListener=no" | sudo tee /etc/systemd/resolved.conf.d/adguardhome.conf
sudo ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf
sudo systemctl reload-or-restart systemd-resolved

# تثبيت AdGuard Home
curl -s -S -L https://raw.githubusercontent.com/AdguardTeam/AdGuardHome/master/scripts/install.sh | sh -s -- -v
```

> يمكنك نسخ ملف `AdGuardHome.yaml` من مجلد Docker القديم إلى `/opt/AdGuardHome/` للاحتفاظ بإعداداتك وإحصائياتك.
{: .prompt-tip }

---

### الخطوة 2: ضبط Upstream DNS (الأهم)

افتح لوحة تحكم AdGuard Home ثم انتقل إلى **Settings** > **DNS settings**.

في مربع **Upstream DNS servers** ضع:

```text
https://dns.cloudflare.com/dns-query
tls://dns.cloudflare.com
[/*.sa/]https://dns.google/dns-query
[/*.sa/]tls://dns.google
[/*.gov.sa/]https://dns.google/dns-query
```

في مربع **Bootstrap DNS servers** ضع:

```text
1.1.1.1
8.8.8.8
```

ثم اضغط **Apply**.

---

### الخطوة 3: ضبط EDNS و DNSSEC

في نفس صفحة DNS settings اضبط الخيارات كالتالي:

| الخيار | الحالة |
| :--- | :--- |
| Enable EDNS client subnet | ✅ مفعّل |
| Use custom IP for EDNS | ⬜ معطّل |
| Enable DNSSEC | ⬜ معطّل |
| Disable resolving of IPv6 addresses | ⬜ معطّل |

> أبقِ DNSSEC معطّلاً — بعض النطاقات الحكومية الفرعية تفشل في فحص التوقيع الصارم من خارج السعودية.
{: .prompt-warning }

> أبقِ IPv6 مفعّلاً — ضروري لسرعة تحميل تطبيقات أبل على شبكات 5G.
{: .prompt-info }

اضغط **Save** ثم **Clear DNS cache**.

---

### الخطوة 4: إضافة قواعد الاستثناء

انتقل إلى **Filters** > **Custom filtering rules** وأضف:

```text
! --- استثناء النطاقات الحكومية السعودية ---
@@||sa^$important
@@||gov.sa^$important
@@||edu.sa^$important
@@||moe.gov.sa^$important

```

اضغط **Apply**.

---

## النتيجة

بعد تطبيق هذه الخطوات، فعّل **وضع الطيران** لثوانٍ لمسح الكاش القديم من الجهاز:

- **المواقع الحكومية والتعليمية** ستعمل فوراً عبر Google DNS الذي يُرسل موقعك الجغرافي السعودي الحقيقي.
- **App Store** والتحديثات ستعمل بأقصى سرعة لأن طلبات أبل تمر عبر Cloudflare مباشرة دون أي تشويش من ECS.
