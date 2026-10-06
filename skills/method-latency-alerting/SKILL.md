---
name: method-latency-alerting
description: >-
  Çok dilli (Java, Go, Python) mikroservislerde seçilen metotların çalışma süresini OTLP ile toplayan,
  OpenTelemetry Collector üzerinden metrics-platform'a ileten, TimescaleDB'de saklayan ve 1 dakika–12 saat
  pencerelerde eşik alarmı üretip email/SMS/push/webhook bildirimi gönderen platformun mimari bilgisi. Bu
  mimariyle ilgili her soruda kullan: yeni bir servisi sisteme eklemek, ölçüm sözleşmesi (method.duration,
  code.namespace, code.function, outcome), Micrometer OTLP veya OpenTelemetry SDK ayarları, Collector YAML'ı,
  Ingest davranışı, ham nokta ve dakikalık görünüm, alarm kuralı ve pencere seçimi, durum makinesi, outbox,
  bildirim gecikmesi, "veri görünmüyor" sorunları, RabbitMQ eklenmesi veya mimari değişiklik önerilerinin
  değerlendirilmesi. Kullanıcı açıkça bu platformun adını vermese bile metot süresi izleme, eşik alarmı, OTLP,
  Collector veya bu bileşenlerden biri konuşuluyorsa bu skill'i kullan.
---

# Metot Süresi İzleme ve Eşik Alarm Platformu

Bu skill, bir mimari tasarımın özünü taşır. Sorulara bu tasarıma göre cevap ver; tasarımla çelişen bir öneri gelirse çelişkiyi ve ilgili kararı açıkça belirt.

## Yanıt biçimi

- Varsayılan olarak mimari düzeyde ve konfigürasyonla anlat. Kullanıcı açıkça istemedikçe uygulama kodu yazma; YAML konfigürasyonu, tablolar ve akış açıklamaları yeterlidir.
- Süreleri ve sayıları somut ver (60 sn, ≤5 sn, 1–2,5 dk). Örnek gerekiyorsa `references/` içindeki örnek senaryoları kullan.
- Bir değişikliği değerlendirirken hangi kararı (ADR) etkilediğini söyle ve bedelini açıkla.

## Sistem tek paragrafta

Servisler (Java'da Micrometer + OTLP registry, Go ve Python'da OpenTelemetry SDK) ekiplerin elle işaretlediği metotların süresini bellekte toplar ve 60 saniyede bir OTLP/HTTP ile `otel-collector:4318`'e gönderir. Collector (`otelcol-contrib`, hazır bileşen, YAML ile yapılandırılır) sırasıyla memory_limiter, `method.duration` dışını atan filtre, kümülatiften delta'ya dönüşüm ve en fazla 5 saniyelik batch uygular; otlphttp exporter ile metrics-platform'un `/v1/metrics` uç noktasına gönderir, başarısızlıkta 30 dakikaya kadar diskte kuyruklar. metrics-platform (Spring Boot) Ingest modülü noktaları ham olarak `method_points` tablosuna idempotent yazar; TimescaleDB sürekli toplamasıyla `method_stats_1m` dakikalık görünümü üretilir. Alarm motoru her dakika başında kuralları pencerelere göre değerlendirir, durum makinesini işletir, bildirim gerekiyorsa durum ve outbox kaydını tek transaction'da yazar. Outbox dağıtıcı 5 saniyede bir bekleyen kayıtları email, SMS, push veya webhook'a gönderir. React panel grafikleri, kuralları ve sözleşme ihlallerini gösterir.

## Değişmez kurallar

Bu kurallar mimarinin temelidir. Bir öneri bunlardan birini bozuyorsa bunu açıkça söyle.

