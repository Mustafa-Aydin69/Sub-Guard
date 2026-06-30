# CLAUDE.md — SubGuard

> Bu dosya Claude Code için proje bağlamıdır. Kod yazmadan önce buradaki
> mimari kararlarına, şemaya ve "Kritik Dikkat Noktaları"na uy.

## Proje Özeti

SubGuard, kullanıcıların unutulan dijital abonelikler ve ücretsiz deneme
(free-trial) süreleri yüzünden yaşadığı para kayıplarını önleyen proaktif bir
hatırlatma platformudur. Kesim/deneme bitiş tarihinden önce, uygulama kapalı
olsa bile, push bildirimiyle kullanıcıyı uyarır.

**Tek cümlede çekirdek değer:** "Para çekilmeden önce haber ver."

## Mimari (4 Direk)

1. **Mobil İstemci (Flutter)** — Abonelik ekleme, kalan günleri dairesel
   progress bar ile gösterme, dark mode. API ile konuşur.
2. **Backend API (Node.js + Express)** — Auth, CRUD, iş mantığı. Veritabanına
   tek yazma/okuma noktası.
3. **Cron Worker (node-cron)** — Projenin kalbi. Her gece çalışıp "yakında
   kesilecek/bitecek" abonelikleri tespit eder ve bildirim tetikler.
4. **Push Altyapısı (Firebase Cloud Messaging)** — Worker risk tespit edince
   FCM üzerinden cihaza anlık bildirim gönderir.

```
Flutter  ──REST──>  Express API  ──>  PostgreSQL (Supabase)
                          ^                  |
                          |                  | (her gece tarar)
                    node-cron Worker  <──────┘
                          |
                          └──> FCM ──> Kullanıcının telefonu
```

## Teknoloji Yığını

- **Mobil:** Flutter (Dart)
- **Backend:** Node.js + Express
- **Veritabanı:** PostgreSQL üzerinden **Supabase** *(varsayılan seçim — MVP
  için Auth + DB + storage'ı tek yerde verdiği için. MS SQL alternatifti ama
  CLAUDE bunu önerdi; eğer MS SQL'e geçilirse şemadaki `BOOLEAN`/`SERIAL` tipleri
  güncellenmeli.)*
- **Bildirim:** Firebase Cloud Messaging (FCM)
- **Zamanlayıcı:** node-cron
- **Deploy:** Backend → Docker → Render / DigitalOcean / AWS

## Repo Yapısı (hedeflenen monorepo)

```
subguard/
├── CLAUDE.md
├── backend/          # Node.js + Express + cron worker
│   ├── src/
│   │   ├── routes/         # auth, subscriptions, services
│   │   ├── controllers/
│   │   ├── db/             # bağlantı + sorgular/migration
│   │   ├── jobs/           # cron worker (reminder tarayıcı)
│   │   └── services/       # fcm.js (push gönderimi)
│   └── package.json
└── mobile/           # Flutter
    └── lib/
        ├── screens/
        ├── widgets/        # circular progress, kart vb.
        ├── models/
        └── services/       # api_client.dart
```

## Veritabanı Şeması

> Senin modeline `notifications_log` ve `created_at` alanlarını ekledim —
> sebepleri "Kritik Dikkat Noktaları"nda.

```sql
-- Users
CREATE TABLE users (
  user_id      SERIAL PRIMARY KEY,
  name         TEXT NOT NULL,
  email        TEXT UNIQUE NOT NULL,
  push_token   TEXT,                 -- FCM cihaz token'ı (değişebilir!)
  created_at   TIMESTAMPTZ DEFAULT now()
);

-- Services (Netflix, Spotify, Amazon...)
CREATE TABLE services (
  service_id   SERIAL PRIMARY KEY,
  name         TEXT NOT NULL,
  logo_url     TEXT
);

-- Subscriptions
CREATE TABLE subscriptions (
  sub_id            SERIAL PRIMARY KEY,
  user_id           INT NOT NULL REFERENCES users(user_id) ON DELETE CASCADE,
  service_id        INT NOT NULL REFERENCES services(service_id),
  cost              NUMERIC(10,2),
  next_billing_date DATE NOT NULL,
  is_free_trial     BOOLEAN DEFAULT FALSE,
  reminder_days     INT DEFAULT 3,        -- kaç gün önceden uyarılacak
  created_at        TIMESTAMPTZ DEFAULT now()
);
CREATE INDEX idx_sub_next_billing ON subscriptions(next_billing_date);

-- [CLAUDE ekledi] Aynı bildirim iki kez gitmesin diye log
CREATE TABLE notifications_log (
  id         SERIAL PRIMARY KEY,
  sub_id     INT NOT NULL REFERENCES subscriptions(sub_id) ON DELETE CASCADE,
  sent_for   DATE NOT NULL,            -- hangi billing tarihi için gönderildi
  sent_at    TIMESTAMPTZ DEFAULT now(),
  UNIQUE (sub_id, sent_for)            -- idempotency garantisi
);
```

