# 0006 — Bildirimlerde transactional outbox

**Durum:** Kabul edildi

## Bağlam

Alarm durumu değiştiğinde bildirim gönderilmesi gerekir. Bildirimi doğrudan alarm turunun içinden göndermek, dış sistemlerin (SMTP, SMS gateway) yavaşlığını alarm değerlendirmesine taşır; durum kaydı ile gönderim arasında tutarsızlık riski doğurur.

## Karar

Alarm motoru durum değişikliğini ve bildirim kayıtlarını aynı transaction içinde yazar. Ayrı bir dağıtıcı outbox tablosunu 5 saniyede bir okuyup kanallara gönderir; kayıtları kilitleyerek alır, kilitli kayıtları atlar, başarısızlıkta artan aralıklarla tekrar dener.

## Sonuçlar

- Durum kaydedilip bildirim kaybolmaz; platform çökse bile bildirim yeniden başlangıçta gönderilir.
- Kanallar birbirinden etkilenmez; her kanal için ayrı kayıt vardır.
- Bildirime en fazla 5 saniye gecikme eklenir (LISTEN/NOTIFY ile kaldırılabilir).
- Mesaj kuyruğu gerekmez. RabbitMQ gerekirse outbox'un arkasına, relay ile eklenir.

## Değerlendirilen alternatifler

- **Alarm turunda doğrudan gönderim.** Basit ama dış sistem yavaşlığı alarm turunu bloklar; çökmede bildirim kaybolabilir.
- **Transaction içinde RabbitMQ'ya yayın.** Veritabanı ile kuyruk arasında tutarsızlık (dual write).
- **Veritabanı eklentisiyle doğrudan AMQP yayını.** Rollback durumunda mesaj gitmiş olabilir; bakım durumu zayıf.
