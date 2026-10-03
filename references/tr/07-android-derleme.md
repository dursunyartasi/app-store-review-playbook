# Android derleme, imzalama ve test (Expo)

Bir Expo projesinden doğru bir AAB çıkarmak ve Play onu görmeden önce çalıştığını kanıtlamak. Konsol tarafı `03-google-play.md` içinde.

Yaşadığımız yalnız-Android hataların neredeyse hepsi tek bir gerçekten çıktı: **`android/` klasörü ile `app.json` farklı şey söylüyor, ve kazanan `android/` klasörü.**

## Yerel araç zinciri (Apple Silicon)

```bash
brew install --cask android-commandlinetools
yes | sdkmanager --licenses
sdkmanager "platform-tools" "platforms;android-36" "build-tools;36.0.0"
export ANDROID_HOME=/opt/homebrew/share/android-commandlinetools
export ANDROID_SDK_ROOT=$ANDROID_HOME
export LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8
```

- `ANDROID_HOME` olmadan Gradle "SDK location not found" der.
- UTF-8 locale olmadan Ruby 4, arka plan kabuklarında `pod install`'u bir `unicode_normalize` `Encoding::CompatibilityError` ile düşürür, fastlane de uyarı verir. Export satırlarını yalnız kabuk profiline değil, build script'ine koy.
- Homebrew'un `avdmanager`'ı `~/Library/Android/sdk` altındaki sistem imajlarını göremiyor ("Package path is not valid"). `sdkmanager --sdk_root=$ANDROID_HOME "cmdline-tools;latest"` kur ve onun yerine `$ANDROID_HOME/cmdline-tools/latest/bin/avdmanager` çağır.
- macOS, ajanın Masaüstü, Belgeler ve İndirilenler erişimini oturum ortasında geri alabiliyor ("Operation not permitted"), bu da build'leri öldürüyor. Dosyalar ve Klasörler (ya da Tam Disk Erişimi) iznini yeniden ver ve uygulamayı yeniden başlat.

## Derleme

```bash
eas build --platform android --profile production --local --non-interactive --output ./app.aab
```

**Yerelde derle.** EAS Free planının **platform başına** aylık bir build kotası var: Android hâlâ çalışırken iOS kotasını bitirdik, çoğunu kimsenin yüklemediği Android AAB'lerine ve başarısız build'lere harcayarak. `--local` build'ler ücretsiz ve sınırsız; `eas submit` kotadan düşmüyor.

**Herhangi bir build'i kuyruğa almadan önce yerelde `npx expo export` çalıştır.** Bir monorepo uygulaması, `dist/` klasörü EAS'ta derlenmeyen paylaşımlı bir workspace paketinden çalışma zamanı sabitleri import ediyordu. Hem Android hem iOS build'i yalnızca `UNKNOWN_ERROR ... See logs of the Bundle JavaScript build phase` ile düştü. Paylaşımlı import'ları yalnız `import type` olarak tut, ya da paketi uygulamanın bir parçası olarak derle.

**İlk Android build'inden önce `npx expo install --check` çalıştır.** SDK 55'te `@react-native-async-storage/async-storage` 3.x, release build'i `Could not find org.asyncstorage.shared_storage:storage-android:1.0.0` ile bozdu; 2.2.0 çalıştı.

**`npm ci` temiz bir klasörde geçmeli.** `--legacy-peer-deps` ile kurulan bağımlılıklar `npm ci`'ın reddettiği bir lockfile bırakıyor, ve bulut build'i ölüyor. `legacy-peer-deps=true` içeren bir `.npmrc` ekle ve lockfile'ı yeniden üret.

### `.easignore`, `.gitignore`'un yerini alır, onu genişletmez

Proje arşivi 187 MB'tı ve `EPIPE` ile düştü, çünkü klasörde 68 MB'lık bir AAB ve mağaza görselleri vardı. Yeni bir `.easignore`'a yalnız bunları eklemek arşivi **2,9 GB** yaptı — `node_modules`, `.git` ve `Pods` artık hariç tutulmuyordu. Önce `.gitignore`'un tamamını `.easignore`'a kopyala, sonra `*.aab`, `*.ipa`, mağaza görsellerini ve imzalama dosyalarını ekle. Çalışan arşiv 3,3 MB'tı. Her başarısız deneme yine de bir build numarası tüketti.

