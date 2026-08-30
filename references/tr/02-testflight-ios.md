# iOS derleme ve TestFlight

Her build için tekrarladığımız döngü. Kabaca 15–40 dakika; aşağıdaki hataların çoğu atlandığında bir tam tura mal oluyor.

## Derlemeden önce şu ikisini kontrol et

İkisi de kontrol etmesi ucuz, sonradan öğrenmesi pahalı.

1. **Build numarası zaten kullanılmış mı?** `GET /v1/builds?filter[app]={id}` — kopya numara yüklemede reddediliyor.
2. **Mevcut sürüm dizesi yayında mı?** App Store sürümü `READY_FOR_SALE` ise o sürümün treni kapalıdır ve yükleme **90186** / **ITMS-90062** ile düşer. Yalnız build numarasını değil **sürüm dizesini** yükseltmen ve yeniden derlemen gerekir — sürüm IPA'nın içine gömülüdür.

Bir sürümde bunu öğrenirken dört build kaybettik.

## Sürüm gerçekte nerede duruyor

- **Managed workflow:** `app.json` → `expo.version` ve `expo.ios.buildNumber`.
- **Bare workflow:** `ios/<App>/Info.plist` → `CFBundleShortVersionString` ve `CFBundleVersion`, artı pbxproj `MARKETING_VERSION`. **`app.json` yok sayılır.**

`app.json`'u bir script'ten düzenliyorsan ayrıştırıp yeniden seri hale getir (`json.load` / `json.dump`). Aynı handle'ı okuyup yazmak dosyayı boşaltıyor — bu bize bir build'e mal oldu.

## Derleme

```bash
npx tsc --noEmit
npx expo export --platform ios --output-dir /tmp/exportcheck   # arşiv öncesi import hatalarını yakala

LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8 \
  eas build --platform ios --profile production --local \
  --non-interactive --output ./app-buildNN.ipa
```

`LANG` öneki isteğe bağlı değil. Ruby 4.0 + CocoaPods 1.16'da `pod install`, `Unicode Normalization not appropriate for ASCII-8BIT` ile düşüyor — özellikle projede ASCII dışı karakter varsa.

Düz `xcodebuild` de çalışır ve EAS kimlik bilgileri bayatladığında kaçış yoludur:

```bash
LANG=en_US.UTF-8 xcodebuild -workspace App.xcworkspace -scheme App \
  -configuration Release -archivePath /tmp/AppNN.xcarchive \
  -allowProvisioningUpdates \
  -authenticationKeyPath ~/.appstoreconnect/private_keys/AuthKey_<KEY_ID>.p8 \
  -authenticationKeyID <KEY_ID> -authenticationKeyIssuerID <ISSUER_ID> archive
```

`ARCHIVE SUCCEEDED` ara, sonra `method=app-store` ve `signingStyle=automatic` ayarlı bir `ExportOptions.plist` ile `-exportArchive`.

### Derleme sürerken

**Arşiv alınırken kaynak dosya düzenleme.** Metro yarım yazılmış bir bundle gömer ve uygulama açılışta çöker. Teorik değil — bir build'e mal oldu ve gizemli bir çökme gibi göründü.

## Yükleme

```bash
xcrun altool --upload-app -f ./app-buildNN.ipa -t ios \
  --apiKey <KEY_ID> --apiIssuer <ISSUER_ID>
```

`.p8` dosyasını `~/.appstoreconnect/private_keys/AuthKey_<KEY_ID>.p8` altına koy; altool oradan bulur. Yaklaşık 15 saniye sürer.

**Bunu `eas submit`'e tercih et.** `eas submit`'in 23 dakika boyunca hiç çıktı vermeden ve hiç yükleme yapmadan asıldığını, ve "Unable to upload archive. Failed to authenticate for session" ile düştüğünü gördük. altool gerçek hatayı söylüyor.

## Yükleme sonrası

**`UPLOAD SUCCEEDED` kabul edildi demek değil.** ASC API'den build `VALID` olana kadar yokla; Apple işleme sırasında da reddediyor. Geçerli olunca önceki build'i sonlandır ki testçiler yalnız yenisini görsün:

```
PATCH /v1/builds/{id}   {"expired": true}
```

## Tanımanız gereken yükleme hataları

| Kod | Anlamı |
|---|---|
| **90062** / ITMS-90062 | Bu sürüm zaten yayında — sürüm dizesini yükselt |
| **90186** | Ön-yayın treni kapalı — aynı sebep |
| **90713** | Bir hedefte `CFBundleIconName` eksik — Watch ve widget hedeflerinin kendi ikonu olmalı |
| **ITMS-90863** | Apple Silicon sembol uyarısı. **Expo uygulamalarında normaldir, red değildir.** Görmezden gel. |

## Ek hedefler

Watch ve Live Activity widget'ları `credentials.json` içinde kendi provisioning profile'larını ister, hepsi aynı dağıtım sertifikası altında. Watch hedefi arşivlemek Mac'te watchOS **cihaz** platformunu gerektirir — simülatör SDK'sı yetmez:

```bash
xcodebuild -downloadPlatform watchOS    # ~4 GB
```

**Ek hedefin kendi akışlarını test et.** Watch hedefimizde giriş hiç çalışmamıştı — backend `identifier` okurken `email` gönderiyordu — ve Apple bunu bizden önce, bir 2.1 reddinde buldu.

## Xcode proje ortasında güncellenirse

Otomatik güncelleme, o sabah build'ler geçmiş olsa bile derlemeleri `iOS <sürüm> Platform Not Installed` ile düşürür:

```bash
xcodebuild -downloadPlatform iOS   # ~8,5 GB, sudo gerekmez
xcodebuild -runFirstLaunch
```

## Temizlik

EAS geçici dosyaları sınırsız büyüyor — bizde `/var/folders/.../eas-cli-nodejs` altında 35 GB'a çıktı. Dolu disk build'i `No space left` ile düşürür. Sürümler arasında temizle. Başarısız denemelerden sonra build numaralarının atlaması normaldir.

Sonraki: `04-app-store-gonderim.md`.
