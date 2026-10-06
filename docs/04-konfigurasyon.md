# 04 — Konfigürasyon

Bu belge kod içermez; her bileşenin nasıl yapılandırılacağını anlatır. Merkezi bir konfigürasyon servisi olmadığı için her bileşen kendi ayarını kendi tutar. Ayarların çoğunun doğru varsayılanı vardır; sadece ortama göre değişenler verilir.

## 1. Java servisler

Ayarlar servisin kendi `application.yml` dosyasındadır. Spring Boot, Micrometer OTLP registry bağımlılığı eklendiğinde bu ayarlarla registry'yi kendisi kurar.

```yaml
spring:
  application:
    name: order-service                 # OTLP'de service.name olarak gider

management:
  otlp:
    metrics:
      export:
        url: http://otel-collector:4318/v1/metrics
        step: 60s                       # gönderim aralığı
        base-time-unit: milliseconds    # sözleşme: ms
  opentelemetry:
    resource-attributes:
      "[service.instance.id]": ${HOSTNAME:localhost}:${server.port:8080}
```

| Ayar | Neden |
|---|---|
| `spring.application.name` | Platformda servisin görüneceği ad. Servis içinde sabit kalmalı. |
| `management.otlp.metrics.export.url` | Collector'ın adresi. Tüm servislerde aynı. |
| `management.otlp.metrics.export.step` | 60 sn; veritabanı çözünürlüğü dakikalık olduğu için daha kısa tutmak fayda getirmez. |
| `management.otlp.metrics.export.base-time-unit` | Sürelerin milisaniye gitmesi için. |
| `management.opentelemetry.resource-attributes` | Instance kimliği. Anahtarın köşeli parantez içinde yazılması, Spring Boot'un noktalı anahtarı bölmemesi içindir. |

Sayaç modu (`aggregation-temporality`) ayarlanmaz; varsayılan kümülatif modu Collector delta'ya çevirir. Bu ayarlar Spring Boot 3.2 ve sonrası için geçerlidir.

## 2. Go ve Python servisler

OpenTelemetry SDK'ları aşağıdaki standart environment değişkenlerini kendiliğinden okur. Değerler servisin deploy ortamına (Docker Compose, systemd unit dosyası, `.env`) yazılır.

| Değişken | Değer | Açıklama |
|---|---|---|
| `OTEL_SERVICE_NAME` | `pricing-service` | Servis adı |
| `OTEL_EXPORTER_OTLP_METRICS_ENDPOINT` | `http://otel-collector:4318/v1/metrics` | Collector adresi (tam yol) |
| `OTEL_EXPORTER_OTLP_METRICS_PROTOCOL` | `http/protobuf` | OTLP/HTTP, protobuf gövde |
| `OTEL_METRIC_EXPORT_INTERVAL` | `60000` | Gönderim aralığı, milisaniye |
| `OTEL_RESOURCE_ATTRIBUTES` | `service.instance.id=pricing-1` | Instance kimliği |

Örnek `.env`:

```
OTEL_SERVICE_NAME=pricing-service
OTEL_EXPORTER_OTLP_METRICS_ENDPOINT=http://otel-collector:4318/v1/metrics
OTEL_EXPORTER_OTLP_METRICS_PROTOCOL=http/protobuf
OTEL_METRIC_EXPORT_INTERVAL=60000
OTEL_RESOURCE_ATTRIBUTES=service.instance.id=pricing-1
```

Python'da birden fazla worker çalıştıran servislerde instance kimliği environment'ta sabit verilemez, çünkü her worker'ın kendi kimliği olmalıdır. Bu servislerde ekip, worker başlarken süreç numarasını kimliğe ekler; bkz. [03 — Ölçüm sözleşmesi](03-olcum-sozlesmesi.md).

## 3. OpenTelemetry Collector

Collector hazır bir uygulamadır; aşağıdaki YAML dosyasıyla çalıştırılır. `otelcol-contrib` dağıtımı kullanılmalıdır, çünkü filter, cumulativetodelta ve disk kuyruğu bileşenleri contrib'dedir.