### EAS logları sıkıştırılmış geliyor

`eas build:list --json` çıktısındaki `logFiles[0]` üzerinde `curl`, `gunzip`'in reddettiği baytlar döndürüyor. `curl --compressed` kullan, ya da build sayfasını expo.dev'de aç.

## Prebuild: app.json değişikliklerinin öldüğü yer

`android/` bir kez oluştuğunda — `expo run:android`'den ya da herhangi bir `expo prebuild`'den sonra — build'ler onu olduğu gibi kullanır. EAS, prebuild'i atladığını log'a yazar. O andan itibaren, yeniden üretmedikçe **versionCode, anahtarlar, izinler ve paketle ilgili app.json değişiklikleri build'e ulaşmaz.**

Bunun bize maliyeti:
- `app.json` ilerlemişken `android/app/build.gradle` hâlâ versionCode 1 / 1.0.8 diyordu.
- **Bir hafta içinde iki kez "Version code already used".** `app.json` artırılmış, prebuild yeniden çalıştırılmamıştı. Build script'i prebuild'den yalnızca bir yorum satırında *bahsediyordu*.

Bu yüzden sürüm script'i prebuild'i her seferinde çalıştırıyor:

```bash
rm -rf android && npx expo prebuild -p android   # a half-finished prebuild leaves a broken android/
grep versionCode android/app/build.gradle         # check before handing off the AAB
```

Sürüm bilgisinin asıl kaynağını da bil. `eas.json` içinde `"cli": { "appVersionSource": "remote" }` varsa EAS kendi sayacını tutar ve `app.json`'u yok sayar; değeri `eas build:version:set` ile ayarla. İptal edilen ve başarısız olan build'ler de o sayaçta bir numara tüketir.

`app.json` içinde `android.package` ayarlı olmalı, yoksa prebuild durur.

### Prebuild, release imzalamasını sessizce debug anahtarına geri döndürür

Expo'nun şablon `build.gradle` dosyasında `release { signingConfig signingConfigs.debug }` var. Yükleme anahtarını kullanacak şekilde elle düzenlersen, bir sonraki `prebuild --clean` debug anahtarını geri koyar — ve `android/` içinde tuttuğun keystore'u siler. Bizimkini projenin eski bir kopyasından kurtardık.

Kalıcı çözüm:
- Keystore'u ve properties dosyasını `android/`'in **dışında**, gitignore'a eklenmiş bir `credentials/` klasöründe tut.
- `../credentials/keystore.properties` dosyasını gösteren bir `signingConfigs.release` ekleyen, release build tipinin onu kullanmasını sağlayan ve **yama uygulanamazsa hata fırlatan** küçük bir config plugin (`withAppBuildGradle`) yaz.

Bunun yerine kimlik bilgilerini EAS'ın yönetmesine izin verirsen bunların hiçbiri geçerli değil — ama oluşturduğu keystore'u yedekle (`eas credentials -p android`).

## AAB'yi bir yere gitmeden önce doğrula

Dört kontrol; hepsi script'lenebilir, hepsi Play'e bozuk ulaşmış şeylerden çıktı:

```bash
# 1. Which key signed it? jarsigner -verify says "jar verified" for a debug-signed bundle too.
keytool -printcert -jarfile app.aab | grep Owner:      # fail unless it's your upload key's CN

# 2. Version code
grep versionCode android/app/build.gradle

# 3. Permissions and meta-data (the manifest is protobuf, but strings still works)
unzip -p app.aab base/manifest/AndroidManifest.xml | strings | grep -iE 'permission|API_KEY'
#    - AD_ID present or absent, matching the Play declaration
#    - com.google.android.geo.API_KEY present if you use react-native-maps
#    - no ACCESS_BACKGROUND_LOCATION if you blocked it

# 4. Compare the signer with Play Console -> App integrity -> Upload key certificate
```

(4. adımdaki konsol yolu Türkçe arayüzde: Play Console → Uygulama bütünlüğü (App integrity) → Yükleme anahtarı sertifikası (Upload key certificate).)

## Emülatörde test

