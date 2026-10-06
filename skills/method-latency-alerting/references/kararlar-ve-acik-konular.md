# Mimari karar kayıtları (ADR)

Her kayıt bir kararın bağlamını, kararın kendisini, sonuçlarını ve değerlendirilen alternatifleri anlatır. Bir karar değiştiğinde eski kayıt silinmez; durumu "yerini aldı" olarak güncellenir ve yeni kayıt eklenir.

| No | Karar | Durum |
|---|---|---|
| 0001 | Servislerle merkez arasında ortak protokol olarak OTLP | Kabul edildi |
| 0002 | Ortada hazır bir OpenTelemetry Collector | Kabul edildi |
| 0003 | Kümülatiften delta'ya dönüşüm Collector'da | Kabul edildi |
| 0004 | Ham noktaların saklanması, dakikalık görünümün onlardan üretilmesi | Kabul edildi |
| 0005 | Alarm kurallarının veritabanında tutulup panelden yönetilmesi | Kabul edildi |
| 0006 | Bildirimlerde transactional outbox | Kabul edildi |
| 0007 | Paylaşılan paket ve merkezi konfigürasyon yok; elle enstrümantasyon ve sözleşme | Kabul edildi |


---

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

---

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

---

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

---

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

---

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

---

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

---

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

---

# 09 — Açık konular

Henüz karar verilmemiş veya ileriye bırakılmış konular.

## Sessizlik alarmı

Bir servis veri göndermeyi tamamen bırakırsa (çöktü, ölçüm kodu kaldırıldı, Collector'a erişemiyor) bugünkü kurallar bunu fark etmez; "en az çağrı" koşulu sağlanmadığı için değerlendirme yapılmaz. Kurallara "pencerede hiç veri yok" türünde bir ölçü eklenmesi önerilir. Karar verilmesi gereken: hangi servisler için zorunlu olacağı ve gece düşük trafikte yanlış alarm üretmemesi için nasıl ayarlanacağı.

## Yüzdelik değerler (p95, p99)

Ortalama, uzun kuyruk gecikmelerini gizler. Sözleşme zaten histogram tipinde olduğu için yüzdelikler eklenebilir; bunun için üç dilde de **aynı sabit kova sınırlarının** (örneğin 5, 10, 25, 50, 100, 250, 500 ms, 1, 2,5, 5, 10 sn) kullanılması, ham nokta tablosunun kova sayaçlarını da saklaması ve dakikalık görünümün kova sayaçlarını toplaması gerekir. Veri hacmi kova sayısı kadar artar.

## Collector yedekliliği

Tek Collector, veri kaybının tek kalıcı kaynağıdır. İki instance çalıştırıldığında cumulativetodelta adımı nedeniyle aynı servis instance'ının verisi hep aynı Collector'a gitmelidir. Seçenekler: servislerin önünde kaynak bazlı yapışkan yönlendirme yapan bir yük dengeleyici, ya da iki katmanlı Collector (ön katmanda servis kimliğine göre yönlendiren loadbalancing exporter, arka katmanda delta dönüşümü). Alternatif olarak servislerin delta göndermesi zorunlu hale getirilirse Collector durumsuz olur ve serbestçe çoğaltılabilir; bu da sözleşmeye bir kural daha ekler.

## Güvenlik

- Servislerden Collector'a gelen veride kimlik doğrulama yok. İç ağda kabul edilebilir; gerekirse Collector'da başlık tabanlı anahtar veya mTLS.
- Collector'dan platforma: başlıkta API anahtarı.
- Panel ve Query + Rule API için kullanıcı kimlik doğrulaması ve kural değişikliklerinin denetim kaydı.

## Ortam ayrımı

Test ve üretim ortamları aynı platformu kullanacaksa kaynak bilgilerine bir ortam etiketi (`deployment.environment`) eklenmeli ve kurallar ortam bazında tanımlanmalı. Aksi halde test trafiği üretim alarmlarını etkiler.

## Instance bazında alarm

Bugün kurallar servis düzeyinde çalışır. Tek bir kopyanın bozulduğu durumlar (örneğin bir makinedeki disk sorunu) servis ortalamasında kaybolabilir. Gerekirse dakikalık görünüme instance kırılımı eklenir; veri hacmi instance sayısı kadar artar.

## RabbitMQ

Bildirim kanalları ayrı servislere bölündüğünde veya alarm olaylarını başka sistemler de dinlemek istediğinde outbox relay'in arkasına eklenir. Ayrıntı: [06 — Alarm ve bildirim](alarm-ve-bildirim.md#ileride-rabbitmq).

## Saklama süreleri

7 gün ham, 30 gün dakikalık başlangıç değerleridir. Uzun dönem trend analizi istenirse dakikalık görünümden saatlik bir görünüm türetilip daha uzun saklanabilir.
