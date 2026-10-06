# 0004 — Ham noktaların saklanması

**Durum:** Kabul edildi

## Bağlam

SDK'ların gönderim anları dakikaya hizalı değildir. Servis yeniden başlatıldığında aynı dakikaya iki farklı nokta düşebilir; Collector, cevabı alamadığı bir paketi tekrar gönderebilir. Dakikalık satıra doğrudan yazıldığında çakışmada "yok sayma" yeniden başlatmada veri kaybettirir, "toplama" ise tekrar gönderimde veriyi iki katına çıkarır.

## Karar

Ingest her noktayı aralığının başlangıç ve bitiş zamanıyla birlikte ham nokta tablosuna yazar. Benzersiz anahtar bu iki zamanı içerir. Dakikalık görünüm TimescaleDB sürekli toplamasıyla ham tablodan üretilir; alarm ve panel bu görünümden okur.

## Sonuçlar

- Tekrar gönderilen paketler yok sayılır, farklı noktalar korunur; iki durumda da sonuç doğrudur.
- Ham veri 7 gün tutulur; instance bazında inceleme bu süre içinde mümkündür.
- Satır sayısı dakikalık tabloya göre instance sayısı oranında fazladır.

## Değerlendirilen alternatifler

- **Dakikalık satıra yaz, çakışmada yok say.** Yeniden başlatmada veri kaybı.
- **Dakikalık satıra yaz, çakışmada topla.** Tekrar gönderimde çift sayım.
