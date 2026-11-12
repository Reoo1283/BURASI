# Haxball Ajan Ekibi — 1. Tur Özeti

**Proje:** Reoo1283/haxball-serverr (`main` @ `ad78869`)
**Tarih:** 1. tur · **Yöneten:** Haxball Proje Yöneticisi
**Görev alan ajanlar:** Oda Çekirdek Mühendisi, Ban Mute Kimlik Mühendisi, Süreç Dayanıklılığı Mühendisi, Sır ve Yapılandırma Mühendisi, Test ve Kanıt Mühendisi
**Kural:** Hiçbir ajan GitHub'a yazmadı, kod değiştirmedi. Bu rapor yalnızca okuma üzerinedir.

> **Ortak sınır:** `src/` klasörü depoda yok. 5 ajanın tamamı bunu bağımsız olarak tespit etti: `server.js:6-23` içindeki 20 modül require ediliyor ama dosyalar yok — **sunucu bu haliyle hiç boot etmiyor**. `src/` gelmeden o kod hakkında hiçbir yargı kesin değil.

---

## 1. Toplu tablo — tekilleştirilmiş bulgular

5 ajanın toplam **76 bulgusu** tekilleştirildi → **21 benzersiz bulgu**. Aynı satıra birden fazla ajan bağımsız olarak değindiyse "kaç ajan" sütununda görünüyor; bu, önem sıralamasının en güçlü kanıtı.

| # | Önem | Yer | Bulgu | Kaç ajan | Sahip |
|---|------|-----|-------|----------|-------|
| 1 | 🔴 | `.env` (tümü), `README.md:2` | 7 canlı sır + README'de e-posta/şifre commit edilmiş | 3 | Sır |
| 2 | 🔴 | `server.js:93-99` | `netstat`+`taskkill /F /T` ile sahiplik kontrolü olmadan yabancı süreç öldürme | 3 | SRE |
| 3 | 🔴 | `server.js:179-183` | Ban anahtarı çıplak `player.name` + `state.username`; isim değiştirerek atlatma, masumun yanlış banlanması | 2 | Ban/Kimlik |
| 4 | 🔴 | `server.js:191-224` | 400 ms timeout **fail-closed**; DB yavaşlarsa odaya kimse giremez (fail-frozen) | 3 | Ban/Kimlik + SRE |
| 5 | 🔴 | `server.js:176-240` | Join async; oyuncu kontroller bitene kadar odada serbest (banlıyken oynayabiliyor) | 2 | Çekirdek |
| 6 | 🔴 | `server.js:139-345` | `room.onPlayerLeave` **hiç tanımlı değil**; state/timer/live-set temizliği yok | 1 | Çekirdek |
| 7 | 🔴 | `server.js:356-364` | `uncaughtException`/`unhandledRejection` sonrası süreç yarı-canlı kalıyor | 2 | SRE |
| 8 | 🔴 | `remove_comments.js:19-31`, `strip_script.js:1`, `verify.js:21-31` | Üç araç betiği: biri sahte başarı, biri sessiz yıkım, biri tanımsız bağımlılık | 2 | Sır + Test |
| 9 | 🟠 | `server.js:106-114` | Token doğrulaması `startWebServer` sonrasında; geçersiz token'da yarım kurulu web | 2 | SRE + Sır |
| 10 | 🟠 | `server.js:160-172` | `initState` red yollarından önce; duplicate-auth kick'inde state sızıntısı, sessiz `catch` kontrolü kapatıyor | 2 | Çekirdek + Ban |
| 11 | 🟠 | `server.js:221-222` | Ban sebebi oyuncuya aynen gidiyor; iç not sızıntısı | 2 | Ban/Kimlik |
| 12 | 🟠 | `server.js:149,162,271,275` | `auth`/`conn` loglara düz yazılıyor (kalıcı kimlik + IP) | 2 | Ban/Kimlik |
| 13 | 🟠 | `server.js:283` | Şifre sohbet satırında isteniyor (`!giris <sifre>`) → ekran görüntüsü/log riski + kimlik avı zemini | 2 | Ban/Kimlik |
| 14 | 🟠 | `server.js:40-57` | `JWT_SECRET === ADMIN_JWT_SECRET` kontrolü yok; yetki yükseltme riski | 2 | SRE + Sır |
| 15 | 🟠 | `.gitignore:9-13` | `*.db`, `*.db-wal`, `*.db-shm`, `*.rar` desenleri yok; canlı SQLite + 49 MB arşiv izleniyor | 2 | Sır |
| 16 | 🟠 | `server.js:368-384` | Shutdown interval/timer temizlemiyor, zorunlu zaman aşımı yok; `closeDB` asılırsa süreç sonsuza kalır | 1 | SRE |
| 17 | 🟡 | `server.js:141-147` | İsim normalizasyonu yalnız `İ`; `ı`, zero-width, NFKC, `id===0` host muafiyeti | 2 | Çekirdek + Ban |
| 18 | 🟡 | `server.js:305-324` | VIP cooldown ham isimle anahtarlanıyor; ayrılıp giden oyuncuya duyuru, isim varyantı cooldown'u atlıyor | 1 | Çekirdek |
| 19 | 🟡 | `server.js:242-252` | `dizilim` modunda erken `return` → hoş geldin, takım, VIP, kimlik gözlemi hiç çalışmıyor | 1 | Çekirdek |
| 20 | 🟡 | `server.js:74-86` | `verifyLatestBackup` reddederse yedek scheduler **hiç başlamıyor**, yalnız log'a yazıyor | 1 | SRE |
| 21 | 🟡 | `package.json:6,9,12` + `scripts/` | `test`, `bot`, `lint`, `typecheck` betiklerinin dosyaları yok; test altyapısı sıfır | 2 | Test |

