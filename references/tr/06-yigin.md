# Rehberin arkasındaki yığın

[Red rehberi](05-redler.md) incelemeyi geçmekle ilgili. Bu dosya altta ne döndüğüyle ilgili — bu uygulamaların üzerinde yayınlandığı yığın ve onu işletirken çarptığımız hatalar.

Rehberdeki kural burada da geçerli: yalnızca gerçekten yaşananlar. Bu **bizim kullandığımız**, senin kullanman gerekeni söyleyen bir iddia değil.

---

## Ne kullanıyoruz

| Katman | Seçim | Neden |
|---|---|---|
| Mobil | **Expo / React Native** (SDK 57) | İki mağaza için tek kod tabanı; EAS ya da tamamen yerel build |
| Web / API | **Next.js** | İki uçta aynı TypeScript |
| Veritabanı | **PostgreSQL**, kendi sunucumuzda | Öngörülebilir maliyet; satır başı fiyat sürprizi yok |
| ORM | **Prisma** | Üretime dokunmadan önce gözden geçirebildiğimiz migration'lar |
| Dosyalar | **MinIO** (S3 uyumlu) ya da Cloudflare R2 | Kendi sunucumuzda nesne depolama; çıkış trafiği faturası yok |
| Barındırma | Düz bir VPS üzerinde **Coolify** | Kendi sunucunda PaaS: git deploy, TLS ve konteynerler, servis başı ücret olmadan |
| E-posta | **Brevo** ücretsiz katman, SMTP ile | Günde 300 posta ücretsiz; OTP ve bildirimler için uzun süre yetiyor |
| Ödeme (Türkiye) | **iyzico** | Yerel kartlar ve taksit — Stripe burada bunu karşılamıyor |

Bir uygulama kendi Postgres'imiz yerine Supabase üzerinde çalışıyor. İkisi de iş görüyor; aşağıda hangi dersin Supabase'e özel olduğu belirtildi.

---

## Coolify ve dağıtım

Coolify kendi VPS'inde çalışan bir PaaS. Servis başı barındırma faturasını ortadan kaldırıyor, karşılığında yönetilen bir platformun senin adına yuttuğu işletme hatalarını sana devrediyor.

### Asıl çarpacağın hata disk baskısı
Sunucu diski kabaca **%80'i** geçtikten sonra, build başarılı olsa bile deploy katman-dışa-aktarma aşamasında patlıyor. Coolify bunu `exit code 255` ya da genel bir `DeploymentException` olarak gösteriyor — **gerçek sebep gizli.** Dışa aktarma için ~20 GB boş alan gerekiyor.

```bash
docker system df           # önce bak
docker builder prune -af   # güvenle silinebilecek olan build cache'idir
```

Temizlik **simetrik değil** ve tersini yapmak bir deploy'a mal oluyor:

- `docker image prune -af` — her zaman güvenli.
- `docker builder prune -af` — **deploy'dan hemen önce asla.** Build cache'ini
  siler, `apt-get` katmanı yeniden çalışıp paketleri indirir; tek bir geçici ağ
  tökezlemesi build'i düşürür. Deploy yolunun dışındaysan elindeki en etkili araç.

