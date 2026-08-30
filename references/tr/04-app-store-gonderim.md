# App Store gönderimi

Build yüklendi ve `VALID`. Sıra metadata, beyanlar ve inceleme notlarında — redlerimizin çoğu aslında burada belirlendi.

## Kaydı oluştur (bir kez)

**App Store Connect uygulama kaydı API ile oluşturulamıyor.** Denedik ve doğruladık; tarayıcıdan yap. Bundle ID o kayda kalıcı olarak bağlanır.

Ondan sonrası — build bağlama, metadata, yaş derecelendirme, inceleme notları, gönderim — tamamen ASC API'sinden sürülebilir.

## Ekran görüntüleri

- **`APP_IPHONE_69` diye bir ekran görüntüsü tipi yok.** API'nin kabul ettiği en büyük tip `APP_IPHONE_67` (1290×2796). 6.9" cihaz için üretilen 1320×2868 görseller **reddediliyor**. 6.7" yükle, Apple büyüğe kendisi ölçekler.
- `whatsNew` **ilk sürümde düzenlenemiyor** — 409, "cannot be edited at this time". Yalnız güncellemelerde var.

## Yaş derecelendirme

- Alan tipleri karışık: bazıları BOOLEAN (`messagingAndChat`, `userGeneratedContent`, `advertising`), bazıları STRING enum (`contests`, `profanityOrCrudeHumor` → `NONE` / `INFREQUENT_OR_MILD` / `FREQUENT_OR_INTENSE`). Yanlış tip 409 döner ve hata doğru kümeyi söyler.
- **Apple 2025'te bandları değiştirdi: 12+ artık yok.** Bandlar 4+, 9+, 13+, 16+, 18+.
- Dürüst cevaplar 4+ üretebilir; `ageRatingOverrideV2` ile yükselt (ör. `THIRTEEN_PLUS`).
- **Uygulamada "insanlarla tanışma / networking" boyutu varsa `matureOrSuggestiveThemes` en az `INFREQUENT_OR_MILD` olmalı.** Boş bırakmak 2.3.6 reddi getirdi.

## App Privacy beyanı

- **Ulusal kimlik numarası "Hassas Bilgi" değil.** Apple'ın hassas listesi ırk, din, cinsel yönelim, biyometri ve benzerlerini sayar; kimlik numarası orada yok, dolayısıyla doğru kova **"Diğer Veri Türleri"**.
- **Kendi tuttuğun banka bilgisi "Toplanıyor"dur.** Apple yalnızca ödeme sağlayıcısı tutuyor ve sen erişemiyorsan muaf tutuyor.
- ⚠️ **Sihirbazda kör tıklama yapma.** Veri türüne göre farklı yükseklikte açılıyor, bu yüzden aynı konuma tekrar tıklamak "Kullanıcı Kimliği takip için kullanılıyor: EVET" gibi düpedüz yanlış cevaplar üretti. Her kalemde son durumu ekran görüntüsüyle doğrula.

## App Review notları — yazacağın en etkili metin

Redlerimizden biri tamamen bu alandan kaynaklandı. Apple'ın 4.2 "small, or niche, set of users" reddi bizim kendi cümlemizi bize geri okudu:

> "We are targeting a deliberately **small**, curated early community — **a few dozen** invited members, **not a mass-market** social network."

**Uygulamayı asla küçük, niş, yalnız-davetli, kapalı, özel, belirli bir topluluğa ait ya da kitlesel-olmayan diye çerçeveleme.** Apple bunu App Store değil Ad Hoc dağıtımı olarak okuyor.

Bunun yerine şunu yaz: herkese açık, ücretsiz, her yerden indirilebilir; kürasyonlu ya da üyelik katmanı *opsiyonel* bir katman. Sonra reviewer'ın **hesapsız** izleyebileceği yolu numaralı adımlarla anlat. Uygulamada gerçekten böyle bir yol yoksa göndermeden önce inşa et — bizim 4.2'yi geçiren şey buydu.