---

## 2. Alan alan analiz

### 2.1 Oda çekirdeği — `onPlayerJoin` ve yaşam döngüsü
En kritik tespit: join akışı **tek bir kilit noktasına sahip değil**. `applyJoinFlow`'a giden yol 5 ayrı yerde `return` ediyor ve yalnızca 2'si gerçekten kick ediyor. Ayrıca `room.onPlayerLeave` tanımlı olmadığı için `initState` ile açılan her state'in tek çıkışı yok — Haxball `player.id`'yi geri dönüştürdüğünde yeni oyuncu eski oyuncunun `teamId`/`mutedUntil`/`dbId` değerlerini miras alabilir.

**Geliştirme önerisi (öncelikli):** `handleJoin(player)` tek fonksiyon + tek `reject(reason, code)` yardımcısı; `playerState`'e `joinSeq` ekleyip her async adımda `if (getState(id)?.joinSeq !== seq) return;`; `pendingJoins` Set ile komut/chat handler'larında erken çıkış.

### 2.2 Moderasyon ve kimlik — ban/mute
Ajanın anahtar değerlendirmesi net: **isimden türetilen anahtar kimlik değildir**, hem atlatılır hem yanlış pozitif üretir. Ayrıca iki farklı fail politikası var: mute fail-closed, ban da fail-closed ama timeout 400 ms — ikisi birlikte odayı DB'ye rehin veriyor.

**Geliştirme önerisi:** anahtarları tiple ayır (`{type:'name'|'user'|'device'}`), ban sebebini nötr metne çevir + `banId` ver, `auth`'u 6 karakter + hash ile logla, misafirler (`auth=null`) için `conn` bazlı kısa süreli katılım limiti ekle.

### 2.3 SRE — süreç dayanıklılığı
Boot sözleşmesi tablosu (raporda tam haliyle) tek bir cümleye indiriyor: **her adım başarısız olursa ne olacağı tanımlı değil**. Özellikle `initDB()`/`migrate()` korumasız, `replays.cleanup()` try dışında, `startWebServer` dönüşü dinlenmiyor, `HBInit` sonrası init'ler try dışında ve `activeRoom` erken set ediliyor.

**Geliştirme önerisi:** boot'u faz numaralı (env → db → web → room) hale getir; `EADDRINUSE`'u fatal yap; `/healthz` ekle (`{db, room, phase}`); `uncaughtException`'da `gracefulShutdown` + `exit(1)`; shutdown'a 10 sn zorunlu tavan.

