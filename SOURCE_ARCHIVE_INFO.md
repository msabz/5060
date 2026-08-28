# source.zip — أرشيف المصدر المُصلَّح

هذا الأرشيف هو شجرة المشروع الكاملة (مجلد قمّي واحد `G1-DirectChat/`) التي يفكّها
‏`.github/workflows/build.yml` ويبني عليها.

- **SHA-256**: `06b3bb5e292cc4fb6a562161cd017a38121d07327e92cfd3539584c5a1f55211`
- **المحتوى**: إصلاح هجرة Make-Before-Break + هوية Aurora + إصلاحات الاختبار الميداني
- **الاختبارات**: 373/373 اختبار Jest أخضر (69 حزمة)

## إصلاحات الجولة الميدانية (2026-08-28)
1. `ChatScreen`: استيراد `Avatar` و`TransportChip` المفقودان — كانا يسقطان
   التطبيق بشاشة حمراء `ReferenceError` عند فتح أي محادثة.
2. `App.js`: إعادة `startPresence` خارج شرط `lanDiscovery.isSupported()` —
   إدخالها فيه كان يقتل طبقة الحضور (UDP) فلا يظهر الأصدقاء ولا يعمل
   QR ولا الإضافة بالرقم ولا الاتصال التلقائي.
3. `signaling.js`: ربط `interface:'wifi'` أصبح حسب الوجهة — عناوين
   ‏192.168.49.x (Wi-Fi Direct) تُترك لموجّه النظام.
4. حُرّاس انحدار جدد: `sourceIntegrity` (no-undef كامل لكل src)،
   ‏`presenceStartupGuard`، `socketInterfacePolicy`.

## الأرشيف السابق
- SHA-256: `20546db71ce4de1a86d955deff5f475bd98657f36d8503efb94dce8e292fc5cd`
  (CI أخضر: 33139458440 / 33139455934 / 33140415646 / 33140417576)