```yaml
extensions:
  health_check:
    endpoint: 0.0.0.0:13133
  file_storage:
    directory: /var/lib/otelcol/queue

receivers:
  otlp:
    protocols:
      http:
        endpoint: 0.0.0.0:4318

processors:
  memory_limiter:
    check_interval: 1s
    limit_percentage: 80
    spike_limit_percentage: 20

  filter/sadece-method-duration:
    error_mode: ignore
    metrics:
      metric:
        - 'name != "method.duration"'

  cumulativetodelta:
    include:
      metrics: ["method.duration"]
      match_type: strict

  batch:
    timeout: 5s
    send_batch_size: 1000

exporters:
  otlphttp/platform:
    metrics_endpoint: http://metrics-platform:8090/v1/metrics
    compression: gzip
    timeout: 10s
    retry_on_failure:
      enabled: true
      initial_interval: 5s
      max_interval: 60s
      max_elapsed_time: 30m
    sending_queue:
      enabled: true
      storage: file_storage
      queue_size: 5000

service:
  extensions: [health_check, file_storage]
  pipelines:
    metrics:
      receivers: [otlp]
      processors: [memory_limiter, filter/sadece-method-duration, cumulativetodelta, batch]
      exporters: [otlphttp/platform]
```

| Bölüm | Görevi |
|---|---|
| `extensions.health_check` | `:13133` adresinde sağlık kontrolü; Collector'ın ayakta olup olmadığı buradan izlenir. |
| `extensions.file_storage` | Gönderilemeyen paketlerin diskte tutulacağı klasör. Kalıcı bir volume'a bağlanmalıdır. |
| `receivers.otlp` | Servislerin gönderdiği OTLP/HTTP isteklerini 4318 portunda kabul eder. |
| `processors.memory_limiter` | Aşırı yükte Collector'ın belleğini korur. İşlem hattında ilk sırada olmalıdır. |
| `processors.filter` | Adı `method.duration` olmayan her metriği düşürür. |
| `processors.cumulativetodelta` | Kümülatif sayaçları dakikalık farka çevirir. |
| `processors.batch` | Noktaları en fazla 5 saniye biriktirip tek pakette gönderir. |
| `exporters.otlphttp/platform` | Paketi platforma gönderir; başarısızlıkta 5 saniyeden başlayıp 60 saniyeye kadar artan aralıklarla 30 dakika boyunca tekrar dener; bu sürede paketleri diskte kuyruklar. |
| `service.pipelines` | Adımların sırasını belirler: bellek koruması → filtre → delta dönüşümü → batch → gönderim. |

**Çalıştırma.** Collector container veya servis olarak çalıştırılır; dışarıya 4318 (servislerin göndereceği port) ve 13133 (sağlık kontrolü) açılır. `/var/lib/otelcol/queue` klasörü kalıcı diske bağlanmazsa Collector yeniden başladığında kuyruktaki veri kaybolur.

## 4. metrics-platform

Platformun kendi `application.yml` dosyası. `platform.*` altındaki anahtarlar bu tasarımın önerdiği ayarlardır.

```yaml
server:
  port: 8090

spring:
  datasource:
    url: jdbc:postgresql://metrics-db:5432/metrics
    username: metrics
    password: ${METRICS_DB_PASSWORD}

platform:
  ingest:
    path: /v1/metrics
    max-request-size: 10MB
    accepted-units: [ms, s]

  alarm:
    evaluation-interval: 60s
    window-lag: 1m
    min-window: 1m
    max-window: 12h

  outbox:
    poll-interval: 5s
    batch-size: 50
    max-attempts: 10
    backoff: [30s, 2m, 10m, 30m]
    listen-notify: false

  channels:
    email:
      smtp-host: smtp.sirket.local
      smtp-port: 25
      from: alarm@sirket.local
    sms:
      gateway-url: http://sms-gateway.sirket.local/api/send
      timeout: 5s
    push:
      websocket-endpoint: /ws
    webhook:
      timeout: 5s
```

| Ayar | Açıklama |
|---|---|
| `ingest.accepted-units` | Kabul edilen birimler. `s` milisaniyeye çevrilir; başka birimler sözleşme ihlali sayılır. |
| `alarm.evaluation-interval` | Alarm turlarının aralığı. |
| `alarm.window-lag` | Pencerenin sonunun şimdiki zamandan ne kadar geride tutulacağı. 1 dakika, içinde bulunulan ve verisi henüz gelmemiş dakikayı dışarıda bırakır. |
| `alarm.min-window`, `alarm.max-window` | Kurallarda izin verilen pencere aralığı. |
| `outbox.poll-interval` | Dağıtıcının bekleyen kayıtlara bakma aralığı. |
| `outbox.max-attempts`, `outbox.backoff` | Gönderim tekrar politikası. Son aralık sonraki tüm denemeler için kullanılır. |
| `outbox.listen-notify` | Açılırsa alarm motoru outbox'a yazdığında dağıtıcı veritabanı bildirimiyle anında uyanır; 5 saniyelik bekleme kalkar, polling yedek olarak sürer. |
| `channels.*` | Kanal bağlantı bilgileri. Şifre ve anahtarlar dosyaya değil environment değişkenlerine yazılır. |

