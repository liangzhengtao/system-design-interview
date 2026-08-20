[English](README.md) | [中文](README.zh.md) | [日本語](README.ja.md) | [Français](README.fr.md) | [Español](README.es.md) | [العربية](README.ar.md) | [한국어](README.ko.md) | [Português](README.pt.md) | [Русский](README.ru.md) | [Deutsch](README.de.md)

<div align="center">

<img src=".banner.svg" width="100%" alt="banner">

</div>


# دليل مقابلات تصميم الأنظمة

**تفوّق في مقابلة تصميم الأنظمة. 10 مفاهيم + 5 تصميمات أنظمة كاملة مع مخططات ASCII.**

---

<p align="center">
  <a href="#المفاهيم">المفاهيم</a> •
  <a href="#تصميمات-الأنظمة">تصميمات الأنظمة</a> •
  <a href="#كيفية-الاستخدام">كيفية الاستخدام</a> •
  <a href="#المساهمة">المساهمة</a>
</p>

---

## لماذا هذا الدليل؟

مقابلات تصميم الأنظمة هي الجزء الأكثر تحديًا في المقابلات التقنية في الشركات الكبرى. يوفر هذا الدليل:

- **إطار منظم** — نهج خطوة بخطوة لأي سؤال تصميم
- **مخططات ASCII** — بنية مرئية يمكنك رسمها على السبورة
- **تحليل المفاضلات** — ليس مجرد إجابة واحدة، بل عدة نهج مع الإيجابيات والسلبيات
- **تقدير السعة** — أرقام حقيقية لجعل تصميمك ملموسًا
- **محتوى ثنائي اللغة** — كل ملف يحتوي نسخة صينية كاملة

---

## قائمة المحتويات

- [المفاهيم](#المفاهيم)
- [تصميمات الأنظمة](#تصميمات-الأنظمة)
- [كيفية الاستخدام](#كيفية-الاستخدام)
- [المساهمة](#المساهمة)
- [الرخصة](#الرخصة)

---

## المفاهيم

المعرفة الأساسية التي تحتاجها قبل مواجهة أسئلة تصميم الأنظمة.

| # | المفهوم | المواضيع الرئيسية |
|---|---------|-----------------|
| 1 | [الأساسيات](concepts/fundamentals.md) | قابلية التوسيع، نظرية CAP، نماذج التناسق، زمن الاستجابة مقابل الإنتاجية |
| 2 | [موازنة الحمل](concepts/load-balancing.md) | L4/L7، التدوير الدوري، التجزئة المتسقة، فحوصات الصحة |
| 3 | [التخزين المؤقت](concepts/caching.md) | Redis، CDN، ذاكرة التخزين المؤقت للمتصفح، استراتيجيات التخزين المؤقت |
| 4 | [تصميم قواعد البيانات](concepts/database-design.md) | SQL مقابل NoSQL، الفهرسة، التجزئة، التكرار، ACID |
| 5 | [طوابير الرسائل](concepts/message-queues.md) | Kafka، RabbitMQ، هندسة موجهة بالأحداث، مرة واحدة بالضبط |

### خريطة المفاهيم

```
                        ┌─────────────────────────┐
                        │  System Design           │
                        │  Fundamentals            │
                        └────────────┬────────────┘
                                     │
                 ┌───────────────────┼───────────────────┐
                 │                   │                   │
          ┌──────▼──────┐    ┌──────▼──────┐    ┌──────▼──────┐
          │ Load Balancing│    │   Caching   │    │  Database   │
          └──────┬──────┘    └──────┬──────┘    └──────┬──────┘
                 │                   │                   │
                 └───────────────────┼───────────────────┘
                                     │
                             ┌───────▼───────┐
                             │ Message Queues │
                             └───────┬───────┘
                                     │
                        ┌────────────▼────────────┐
                        │    System Designs        │
                        └─────────────────────────┘
```

---

## تصميمات الأنظمة

حلول كاملة من البداية للنهاية لأسئلة مقابلات تصميم الأنظمة الشائعة.

| # | التصميم | الصعوبة | المفاهيم الرئيسية |
|---|---------|---------|-----------------|
| 1 | [مُقصّر URL](designs/url-shortener.md) | ⭐⭐ | ترميز Base62، التجزئة، إعادة التوجيه، التحليلات |
| 2 | [نظام المحادثة](designs/chat-system.md) | ⭐⭐⭐ | WebSocket، طابور الرسائل، الحضور، التسليم |
| 3 | [خلاصة الأخبار](designs/news-feed.md) | ⭐⭐⭐ | البث المتفرع، الترتيب، الجدول الزمني |
| 4 | [محرك البحث](designs/search-engine.md) | ⭐⭐⭐⭐ | الفهرس المعكوس، الزحف، الترتيب، NLP |
| 5 | [ذاكرة تخزين مؤقت موزعة](designs/distributed-cache.md) | ⭐⭐⭐⭐ | التجزئة المتسقة، التكرار، الإزالة |

---

## كيفية الاستخدام

### للتحضير للمقابلة

```
Step 1: Read concepts/fundamentals.md to build your foundation
Step 2: Study each concept file (2-3 hours total)
Step 3: Practice system designs (1-2 hours each)
Step 4: Time yourself — aim for 45 minutes per design
Step 5: Review trade-offs and practice explaining decisions
```

### ليوم المقابلة

```
┌─────────────────────────────────────────────────┐
│           45-Minute Interview Structure          │
├─────────────────────────────────────────────────┤
│                                                 │
│  0-5 min    Requirements clarification           │
│                                                 │
│  5-10 min   High-level design                   │
│                                                 │
│  10-25 min  Deep dive into core components       │
│                                                 │
│  25-35 min  Scaling & trade-offs                │
│                                                 │
│  35-45 min  Monitoring & failure scenarios       │
│                                                 │
└─────────────────────────────────────────────────┘
```

### إطار RESHADED

كل تصميم في هذا الدليل يتبع إطار **RESHADED**:

```
R - Requirements       Requirements analysis
E - Estimation          Capacity estimation
S - Storage design      Storage design
H - High-level design   High-level design
A - API design          API design
D - Detailed design     Detailed design
E - Evaluation          Trade-off evaluation
D - Distinctive component  Differentiating component
```

---

## خارطة الطريق

- [x] 5 ملفات مفاهيم أساسية
- [x] 5 تصميمات أنظمة كاملة
- [x] محتوى ثنائي اللغة (إنجليزي + صيني)
- [ ] 5 تصميمات إضافية (مُحدد المعدل، نظام الإشعارات، بث الفيديو، زاحف الويب، متجر مفتاح-قيمة)
- [ ] بطاقات Anki للتكرار المتباعد
- [ ] مخططات تفاعلية

---

## المساهمة

نرحب بالمساهمات! راجع [CONTRIBUTING.md](CONTRIBUTING.md) للتفاصيل.

---

## سجل النجوم

إذا وجدت هذا الدليل مفيدًا، يرجى منحه نجمة!

---

## الرخصة

هذا المشروع مرخص بموجب رخصة MIT - راجع ملف [LICENSE](LICENSE) للتفاصيل.

---

<p align="center">
  صُنع بـ ❤️ بواسطة <a href="https://github.com/liangzhengtao">liangzhengtao</a>
</p>
