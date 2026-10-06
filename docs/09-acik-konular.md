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

Bildirim kanalları ayrı servislere bölündüğünde veya alarm olaylarını başka sistemler de dinlemek istediğinde outbox relay'in arkasına eklenir. Ayrıntı: [06 — Alarm ve bildirim](06-alarm-ve-bildirim.md#ileride-rabbitmq).

## Saklama süreleri

7 gün ham, 30 gün dakikalık başlangıç değerleridir. Uzun dönem trend analizi istenirse dakikalık görünümden saatlik bir görünüm türetilip daha uzun saklanabilir.
