# Sıfırdan çalışan bir prototipe

Hedef: kullanıcının kendi telefonunda çalışan bir uygulama — ve `05-redler.md`'deki mağaza şartları sonradan eklenmiş değil, baştan sağlanmış olsun.

## İlk dosyadan önce verilecek kararlar

**Managed mi bare workflow mü.** Managed `ios/` ve `android/` klasörlerini üretilmiş halde tutar; bare bunları repoya koyar. Bare sana native kontrol verir ve şunu götürür: **`app.json` artık doğruluk kaynağı değildir.** İzin metinleri (purpose strings) `Info.plist`'e artık geçmez, sürüm ise `Info.plist` `CFBundleShortVersionString` ve pbxproj `MARKETING_VERSION`'dan gelir. İkisi de bize red getirdi. Bir native modül zorlamadıkça managed ile başla.

**Bundle identifier.** Şimdi seç ve emin ol — bir App Store Connect kaydı oluştuğu anda **bundle ID kalıcıdır.** Sahip olduğun bir alan adı üzerinden ters-DNS kullan.

**Apple'ın göstereceği ad.** Bireysel Apple Developer hesabında App Store'da görünen geliştirici adı, kayıtlı yasal adındır. ASCII dışı karakterler kayıt sırasında sessizce düşebiliyor (bizde Türkçe harfler düştü) ve App Store Connect'in kendi düzeltme akışı **çalışmıyor** — seni adres doğrulama ve Paid Apps Agreement zincirine sokup adı hiç kaydetmiyor. Düzeltmek kimlik doğrulamalı bir destek talebi gerektiriyor. **Kayıt sırasında adını harf harf kontrol et.**

## İskelet

```bash
npx create-expo-app@latest my-app
cd my-app
npx expo start            # QR'ı Expo Go ile okut
```

Native bir modül eklemene ya da imzalı build'e ihtiyaç duymana kadar Expo Go yeter. Ondan sonrası development build ya da gerçek arşiv ister.

## Mağaza şartlarını şimdi bağla

Bunlar birinci gün ucuz, red sonrası pahalı.

**Kullanıcılar giriş yapabiliyorsa — hesap silme (5.1.1(v)).** Uygulama içinden erişilebilir, anında ve kalıcı olmalı. Deaktivasyon yok, bekleme süresi yok, "silmek için destekle e-postalaş" yok, web sitesine yönlendirme yok. Şifreyi tekrar sor, yıkıcı bir onay göster, nelerin silineceğini listele ve üçüncü taraf izinlerini sağlayıcı tarafında da revoke et.

**Kullanıcılar içerik üretebiliyorsa — 1.2'nin dört şartı.** Her mesaj, gönderi ve yorumda görünür bir eylem ("⋯" düğmesi; uzun-bas jesti reviewer'a görünmez ve reddedildi), iki yönlü çalışan engelleme, **her** yazma ucunda içerik filtresi, ve bir yabancının birden fazla doğrudan mesaj göndermesinden önce bir onay adımı.

**Yasal bağlantılar TIKLANABİLİR olmalı (2.1(a)).** "Kaydolarak Koşulları kabul etmiş olursun" cümlesinin düz metin olması bir red sebebi. Gerçek bağlantı yap, uygulama içi tarayıcıda aç (kullanıcıyı Safari'ye atma) ve yalnız kayıt ekranına değil **giriş ekranına da** koy.

**İzinler.** Kullanmadığın hiçbir şeyi isteme. Seçici açılmadan önce tüm fotoğraf kütüphanesi izni istemek bir red sebebi oldu — modern iOS seçicisi hiç izin almadan tek fotoğraf döndürüyor. Priming ekranlarında yönlendirici buton metni olmasın: "Konumumu Kullan" değil, "Devam Et".

**Gerçekten posta alan bir iletişim adresi.** Listelemende ya da uygulama içi kurallarında bir adres yayımlıyorsan alan adının MX kaydı olmalı. `06-yigin.md`'deki MX tuzağına bak — bizimki gönderebiliyor ama alamıyordu, yayımlanmış kurallarımızdaki moderasyon adresi kimseye ulaşmıyordu.

## Ortam değişkenleri

```
.env            → her zamanki gibi .gitignore'a girer
.easignore      → EAS'in okuduğu BUDUR ve .gitignore'un yerine geçer
```

**Yok sayılan bir `.env` EAS arşivine hiç ulaşmaz.** Bundle boş değişkenlerle çıkar ve uygulama açılışta çöker — yalnız standalone build'lerde, yani simülatör de dev client de sorunsuz görünür. Uygulamalarımızdan birinde bunu bulana kadar **her** build bu yüzden çöküyordu. Ya EAS ortam değişkenlerini yapılandır ya da `.easignore`'un build'in ihtiyacı olanı dışlamadığından emin ol.

## Gerçek cihaza indir

Simülatör uygulamanın çalıştığını kanıtlamaz. Yalnız-standalone çökmeler reviewer'a ulaşan sınıftır:

```bash
npx expo export --platform ios --output-dir /tmp/exportcheck   # import hatalarını erken yakalar
```

Sonra kabloyla derleyip kur ve `devicectl --console` ile logu izle. Dinamik `import()` edilen, kurulu olmayan bir native modül geliştirmede görünmez — Metro sunar — ve standalone'da `RCTFatalException: Cannot find module` ile çöker.

## Devam etmeden önce

`npx tsc --noEmit` ve testlerini çalıştır, temiz olsunlar. Bu noktadan sonraki her build turu 5–40 dakika, incelemeye girdikten sonra ise gün maliyetli.

Sonraki: `02-testflight-ios.md` ya da `03-google-play.md`.
