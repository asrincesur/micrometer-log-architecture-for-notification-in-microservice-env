# 08 — Hata senaryoları

| Senaryo | Etkisi | Sistemin davranışı | Nasıl fark edilir |
|---|---|---|---|
| metrics-platform kapalı | Veri yazılamaz, alarm değerlendirilemez, bildirim gitmez | Collector paketleri diskte kuyruklar ve 30 dakikaya kadar tekrar dener; platform açılınca sırayla iletir. Dakikalık görünüm geç gelen noktaları son 3 saati yenilerken yansıtır. | Collector exporter hata logları; platform sağlık kontrolü |
| Veritabanı kapalı veya meşgul | Platform kapalıyla aynı | Ingest 503 döner, Collector kuyruklar. Alarm turu ve dağıtıcı hata alır, bir sonraki turda tekrar dener. | Platform logları, veritabanı izleme |
| Collector kapalı | **Veri kaybı.** Servislerde tampon yoktur; SDK'lar gönderemedikleri aralığı düşürür. | Servisler çalışmaya devam eder, iş akışı etkilenmez; sadece ölçüm verisi kaybolur. | Collector sağlık kontrolü (`:13133`); servis loglarında gönderim hataları |
| Collector yeniden başladı | Her seri için bir aralık kaybolur | cumulativetodelta her serinin son değerini hafızada tuttuğu için yeniden başlangıçtan sonraki ilk kümülatif değerin farkını hesaplayamaz; o nokta atlanır. Disk kuyruğundaki paketler korunur. | Collector başlangıç logu |
| Collector diski doldu | Kuyruğa alınamayan paketler kaybolur | Exporter yeni paketleri reddeder | Disk izleme |
| Servis yeniden başlatıldı (deploy) | Yok | Eski süreç kapanırken son veriyi gönderir; yeni süreç sıfırdan başlar. Collector sayaç sıfırlanmasını algılar, ham nokta saklama iki noktayı da korur. | — |
| Servis çöktü | Son en fazla 60 saniyenin verisi kaybolur | Kapanış gönderimi yapılamaz | Servis izleme |
| Aynı instance kimliğiyle iki kopya | Veriler karışır, değerler tutarsızlaşır | Collector iki farklı sayacı tek seri sanar | Grafiklerde anlamsız sıçramalar; sözleşme kontrol listesi |
| Sözleşmeye uymayan veri | Nokta reddedilir veya düzeltilir | Ingest ihlali kaydeder, Collector'a kısmi başarı döner | Paneldeki sözleşme ihlalleri listesi |
| SMTP veya SMS gateway kapalı | O kanalın bildirimleri gecikir | Dağıtıcı artan aralıklarla tekrar dener; diğer kanallar etkilenmez, çünkü her kanal ayrı outbox kaydıdır. Deneme sınırı aşılınca kayıt "başarısız" olur. | Paneldeki başarısız bildirimler |
| Alarm turu uzun sürdü | Bir sonraki tur gecikir | Kilit süresi dolmadan ikinci tur başlamaz | Platform logları |
| Servisin saati yanlış | Noktalar yanlış kovaya düşer | Ingest noktanın kendi zaman damgasını kullanır | Ham noktada `received_at` ile `end_time` arasındaki fark |
| Ağ bölünmesi (Collector ↔ platform) | Platform kapalıyla aynı | Collector kuyruklar | Collector exporter hata logları |

## Öncelikli önlemler

1. **Collector'ın sürekli ayakta olması.** Veri kaybının tek kalıcı kaynağı Collector'ın kapalı olmasıdır. Otomatik yeniden başlatma politikasıyla çalıştırılmalı, sağlık kontrolü izlenmelidir.
2. **Collector disk kuyruğunun kalıcı volume'da olması.** Aksi halde Collector yeniden başladığında kuyruk kaybolur.
3. **Tüm makinelerde saat senkronizasyonu (NTP).** Kova ataması noktaların kendi zaman damgasına dayanır.
4. **Ingest'in geçici hatalarda 503 dönmesi.** 400 dönülürse Collector paketi kalıcı olarak atar.
5. **Benzersiz instance kimliği.** Özellikle çok worker'lı Python servislerinde.

## İzlemenin izlenmesi

Platformun kendisi de bozulabilir. En azından şunlar izlenmelidir: Collector sağlık kontrolü, Collector'ın gönderemediği paket sayısı ve kuyruk doluluğu, Ingest'e son verinin ne zaman geldiği, outbox'ta bekleyen ve başarısız kayıt sayısı. Bir servisin tamamen veri göndermeyi bırakmasını yakalayan "sessizlik alarmı" [09 — Açık konular](kararlar-ve-acik-konular.md) belgesindedir.
