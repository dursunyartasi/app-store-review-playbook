# Google Play Console

Play, App Review'dan daha bağışlayıcı; ama sürümleri koddan çok evrak yüzünden kilitliyor — ve o kilitlere takılmak kolay. Bu dosya işin konsol tarafı. AAB'yi derlemek, imzalamak ve test etmek `07-android-derleme.md` içinde.

## 1. gün: uygulamayı oluştur ve *bir şey* yükle

Android'de yaptığımız en pahalı hata bir red değildi. **Play Console'da hiç var olmamış bir uygulama için derlenmiş on altı üretim AAB'siydi.** Her rapor "elle yüklemeye hazır" diyordu; iOS TestFlight'ta ilerledi; Android kanalı günlerce sıfırda kaldı.

Bu yüzden 1. gün, hiçbir şeyi cilalamadan önce:

1. **Uygulamayı Play Console'da elle oluştur.** Bunun API yolu yok.
2. **İlk AAB'yi Dahili test (Internal testing) kanalına yükle.** Bu, Play'in uygulama imzalama anahtarını oluşturur, ihtiyacın olacak ikinci SHA-1'i verir (aşağıya bak) ve `eas submit -p android` yolunu açar.
3. Bu yapılana kadar **Android üretim build'i almayı bırak.** Build kotasını yer ve Masaüstünde birikir.

Geri alamayacağın iki seçim:
- **"Ücretsiz", yayından sonra ücretliye çevrilemez.**
- Paket adı, iOS bundle ID gibi kalıcıdır.

### Kapalı test şartı — hangi kuralın sana uyduğunu kontrol et

**13 Kasım 2023'ten sonra açılmış kişisel geliştirici hesapları**, üretim erişimine başvurabilmek için bile **en az 12 testçinin katıldığı, 14 gün kesintisiz süren** bir Kapalı test (Closed testing) yürütmek zorunda. (Eskiden 20'ydi; bir kullanıcıya bir keresinde eski sayıyı söyledik. 12.) 14 gün ancak bir kapalı test sürümü gerçekten yayına girince başlıyor — 1. gün yüklemek için bir sebep daha.

**Eski hesaplar ve kuruluş hesapları muaf.** Bizimki 2016'dandı ve her uygulamayla doğrudan Üretim (Production) kanalına gitti. Var olmayan iki haftalık bir gecikmeyi planlamadan — ya da tutturulamayacak bir yayın tarihi vermeden — önce hesabın açılış tarihine bak.

### Geliştirici doğrulaması

Google 2026'dan itibaren geliştiricilerin kimliklerini doğrulamasını ve paket adlarını kaydettirmesini istiyor. Zaten yayında olan paketler kimlik sekmesinde "Kayıtlı" ("Registered") olarak görünüyordu. **Her yeni paket adı için kontrol et** ve hatırlatma e-postalarını bir kenara atmak yerine gereğini yap.

## İmzalama ve herkesi yakalayan SHA-1

Sen bir **yükleme anahtarıyla** imzalarsın; Play kendi **uygulama imzalama anahtarıyla** yeniden imzalar — ve o anahtarı ancak ilk AAB yüklemenden sonra üretir. İki parmak izi de Play Console → Test et ve yayınla (Test and release) → Uygulama bütünlüğü (App integrity) sayfasında (URL: `.../app/<id>/keymanagement`).

**Parmak izi isteyen her şey İKİSİNİ de ister** — Google ile Giriş için Android OAuth istemcisi, Google Maps API anahtarı kısıtlaması, App Links veya TWA için `assetlinks.json`. Play'in anahtarını atlarsan özellik *tam da mağaza build'inde* bozulur, yerel build'in ise çalışır.