1. **Paylaşılan kod, paket deposu ve merkezi konfigürasyon servisi yok.** Ölçüm kodu her serviste elle yazılır; ortak olan tek şey ölçüm sözleşmesidir. (ADR 0007)
2. **Protokol OTLP/HTTP.** Java'da Micrometer şarttır. (ADR 0001)
3. **Servisler yalnızca Collector'ı bilir.** Platforma doğrudan gönderim yok. (ADR 0002)
4. **Sayaç modu servislerde ayarlanmaz; delta dönüşümü Collector'dadır.** Bu yüzden Collector durumludur. (ADR 0003)
5. **Ingest ham noktayı aralığıyla saklar; benzersiz anahtar başlangıç ve bitiş zamanını içerir.** Dakikalık satıra doğrudan yazılmaz. (ADR 0004)
6. **Ortalama her zaman ağırlıklıdır:** toplam süre ÷ toplam çağrı.
7. **Pencere son tamamlanmış ve verisi gelmiş dakikada biter** (`window-lag` 1 dk).
8. **Alarm kuralları veritabanında, panelden yönetilir.** (ADR 0005)
9. **Bildirim yalnızca outbox üzerinden gider; alarm motoru dış sisteme doğrudan çağrı yapmaz.** (ADR 0006)
10. **Ingest geçici hatalarda 503 döner, asla 400 dönmez;** 400 Collector'ın veriyi kalıcı olarak atmasına yol açar.

## Ölçüm sözleşmesi

| Alan | Değer |
|---|---|
| Metrik adı | `method.duration` |
| Tip | Histogram (count, sum, max) |
| Birim | `ms` (`s` gelirse Ingest çevirir) |
| Gönderim aralığı | 60 sn |
| Resource | `service.name`, `service.instance.id` (her kopya ve worker için benzersiz) |
| Etiketler | `code.namespace` (sınıf/tip), `code.function` (metot), `outcome` (`SUCCESS`/`ERROR`) — başka etiket yok |
| Hedef | `http://otel-collector:4318/v1/metrics` |

Sınırsız değer alabilen bilgiler (kullanıcı, istek, sipariş kimliği) asla etiket olamaz.

## Görev: yeni bir servisi sisteme eklemek

1. Dili belirle ve [references/konfigurasyon.md](references/konfigurasyon.md) içindeki ilgili bölümü uygula:
   - **Java:** Micrometer OTLP registry bağımlılığı; `application.yml` içinde `management.otlp.metrics.export.url`, `step: 60s`, `base-time-unit: milliseconds`, `management.opentelemetry.resource-attributes."[service.instance.id]"`. Metotlarda `method.duration` adlı ve üç etiketli `Timer`.
   - **Go:** OpenTelemetry Go SDK + OTLP/HTTP metrik exporter; `OTEL_*` environment değişkenleri; metodun başına süreyi kaydeden tek satır; ölçülen metotlarda isimlendirilmiş dönüş hatası.
   - **Python:** OpenTelemetry Python SDK + OTLP/HTTP metrik exporter; `OTEL_*` environment değişkenleri; ekibin yazdığı dekoratör (senkron ve async ayrı). Çok worker'lı sunucularda instance kimliğine süreç numarası ekle ve sağlayıcıyı fork'tan sonra kur.
2. [references/olcum-sozlesmesi.md](references/olcum-sozlesmesi.md) içindeki uyum kontrol listesinden geç.
3. Servis açıldıktan birkaç dakika sonra panelde metotların göründüğünü ve sözleşme ihlali listesinde olmadığını doğrula.
4. Gerekli alarm kurallarını panelden tanımla.

## Görev: alarm kuralı tasarlamak

Kuralın alanları: servis, sınıf (boş = tümü), metot (boş = tümü), ölçü (ortalama, en yüksek, hata oranı, saniyedeki çağrı), karşılaştırma, eşik, pencere (1 dk–12 sa), en az çağrı, art arda ihlal, tekrar bildirim aralığı, önem, kanallar.

| Pencere | Kullanım | Öneri |
|---|---|---|
| 1–5 dk | Ani, sert bozulmalar | Art arda ihlal ≥ 2 |
| 10–30 dk | Kalıcı yavaşlamalar | Çoğu kural için varsayılan |
| 1–12 sa | Yavaş kötüleşen trendler | En az çağrıyı yüksek tut |

"En az çağrı" altında değerlendirme yapılmaz ve durum değişmez. Aynı metoda farklı pencerelerle birden fazla kural tanımlanabilir. Hesap ve durum makinesi ayrıntısı: [references/alarm-ve-bildirim.md](references/alarm-ve-bildirim.md).

## Görev: bildirim gecikmesini açıklamak