## 5. PostgreSQL + TimescaleDB

| Ayar | Değer | Açıklama |
|---|---|---|
| Ham nokta tablosu | Hypertable, 1 günlük bölümler | Zaman bazlı bölümleme; eski veri bölüm bölüm silinir |
| Ham nokta saklama süresi | 7 gün | Dakikalık görünüm hesaplandıktan sonra ham veriye kısa süre ihtiyaç vardır |
| Dakikalık görünüm | Sürekli toplama (continuous aggregate), kova 1 dakika, aralık başlangıcına göre | Alarm ve panel buradan okur |
| Görünüm yenileme | Her 1 dakikada, son 3 saati kapsayacak şekilde | Geç gelen veriler (Collector kuyruğundan çıkanlar) de yansır |
| Gerçek zamanlı toplama | Açık | Henüz yenilenmemiş en yeni noktalar da sorguya dahil olur |
| Dakikalık görünüm saklama süresi | 30 gün | |
| Sıkıştırma (isteğe bağlı) | 1 günden eski ham bölümler | Disk kullanımını azaltır |

Görünüm yenileme aralığı (3 saat), ham veri saklama süresinden (7 gün) kısa olmalıdır; aksi halde silinmiş ham veri görünümü bozar.

## 6. Alarm kuralları

Kurallar panelden tanımlanır ve veritabanında tutulur.

| Alan | Anlamı | Değer aralığı |
|---|---|---|
| Ad | Kuralın okunabilir adı | Serbest metin |
| Servis | Hangi servis | `service.name` değeri |
| Sınıf | Hangi sınıf | Boş bırakılırsa servisteki tüm sınıflar |
| Metot | Hangi metot | Boş bırakılırsa sınıftaki tüm metotlar |
| Ölçü | Neye bakılacak | Ortalama süre, en yüksek süre, hata oranı, saniyedeki çağrı |
| Karşılaştırma | Eşikle nasıl karşılaştırılacak | Büyük, büyük-eşit, küçük, küçük-eşit |
| Eşik | Sınır değer | Süre için ms, hata oranı için 0–1, hız için çağrı/sn |
| Pencere | Değerlendirilecek süre | 1 dakika – 12 saat |
| En az çağrı | Pencerede bu kadar çağrı yoksa değerlendirme yapılmaz | Ör. 100 |
| Art arda ihlal | Kaç turda üst üste aşılırsa alarm | Ör. 2 |
| Tekrar bildirim aralığı | Alarm sürerken hatırlatma sıklığı | Ör. 30 dakika |
| Önem | Bildirim metninde ve panelde | Uyarı, kritik |
| Kanallar | Bildirimin gideceği yerler ve hedefleri | Email adresleri, telefon numaraları, push, webhook |
| Etkin | Kural açık mı | Evet / hayır |

**Örnek kurallar:**

| Kural | Servis / metot | Ölçü | Eşik | Pencere | En az çağrı | Art arda | Kanallar |
|---|---|---|---|---|---|---|---|
| Sipariş yavaşladı | order-service / OrderService.createOrder | Ortalama | > 50 ms | 10 dk | 100 | 2 | SMS, push |
| Sipariş takıldı | order-service / OrderService.createOrder | En yüksek | > 5 000 ms | 5 dk | 1 | 1 | Push |
| Fiyat hesabı hata veriyor | pricing-service / PriceService.calculate | Hata oranı | > 0,02 | 15 dk | 200 | 2 | Email |
| Gece yavaş kötüleşme | order-service / tümü | Ortalama | > 40 ms | 12 sa | 10 000 | 1 | Email |

**Pencere seçimi rehberi:**

| Pencere | Ne yakalar | Dikkat |
|---|---|---|
| 1–5 dk | Ani ve sert bozulmalar | Gürültülüdür; "art arda ihlal" en az 2 olmalı |
| 10–30 dk | Kalıcı yavaşlamalar | Çoğu kural için uygun varsayılan |
| 1–12 sa | Yavaş yavaş kötüleşen trendler | Kısa sıçramaları görmez; "en az çağrı" yüksek tutulmalı |

Aynı metoda farklı pencerelerle birden fazla kural tanımlanabilir.
