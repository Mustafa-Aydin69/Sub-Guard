# SubGuard

**Abonelik ve Ücretsiz Deneme (Free-Trial) Bekçisi**

SubGuard, kullanıcıların unutulan dijital abonelikler ve ücretsiz deneme (free-trial) süreleri yüzünden yaşadığı gereksiz para kayıplarını önleyen, proaktif bir kişisel finans ve hatırlatma platformudur. Sistem, arka planda çalışan akıllı zamanlayıcılar sayesinde kesim tarihinden önce kullanıcıyı uyararak abonelik yönetimini tamamen zahmetsiz hale getirir.

## Proje Odak Noktaları ve Temel Bileşenler

Sistem, anlık bildirimler ve zamanlanmış görevler (cron jobs) ekseninde, dört ana mimari direk üzerine kuruludur:

*   **Mobil İstemci (Frontend):** Kullanıcının aboneliklerini eklediği, kalan günleri dairesel grafiklerle (progress bar) görselleştirdiği, karanlık mod (dark mode) destekli, temiz ve akıcı bir Flutter arayüzü.
*   **Arka Uç (Backend API):** Mobil uygulamadan gelen verileri alıp veritabanına yazan, kullanıcı doğrulamasını (Authentication) sağlayan ve tüm iş mantığını yöneten Node.js (Express) servisi.
*   **Zamanlanmış Görevler (Cron Jobs):** Projenin kalbi olan bu yapı, her gece (örneğin 00:00'da) veritabanını tarayarak ertesi gün veya belirlenen süre içinde deneme süresi bitecek/faturası kesilecek abonelikleri tespit eden otonom arka plan işçisidir (Worker).
*   **Bildirim Altyapısı (Push Notifications):** Firebase Cloud Messaging (FCM) entegrasyonu sayesinde, arka uç riskli bir tarih tespit ettiğinde, uygulama kapalı dahi olsa kullanıcının telefonuna anlık bildirim (Örn: "Dikkat: Coursera deneme süreniz yarın doluyor, kartınızdan çekim yapılacak!") düşürülür.

## Kritik Veritabanı Modeli

Projenin temeli olan ilişkisel veritabanı (MS SQL veya PostgreSQL/Supabase) tasarımı şu şekildedir:

*   **Users:** `UserID`, `Name`, `Email`, `PushToken` (Anlık bildirim göndermek için cihazın eşsiz kodu).
*   **Services:** `ServiceID`, `Name` (Netflix, Spotify, Amazon vb.), `LogoURL` (Arayüzde logoların otomatik gösterilmesi için).
*   **Subscriptions:** `SubID`, `UserID`, `ServiceID`, `Cost`, `NextBillingDate` (Sonraki kesim tarihi), `IsFreeTrial` (Boolean - Ücretsiz deneme durumu), `ReminderDays` (Kaç gün önceden uyarı verileceği, örn: 3 gün).

## Adım Adım Geliştirme Haritası

1.  **Temel API ve Veritabanı (Backend):** Node.js projesinin oluşturulması, SQL bağlantılarının yapılması. Kullanıcı kaydı ve abonelik yönetimi (Ekle, Sil, Listele vb.) için CRUD uç noktalarının yazılması.
2.  **Cron Job ve Sorgu Yazımı:** `node-cron` vb. kütüphaneler ile her gün belirli saatte çalışacak fonksiyonun yazılması. Bu fonksiyonun veritabanına; "NextBillingDate değeri bugünden tam 'ReminderDays' kadar sonra olan kayıtları getir" sorgusunu atması.
3.  **Mobil Arayüz (Flutter):** API entegrasyonlarının tamamlanması. Kullanıcının hızlıca platform seçebileceği, "Yeni Abonelik Ekle" fonksiyonunu barındıran şık ekran tasarımlarının kodlanması.
4.  **DevOps ve Yayına Alma:** Arka ucun bir Docker container içine alınarak AWS, Render veya DigitalOcean gibi bir bulut sunucusunda çalıştırılması. Veritabanının sürekli erişilebilir (always-on) bulut mimarisine taşınması.

## Gelecek Vizyonu (V2 Özellikleri)

Projenin ana iskeleti oturduktan sonra platformu profesyonel bir seviyeye taşıyacak eklentiler:
*   **Makine Öğrenmesi (OCR) Entegrasyonu:** Kullanıcının gelen e-faturasının veya dijital fişinin ekran görüntüsünü yüklemesi durumunda; Optik Karakter Tanıma algoritmalarının markayı, tutarı ve fatura tarihini otomatik okuyup aboneliği veritabanına kendi kendine eklemesi.

---
