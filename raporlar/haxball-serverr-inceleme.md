# Reoo1283/haxball-serverr — Kod İncelemesi

**İnceleme kapsamı:** `main` dalındaki tüm metin dosyaları (`server.js`, `package.json`, `.env.example`, `.gitignore`, `README.md`, `verify.js`, `remove_comments.js`, `strip_script.js`, `jsconfig.json`).
**Not:** Repoda açık PR yok (0 PR, tek dal: `main`), bu yüzden inceleme doğrudan `main` üzerinde yapıldı.
**Sınır:** `src/**` klasörü repoda **yok** — kodun büyük kısmı (db, web, match, commands, joinGate, identity, vip, bans, mutes, onboarding, slotWatch, team, helper, replays, dbBackup, logger, metrics, styles) yüklenmemiş. Bu dosyalar olmadan kod çalışmaz; aşağıdaki bulgular yalnızca depoda bulunan dosyalara dayanır.

---

## 1. Özet

| # | Önem | Bulgu | Yer |
|---|------|-------|-----|
| 1 | 🔴 Kritik | Canlı sırlar (token, şifre, JWT secret) repoya commit edilmiş | `.env:1-11` |
| 2 | 🔴 Kritik | README'de e-posta + şifre yazılı | `README.md:2` |
| 3 | 🔴 Kritik | Veritabanı ve arşiv dosyaları repoda | kök dizin |
| 4 | 🔴 Kritik | Çalışma anında `taskkill` ile başka süreç öldürülüyor | `server.js:93-101` |
| 5 | 🟠 Yüksek | Var olmayan `src/` modülleri → uygulama açılmaz | `server.js:6-23` |
| 6 | 🟠 Yüksek | Join akışında yarış durumu (async gate + eşzamanlı join) | `server.js:148-241` |
| 7 | 🟠 Yüksek | Giriş kontrolünde fail-open davranış | `server.js:196-215` |
| 8 | 🟠 Yüksek | Hata mesajlarıyla bilgi sızdırma / kullanıcı adına göre ban | `server.js:213-215`, `:270-290` |
| 9 | 🟡 Orta | `uncaughtException` sonrası süreç ayakta kalıyor | `server.js:355-362` |
| 10 | 🟡 Orta | Test altyapısı ve `scripts/` klasörü yok | `package.json:6` |
| 11 | 🟡 Orta | `strip_script.js` bağımlılığı tanımsız (`strip-comments`) | `strip_script.js:1` |
| 12 | 🟢 Düşük | `remove_comments.js` / `verify.js` çalışma dizinindeki her dosyayı yeniden yazıyor | `remove_comments.js:19-31`, `verify.js:17-30` |
| 13 | 🟢 Düşük | `.gitignore` sırları geç yakalıyor, `*.rar` yok | `.gitignore:5-13` |

---

## 2. Ayrıntılı bulgular