- **Parmak izlerini konsolun kopyala düğmeleriyle kopyala.** Sayfa iki anahtar için SHA-1 ve SHA-256 listeliyor; az kalsın bir SHA-1 alanına SHA-256 yapıştırıyorduk. Tek yanlış karakter sessizce başarısız olur.
- **Yükleme anahtarı kalıcı bir kimliktir.** Bir ajan, kullanıcının açık onayı olmadan bir tane üretmemeli — ve sessizce üretilmesine de izin vermemeli. EAS tam olarak bunu yapıyor: ilk `eas build -p android --non-interactive` çalıştırması `Created keystore` yazar ve keystore'u Expo'nun sunucularında saklar, yerel kopya yok. Hemen `eas credentials -p android` ile yedekle.
- Keystore'u **proje klasörünün dışında** tut. Bir tanesini repo kökünde, bir iOS `.p12` ve provisioning profile'ın yanında bulduk — build arşivine girmesine elle yazılmış tek bir `.easignore` kalmıştı.

Her yüklemeden önce AAB'yi gerçekte hangi anahtarın imzaladığını doğrula (kontrol `07-android-derleme.md` içinde). Debug anahtarıyla imzalanmış bir paket `jarsigner -verify`'dan gönül rahatlığıyla geçer.

## Yükleme

### Bir yapay zekâ ajanı AAB'yi tarayıcıdan yükleyemez

Tarayıcı otomasyonunun dosya yükleme sınırı 10 MB; tipik bir AAB 55–100 MB. macOS dosya seçicisini AppleScript ile sürmek Erişilebilirlik izni istiyor ve `-25211` ile başarısız oldu. Üç uygulamanın her sürümü, bununla uğraşmayı bırakana kadar bir insan turu gerektirdi.

**İşe yarayan devir teslim:**
1. AAB'yi derle ve doğrula.
2. `~/Desktop/<app>-<version>-vc<N>.aab` konumuna kopyala ve Finder dosyayı göstersin diye üzerinde `open -R` çalıştır.
3. Play Console'da sürüm sayfasını aç.
4. Kullanıcı dosyayı sürükleyip bırakır; ajan sürüm notları ve gönderimle devam eder.

Play Console otomasyona yanıt vermeyi keserse (sayfa hiç boşa düşmüyor ve her tıklama zaman aşımına uğruyorsa), **yeniden denemeyi bırak ve devret.** Bir oturumu buna kaybettik.

**Asıl çözüm bir Play servis hesabı** ve `eas.json > submit.android` (`serviceAccountKeyPath`, `track: "internal"`). Uygulama oluşur oluşmaz kur; ondan sonra bütün bu dansın yerini `eas submit -p android` alır. İki ay boyunca her sürümde önerdik ve hiç yapmadık — bunu tekrarlama.

### Karşılaşacağın sürüm hataları

- **"N sürüm kodu zaten kullanıldı." ("Version code N has already been used.")** Bir sürüm kodu, *herhangi bir* yükleme Play'e ulaştığı anda tüketilir — hiç yayınlamadığın bir taslaktaki yükleme dahil. Artır ve yeniden derle. Artırdıysan, büyük ihtimalle native proje bunu almamıştır: `07-android-derleme.md` içindeki prebuild bölümüne bak. Bu bizi bir hafta içinde iki kez vurdu.
- **"Bu sürüm, mevcut kullanıcıların yeni eklenen uygulama paketlerine geçmelerine izin vermediği için kullanıma sunulamaz." ("This release will not be available to existing users because it doesn't allow them to upgrade to the newly added app bundles.")** → sürüm kodunu yükselt, ya da önce Dahili test / Kapalı test üzerinden yayınla.
- **"Bu sürüm hiçbir uygulama paketi eklemiyor veya kaldırmıyor." ("This release adds or removes no app bundles.")** → AAB sürüme eklenmemiş. App bundle gezgininde (App bundle explorer) duruyor ama taslakta olmayabilir; yeniden yüklemek yerine sürümde **Kitaplıktan ekle (Add from library)** kullan.
- **Native debug sembolleri** ABI klasörleri içeren bir `native-debug-symbols.zip` olmalı — `armeabi-v7a/`, `arm64-v8a/`, `x86_64/`, her birinde `libapp.so` — ve **`__MACOSX` veya `.DS_Store` girdisi bulunmamalı**.
- **"Kod gizleme dosyası yok" ("No deobfuscation file")** R8 kapalıyken (Expo varsayılanı) bir uyarıdır, engel değildir.

