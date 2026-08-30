---
name: app-store-review-playbook-tr
description: App Store ve Google Play yayın süreci - gerçek redlerden çıkarılmış kontrol listesi, build/yükleme tuzakları ve çözümleri. Şunlarda kullan - "App Store'a gönderelim", "Play'e yükleyelim", "uygulama reddedildi", "Apple red", "Guideline", "App Review", "TestFlight", "AAB yükle", "eas submit", "altool", "provisioning profile", "mağaza kaydı", "yayına alalım", "inceleme notu yazalım", "review notes", "gizlilik beyanı", "yaş derecelendirme". Yeni bir mobil uygulamayı yayına hazırlarken gönderimden ÖNCE oku.
metadata:
  version: 1.0.0
  kaynak: 8 yayınlanmış iOS/Android uygulaması, 2026
---

# App Store & Play inceleme rehberi

Bu dosya genel mağaza tavsiyesi değil. Her madde sekiz uygulamanın yayın sürecinde **gerçekten yaşandı** ve sonunda bulunan kök nedeniyle birlikte yazıldı. Uygulamalar anonim: **A Uygulaması** (sosyal etkinlik), **B Uygulaması** (harita/mekan rehberi), **C Uygulaması** (Expo tüketici uygulaması).

**Kullanım:** Gönderimden önce "Gönderim öncesi kontrol listesi"ni uygula. Red geldiyse "Red kataloğu"ndan eşleşeni bul. Build/yükleme takılıyorsa "Build ve yükleme tuzakları"na bak.

---

## 🔴 En pahalı ders — App Review Notes'ta ne YAZMAYACAĞIN

A Uygulaması build 49, Apple **Guideline 4.2 (Minimum Functionality)** ile reddedildi. Red metni, gönderimi hazırlayan tarafın App Review Notes'a kendi yazdığı cümlelerin **birebir kelimelerini** kullandı:

> "We are targeting a deliberately **small**, curated early community — **a few dozen** invited members, **not a mass-market** social network. **Staying small is a core part of the product.**"

Apple bunu "bu uygulama App Store'a değil Ad Hoc dağıtıma ait" diye okudu.

**Kural: App Review Notes'ta asla şu çerçeveyi kurma** — "small", "niche", "few dozen", "invite-only", "not mass-market", "private group", "closed community", "belirli bir topluluk için".

**Doğru çerçeve:** "herkese açık, ücretsiz, her şehirden/ülkeden indirilebilir; kürasyonlu/üyelik katmanı OPSİYONEL bir özellik". Ürün gerçekten davetliyse bile, uygulamanın **hesapsız görülebilen** bir yüzü olmalı ve notlar o yüzü anlatmalı.

Ek: **ASC review notes sınırı 4000 karakter.** Aşarsan API 409 verir.

---

## Gönderim öncesi kontrol listesi

Her madde bir redden geliyor. Gönderimden önce tek tek doğrula.

### Hesap ve veri (5.1.1)
- [ ] **Hesap silme var mı?** Hesap açılabiliyorsa uygulama İÇİNDEN silinebilmeli — 5.1.1(v). Onay + "geri alınamaz" uyarısı. *(İki uygulamamızda red sebebi oldu.)*
- [ ] **İzin metinleri (purpose strings) doğru mu?** — 5.1.1(ii). **Bare workflow tuzağı:** `ios/` klasörü repoda ise `app.json`'daki purpose string'ler Info.plist'e OTOMATİK GİTMEZ; Info.plist'i elle kontrol et.
- [ ] **Kullanmadığın izni isteme.** B Uygulaması 5.1.1'den reddedildi: `pickImage` galeriyi açmadan önce tüm kütüphane izni istiyordu. Modern iOS PHPicker izinsiz tek foto seçtirir → **izin çağrısını kaldır**.
- [ ] **Konum izni priming ekranı** — 5.1.1(iv). Yönlendirici kelime kullanma: "Konumumu Kullan" ❌ → "Devam Et" ✅. Kafa karıştıran çift "Atla" olmasın.

