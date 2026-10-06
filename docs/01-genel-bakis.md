# 01 — Genel bakış

## Problem

Mikroservis ortamında her servis farklı bir ekip tarafından, farklı dillerde (Java, Go, Python) geliştiriliyor. Servislerin içindeki kritik metotların ne kadar sürede çalıştığını merkezden görmek, belirli zaman aralıklarında bu sürelerin kabul edilebilir sınırların dışına çıkıp çıkmadığını otomatik olarak kontrol etmek ve sınır aşıldığında ilgili kişilere bildirim göndermek isteniyor.

Bugün bu bilgi ya hiç toplanmıyor ya da her ekip kendi yöntemiyle, karşılaştırılamaz biçimde topluyor. Yavaşlamalar çoğu zaman kullanıcı şikayetiyle fark ediliyor.

## Hedefler

1. Herhangi bir dilde yazılmış bir servisin seçilen metotlarının çalışma süresi, çağrı sayısı ve başarı/hata bilgisi merkezde toplanabilmeli.
2. Toplanan veri 1 dakika ile 12 saat arasındaki herhangi bir pencerede gözlemlenebilmeli (grafik, tablo).
3. Pencere bazlı eşik kuralları tanımlanabilmeli: ortalama süre, en yüksek süre, hata oranı, çağrı hızı.
4. Eşik aşıldığında email, SMS, push veya webhook ile bildirim gitmeli; aynı sorun için bildirim yağmuru olmamalı; sorun çözüldüğünde "çözüldü" bildirimi gitmeli.
5. Ölçümün servise maliyeti ihmal edilebilir olmalı; her metot çağrısında ağ isteği yapılmamalı.
6. Merkezdeki bir bileşenin kısa süreli kesintisinde veri kaybolmamalı.

## Kısıtlar

Ortam bir girişim (startup) mikroservis ortamına benzer: olgunlaşmamış, merkezi altyapısı sınırlı.

| Kısıt | Tasarıma etkisi |
|---|---|
| Merkezi konfigürasyon servisi yok | Her servis kendi ayarını tutar (Java'da `application.yml`, Go/Python'da environment değişkenleri). Doğru varsayılanlar önemlidir. |
| Paylaşılan paket deposu (ör. Nexus) yok, ortak kütüphane dağıtılmayacak | Ölçüm kodu her serviste ekip tarafından elle yazılır. Paylaşılan tek şey sözleşme dokümanıdır. |
| Servisler farklı dillerde | Dile bağımsız standart protokol gerekir: OTLP. |
| Java servislerde Micrometer kullanılması şart | Java tarafı Micrometer + Micrometer OTLP registry ile ölçer ve gönderir. |
| Ölçüm noktası koda eklenebilir | Go'da metoda bir satır, Python'da dekoratör, Java'da `Timer` veya `@Timed` kabul edilebilir. Otomatik yakalama (AOP, monkeypatching) gerekmez. |

## Kapsam dışı

- Dağıtık izleme (distributed tracing) ve span'ler.
- Log toplama.
- Tüm metotların otomatik enstrümantasyonu; sadece ekiplerin işaretlediği metotlar ölçülür.
- Her çağrının tek tek saklanması; sadece dakikalık özetler saklanır.
- Yüzdelik değerler (p95, p99). Sözleşme histogram tipinde olduğu için ileride eklenebilir; bkz. [09 — Açık konular](09-acik-konular.md).

## Terimler sözlüğü

| Terim | Anlamı |
|---|---|
| OTLP | OpenTelemetry Protocol. Ölçüm verisini taşıyan, dile ve üreticiye bağımsız standart protokol. HTTP (4318) ve gRPC (4317) üzerinden çalışır. |
| OpenTelemetry Collector | Hazır, açık kaynak bir ara bileşen. OTLP verisini alır, işler ve başka bir yere iletir. Kod yazılmaz, YAML ile yapılandırılır. |
| Kümülatif (cumulative) | Sayaçların servis açıldığından beri biriken toplamı göndermesi. |
| Delta | Sadece son gönderimden bu yana geçen aralığın değerlerinin gönderilmesi. |
| Nokta (data point) | Bir metodun belirli bir zaman aralığındaki çağrı sayısı, toplam süresi ve en yüksek süresi. |
| Kova (bucket) | Noktaların gruplandığı bir dakikalık zaman dilimi. |
| Pencere (window) | Bir alarm kuralının değerlendirdiği zaman aralığı; 1 dakika ile 12 saat arası. |
| Ağırlıklı ortalama | Toplam süre bölü toplam çağrı sayısı. Dakikalık ortalamaların ortalaması değildir. |
| Outbox | Bildirimlerin önce veritabanındaki bir tabloya yazılıp sonra ayrı bir süreç tarafından gönderildiği desen. |
| Sözleşme ihlali | Bir servisin gönderdiği verinin ölçüm sözleşmesine uymaması (eksik etiket, beklenmeyen birim vb.). |
