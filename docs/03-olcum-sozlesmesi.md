# 03 — Ölçüm sözleşmesi

Servisler arasında paylaşılan kod yoktur. Her ekip kendi servisinde, kendi dilinde ölçüm kodunu yazar. Bu belge, o kodun üretmesi gereken veriyi tanımlar. Sözleşmeye uyan her servis, dili ne olursa olsun, platform tarafından aynı şekilde işlenir.

## Taşıma

| Kural | Değer |
|---|---|
| Protokol | OTLP/HTTP, protobuf gövde |
| Hedef | `http://otel-collector:4318/v1/metrics` |
| Gönderim aralığı | 60 saniye |
| Sayaç modu | Serbest. Micrometer ve OpenTelemetry SDK'larının varsayılanı kümülatiftir; Collector delta'ya çevirir. Delta gönderen servisler de sorunsuz işlenir. |

## Metrik

| Alan | Değer |
|---|---|
| Ad | `method.duration` |
| Tip | Histogram (en azından çağrı sayısı, toplam süre ve en yüksek süre) |
| Birim | `ms` (milisaniye). `s` gelirse Ingest çevirir, ama hedef milisaniyedir. |

## Kaynak (resource) bilgileri

Her gönderim, veriyi kimin ürettiğini taşır.

| Anahtar | Anlamı | Örnek |
|---|---|---|
| `service.name` | Servisin adı | `order-service` |
| `service.instance.id` | Aynı servisin çalışan kopyalarını ayırt eden kimlik | `order-7c9f-2:8080` |

Instance kimliği aynı servisin farklı kopyaları arasında **benzersiz** olmalıdır. Aynı kimlikle iki kopya gönderirse verileri birbirine karışır.

## Nokta etiketleri

| Anahtar | Anlamı | Değerler |
|---|---|---|
| `code.namespace` | Metodun ait olduğu sınıf veya tip | `OrderService` |
| `code.function` | Metodun adı | `createOrder` |
| `outcome` | Çağrının sonucu | `SUCCESS` veya `ERROR` |

Bu üç etiket dışında etiket eklenmemelidir. Özellikle kullanıcı kimliği, istek kimliği, sipariş numarası gibi sınırsız sayıda değer alabilen bilgiler etiket olamaz; her farklı değer ayrı bir seri demektir ve hem servisin belleğini hem veritabanını şişirir.

## İsimlendirme

Sınıf ve metot adları dilin kendi yazım biçimiyle gönderilir. Paket veya modül adı eklenmez.

| Dil | `code.namespace` | `code.function` |
|---|---|---|
| Java | `OrderService` | `createOrder` |
| Go | `OrderService` (struct adı) | `CreateOrder` |
| Python | `PriceService` | `calculate` |

Paket bilgisi gönderilmediği için bir servis içinde aynı adlı iki sınıf olmamalıdır. Alarm kuralları servis adı, sınıf adı ve metot adının birlikte kullanılmasıyla tanımlandığı için farklı servislerdeki aynı adlar çakışmaz.

## Ekiplerin servislerinde yapacakları

### Java servisler

1. Micrometer OTLP registry bağımlılığını eklemek. Spring Boot registry'yi kendisi kurar.
2. `application.yml` içinde Collector adresini, 60 saniyelik gönderim aralığını, milisaniye birimini ve instance kimliğini tanımlamak. Servis adı `spring.application.name` değerinden gelir. Ayrıntı: [04 — Konfigürasyon](04-konfigurasyon.md).
3. Ölçülecek her metotta süreyi `method.duration` adlı ve sözleşmedeki üç etiketi taşıyan bir `Timer` ile kaydetmek. Hata durumunda sonuç `ERROR` olmalıdır.

Spring Boot, bu registry üzerinden JVM ve HTTP metriklerini de gönderir. Ekiplerin bunu kapatmasına gerek yoktur; Collector'daki filtre bu metrikleri düşürür.

### Go servisler

1. OpenTelemetry Go SDK'sını ve OTLP/HTTP metrik exporter'ını eklemek.
2. Uygulama açılışında bir kez ölçüm sağlayıcısını kurmak; adres, aralık ve kimlik bilgileri standart environment değişkenlerinden okunur.
3. Ölçülecek her metodun başına, metot bittiğinde süreyi kaydeden tek bir satır eklemek.
4. Hatanın yakalanabilmesi için ölçülen metotlarda dönüş hatası isimlendirilmiş (named return) olmalıdır. Bu, ekip için bir kodlama kuralıdır.

### Python servisler

1. OpenTelemetry Python SDK'sını ve OTLP/HTTP metrik exporter'ını eklemek.
2. Uygulama açılışında bir kez ölçüm sağlayıcısını kurmak.
3. Süreyi ölçüp kaydeden küçük bir dekoratör yazmak ve ölçülecek metotlara koymak. Dekoratör senkron ve asenkron (`async`) fonksiyonları ayrı ele almalıdır.

**Çoklu süreç uyarısı.** Gunicorn veya uWSGI gibi sunucular aynı makinede birden fazla worker süreci çalıştırır. Her worker ayrı gönderim yapar; bu yüzden instance kimliğine süreç numarası da eklenmelidir. Ayrıca ölçüm sağlayıcısı fork'tan **sonra**, her worker'ın içinde kurulmalıdır; fork öncesinde kurulan arka plan gönderim thread'i worker'lara geçmez ve veri hiç gönderilmez.

## Uyum kontrol listesi

Yeni bir servis devreye alınırken:

- [ ] Metrik adı tam olarak `method.duration`.
- [ ] Birim `ms`.
- [ ] `service.name` doğru ve servis içinde sabit.
- [ ] `service.instance.id` her kopya ve her worker için benzersiz.
- [ ] Her noktada `code.namespace`, `code.function`, `outcome` var; başka etiket yok.
- [ ] Hatalı çağrılar `outcome=ERROR` olarak kaydediliyor.
- [ ] Gönderim aralığı 60 saniye.
- [ ] Servis açıldıktan birkaç dakika sonra panelde metotları görünüyor, sözleşme ihlali listesinde yer almıyor.

## Sözleşme ihlalleri

Ingest, sözleşmeye uymayan veriyi sessizce atmaz:

| Durum | Ingest'in davranışı |
|---|---|
| Birim `s` | Milisaniyeye çevirir, kabul eder |
| Birim tanımsız veya başka bir değer | Noktayı reddeder, ihlal kaydı oluşturur |
| `code.namespace` veya `code.function` eksik | Noktayı reddeder, ihlal kaydı oluşturur |
| `outcome` eksik | `SUCCESS` kabul eder, ihlal kaydı oluşturur |
| `outcome` beklenmeyen değer | Noktayı reddeder, ihlal kaydı oluşturur |
| Beklenmeyen ek etiket | Etiketi yok sayar, noktayı kabul eder, ihlal kaydı oluşturur |

İhlaller servis bazında gruplanarak panelde gösterilir. Bir ekibin kodu sözleşmeye uymuyorsa bu merkezden görülür ve ekibe iletilir.