### İçerik ve topluluk (1.2) — kullanıcı içeriği varsa ZORUNLU
A Uygulaması build 51 buradan reddedildi. Dördü birden gerekiyor:
- [ ] **Görünür** şikayet mekanizması (uzun-bas jesti YETMEZ — reviewer bulamaz). Her mesaj/gönderi/yorum yanında görünür "⋯" düğmesi.
- [ ] Engelleme (iki yönlü — engellenen yazamaz).
- [ ] İçerik filtresi — **yazma uçlarının TAMAMINDA**, sadece birkaçında değil.
- [ ] DM'de sohbet isteği/kabul akışı: başlatan taraf kabul gelene dek tek mesaj.

### Metadata
- [ ] **Yaş derecelendirme** — 2.3.6. "Tanışma/networking" içeren uygulamada `matureOrSuggestiveThemes` en az `INFREQUENT_OR_MILD` olmalı. ASC API ile düzeltilebilir: `PATCH /v1/ageRatingDeclarations/{id}`.
- [ ] **Gizlilik politikası URL'si 200 dönüyor mu?** İlk Play üretim gönderimimiz sırf beyan edilen URL 404 verdiği için reddedildi.

### Reviewer'ın göreceği hal
- [ ] **Demo hesap GERÇEKTEN çalışıyor mu?** Cihazda dene. A Uygulaması 2.1'den reddedildi: Watch'tan giriş hiç çalışmamıştı (`email` gönderiyordu, backend `identifier` okuyor → 422). iPhone'da çalıştığı için fark edilmemişti.
- [ ] **Demo veri taze mi?** A Uygulamasında demo etkinliklerin 16'sı geçmişti, reviewer boş uygulama görecekti. Tarihleri ileri kaydıran idempotent bir script tut.
- [ ] **Doğrulama duvarı reviewer'ı kilitliyor mu?** Kaydolup e-postasını doğrulamamış kullanıcı hiçbir şey göremiyorsa uygulama "kapalı" görünür. Misafir gezebilsin, doğrulama yalnız yazma eyleminde istensin.
- [ ] **Boş/kapalı modül bırakma.** A Uygulamasında kapalı bir özellik bayrağı yüzünden boş görünen "Kurslar" bölümü üst üste 2.1 (App Completeness + Information Needed) redlerine sebep oldu; sonunda özellik tamamen kaldırıldı. **Yarım özelliği gönderme, kaldır.**
- [ ] **Reviewer hangi cihazda bakacak belli değil.** Bizde iPad Air, Apple Watch çıktı. Ana cihazın dışındaki hedefleri de test et.

### Platform entegrasyonu (4.0 tasarım)
- [ ] **Harita/konum özelliği yerel uygulamayla entegre mi?** B Uygulaması 4.0.0'dan reddedildi: yalnız Google Maps'e yönlendiriyordu. Apple Haritalar seçeneği (`maps.apple.com`) sunulmalı.

### Android tarafı
- [ ] **Reklam kimliği beyanı manifest'le tutarlı mı?** `.aab`'yi açıp `com.google.android.gms.permission.AD_ID` var mı bak. Yoksa beyan "kullanılmıyor" olmalı — yanlış beyan gönderimi kilitliyor.
- [ ] **Dağıtım ülkeleri.** Bir uygulamamız üretimde kazara **1 ülkeye** kilitliydi (iOS 175 ülkedeyken). Üretim → Ülkeler/bölgeler.
- [ ] **Arka plan konumu** kullanıyorsan Play "Konum İzni Beyanı" isteyebilir.
- [ ] **Maps API anahtarı** `app.json > android.config.googleMaps.apiKey` — YOKSA `react-native-maps` Android'de native init'te **çöker**. iOS'ta çalışır (orada Apple Maps varsayılan), o yüzden gözden kaçar.
- [ ] **Google ile Giriş için İKİ SHA-1** gerekiyor: yükleme anahtarı + **Play uygulama imzalama anahtarı**. Play kendi anahtarını AAB yüklendikten sonra üretir; o SHA-1 Android OAuth istemcisine eklenmezse Play sürümünde Google girişi çalışmaz. Emülatörde de basılamaz (debug SHA-1 kayıtlı değil) — fonksiyonel test yalnız Play-imzalı cihazda.

---

## Red kataloğu (yaşanmış, kök nedenli)