**Cron'un temel sorgusu:**
"`next_billing_date` değeri `CURRENT_DATE + reminder_days` olan ve bu tarih için
`notifications_log`'da kaydı OLMAYAN abonelikleri getir."

## Komutlar

```bash
# Backend
cd backend
npm install
npm run dev          # nodemon ile API
npm run worker       # cron worker'ı ayrı çalıştırmak için (önerilen — aşağıya bak)
npm run migrate      # şema/migration

# Mobil
cd mobile
flutter pub get
flutter run
```

## Kod Konvansiyonları

- API yanıtları JSON; hata formatı tutarlı: `{ "error": { "code", "message" } }`.
- DB'ye erişim yalnızca `backend/src/db/` üzerinden — controller'lar ham SQL
  çağırmaz.
- Gizli anahtarlar (`.env`): `DATABASE_URL`, `FCM_SERVER_KEY`, `JWT_SECRET`.
  Repoya **asla** commit'lenmez.
- Tarih alanları her zaman timezone bilinçli — aşağıyı oku.

## Kritik Dikkat Noktaları (gotchas)

1. **Saat dilimi.** "Her gece 00:00" ve `next_billing_date` karşılaştırmaları
   sunucu UTC'de çalışırsa Türkiye (UTC+3) için yanlış güne kayar. Cron ve tarih
   sorgularını açıkça `Europe/Istanbul` ile sabitle.
2. **Idempotency / mükerrer bildirim.** Worker yeniden başlarsa veya iki kez
   tetiklenirse aynı uyarı iki kez gitmemeli. `notifications_log` + `UNIQUE`
   kısıtı bunun için var; göndermeden önce kontrol et, gönderdikten sonra yaz.
3. **Çoklu instance.** Backend birden fazla kopya çalışırsa node-cron her
   kopyada tetiklenir → mükerrer bildirim. MVP'de worker'ı **tek bir process**
   olarak ayır (`npm run worker`), API'den bağımsız tek instance çalıştır.
4. **FCM token bayatlaması.** Push token'ları döner/geçersizleşir. Gönderim
   `messaging/registration-token-not-registered` dönerse ilgili `push_token`'ı
   temizle/güncelle.
5. **`reminder_days` mantığı.** "Tam X gün sonra" mı yoksa "X gün içinde her gün"
   mü? MVP'de "tam X gün önce bir kez" tutuyoruz (log bunu da sağlıyor).

## Geliştirme Yol Haritası

1. **Faz 1 — Backend + DB:** Express iskeleti, Supabase bağlantısı, auth,
   subscriptions/services için CRUD uçları.
2. **Faz 2 — Cron + sorgu:** node-cron worker, yukarıdaki idempotent reminder
   sorgusu, FCM gönderimi.
3. **Faz 3 — Flutter:** API entegrasyonu, "Yeni Abonelik Ekle" akışı, dairesel
   progress bar'lı liste ekranı, dark mode.
4. **Faz 4 — DevOps:** Backend'i Docker'a al, Render/DO/AWS'ye deploy,
   always-on DB.

## V2 Vizyonu (sonraya)

- **OCR/ML:** e-fatura/fiş ekran görüntüsünden marka + tutar + tarih okuyup
  aboneliği otomatik ekleme.
- *(TR fırsatı — daha önce konuştuk):* e-Arşiv entegrasyonu ve BDDK open banking
  ile tekrarlayan ödemeleri otomatik tespit, manuel giriş sürtünmesini azaltmak.