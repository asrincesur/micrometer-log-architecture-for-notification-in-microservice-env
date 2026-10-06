# 0007 — Paylaşılan paket ve merkezi konfigürasyon yok

**Durum:** Kabul edildi. Daha önce değerlendirilen "Nexus'tan dağıtılan ortak starter ve merkezi konfigürasyon servisi" yaklaşımının yerini aldı.

## Bağlam

Ortam bir girişim mikroservis ortamına benzer: paylaşılan paket deposu ve merkezi konfigürasyon servisi yok. Servisler farklı dillerde. Ölçüm noktasının koda eklenmesi kabul edilebilir.

## Karar

- Ölçüm kodu her serviste ekip tarafından elle yazılır.
- Paylaşılan tek şey ölçüm sözleşmesi dokümanıdır.
- Her servis kendi ayarını tutar: Java'da `application.yml`, Go ve Python'da OpenTelemetry standart environment değişkenleri.
- Elle yazılan koddaki olası farklılıklar merkezde (Collector filtresi ve delta dönüşümü, Ingest normalizasyonu ve sözleşme ihlali kaydı) tolere edilir veya görünür kılınır.

## Sonuçlar

- Bağımlılık ve dağıtım altyapısı gerekmez; her ekip kendi hızında ilerler.
- Ölçüm noktası değişikliği o servisin yeniden deploy edilmesini gerektirir.
- Sözleşme uyumu zamanla kayabilir; paneldeki sözleşme ihlalleri listesi bunu görünür kılar.

## Değerlendirilen alternatifler

- **Ortak starter / kütüphane.** Otomatik yakalama ve merkezi metot listesi sağlar, ancak her dil için ayrı paket, paket deposu ve sürüm yönetimi gerektirir.
- **Merkezi konfigürasyon servisi.** Ölçüm listesini restart'sız değiştirmeyi kolaylaştırır, ancak ortamda yok ve sadece bu ihtiyaç için kurulması orantısız.