### 2.4 Sırlar ve repo hijyeni
7 canlı sır + README'de hesap bilgisi + 49 MB canlı veri/arşiv. Ajanın vurgusu doğru sıralama: **önce rotasyon, sonra temizlik** — `.gitignore` zaten izlenen dosyaları kurtarmıyor, `git rm --cached` de geçmişi temizlemiyor.

**Geliştirme önerisi:** rotasyon sırası (token → admin JWT → JWT → admin şifresi → BOT_API_KEY → Discord → oda şifresi) → `git rm --cached` → `git filter-repo` → boot'ta eşitlik/uzunluk kontrolleri → `process.env` ↔ `.env.example` sözleşme testi.

### 2.5 Test ve kanıt
Altyapı **sıfır**: `scripts/run-tests.js` yok, `tests/` yok, `npm test` `MODULE_NOT_FOUND` veriyor. Ajan sıfır bağımlılıklı bir harness önerdi (`node:test` + `--test`) ve `fakeRoom.js` ile join akışının canlı oda olmadan test edilebileceğini gösterdi. Üç araç betiğinin hiçbiri gerçek doğrulama yapmıyor; `verify.js` her koşulda `exit 0` diyor.

**Geliştirme önerisi:** önce harness, sonra 9 regresyon testi (P0: isim normalizasyonu, fail-closed, ban anahtarları). Test edilemeyen kararlar için `createRoomFactory` ve in-memory stub enjeksiyonu.

---

## 3. Ajanların mutabık olduğu "dokunma" listesi

Beş ajanın da korunmasını istediği kararlar:

- `server.js:40-58` fail-fast env kontrolü ve `ADMIN_JWT_SECRET` ayrımı (eksik olan: eşitlik kontrolü)
- `server.js:29-34` `withTimeout` + `Promise.allSettled` ile tek hatanın tüm gate'i düşürmemesi
- `server.js:165` `auth === null` iken duplicate-auth kick yapılmaması
- `server.js:145` `p.id !== 0` host muafiyeti
- `server.js:61-63` `activeRoom` modül scope'u + `:367` `_shuttingDown` idempotency
- `server.js:341-342` `InvalidRoomTokenError`'ın ayrı ve eyleme dönük ele alınması
- `.env` yolunun `__dirname` tabanlı olması (`server.js:4`)

---

## 4. İlk 3 aksiyon (sahip kararı bekliyor)

| # | Aksiyon | Kim | Neden şimdi |
|---|---------|-----|-------------|
| 1 | `.env` sırlarını rotasyona sok + `README.md:2` satırını sil | **Yalnızca sen** | Sırlar commit atıldığı an itibarıyla yanmış sayılır; kod düzeltmesi bunu geri getirmez |
| 2 | `src/` klasörünü repoya ekle | **Yalnızca sen** | 5 ajanın 5'i de bu yüzden analizini sınırlamak zorunda kaldı; sunucu bu haliyle boot etmiyor |
| 3 | `taskkill` bloğunu kaldır (`server.js:93-99`) | Oda Çekirdek + SRE | Tek satırlık, geri dönüşü kolay, en yüksek etkili kod düzeltmesi |

**Karar bekleyen soru:** İkinci turda hangi üçlüye gidelim — (a) join akışının kilitlenmesi ve state sızıntısı, (b) ban/mute anahtarlarının auth'a taşınması, yoksa (c) test harness'inin kurulması?

---

## 5. Bu turun sınırları

- `src/` klasörü, `scripts/`, `tests/`, `.rar` arşivlerinin içi ve `database*.sqlite` okunmadı.
- Git geçmişi incelenmedi; sırların kaç commit geriye gittiği doğrulanmadı.
- Sunucu çalıştırılmadı — `node_modules` yok, `haxball.js` kurulu değil.
- Bu rapordaki her satır numarası `main` dalındaki `server.js` içindir; dosya değişirse doğrulanmalıdır.