| Guideline | Ne dedi | Gerçek kök neden | Çözüm |
|---|---|---|---|
| **4.2** Minimum Functionality | "small, niche set of users" | **Review Notes'taki kendi cümlem** | Hesapsız keşif akışı + public API uçları + içerik dengesi |
| **1.2** UGC | filtre/şikayet/engelleme yok | Şikayet yalnız görünmez uzun-bas jestindeydi; DM ve lobide hiç yoktu | Görünür "⋯" menüsü 8 yüzeyde, filtre 9 yazma ucunda, DM kabul akışı |
| **2.1** Demo hesapla girilemedi | Watch'ta giriş hatası | Watch `email` gönderiyor, backend `identifier` okuyor → 422 | Alan adı düzeltildi; Review Notes'a "WATCH — PLEASE READ FIRST" bölümü |
| **2.1** App Completeness | "could not access the courses" | Modül kapalıydı (`lms_enabled=0`), bölüm boş görünüyordu | Özellik tamamen kaldırıldı |
| **2.1** Information Needed | "kaç kullanıcı hedefliyorsunuz?" | Aynı boş modül + 4.2 çerçevesi | Notlar yeniden yazıldı |
| **2.3.6** Yaş derecelendirme | "Mature or Suggestive Themes" | Tanışma/networking teması beyan edilmemiş | ASC API `ageRatingDeclarations` PATCH |
| **5.1.1(v)** Data Collection | hesap silme yok | — | Uygulama içi hesap silme |
| **5.1.1(ii)** | izin metni eksik | Bare workflow'da app.json → Info.plist senkronu yok | Info.plist elle |
| **5.1.1(iv)** Konum akışı | priming ekranı yönlendirici | Buton metinleri + çift Atla | "Devam Et", tek çıkış |
| **5.1.1** Foto erişimi | galeri izni istiyor | PHPicker izin gerektirmiyordu | İzin çağrısı kaldırıldı |
| **4.0.0** Tasarım | "not integrated with built-in mapping" | Yalnız Google Maps'e yönlendirme | Apple Haritalar seçeneği |
| **Play** (üretim) | gönderim reddi | Gizlilik politikası URL'si 404 | Kalıcı takma ad + konsol kaydı düzeltildi |

**Not:** Bir redi çözerken bir sonrakini davet edebilirsin. Bir uygulamada 4 ardışık red oldu, bir diğerinde 3. Her düzeltmeden sonra listenin TAMAMINI yeniden gözden geçir.

---

## Build ve yükleme tuzakları