### 🔴 1. Sırlar repoda — `.env` commit edilmiş
**Yer:** `.env` (447 bayt, tam içerik repoda), `.gitignore:5`
`.env` içinde `HAXBALL_TOKEN`, `JWT_SECRET`, `ADMIN_JWT_SECRET`, `ADMIN_PASSWORD`, `ROOM_PASSWORD`, `DISCORD_BOT_TOKEN`, `BOT_API_KEY` gerçek değerleriyle duruyor. `.gitignore`'da `.env*` kuralı var ama dosya zaten izleniyor olduğu için kural işlemiyor.
**Risk:** Depoya erişen herkes oda token'ını, admin şifresini, Discord bot token'ını alır → oda ele geçirme, admin paneline tam yetki, Discord botunun çalınması. JWT secret sızarsa tüm kullanıcı oturumları sahte üretilebilir.
**Yapılacak:** Tüm sırları **rotasyona sokun** (token yenile, şifreleri değiştir, Discord token'ı resetle), `git rm --cached .env`, geçmişten temizlik (BFG/filter-repo), `.env.example`'ı tek referans yapın.

### 🔴 2. README'de kimlik bilgisi
**Yer:** `README.md:2` — `... suleymanderste@hotmail.com ile giriş yap hikobaba00`
**Risk:** Herkese açık (şu an private, ama private→public kazası veya fork/arşiv yoluyla sızar) e-posta + şifre. Aynı şifre başka hesaplarda da kullanılıyorsa zincirleme ihlal.
**Yapılacak:** Satırı silin, şifreyi değiştirin, mümkünse 2FA açın.

### 🔴 3. Veritabanı ve arşiv dosyaları repoda
**Yer:** kök dizin — `database.sqlite` (8.0 MB), `database_o.sqlite` (4.2 MB), `7dafe4b4-….db` + `-shm` + `-wal` (SQLite WAL dosyaları), `Haxball Server.rar` (7.9 MB), `node_modules.rar` (25.9 MB), `tasarlanan web uygulaması.rar` (11.2 MB), `veri.rar` (2.2 MB). Toplam ~49 MB.
**Risk:** `.gitignore:9-13` `*.sqlite` diyor ama dosyalar zaten izlendiği için kural işlemiyor. Canlı SQLite + WAL commit etmek: (a) kullanıcı verisi/kimlik bilgisi sızıntısı, (b) WAL tutarsız snapshot → repodan alınan DB bozuk açılabilir, (c) her commit'te 49 MB şişme.
**Yapılacak:** `git rm --cached` ile izlemeden çıkarın, geçmişi temizleyin, yedekleri repoya değil yedek depolamaya koyun.

### 🔴 4. `taskkill` — çalışma anında başka süreç öldürme
**Yer:** `server.js:93` (`netstat … findstr`), `server.js:99` (`taskkill /F /PID … /T`)
**Sorun:**
- Yalnızca Windows'a özgü; Linux/macOS'ta sessizce çalışmaz (catch yutuyor), `findstr` yoksa hata yutulur.
- Aynı portu dinleyen **herhangi bir** süreç (başka bir uygulama, DB, ters proxy) öldürülür; sahiplik kontrolü yok.
- `/F /T` → zorla, alt süreç ağacıyla birlikte. Veri kaybı riski.
- `catch (e) {}` → gerçek hatalar (izin yok, PID ayrıştırılamadı) görünmez.
**Yapılacak:** Bu bloğu kaldırın; port meşgulse net bir hata verip çıkın (`EADDRINUSE`), temizliği geliştiricinin betiğine bırakın.

### 🟠 5. Var olmayan `src/` modülleri
**Yer:** `server.js:6-23` ve `:70, 72, 162, 166, 270, 379` (`require('./src/...')`)
`src/db`, `src/web`, `src/match`, `src/commands`, `src/playerState`, `src/styles`, `src/onboarding`, `src/bans`, `src/mutes`, `src/logger`, `src/metrics`, `src/vip`, `src/replays`, `src/dbBackup`, `src/joinGate`, `src/slotWatch`, `src/team`, `src/identity`, `src/helper` — hiçbiri depoda yok.
**Etki:** `node server.js` ilk satırda `MODULE_NOT_FOUND` ile düşer. `boot()` içindekiler `try/catch` ile sarılı olduğu için bazı özellikler sessizce devre dışı kalır (örn. `:70`, `:72`, `:162`, `:166`) — yani kimlik kontrolü/duplicate-auth kick **çalışmıyor gibi** görünüp hata da vermez.
**Risk:** Sessiz yutulan `require` hatası, güvenlik kontrolünün (duplicate auth kick, `:166`) fark edilmeden kapalı kalmasına yol açar.
**Yapılacak:** `src/` klasörünü repoya ekleyin (veya `.rar` içinden çıkarıp düzgün commit edin); güvenlikle ilgili `require`'ları sessiz `catch` yerine açık hataya çevirin.

### 🟠 6. Join akışında yarış durumu
**Yer:** `server.js:178-241` (async IIFE) + `:183-206` (mute), `:207-224` (ban), `:225-227` (join gate)
**Sorun:** Ban/mute/gate kontrolleri `await` ile asenkron yapılıyor, ancak oyuncu bu süre boyunca **odada kalıyor** (kick yalnızca kontrol biterince). Aynı anda gelen iki join birbirinin kontrolünü görmez: slot kontrolü (`gate.enforce`) ile `applyJoinFlow()` arasında kilit yok.
**Somut senaryo:** 15 kişilik dolu odaya aynı anda 3 kişi girer → hepsi `getPlayerList()`'te 15 görür → üçü de geçer → 17/15. Aynı şekilde ban kontrolü geciken oyuncu, banlanmış haliyle birkaç yüz ms oynayabilir.
**Yapılacak:** Join için oyuncu kimliğine (auth) göre mutex/kuyruk; `gate.enforce` sonucunu atomik sayaçla doğrula; kontrol sürerken `room.kickPlayer` yerine önce `setPlayerTeam`/hareket kısıtı uygulayın.

### 🟠 7. Giriş kontrolünde fail-open
**Yer:** `server.js:196-215`
Mute/ban sorgusu `JOIN_CHECK_TIMEOUT_MS = 400` (`:29`) ile sınırlı. Timeout veya hata → `rejectJoin(...)` çağrılıyor (fail-closed, doğru), **ancak** `:236-239` dış `catch` bloğu her hatada kickliyor — bu iyi. Asıl sorun: `:207`'de `bResults` için `Promise.allSettled` kullanılıyor ve `:222`'de `r.status === 'rejected'` → kick. Yani DB yavaşladığında **herkes** giremez (DoS'a açık): 400 ms'lik timeout, yavaş bir SQLite yazması altında kolayca aşılır → odaya kimse giremez.
**Yapılacak:** Timeout'u ayarlanabilir yapın (`JOIN_CHECK_TIMEOUT_MS` env), timeout'ta "retry" yerine son bilinen ban/mute cache'ini kullanın; art arda hata sayısını metrikleyin.

### 🟠 8. Bilgi sızdırma ve isim tabanlı ban
**Yer:** `server.js:213-215`, `:270-290`, `:291`
- Ban sebebi oyuncuya **aynen** gösteriliyor: `Ban: ${b.reason}` (`:214`) — admin notları/kimlik bilgileri sızabilir.
- Ban anahtarları `player.name` içeriyor (`:186`) → isim değiştirerek ban atlatma, ayrıca **masum bir oyuncunun** aynı ismi almasıyla yanlış ban.
- `:173` `console.log('[Katildi] …')` ve `:270-280` kimlik gözlemi (`auth`, `conn`) loglara düz yazılıyor; `auth` kişisel veri.
- `:291` "Admin/owner hesabin varsa: `!giris <sifre>`" — şifrenin **sohbet satırında** yazılması isteniyor. Sohbet loglarına/ekran görüntüsüne düşer.
**Yapılacak:** Ban sebebini genel metne çevirin; ban anahtarını yalnızca `auth`/device üzerinden kurun; `auth`'u loglarda hash'leyin; girişi komut satırı yerine panel/özel mesaj akışına taşıyın.

### 🟡 9. `uncaughtException` sonrası devam
**Yer:** `server.js:355-362`
Yakalanmamış istisnada yalnızca log atılıp süreç ayakta bırakılıyor. Node'da bu, tanımsız durumda çalışmaya devam etmek demek (kapanmamış DB tutamacı, yarım kalmış maç durumu). Oyun sunucusunda tutarsız state → hayalet oyuncular.
**Yapılacak:** Logladıktan sonra `gracefulShutdown('uncaughtException')` çağırın; süreç yöneticisi (pm2/systemd) yeniden başlatsın.

### 🟡 10. Testler yok
**Yer:** `package.json:6` → `"test": "node scripts/run-tests.js"`, ama `scripts/` klasörü repoda yok; `tests/` de yok (`jsconfig.json:18`'de `tests/**/*.js` bekleniyor).
**Eksik testler (öncelik sırasıyla):**
1. `joinGate` / slot limiti — eşzamanlı join (bulgu 6).
2. `bans.isBanned` / `mutes.isMuted` timeout davranışı (bulgu 7).
3. Duplicate-name kick (`server.js:135-155`) — Türkçe `İ` normalizasyonu (`:137`), boş isim, `id === 0`.
4. Duplicate-auth kick (`:166`) — `auth === null` durumunda **asla** kicklenmemeli.
5. `gracefulShutdown` — iki kez sinyal geldiğinde ikinci çağrının çıkmaması (`:367`).
6. `.env` doğrulaması — eksik `JWT_SECRET`/`ADMIN_JWT_SECRET` ile başlatmanın çıkması (`:41-58`).

### 🟡 11. `strip_script.js` tanımsız bağımlılık
**Yer:** `strip_script.js:1` → `require('strip-comments')`; `package.json`'da böyle bir bağımlılık yok (ne dependencies ne devDependencies).
**Etki:** `MODULE_NOT_FOUND`. Ayrıca `catch (e) {}` (`:33`) sessizce yutuyor → "Modified 0 files" der, kimse fark etmez.
**Yapılacak:** Ya bağımlılığı ekleyin ya da dosyayı silin. Araç betikleri repoda tutulacaksa `devDependencies`'e girmeli.

### 🟢 12. `remove_comments.js` ve `verify.js` tehlikeli
**Yer:** `remove_comments.js:19-31`, `verify.js:17-30`
- `process.cwd()` üzerinden **tüm** dizini geziyor; `node_modules` ve `.git` dışlansa da `backups/` (verify.js'te var), `maps/`, `.env` gibi yerlere dokunabilir. `remove_comments.js` `.json` de dahil her `.js/.json/.ts` dosyasını **yerinde yeniden yazıyor** → yorum sanıp string/URL satırlarını silebilir (regex `^\s*(\/\/|#)` — `#` ile başlayan bir YAML/Markdown satırını da siler).
- `verify.js` hiçbir şey doğrulamıyor; `readFileSync` yapıp `newContent`'i kullanmıyor (`:22-28`) → işe yaramaz kod, yanlış güven veriyor.
**Yapılacak:** İkisini de kaldırın; gerekliyse `eslint`/`prettier` kullanın.

### 🟢 13. `.gitignore` yetersiz
**Yer:** `.gitignore:5-13`
- `.env*` / `*.sqlite` kuralları var ama ilgili dosyalar **zaten izlendiği için** etkisiz (bulgu 1 ve 3).
- `*.rar` yok → 49 MB'lık arşivler tekrar commit edilebilir.
- `*.db`, `*-wal`, `*-shm` deseni yok (`7dafe4b4-….db` bu yüzden repoda).
- `package-lock.json` izleniyor (iyi), ama `node_modules.rar` izleniyor (kötü).

---

## 3. Doğru yapılmış şeyler (korunmalı)

- `server.js:41-58` — `JWT_SECRET`, `ADMIN_PASSWORD`, `ADMIN_JWT_SECRET` için fail-fast kontrol; admin JWT'sinin kullanıcı JWT'sinden **ayrı** olması doğru tasarım.
- `server.js:44-47` — kısa `ADMIN_PASSWORD` için uyarı.
- `server.js:135-155` — aynı isimle giriş engeli, Türkçe `İ` normalizasyonu (`:137`) düşünülmüş.
- `server.js:165-166` — `auth === null` iken duplicate-auth kick yapılmaması (yanlış pozitif önlenmiş).
- `server.js:196-206` — `Promise.allSettled` ile tek tek sonuç değerlendirme, tek hata tüm kontrolü boşa düşürmüyor.
- `server.js:303-307` — VIP karşılama cooldown'u (60 sn) ve 500 kayıt üstü temizlik: spam koruması.
- `server.js:366-384` — `SIGINT`/`SIGTERM` ile oda kapatma + DB kapatma sırası doğru; ikinci sinyalde tekrar çalışmama koruması (`:367`).
- `server.js:20-29` — `withTimeout` yardımcı fonksiyonu ile dışa bağımlılık çağrılarını sınırlama fikri doğru.

---

## 4. Öncelikli aksiyon listesi

1. **Bugün:** `.env`'deki tüm sırları rotasyona sokun; `README.md:2`'deki e-posta/şifre satırını silin; repoyu private tutun.
2. **Bu hafta:** `src/` klasörünü ekleyin (yoksa kod çalışmıyor); `taskkill` bloğunu (`server.js:93-101`) kaldırın; `.env`, `*.sqlite`, `*.db`, `*.rar` dosyalarını izlemeden çıkarıp geçmişi temizleyin.
3. **Sonra:** Join akışına kilit ekleyin (bulgu 6); timeout'u cache ile yumuşatın (bulgu 7); ban sebebini genelleştirin ve ban anahtarından ismi çıkarın (bulgu 8).
4. **Sürekli:** `scripts/run-tests.js` + `tests/` ekleyin; yukarıdaki 6 test başlığıyla başlayın. `remove_comments.js`, `strip_script.js`, `verify.js` dosyalarını kaldırın.

---

*Bu inceleme yalnızca okuma amaçlıdır: GitHub'a hiçbir yorum, review veya commit gönderilmedi; repoda hiçbir dosya değiştirilmedi.*