### Sürüm notları

- **Notları dil etiketleriyle sar** — `<tr-TR>…</tr-TR>`, `<en-US>…</en-US>` — her mağaza girişi dili için bir blok; yoksa alan kırmızıya döner.
- **Dil başına 500 karakter.** İlk taslağımız uzunluk yüzünden reddedildi.
- Alanı JavaScript veya form otomasyonuyla doldurmak etiketleri HTML-escape edip `&lt;tr-TR&gt;` haline getirebilir. Ham değeri ata ve kaydetmeden önce gözünle bak.

## Uygulama içeriği: bizi kilitleyen her beyan

Giriş noktası: `.../app/<id>/app-content/overview`. Tek tek beyan sayfalarına giden derin bağlantılar güvenilmez; genel bakış sayfasından başla. Her şeyi uygulamanın *gerçekte ne yaptığına* göre yanıtla — yanıtlar manifest'inle karşılaştırılıyor. **Bir ajan bunları kullanıcı adına işaretlememeli**; bunlar kullanıcının uygulaması hakkında yasal beyanlar.

| Beyan | Bizi nerede düşürdü |
|---|---|
| **Gizlilik politikası (Privacy policy)** | URL 200 dönmeli. İlk üretim gönderimimiz sırf adres 404 verdiği için reddedildi. |
| **Uygulama erişimi (App access)** | Aşağıdaki demo hesap bölümüne bak. |
| **Reklamlar (Ads)** | Reklam kimliği beyanından ayrı. İkisi de var. |
| **İçerik derecelendirmesi (Content rating)** | Anket; hızlı, ama üretimden önce zorunlu. |
| **Hedef kitle (Target audience)** | Aileler politikasının dışında kalmak için 18+ seçtik. 13 yaş altı bir aralık seçmek bütün bir ek incelemeyi beraberinde getirir. |
| **Veri güvenliği (Data safety)** | CSV içe aktarmayı kullan (aşağıda). Uygulamada hesap varsa **herkese açık bir hesap silme URL'si gerekiyor** — uygulama içi silmeden ayrı, ve Google onu çekiyor: yayına alınmamış bir sayfa 404 olarak görünür. Kısmi silme URL'si de isteniyor. |
| **Reklam kimliği (Advertising ID)** | Manifest'le birebir tutmalı. Aşağıya bak — bize en pahalıya patlayan bu oldu. |
| **Devlet uygulamaları, Finansal özellikler, Sağlık (Government apps, Financial features, Health)** | Geçerli değilse her biri için açıkça "yok" denmeli. Para hareketi olmadan IBAN saklamak "finansal özellik yok" sayıldı. Adım sayar bir **Sağlık** beyanıdır ("Aktivite ve fitness" / "Activity and fitness"). |
| **Mağaza girişi (Store listing)** | Yapay zekâyla üretilmiş mağaza görselleri beyan edilmeli. Mağaza ayarlarındaki (Store settings) iletişim bilgileri **herkese açık** — telefon numarasını bilerek boş bıraktık. |
| **İçerik hakları, kategori, fiyat** | Ana kategori, içerik hakları, fiyat (ücretsiz) ve ülkelerin hepsi ayarlanana kadar gönderim "bu kaynak incelenemiyor" ("this resource can't be reviewed") ile kilitliydi. |
| **Ülkeler / bölgeler** | Uygulama düzeyinde **ve** Üretim kanalının boş başlayan kendi Ülkeler sekmesinde ayarlanır. Uygulamalarımızdan biri üretimde tek bir ülkede çalışırken iOS 175 ülkede yayındaydı. Yüklemeler sıfır görünüyorsa her şeyden önce buna bak. |

### Veri güvenliği: tıklama, içe aktar