```bash
sdkmanager "emulator" "system-images;android-36;google_apis;arm64-v8a"
avdmanager create avd -n test -k "system-images;android-36;google_apis;arm64-v8a" -d pixel_7
emulator -avd test -no-snapshot-load -gpu swiftshader_indirect &
adb wait-for-device
until [ "$(adb shell getprop sys.boot_completed | tr -d '\r')" = 1 ]; do sleep 2; done
```

**Geliştirme build'ini değil, release build'i test et:**

```bash
cd android && ./gradlew installRelease
adb shell am start -n <package>/.MainActivity
adb logcat | grep FATAL
adb shell screencap -p /sdcard/s.png && adb pull /sdcard/s.png
```

Bize bir oturuma mal olan yanlış alarmlar:
- `npx expo run:android --device emulator-5554` → "Could not find device". `--device` **AVD adını** bekliyor; tek emülatör çalışıyorsa hiç yazma.
- Metro durunca debug build ölür. Bu bir çökme değil — release build'ler JS'yi içine gömer.
- Düşük RAM'li bir emülatör uygulamayı **logcat'te hiç FATAL olmadan** öldürdü. Kodu suçlamadan önce emülatöre daha fazla bellek ver.
- Emülatörde push token'ı asla verilmez, ve Expo Go'da (SDK 53+) uzaktan push yok. Beklenen durum.
- Debug keystore'un SHA-1'i de kayıtlı değilse Google ile Giriş başarısız olur. Girişin fonksiyonel testi, gerçek bir cihazda Play-imzalı bir build gerektirir.

**İlk emülatör çalıştırmasının bir release build'de yakaladıkları** — iOS testinin hiç göstermediği hatalar:
- **Android otomatik düzeltmesi, yazılan bir slug'ı** iki sözlük kelimesine çevirdi. Tanımlayıcı alanlarda `autoCorrect={false}` ve `autoCapitalize="none"` ayarla, ve sunucu tarafında normalleştir.
- **Donanım geri tuşu, form ortasında uygulamayı kapattı** ve girilenler kayboldu. Onay soran bir `BackHandler` ekle.
- **`react-native`'den gelen `SafeAreaView` Android'de hiçbir şey yapmıyor.** 33 ekranda içerik durum çubuğunun altına kaydı. Kökte bir `SafeAreaProvider` ve `edges={['top']}` ile `react-native-safe-area-context` kullan.

## Bir izni bloklamak yetmez

`blockedPermissions` arka plan konumunu manifest'ten kaldırır (`03-google-play.md`'ye bak), ama kod onu hâlâ istiyordu. targetSdk 36'da uygulama anlamsız bir izin penceresi gösterdi, sonra arka plan görevini başlatırken hata fırlattı. `requestBackgroundPermissionsAsync` ve `startLocationUpdatesAsync` çağrılarını `Platform.OS === 'ios'` koşuluna bağla, ve Android kullanıcılarına takibin yalnız ekran açıkken çalıştığını söyle.

## Google Maps

`react-native-maps` Android'de Google Maps kullanır ve `app.json > android.config.googleMaps.apiKey` eksikse **native init'te çöker**. iOS etkilenmez çünkü orada varsayılan Apple Maps'tir — tam da bu yüzden fark edilmeden yayına gider. Bizde girişten hemen sonra bir çökme olarak yayına çıktı: tüm sekmeler aynı anda mount ediliyordu, bu yüzden kullanıcı giriş yaptığı anda bir `MapView` ekrana geliyordu.

- Anahtarın AAB'de olduğunu kontrol et (yukarıdaki manifest kontrolü).
- **Sekmeleri tembel (lazy) mount et**, ve anahtar eksikse `MapView` yerine bir yer tutucu göster. Böylece eksik anahtar bir çökme değil, gri bir kutu olur.
- Anahtarı pakete ve **iki** SHA-1'e de (yükleme ve Play imzalama) kısıtla. **Yalnızca kısıtlamasız anahtarın çalıştığını doğruladıktan sonra kısıtla**, böylece bir hata tek bir sebebe işaret eder.

## Android'de Google ile Giriş

Bu, tek başına en büyük çıkmaz sokağımızdı: bir inceleme turu ve boşa giden bir sürüm.

