# Android derleme ve Google Play

Play, App Review'dan daha bağışlayıcı; ama sürümleri koddan çok evrak yüzünden kilitliyor — ve o kilitlere takılmak kolay.

## Bir kerelik kurulum

**Play Console'da uygulamayı ilk kez oluşturmak manuel.** API yolu yok.

Geri alamayacağın iki seçim:
- **"Ücretsiz", yayından sonra ücretliye çevrilemez.**
- Paket adı, iOS bundle ID gibi kalıcıdır.

## İmzalama ve herkesi yakalayan SHA-1

Sen bir **yükleme anahtarıyla** imzalarsın; Play kendi **uygulama imzalama anahtarıyla** yeniden imzalar — ve o anahtarı ancak ilk AAB yüklemenden sonra üretir.

```bash
keytool -genkey -v -keystore ~/app-release-key.jks \
  -keyalg RSA -keysize 2048 -validity 10000 -alias app
```

Keystore'u repodan uzak tut. Sonra:

**Google ile Giriş kullanıyorsan Android OAuth istemcisine İKİ SHA-1 parmak izi de gerekiyor** — kendi yükleme anahtarın *ve* Play'in uygulama imzalama anahtarı (Play Console → Uygulama imzalama). İkincisini atlarsan Google ile Giriş yalnız Play sürümünde bozulur, yerel build'in çalışmaya devam eder. Emülatörde de test edilemez, çünkü debug SHA-1'i de kayıtlı değildir. Fonksiyonel test yalnız cihazda, Play-imzalı bir build ile mümkün.

## Derleme

```bash
export ANDROID_HOME=/opt/homebrew/share/android-commandlinetools   # ya da kendi SDK yolun
eas build --platform android --profile production --local --output ./app.aab
```

`ANDROID_HOME` olmadan Gradle "SDK location not found" der.

**Anahtarsız harita çöker.** `react-native-maps` Android'de Google Maps kullanır ve `app.json > android.config.googleMaps.apiKey` eksikse **native init'te çöker**. iOS etkilenmez çünkü orada varsayılan Apple Maps'tir — bu yüzden fark edilmeden yayına gider. Anahtarın girdiğini doğrula: AAB'yi aç ve manifest'te `com.google.android.geo.API_KEY` ara.

## Yükleme

Elle sürükle-bırak çalışır ama sonsuza dek manuel kalır; tipik bir AAB 60 MB üstü, her tarayıcı otomasyonu sınırının ötesinde. Play servis hesabı ve `eas.json > submit.android` ile otomatikleştir.

### Karşılaşacağın sürüm hataları

- **"Bu sürüm, mevcut kullanıcıların yeni eklenen uygulama paketlerine geçmelerine izin vermediği için kullanıma sunulamaz."** → sürüm kodunu artır, ya da (ilk sürüm için önerilen) İç Test / Kapalı Test kanalından yayınla.
- **"Bu sürüm hiçbir uygulama paketi eklemiyor veya kaldırmıyor."** → AAB düzgün yüklenmemiş. Sürüm kodunu kontrol et ve yeniden yükle.
- **Native debug sembolleri** `native-debug-symbols.zip` içinde ABI klasörleriyle olmalı — `armeabi-v7a/`, `arm64-v8a/`, `x86_64/`, her birinde `libapp.so` — ve **`__MACOSX` veya `.DS_Store` bulunmamalı**.

## Yayını kilitleyen beyanlar

**Reklam kimliği.** AAB'yi aç ve `com.google.android.gms.permission.AD_ID` ara. Firebase Analytics hem izni hem "kullanılıyor" beyanını ister; reklamı olmayan bir uygulamada ikisi de olmamalı. **Kural şu: beyan manifest'le birebir tutmalı** — iki yönde de uyumsuzluk yayını kilitler, ve Play'in kendi uyarı metni hangi tarafın yanlış olduğu konusunda yanıltıcı olabilir.

**Gizlilik politikası adresi.** 200 dönmeli. İlk üretim gönderimimiz sırf beyan edilen adres 404 verdiği için reddedildi — uygulamada başka hiçbir sorun yoktu.

**Veri güvenliği formu ve içerik derecelendirme anketi.** İkisi de üretimden önce zorunlu. Uygulamanın gerçekte ne yaptığına göre doldur; beyan ettiğin izinlerle karşılaştırılıyorlar.

**Dağıtım ülkeleri.** Kontrol et. Uygulamalarımızdan biri üretimde **tek ülkeye** kilitliyken iOS 175 ülkede yayındaydı — kimsenin bilerek seçeceği bir durum değil.

## Hassas izinler

Arka plan konumu ve `FOREGROUND_SERVICE_LOCATION`, **tanıtım videosu** ve inceleme gerektiren bir Play izin beyanını tetikliyor. Henüz ihtiyacın yoksa gönderip takılmak yerine açıkça blokla:

```json
"android": { "blockedPermissions": ["android.permission.ACCESS_BACKGROUND_LOCATION",
                                    "android.permission.FOREGROUND_SERVICE_LOCATION"] }
```

Sonradan, beyanı ve videosu hazır halde bilerek ekle.

## Hedef API düzeyi son tarihleri

Play, hedef API düzeyini yükseltme tarihini kaçıran uygulamalarda güncelleme kabul etmeyi durduruyor. Tarih her yıl kayıyor. **Takip et** — sürüm gününde öğrenmek kötü bir gün.

## Play'in hızı üzerine bir not

Play hızlı onaylıyor, bu iki tarafı da kesiyor: bozuk bir sürüm yaklaşık bir saat içinde yayında olabiliyor ve **geri çekilemiyor**. Bizimki giriş ekranında çöken bir sürümle çıktı; tek çare düzeltilmiş bir sürüm kodu gönderip beklemek oldu. Önce İç Test kullan. Sürüm sonrası Play Vitals çökme sayılarını izle — düzeltmenin oturduğunu böyle doğruladık (10 çökme → 0).