En kötü durum: dakika biter → servis ≤ 60 sn → Collector ≤ 5 sn → Ingest ms → alarm turu ≤ 60 sn → outbox ≤ 5 sn → bildirim. Toplam ≈ 1–2,5 dk; art arda ihlal 1'den büyükse her ek tur +1 dk; email/SMS gateway süresi ayrıca eklenir. Gönderim anları dakikaya hizalı değildir; kova, aralığın başlangıç dakikasıdır. Örnekler: [references/zamanlama-ve-gecikme.md](references/zamanlama-ve-gecikme.md).

## Görev: "veri görünmüyor" sorununu teşhis etmek

Sırayla kontrol et:

1. **Collector ayakta mı?** Sağlık kontrolü `:13133`. Kapalıysa servis verisi o süre boyunca kaybolur.
2. **Servis Collector'a gönderebiliyor mu?** Servis loglarında gönderim hatası; adres `otel-collector:4318/v1/metrics` mi, aralık 60 sn mi?
3. **Metrik adı tam olarak `method.duration` mı?** Değilse Collector filtresi atar; iz bırakmaz.
4. **Collector platforma gönderebiliyor mu?** Exporter hata logları, kuyruk doluluğu. Ingest 503 dönüyorsa veritabanına bak.
5. **Sözleşme ihlali var mı?** Paneldeki ihlal listesi: eksik etiket, bilinmeyen birim, beklenmeyen `outcome`.
6. **Instance kimliği benzersiz mi?** Aynı kimlikle iki kopya grafikte anlamsız sıçramalar üretir.
7. **Zaman doğru mu?** Ham noktada `received_at` ile `end_time` farkı büyükse servis saati yanlıştır (NTP).
8. **Collector yakın zamanda yeniden başladı mı?** Her seri için bir aralık kaybı beklenen davranıştır.
9. **Pencere gecikmesi:** En yeni dakika alarm turuna bir dakika sonra girer; bu hata değildir.

Ayrıntılı tablo: [references/hata-senaryolari.md](references/hata-senaryolari.md).

## Görev: mimari değişiklik önerisini değerlendirmek

Öneriyi değişmez kurallar ve ADR'lerle karşılaştır. Sık gelen öneriler ve tasarımın cevabı:

| Öneri | Değerlendirme |
|---|---|
| Collector'ı kaldırıp doğrudan platforma göndermek | Platform kesintisinde veri kaybı; filtre ve delta dönüşümü servislere geri döner (ADR 0002) |
| Collector'dan doğrudan veritabanına yazmak | PostgreSQL/TimescaleDB için resmi exporter yok; doğrulama ve idempotent yazma kaybolur |
| Dakikalık satıra doğrudan yazmak | Yeniden başlatmada kayıp veya tekrar gönderimde çift sayım (ADR 0004) |
| Bildirimi alarm turundan doğrudan göndermek | Dış sistem yavaşlığı turu bloklar; çökmede kayıp (ADR 0006) |
| RabbitMQ eklemek | Gerekirse outbox'un arkasına relay ile; transaction içinden doğrudan yayın yok; tüketiciler outbox kimliğiyle tekrarları ayıklamalı |
| Collector'ı iki instance yapmak | Delta dönüşümü durumlu: aynı servis instance'ı hep aynı Collector'a gitmeli ya da servisler delta göndermeli (ADR 0003) |
| p95 eklemek | Üç dilde aynı sabit histogram kovaları, kova sayaçlarının saklanması |
| Ortak kütüphane veya merkezi konfigürasyon | Ortamın kısıtlarıyla çelişir (ADR 0007) |

Kararların tamamı ve açık konular: [references/kararlar-ve-acik-konular.md](references/kararlar-ve-acik-konular.md).

## Referans dosyaları

| Dosya | Ne zaman oku |
|---|---|
| `references/olcum-sozlesmesi.md` | Servis ekibine sözleşmeyi anlatırken, uyum kontrolünde |
| `references/konfigurasyon.md` | Java/Go/Python, Collector, platform, veritabanı ve kural ayarları sorulduğunda |
| `references/alarm-ve-bildirim.md` | Pencere hesabı, durum makinesi, outbox, kanallar, RabbitMQ |
| `references/zamanlama-ve-gecikme.md` | Saniye saniye akış, hizalama, yeniden başlatma ve tekrar gönderim örnekleri |
| `references/hata-senaryolari.md` | Bir bileşen düştüğünde ne olur |
| `references/kararlar-ve-acik-konular.md` | ADR özetleri ve açık konular |