Tıklayarak doldurmak yaklaşık 50 diyalog demek ve seçimleri defalarca kaybetti ("0/9 kaydedildi"). Onun yerine:
1. Veri güvenliği sayfasından CSV şablonunu dışa aktar.
2. Bir script'le doldur, **dışa aktarılan dosyadaki anahtarları birebir kullanarak** — uydurma adlar başarısız olur. Gerçek olanlar `PSL_DEVICE_ID`, `PSL_FILES_AND_DOCS`, `PSL_OTHER_MESSAGES` gibiydi. Bir push token'ı "Cihaz veya diğer kimlikler" ("Device or other IDs") altına girdi.
3. İçe aktar — sonra ayrıca çıkan **İçe aktar** onayına tıkla. İlk seferde onu kaçırdık.
4. Kaydedildiğini kanıtlamak için yeniden dışa aktar ve farkına bak.

### Reklam kimliği kilidi

AAB'yi aç ve `com.google.android.gms.permission.AD_ID` ara (komut `07-android-derleme.md` içinde). Firebase Analytics izni projeye çeker ve buna uyan bir "kullanılıyor" beyanı ister; reklamı ve analitik SDK'sı olmayan bir uygulamada ikisi de olmamalı. **Beyan manifest'le birebir tutmalı** — iki yönde de uyumsuzluk yayını kilitler, ve Play'in kendi uyarı metni hangi tarafın yanlış olduğu konusunda yanıltıcı olabilir.

**İlk gönderimde doğru yap.** Bir uygulamada Play, incelemeye gönder düğmesi kilitliyken sürekli "Reklam kimliği beyanı eksik — tüm değişikliklerinizi etkiliyor" ("Advertising ID declaration missing — affects all your changes") gösterdi; bu sırada beyan sayfasında "Hayır" kayıtlı görünüyordu ve yapılacaklar listesi boştu. Birkaç güne yayılan yaklaşık on beş saat harcadık:
- ön kontrollerin bitmesini bekledik (~12 dakikaya kadar sürüyor) — değişiklik yok;
- Kaydet'i yeniden etkinleştirmek için Evet → Hayır arasında gidip geldik — değişiklik yok;
- yeni bir AAB yükledik — değişiklik yok;
- ayrı Reklamlar beyanını kontrol ettik — değişiklik yok.

Uyarı görünürken iki sürüm bile yayına çıktı. Sonunda açık bir destek talebi olarak kaldı. **Kilidi açmak için "Evet"e çevirme** — "Evet" seni bir amaç seçmeye zorlar (işlevsellik, analitik, reklam), ve bu yanlış bir beyan olur.

### Hassas izinler

Arka plan konumu ve `FOREGROUND_SERVICE_LOCATION`, **tanıtım videosu** ve inceleme gerektiren bir izin beyanını tetikliyor. Bunlarla derlenmiş bir yürüyüş uygulaması aynı anda üç engele takıldı:

- "Bu sürüm, Play Console'da beyan edilmemiş izinler içeriyor" ("This release contains permissions that haven't been declared in Play Console")
- "Uygulamanızın ön plan hizmeti izinlerini kullanıp kullanmadığını bize bildirmeniz gerekiyor" ("You need to tell us whether your app uses foreground service permissions")
- "Sağlık beyan formunu doldurmanız gerekiyor" ("You need to complete the Health declaration form")

Henüz arka plan konumuna ihtiyacın yoksa onu blokla ve yeniden derle:

```json
"android": { "blockedPermissions": ["android.permission.ACCESS_BACKGROUND_LOCATION",
                                    "android.permission.FOREGROUND_SERVICE_LOCATION"] }
```

Ön plan hizmeti uyarısı kendiliğinden kalktı; Sağlık formu ise adım sayar yüzünden yine de doldurulmak zorundaydı. **Kod da uymalı** — `07-android-derleme.md` içindeki "Bir izni bloklamak yetmez" bölümüne bak.

## İnceleme

### Demo hesap: 2FA yüzünden reddedildik

