# 07 — Zamanlama ve gecikme

## Adım adım veri yolculuğu

| Adım | Nerede | Ne olur | Ne zaman |
|---|---|---|---|
| 1 | Servis belleği | Her çağrıda süre ilgili seriye eklenir: çağrı sayısı artar, toplam süre büyür, en yüksek süre güncellenir. Ağ yok. | Her çağrıda, nanosaniyeler |
| 2 | Servis → Collector | Tüm serilerin değeri tek OTLP isteğiyle gönderilir | 60 sn'de bir |
| 3 | Collector receiver | İstek çözülür | Anında |
| 4 | Collector filter | `method.duration` dışındaki metrikler atılır | Anında |
| 5 | Collector cumulativetodelta | Her serinin bir önceki değeriyle farkı alınır; servis yeniden başlatıldıysa sayaç sıfırlanması algılanır | Anında |
| 6 | Collector batch + exporter | Noktalar biriktirilip platforma gönderilir; platform kapalıysa diskte bekletilir | En fazla 5 sn |
| 7 | Ingest | Etiketler çıkarılır, birim normalize edilir, sözleşme kontrol edilir, ham nokta idempotent yazılır | Milisaniyeler |
| 8 | Dakikalık görünüm | Ham noktalar dakikalık kovalara toplanır; gerçek zamanlı toplama sayesinde yeni noktalar hemen sorgulanabilir | Anında / her dakika yenileme |
| 9 | Alarm motoru | Pencereler hesaplanır, durum makinesi işletilir, gerekiyorsa outbox kaydı yazılır | Her dakika başı |
| 10 | Outbox dağıtıcı | Bekleyen bildirimler kanallara gönderilir | En fazla 5 sn |

## Uçtan uca gecikme

Bir dakikada oluşan yavaşlığın bildirime dönüşmesi için en kötü durum:

```mermaid
flowchart LR
  a["Dakika biter"] -->|"≤ 60 sn"| b["Servis gönderir"]
  b -->|"≤ 5 sn"| c["Collector iletir"]
  c -->|"ms"| d["Veritabanında"]
  d -->|"≤ 60 sn"| e["Alarm turu"]
  e -->|"≤ 5 sn"| f["Outbox gönderir"]
  f --> g["Bildirim"]
```

Toplam yaklaşık 1 ile 2,5 dakikadır; "art arda ihlal" değeri 1'den büyükse her ek tur bir dakika ekler. Email ve SMS gateway'inin kendi teslim süresi bunun üstüne eklenir. Panelde grafik verisi alarm turunu beklemez; bir dakikanın verisi, dakika bittikten yaklaşık bir dakika sonra görünür.

## Gönderim anları dakikaya hizalı değildir

SDK'lardaki periyodik gönderim "her dakikanın başında" değil, "servis açıldıktan sonra her 60 saniyede bir" çalışır. Servis 10:12:23'te açıldıysa gönderimler her dakikanın 23. saniyesinde olur ve her nokta iki dakikaya yayılan bir aralığı kapsar.

**Örnek: pricing-service, PriceService.calculate, servis 10:12:23'te açıldı.**

| Gönderim anı | Servisin gönderdiği (kümülatif) | Collector'ın ürettiği delta aralığı | Delta değeri | Kova |
|---|---|---|---|---|
| 10:13:23 | 400 çağrı, 12 000 ms | 10:12:23 → 10:13:23 | 400 çağrı, 12 000 ms | 10:12 |
| 10:14:23 | 1 000 çağrı, 30 500 ms | 10:13:23 → 10:14:23 | 600 çağrı, 18 500 ms | 10:13 |
| 10:15:23 | 1 700 çağrı, 51 500 ms | 10:14:23 → 10:15:23 | 700 çağrı, 21 000 ms | 10:14 |
| 10:16:23 | 2 300 çağrı, 69 500 ms | 10:15:23 → 10:16:23 | 600 çağrı, 18 000 ms | 10:15 |

Kova, aralığın başlangıç dakikasıdır. "10:14" olarak görünen satır aslında 10:14:23 ile 10:15:23 arasını temsil eder.

## Aynı kovaya iki nokta düşebilir

Servis yeniden başlatıldığında normalde dakikada bir olan nokta sayısı ikiye çıkabilir. Deploy sırasında servis 10:15:50'de kapanıp 10:15:52'de yeniden açıldı:

| Olay | Delta aralığı | Değer | Kova |
|---|---|---|---|
| Eski süreç kapanırken son veriyi gönderir | 10:15:23 → 10:15:48 | 200 çağrı, 6 000 ms | 10:15 |
| Yeni sürecin ilk gönderimi (10:16:52) | 10:15:52 → 10:16:52 | 550 çağrı, 16 500 ms | 10:15 |

Doğru sonuç, 10:15 kovasında 750 çağrı ve 22 500 ms'dir.

## Collector aynı paketi tekrar gönderebilir

Platform paketi yazdığı halde cevabı Collector'a ulaşmazsa (zaman aşımı), Collector aynı paketi tekrar gönderir. Aynı nokta iki kez gelir.

## Neden ham nokta saklanıyor

İki olası basit yaklaşımın ikisi de bir durumda yanlış sonuç verir:

| Yaklaşım | Yeniden başlatma | Tekrar gönderim |
|---|---|---|
| Dakikalık satıra yaz, çakışmada yok say | 550 çağrı kaybolur (yanlış) | Doğru |
| Dakikalık satıra yaz, çakışmada topla | Doğru | 700 çağrı 1 400 olur (yanlış) |
| **Ham noktayı aralığıyla sakla, dakikalık görünümü ondan üret** | **Doğru** | **Doğru** |

Ham nokta tablosunda benzersiz anahtar aralığın başlangıç ve bitiş zamanını içerir. Tekrar gönderilen paket birebir aynı anahtarı taşıdığı için yok sayılır; yeniden başlatmadaki iki noktanın başlangıç zamanları farklı olduğu için ikisi de saklanır. Dakikalık görünüm bu noktaları kovalarına göre toplar.

## Kaymanın pencerelere etkisi

**Uzun pencereler (10 dakika ve üstü).** Kenarlarda en fazla bir dakikalık kayma olur; ortalama pratikte değişmez.

**1 dakikalık pencere.** "10:14 dakikası" yerine "10:14 civarında başlayan 60 saniye" ölçülür. Kısa bir sıçrama, gönderim anına göre bir önceki veya sonraki satırda görünebilir. Kullanılabilir, ama dakika sınırlarına güvenilmemelidir.

**Alarm turu ile geç gelen veri.** 10:15 kovasındaki nokta 10:16:23 civarında üretilir; 10:17:00 turu 10:15'e kadar olan kovaları kullandığı için bu noktayı yakalar. Gönderim anı dakikanın sonuna denk gelen bir serviste (örneğin her dakikanın 59. saniyesi) nokta, 10:17:00 turundan birkaç saniye sonra gelebilir; o turda eksik kalır, bir sonraki turda pencereye girer. Alarm en kötü durumda bir dakika gecikir, veri kaybolmaz.

**Collector kuyruğundan geç çıkan veri.** Platform bir süre kapalı kaldıysa Collector kuyruğu boşalırken eski noktalar gelir. Dakikalık görünüm son 3 saati her dakika yeniden hesapladığı için bu noktalar görünüme yansır. Kapalı kalınan süre boyunca alarmlar değerlendirilemez; platform açıldıktan sonraki turlar güncel veriyle çalışır.
