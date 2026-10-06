# 0002 — Ortada hazır bir OpenTelemetry Collector

**Durum:** Kabul edildi

## Bağlam

Servislerin veriyi doğrudan metrics-platform'a göndermesi mümkün. Ancak Micrometer'ın OTLP registry'sinde ve OpenTelemetry SDK'larında gönderilemeyen veri için kalıcı tampon yoktur; platform kapalıyken veri kaybolur. Ayrıca elle yazılan ölçüm kodundaki farklılıkların bir yerde düzeltilmesi gerekir.

## Karar

Servislerle platform arasına `otelcol-contrib` dağıtımıyla bir OpenTelemetry Collector konur. Collector yazılmaz; YAML ile yapılandırılır. Servisler yalnızca Collector'ın sabit adresini bilir.

## Sonuçlar

- Platform kapalıyken veri Collector'ın diskinde kuyruklanır.
- Filtreleme ve delta dönüşümü servislerden alınıp merkeze taşınır.
- Platformun yeri ve sürümü servisleri etkilemez.
- İşletilecek bir bileşen daha eklenir. Collector'ın kendisi kapalıyken veri kaybolur; bu, tasarımdaki tek kalıcı veri kaybı kaynağıdır.

## Değerlendirilen alternatifler

- **Doğrudan platforma gönderme.** Daha az bileşen, ama platform bakımı veya kesintisi sırasında veri kaybı ve servis tarafında düzeltilmesi gereken farklılıklar.
- **Collector'dan doğrudan veritabanına yazma.** PostgreSQL/TimescaleDB için resmi bir exporter yok; hazır exporter'ı olan veritabanlarında (ClickHouse gibi) şema Collector'ın genel şeması olur ve Ingest'teki doğrulama, normalizasyon ve idempotent yazma yapılacak yer kalmaz.