Image'ların çoğu referanslı olduğu için image prune az yer açar. Daha fazla
Coolify arıza modu — aynı hatayı üreten üç ayrı sebep dahil —
[Coolify işletim rehberinde](https://github.com/dursunyartasi/coolify-operations-playbook). **Volume'lara DOKUNMA — onlar uygulama verisi.** Bir olayda bu, diski %92'den %83'e indirip 7,6 GB açtı; deploy tekrar denemede geçti.

Aynı disk baskısı, build yardımcı konteyneri build ortasında ölünce geçici `No such container: <uuid>` olarak da görünüyor. Bellek baskısı da aynı belirtiyi veriyor, ikisine birden bak.

### Bilinmesi gereken diğer deploy davranışları
- **Deploy, compose dosyasındaki her servisi yeniden oluşturur**, yalnız değişeni değil — veritabanı konteynerin dahil, ve onun **adı da değişir**. Konteyner adına bağlı ne varsa kırılır; her deploy sonrası yeniden çöz.
- **Deploy kabaca 200–300 saniye sürüyor.** Yeni konteyner + HTTP 200 için yokla; tetikleme çağrısının döndüğüne bakıp başarı varsayma.
- **İlk deneme sebepsiz başarısız olabiliyor** (compose aşaması). Tekrar denemek genelde çözüyor ve üretim düşmüyor.
- **Deploy varsayılan olarak webhook ile tetiklenmiyor** — manuel ya da API çağrısı.
- VPS'in **Cloudflare arkasındaysa** varsayılan `urllib` kullanıcı ajanı engelli. Kendi API'ne script yazarken curl kullan ya da tarayıcı UA'sı ver.

### Postgres notları
- **Supabase / PostgREST:** yeni bir tablo, tablo gerçekten var olduğu halde `PGRST205 "Could not find the table in schema cache"` döndürüyor. REST cache'i bayat. Çözüm: `NOTIFY pgrst, 'reload schema'`.
- **Realtime `wal_level=logical` istiyor.** Varsayılan `replica`'da `postgres_changes` sorunsuz abone oluyor ve sonra hiç olay göndermiyor — istemci hatası gibi görünen sessiz bir arıza. Değiştirmek konteyner yeniden başlatması gerektiriyor, bakım penceresi al.

---

## Ücretsiz katmanda e-posta — ve neredeyse her şeyi bozan DNS tuzağı

Brevo'nun ücretsiz katmanı (günde 300 posta) OTP, şifre sıfırlama ve bildirimler için uzun süre yetiyor. Uygulamayı `smtp-relay.brevo.com:587`'ye bağla.

Postaların çöpe değil kutuya düşmesi için alan adının Brevo'da **Authenticated** görünmesi gerekiyor:
- **DKIM** — Brevo'nun verdiği iki CNAME kaydı
- **DMARC** — `p=none` ile başla
- **SPF** — `include:spf.brevo.com`
- Brevo'nun doğrulama TXT kaydı

### ⚠️ SPF tuzağı
Aynı alan adında posta *almak* için Cloudflare Email Routing'i açtık. Cloudflare "eksik kayıtları ekleyeyim" dedi, alan adında Brevo için zaten bir SPF kaydı olduğunu gördü ve çakışmayı **Brevo kaydını silerek** çözmeyi önerdi.

Bunu kabul etmek, uygulamanın gönderdiği her postadan — OTP, bildirim, şifre sıfırlama — kimlik doğrulamasını söker ve hepsini spam'e düşürürdü. Doğrusu iki include'u **tek** kayıtta birleştirmek:

```
v=spf1 include:spf.brevo.com include:_spf.mx.cloudflare.net ~all
```

**Bir alan adında tam olarak bir SPF kaydı olmalı.** Birden fazlası RFC'ye aykırı ve tüm gönderimi bozar. `dig` ile doğrula, panele güvenme.

### MX tuzağı — ve bunun neden bir mağaza sorunu olduğu
Aynı alan adının **hiç MX kaydı yoktu**. Gönderebiliyor ama alamıyordu. Yayımladığımız moderasyon iletişim adresi kimseye ulaşmıyordu.

Bu yalnız bir e-posta hatası değil. App Store **Guideline 1.2** içerik şikâyeti için çalışan bir yol bekliyor ve kendi kurallarımız üç iş günü içinde yanıt sözü veriyordu. Postayı sessizce yutan bir adres hem tutulmayan bir taahhüt hem inceleme riski. **Mağaza listelemende ya da uygulama içi kurallarında bir iletişim adresi yayımlıyorsan, oraya test postası gönder.**

Bir de: Brevo göndermeyi bir IP izin listesiyle kısıtlayabiliyor. Hem geliştirme makineni hem sunucunu ekle, yoksa yerel testler geçerken üretim postası ölür.

---

## Mobil build notları

Build ve yükleme tuzaklarının tamamı [rehberde](05-redler.md). Arkasındaki yığın kararları:

- **İterasyon sırasında yerel build EAS remote'u yener.** Remote kuyruklar doluyor ve non-interactive EAS kimlik bilgilerini güncellemiyor — yani yeni bir capability'den eski kalmış bir provisioning profile seni çıkışsız bırakıyor. Yerel `xcodebuild` + `xcrun altool` kaçış yolu.
- **`.env`'i EAS açısından düşün.** Gitignore'daki bir `.env` EAS arşivine hiç gitmez; bundle'da boş değişken ve yalnız standalone build'de görünen bir açılış çökmesi üretir.
- **Yerel Android build `ANDROID_HOME` istiyor**, yoksa Gradle "SDK location not found" der.
- **Play yüklemesini servis hesabıyla otomatikleştir** (`eas.json > submit.android`). Elle `.aab` yükleme en uzun süre manuel kalan adım ve tarayıcı otomasyonu yardım edemiyor — dosyalar her yükleme sınırının çok üstünde.

---

## VPS nereden

Coolify root erişimli düz bir VPS istiyor; yönetilen bir platform gerekmiyor. Docker çalıştıran ve genel IP'si olan her sağlayıcı iş görür. Çalıştırdıklarımızdan ölçekleme: uygulama için küçük bir instance yeterli, ama **diske gerekli hissettiğinden fazla yer ver** — yukarıdaki katman-dışa-aktarma arızası CPU değil disk sorunu. Image'larının ötesinde 20 GB boşluk bütçele.

Bizimkiler Hostinger'da. **Referans linki — [hostinger.com](https://www.hostinger.com/tr?REFERRALCODE=KAWDURSUNLTO)** — kullanırsan yazara komisyon kazandırır, sana da indirim sağlar. Zorunlu değil: Coolify, Docker ve root erişimi olan her sağlayıcıda çalışır ve bu rehberdeki hiçbir şey barındırıcıya bağlı değildir.

---

## Bunun incelemeyle bağlantısı

Rehberdeki mağaza redlerinin birkaçı, üzerine guideline numarası giydirilmiş altyapı sorunlarıydı:

| Şuna benziyordu | Aslında şuydu |
|---|---|
| Guideline 1.2, içerik şikâyeti yolu yok | MX kaydı olmayan, yayımlanmış bir iletişim adresi |
| Play gönderimi reddedildi | Beyan edilen gizlilik politikası adresi 404 veriyordu |
| 2.1 App Completeness, "açılışta çöktü" | `.env` build'e hiç ulaşmamıştı |
| 2.1, "özelliğe erişemedik" | Üretimde kapalı bırakılmış bir özellik bayrağı |

Reviewer'ı suçlamadan önce, erişemediği şeyin senin makinenin dışından gerçekten erişilebilir olup olmadığına bak.
