# 0001 — Ortak protokol olarak OTLP

**Durum:** Kabul edildi

## Bağlam

Servisler Java, Go ve Python ile yazılıyor. Java servislerde Micrometer kullanılması şart. Merkezin tüm dillerden gelen veriyi tek biçimde alması gerekiyor. Micrometer yalnızca JVM'de çalışır.

## Karar

Servislerle merkez arasındaki protokol OpenTelemetry Protocol (OTLP), HTTP üzerinden, protobuf gövdeyle. Java'da Micrometer OTLP registry, Go ve Python'da resmi OpenTelemetry SDK'ları kullanılır.

## Sonuçlar

- Üç dil de protokolü hazır kütüphanelerle konuşur; toplama, sayaç yönetimi ve gönderim mantığı ekipler tarafından yazılmaz.
- Micrometer şartı korunur.
- Protokol, CNCF'nin standardıdır; ileride başka araçlarla (Prometheus uyumlu sistemler, ticari APM'ler) entegrasyon Collector konfigürasyonuyla yapılabilir.
- Ingest'in protobuf çözmesi gerekir; bunun için resmi proto sınıfları kullanılır.

## Değerlendirilen alternatifler

- **Kendi JSON formatımız.** Basit görünür, ancak her ekip bellek içi toplama, dakikalık sıfırlama, tekrar deneme ve kapanışta son veriyi gönderme mantığını kendisi yazmak zorunda kalır. Hata ve tutarsızlık riski yüksektir.
- **Prometheus pull modeli.** Her servisin bir HTTP endpoint'i açması ve merkezin tüm servisleri keşfetmesi gerekir; merkezi servis keşfi olmayan bu ortamda yönetimi zordur.
