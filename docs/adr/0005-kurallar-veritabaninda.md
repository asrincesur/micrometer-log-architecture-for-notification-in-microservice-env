# 0005 — Alarm kurallarının veritabanında tutulması

**Durum:** Kabul edildi

## Bağlam

Eşikler, pencereler ve bildirim kanalları operasyonel olarak sık değişir. Merkezi bir konfigürasyon servisi yok.

## Karar

Alarm kuralları veritabanında tutulur ve panelden yönetilir. Değişiklik bir sonraki alarm turunda geçerli olur; deploy gerekmez.

## Sonuçlar

- Ölçüm noktalarına servis ekipleri, alarm kurallarına merkez karar verir.
- Kural değişikliklerinin kim tarafından ne zaman yapıldığının kaydı tutulmalıdır (bkz. açık konular).

## Değerlendirilen alternatifler

- **Kuralları dosyada tutmak (Prometheus kural dosyaları gibi).** Her değişiklik için dosya güncelleme ve yeniden yükleme gerekir; panelden yönetim mümkün olmaz.