**Tarayıcı tabanlı OAuth akışı bir Android OAuth istemcisiyle çalışmaz.** Android istemci kimliğiyle `expo-auth-session` / `Google.useIdTokenAuthRequest`, denediğimiz her yönlendirme varyantı için (`<pkg>:/oauthredirect`, `<pkg>://`, ters çevrilmiş istemci kimliği) `Error 400: invalid_request` döndürdü. Android istemcileri yalnız Play Hizmetleri ve Credential Manager içindir.

Akılda tutmaya değer bir teşhis hilesi: her istemci kimliğiyle `https://accounts.google.com/o/oauth2/v2/auth` adresine istek at. Uydurma bir kimlik `invalid_client` verir; Android istemcisi `invalid_request` verir; iOS ve web kabul edilir. Yani istemci vardı — yanlış olan *istek türüydü*.

İşe yarayan:
- **Web** istemci kimliğiyle yapılandırılmış native `@react-native-google-signin/google-signin`. Dönen ID token'ın audience değeri web istemcisidir.
- **İki Android OAuth istemcisi**, ikisi de paket adıyla: biri yükleme anahtarının SHA-1'iyle (yerel build'ler), biri Play uygulama imzalama anahtarının SHA-1'iyle (mağazadan kurulumlar). Yalnız yükleme anahtarı kayıtlıyken mağaza build'i aynı 400'ü verdi.
- **Backend'in token audience kontrolü web, iOS ve Android istemci kimliklerini kabul etmeli**, yoksa sunucu, uygulamanın sorunsuzca aldığı token'ları reddeder.
- OAuth izin ekranı **Üretimde (In production)** olarak ayarlı olmalı. Varsayılan email/profile kapsamlarında kalmak, doğrulanmamış uygulama uyarısından ve 100 kullanıcı sınırından kurtarır.
- Bunların hiçbiri Expo Go'da çalışmaz; bu native kod.

## Push bildirimleri (FCM)

Bir uygulamada Android push aynı anda üç yerden bozuktu:
1. Firebase projesinde yalnız bir **web** uygulaması vardı. Bir Android uygulaması ekle ve onun `google-services.json` dosyasını uygulamayla birlikte gönder.
2. İstemci `Platform.OS !== 'ios'` olunca erkenden dönüyordu ve hiç kayıt olmuyordu.
3. Backend yalnızca `provider = 'apns'` olan token'ları sorguluyordu.

Ayrıca:
- **Bir bildirim kanalı oluştur.** Android 8+ kanal olmadan bildirimleri sessizce düşürür.
- Eski FCM sunucu anahtarı artık yok. **FCM HTTP v1** kullan: servis hesabı JWT → OAuth2 erişim token'ı → gönder.
- Sunucu tarafı için duman testi: sahte bir token'a gönderim `INVALID_ARGUMENT` dönmeli. Kimlik doğrulama hatası alıyorsan kimlik bilgilerin yanlış; `INVALID_ARGUMENT` kimlik doğrulamanın çalıştığını kanıtlar.
- Expo push ile FCM kimlik bilgilerini EAS'a yükle, ve gerçek bir build'de test et — asla Expo Go'da değil.

## Sarmalanmış web siteleri (TWA, uzak URL'li Capacitor)

- **TWA:** `/.well-known/assetlinks.json` dosyasını yalnız yükleme anahtarınınkiyle değil, **Play uygulama imzalama SHA-256**'sıyla da sun — yoksa uygulama tarayıcı adres çubuğunu gösterir. Dosyayı ortam değişkenlerinden sunmak (ve ayarlanana kadar `[]` döndürmek) değeri koddan uzak tutar. Yolda iki tuzak: Apache'nin `mod_headers` modülü kapalıydı, bu yüzden `.htaccess` içindeki her header kuralı sessizce yok sayıldı; ve Cloudflare'in tarayıcı önbelleği geçersiz kılma ayarı sitenin kendi header'larını ezdi.
- **Yalnızca uzak bir URL yükleyen bir uygulama**, Apple'da olduğu gibi Play'de de minimum işlevsellik riski taşır. Göndermeden önce ona en az bir gerçek native özellik ver (push, paylaşım hedefi, çevrimdışı).
