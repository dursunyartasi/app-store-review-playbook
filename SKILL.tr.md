---
name: mobile-app-shipping-tr
description: Mobil uygulamayı sıfırdan App Store ve Google Play'e taşıma rehberi - gerçek redlerden ve gerçek build hatalarından yazıldı. Şunlarda kullan - mobil prototip başlatma, Expo/React Native kurulumu, IPA veya AAB derleme, TestFlight'a yükleme, Google Play'e yükleme, App Store incelemesine gönderme, App Review notu yazma, mağaza beyanlarını doldurma, mağaza reddini çözme. Ayrıca - "uygulama reddedildi", "Guideline", "App Review", "TestFlight", "eas build", "altool", "provisioning profile", "AAB", "Play Console", "App Store Connect", "yaş derecelendirme", "gizlilik beyanı", "ekran görüntüsü".
metadata:
  version: 2.0.0
  kaynak: 8 yayınlanmış iOS/Android uygulaması, 2026
---

# Mobil uygulama yayınlama

Buradaki her şey gerçekten yayınlanmış uygulamalardan ve yol boyunca yenen redlerle karşılaşılan build hatalarından geliyor. Hiçbiri resmî dokümandan aktarılmadı.

## Önce kullanıcının nerede olduğunu bul

Yalnız hâlâ ihtiyacın olanı sor. Kullanıcının mesajı bir soruyu zaten yanıtlıyorsa atla. **Beşten fazla soru sorma.**

1. **Şu an ne yapmak istiyorsun?**
   `yeni prototip` · `derleyip telefonuma atmak` · `TestFlight` · `Google Play` · `App Store incelemesine göndermek` · `red yedim`
2. **Hangi platformlar?** iOS, Android ya da ikisi.
3. **Kullanıcılar giriş yapıyor ya da içerik üretiyor mu?** (hesap, gönderi, yorum, mesaj, fotoğraf)
4. **Geliştirici hesapların var mı?** Apple Developer yılda 99 $ ve simülatör dışında hiçbir şeye ulaşmadan önce şart. Google Play tek seferlik 25 $.
5. **Arka uç var mı, yoksa gerekiyor mu?**

Sonra eşleşen dosyaya git. Yalnız onu oku.

| Yanıt | Oku |
|---|---|
| yeni prototip | `references/tr/01-prototip.md` |
| derleme / TestFlight | `references/tr/02-testflight-ios.md` |
| Google Play | `references/tr/03-google-play.md` |
| incelemeye gönderme | `references/tr/04-app-store-gonderim.md` |
| red yedim | `references/tr/05-redler.md` |
| arka uç, veritabanı, e-posta | `references/tr/06-yigin.md` |

Türkçe çeviri tamdır. İngilizce karşılıkları `references/` altında; İngilizce sürüm kanoniktir.

## Yanıtlar neyi değiştiriyor

**En kritik olan 3. soru.** Kullanıcılar giriş yapabiliyorsa Apple'a uygulama içi hesap silme borçlusun (5.1.1(v)) — yoksa reddedilirsin. Kullanıcılar başkalarının göreceği içerik üretebiliyorsa görünür şikâyet, engelleme, içerik filtresi ve DM onayı borçlusun (1.2) — dördü ayrı ayrı, ve uzun-bas jesti "görünür" sayılmıyor. Bunları prototipe göm. Red sonrası eklemek tam bir inceleme turu, yani günler.

**4. soru her şeyin kapısı.** Ücretli Apple hesabı olmadan TestFlight yok, 7 günlük ücretsiz profilin ötesinde cihaza kurulum yok, gönderim yok. Kullanıcı bir gününü harcamadan önce bunu söyle.

**5. sorunun ucuz bir yanıtı var.** `references/tr/06-yigin.md` servis başı ücret ödemeyen bir kurulumu anlatıyor: düz VPS üzerinde Coolify, PostgreSQL ve işlemsel posta için Brevo'nun ücretsiz katmanı.

## Her aşamada geçerli kurallar

- **Kullanıcının Apple/Google parolasını ve 2FA kodunu asla yazma.** App Store Connect ayrı giriş istiyor, geliştirici portalı oturumu taşınmıyor. Kullanıcıdan giriş yapmasını iste, onayını bekle, sonra API ve konsol adımlarını sen sürdür.
- **Beyan ve onay kutularını onun yerine işaretleme.** Bunlar uygulaması hakkında hukuki beyanlar.
- **"Upload succeeded" kabul edildi demek değil.** Apple işleme sırasında da reddediyor. Build `VALID` olana kadar yokla.
- **Kendi gördüğünü değil, reviewer'ın göreceğini test et.** `tr/05-redler.md`'deki redlerin çoğu geliştiricinin kendi cihazında ve kendi hesabında çalışan şeylerdi.
- **Reviewer'ı suçlamadan önce, erişemediği şeyin senin makinenin dışından erişilebilir olduğunu doğrula.** Birkaç guideline reddi aslında bir 404, bayat bir özellik bayrağı ya da eksik bir DNS kaydıydı.

## Sırası değişmeyecek adımlar

Bunları yanlış sıralamak tam build'lere mal oluyor:

1. Mevcut **sürüm dizesi** onaylı ya da yayındaysa, derlemeden ÖNCE yükselt — sürüm treni kapalı, yükleme reddedilir.
2. **Build numarasının boş olduğunu** derlemeden önce doğrula, sonra değil.
3. Yeni build bağlamadan önce **incelemedeki gönderimi iptal et.** Aynı anda iki sürüm incelenemez.
4. Göndermeden önce **bağlı build'i doğrula.** İptalden sonra attach çağrısı düşerken akış devam edip eski binary'yi gönderebiliyor.
