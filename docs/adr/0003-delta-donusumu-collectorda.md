# 0003 — Kümülatiften delta'ya dönüşüm Collector'da

**Durum:** Kabul edildi

## Bağlam

Veritabanı dakikalık değerler saklıyor. Micrometer ve OpenTelemetry SDK'ları varsayılan olarak kümülatif değer gönderir. Ekiplerin her serviste delta modunu doğru ayarlaması elle yazılan kodda unutulabilecek bir ayardır.

## Karar

Collector'daki `cumulativetodelta` işlemcisi kümülatif değerleri dakikalık farka çevirir. Servislerde sayaç modu ayarlanmaz; delta gönderen servisler de sorunsuz işlenir.

## Sonuçlar

- Sözleşmeden bir kural ekiplerin üzerinden kalkar.
- Servis yeniden başlatıldığında sayaç sıfırlanması merkezde algılanır.
- Collector her serinin son değerini hafızada tutar, yani durumludur. Collector yeniden başladığında her seri için bir aralık kaybolur. Birden fazla Collector çalıştırılırsa aynı servis instance'ının verisi hep aynı Collector'a gitmelidir.

## Değerlendirilen alternatifler

- **Servislerin delta göndermesini zorunlu kılmak.** Collector durumsuz olur ve serbestçe çoğaltılabilir, ancak her ekibin bu ayarı doğru yapması gerekir. Collector yedekliliği gündeme geldiğinde yeniden değerlendirilecek.
- **Ingest'te dönüşüm.** Platformun her serinin son değerini tutması gerekir; platform yeniden başlatmaları ve çoğaltılması zorlaşır.