**Alan 4000 karakterle sınırlı.** Aşınca 409 döner.

Uygulamanın alışılmadık bir hedefi varsa (Watch, widget, cihaza özel bir akış), en üste açık giriş adımlarıyla bir "PLEASE READ FIRST" bölümü koy.

## Demo hesap

"Sign-In Required"ı işaretle ve bilgileri gir.

- **Önce cihazda test et.** Bir 2.1 reddi, Watch hedefinde hiç çalışmamış bir girişten geldi.
- **Hesapta içerik olsun.** Bir uygulamada tohumlanan 17 etkinliğin 16'sı geçmişte kalmıştı; reviewer boş bir uygulama açacaktı. Demo tarihlerini ileri kaydıran idempotent bir script tut ve her gönderimden önce çalıştır.
- **Doğrulama duvarı reviewer'ı kilitler.** Kaydolmuş ama doğrulamamış kullanıcı hiçbir şey göremiyorsa uygulama kapalı görünür. Misafir gezebilsin, doğrulama yalnız yazma eyleminde istensin.
- **Onaydan sonra demo hesabı kapat.** Şifresi App Store Connect'te duruyor.

## Yasal bağlantılar

Koşullar ve Gizlilik bağlantıları **tıklanabilir** olmalı, kullanıcıyı Safari'ye atmak yerine uygulama içi tarayıcıda açmalı ve yalnız kayıt değil **giriş** ekranında da bulunmalı. Tıklanamayan düz metin bir 2.1(a) reddiydi: reviewer koşulları okuyamadı ve yalnız bu yüzden reddetti.

## Uygulama ücretsiz ama bir yerde bir şey satıyorsan

3.1.1, B2B ve ücretsiz uygulamaların tuzağı. **Her fiyatı, plan adını, kredi sayacını, paywall'ı, yükseltme butonunu ve dışarıya satın alma bağlantısını kaldır.** Tek başına bir plan adı bir build'i düşürmeye yetti.

3.1.3(f) "Free Stand-alone Apps" argümanı **tek başına bizde işe yaramadı.** Zayıf halka herkese açık bir kayıt ekranıydı — tüketiciye self-servis satış gibi okunuyor ve 3.1.3(c)'nin "only sold directly by you to organizations" şartıyla çelişiyor. Kayıt ekranını sildik ve yalnız girişle yayınladık.

## Gönderim ve red sonrası yeniden gönderim

Sıra önemli. Yanlış sıra sessizce yanlış binary'yi gönderiyor.

1. **Aynı anda iki sürüm incelenemez.** Mevcut `reviewSubmission`'ı iptal et (`canceled=true`) ve CANCELING → COMPLETE'i bekle.
2. Sürüm `DEVELOPER_REJECTED` olur ve düzenlenebilir. Önce sürüm dizesini, sonra build ilişkisini PATCH'le.
3. ⚠️ **Takas tuzağı.** İptalden hemen sonra build-attach çağrısı 409 döner. Script'in yine de devam ederse **eski** build'i gönderir. Attach'i tekrar dene, sonra göndermeden önce `GET /appStoreVersions/{id}/build` ile **doğrula**. Bir kez bu şekilde yanlış build gönderdik.
4. ⚠️ `POST reviewSubmissionItems` geçiş tamamlanırken 409 `ENTITY_STATE_INVALID` dönebilir. Saniyeler sonra geçiyor — tekrar denenebilir yaz.

Yayın tipi varsayılan olarak **manuel**: onaydan sonra birinin yayınla düğmesine basması gerekiyor.

## Birden fazla tur bekle

Bir uygulama dört ardışık red yedi, bir diğeri üç. Birini düzeltmek bir sonrakini açığa çıkarabiliyor, ve bir alandaki düzeltme başka bir alanda sorun yaratabiliyor. **Her düzeltmeden sonra yalnız değiştirdiğin maddeyi değil, `05-redler.md`'deki listenin tamamını yeniden oku.**