Play bir üretim gönderimini **"çok faktörlü kimlik doğrulama erişimi engelliyor"** ("multi-factor authentication blocks access") gerekçesiyle reddetti. İnceleme hesabında 2FA açıktı ve inceleyici e-postayla gelen altı haneli kod ekranına takıldı. Uygulama erişimi talimatlarında "2FA yok" yazıyordu — önceki bir taslaktan kopyalanmış, veritabanındaki bayrakla hiç karşılaştırılmamıştı.

Aynı reddin kanıt ekran görüntüsünde Google ile giriş düğmesinin 400 ile başarısız olduğu görülüyordu. **İnceleyiciler sosyal girişi de deniyor**, yalnız onlara verdiğin kimlik bilgilerini değil.

Göndermeden önce **demo hesapla temiz bir cihazdan, gerçek bir istekle giriş yap** ve inceleyicinin ne göreceğini doğrula.

**Uygulama erişimi formunun davranışı:**
- Kimlik bilgisi satırını **Kaydet**'ten önce **Ekle** ile ekle; yoksa sayfadan ayrılma uyarısı alırsın ve satırı kaybedersin.
- Talimatlar: **İngilizce, en fazla 500 karakter.**
- Bir retten sonra kimlik bilgileri tablosu boş *görünebilir* ama girdi hâlâ vardır — yenisini oluşturmak "ad zaten kullanılıyor" ("name already used") ile başarısız olur. Mevcut olanı düzenle.
- Şifre alanını kullanıcı doldurur. Ajan şifreyi girmez.

### Gerçekte gördüğümüz süreler

| Olay | Süre |
|---|---|
| Dahili test | anında, inceleme yok |
| İlk üretim incelemesi | birkaç gün (biri dört gün sürdü ve redle bitti) |
| Sonraki güncellemeler | yaklaşık 1 saatten birkaç saate |
| Mağaza girişi değişiklikleri | 1–3 gün; bu sırada yayındaki uygulama yerinde kalır |

- **İnceleme beklerken yeni değişiklik göndermek sıradaki yerini sıfırlar.**
- Bir retten sonra yeniden göndermek "inceleme sıfırlanacak" uyarısı göstermez, çünkü kaybedecek bir yer kalmamıştır.

### Play'in hızı iki tarafı da keser

Bozuk bir sürüm bir saat içinde yayında olabilir ve **geri çekilemez**. Bizimkilerden biri girişten hemen sonra çöken bir hatayla çıktı; tek çare yeni bir sürüm kodu ve bir bekleme daha oldu. **Önce Dahili test kullan.**

## Yayından sonra

**Play Vitals, kabloyla bağlı bir telefon olmadan gerçek stack trace'i verir.** O giriş çökmesi Vitals'tan kanıtlandı — `IllegalStateException: API key not found ... com.rnmaps.maps.MapView.onCreate` — ve düzeltmeyi de Vitals ile doğruladık (10 çökme → 0).

Gördüğümüz **Lansman öncesi rapor (Pre-launch report) ve Vitals uyarıları**, hiçbiri engel değil:
- Bellek kullanımı aylar sonrasına son tarihli "kötü davranış" ("bad behaviour") olarak işaretlendi. Kaçırmak bulunabilirliği düşürür; uygulamayı kaldırmaz.
- R8 kapalı olduğu için kod küçültme %2'de. Açmak AAB boyutunu düşürür, ama tam bir test turu gerektirir.
- Android 15'te kenardan kenara (edge-to-edge) için kullanımdan kaldırılmış API'ler. Bir Expo SDK yükseltmesi çoğunu düzeltir.
- Yalnız dikey / büyük ekran yönlendirme kısıtlamaları.
- "4 derin bağlantı başarısız olabilir" ("4 deep links may fail") — `assetlinks.json` dosyası olmayan web alan adları.

**"0+ indirme", "yayınlanmadı" demek değil.** Bir kullanıcı bunu öyle okudu. Önce ülkelere bak.

## Hedef API düzeyi son tarihleri

Play, hedef API düzeyini yükseltme son tarihini kaçıran uygulamalarda güncelleme kabul etmeyi durduruyor. Tarih her yıl kayıyor. **Takip et** — sürüm gününde öğrenmek kötü bir gün.