### Sürüm treni
**Onaylanan/yayınlanan bir sürüm dizesine yeni build yüklenemez** — altool hataları 90062 / 90186 ("Invalid Pre-Release Train ... closed"). Yeni build için `app.json > version` artırılıp **yeniden derlenmeli** (sürüm dizesi ipa'ya gömülü). Bir uygulamada bir build bu yüzden boşa gitti.

### Yükleme
- `eas submit` takılabiliyor (23+ dk, 0 çıktı) veya "Failed to authenticate for session" veriyor. **Güvenilir yol doğrudan altool:**
  ```
  xcrun altool --upload-app -f build.ipa -t ios --apiKey <KEY_ID> --apiIssuer <ISSUER>
  ```
  `.p8` anahtarı `~/.appstoreconnect/private_keys/AuthKey_<KEY_ID>.p8` altında olmalı — altool oradan okur. 15 saniyede bitiyor.
- **"Upload succeeded" ≠ kabul edildi.** Apple işleme aşamasında da reddedebilir; VALID durumunu izle. VALID olunca eski build'i expire et (`PATCH /v1/builds/{id}` `{expired:true}`).
- **Watch / widget hedefine ikon şart** (`CFBundleIconName`) — yoksa Apple yüklemeyi hata **90713** ile reddeder.

### Yeniden gönderim (resubmit) sırası
1. Aynı anda **iki sürüm incelenemez** — önce mevcut `reviewSubmission`'ı iptal et (`canceled=true`), CANCELING→COMPLETE bekle.
2. Sürüm `DEVELOPER_REJECTED` (düzenlenebilir) olur → versionString PATCH → build relationship PATCH.
3. ⚠️ **Swap tuzağı:** iptalden hemen sonra build attach 409 verir; akış devam ederse **ESKİ build'le** gönderilir. Attach'i retry'la ve submit'ten ÖNCE bağlı build'i doğrula (`GET /appStoreVersions/{id}/build`).
4. ⚠️ `reviewSubmissionItems` POST'u 409 `ENTITY_STATE_INVALID` verebilir (geçiş bitmemiştir) — birkaç saniye sonra geçiyor, retry'lı olmalı.

### Yerel build ortamı
- **Xcode otomatik güncellenirse** build "iOS X Platform Not Installed" ile düşer. Fix: `xcodebuild -downloadPlatform iOS` (~8.5 GB, sudo gerekmez) + `xcodebuild -runFirstLaunch`. Uzun oturumda ortam değişebilir — aynı gün önceki build'ler geçmiş olabilir.
- **CocoaPods + Ruby 4.0:** `pod install` `Unicode Normalization not appropriate for ASCII-8BIT` ile düşer. Fix: `LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8` ile çalıştır.
- **Podfile modular headers:** GoogleSignIn 9.2.0 → `AppCheckCore`/`GoogleUtilities`/`RecaptchaInterop` için `:modular_headers => true`.
- **Provisioning profile yeni capability'yi içermiyorsa** lokal build düşer. Non-interactive EAS credential DEĞİŞTİRMEZ. Ya `eas credentials` interaktif yenile ya ASC API'den.
- **Apple Developer capability API'den açılabiliyor** (portal gerekmez): `POST /v1/bundleIdCapabilities` — `settings` olmadan 409 verir.
- **`ANDROID_HOME` şart:** `export ANDROID_HOME=/opt/homebrew/share/android-commandlinetools`, yoksa Gradle "SDK location not found".
- **Arşiv derlenirken kaynak dosya DEĞİŞTİRME** — Metro yarım paket gömer, açılışta çöker.
- **EAS temp şişiyor** (`/var/folders/.../eas-cli-nodejs` 35 GB'a çıktı) — build sonrası temizle. Disk dolunca "No space left" ile düşer.
- Build numaraları başarısız denemelerle atlar; bu normal.

### Standalone'a özgü çökmeler
- **Simülatör/dev bunları YAKALAMAZ.** Kablo + `devicectl --console` ile gerçek cihazda test et.
- `.env` gitignore'daysa EAS arşivine gitmez → bundle'da boş değişken → açılışta çökme. (Bir uygulamada TÜM build'ler bu yüzden çöküyordu.)
- Dinamik `import()` edilen native paket kurulu değilse dev'de görünmez, standalone'da `RCTFatalException: Cannot find module` ile çöker. (Örnek: `@better-auth/expo` → kurulu olmayan `expo-network`.)
- **Hermes string'leri UTF-16.** Bundle'da Türkçe metin ararken UTF-8 grep 0 döner — UTF-16 ile doğrula.

---

## Mağaza kaydı — bir kerelik, elle

- **App Store Connect'te uygulama kaydı API ile AÇILAMIYOR** (denendi, doğrulandı). Tarayıcıdan elle.
- **Play Console'da uygulama oluşturma da ilk seferde elle** zorunlu.
- **Bundle ID kayda kalıcı bağlanır, değiştirilemez.**
- **Play'de "Ücretsiz" seçimi yayından sonra ücretliye ÇEVRİLEMEZ.**
- Play Console'da uygulama-içi sayfaların gerçek yolu `app-content/**` (ör. `app-content/ad-id-declaration`); tek başına `/app-content` app listesine yönlendirir.

## Bu rehberi yürüten yapay zekâ asistanı için sınırlar

- **Asistan Apple/Google parolası ve 2FA girmemeli.** ASC web ayrı giriş istiyor (Developer oturumu taşınmıyor). Akış: kişi tarayıcıda giriş yapar, "girdim" der, asistan API/konsol adımlarını oradan sürdürür.
- **Büyük dosya yüklenemez:** tarayıcı `file_upload` sınırı 10 MB, `.aab` tipik 60+ MB. Kişi elle yükler. Kalıcı çözüm: Play servis hesabı + `eas.json > submit.android` → `eas submit`.
- **Beyan/onay kutuları kişinin açık onayı olmadan işaretlenmemeli.**

---

Yeni bir red geldiğinde önce kök nedeni bul, sonra buraya bir satır ekle. Bir rehber, son olaydan öğrendiği kadar değerlidir.
