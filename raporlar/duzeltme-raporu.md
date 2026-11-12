# Düzeltme Raporu — İlk İnceleme Eksik Veriyle Yapılmış

**Tarih:** 1. tur düzeltmesi · **Yöneten:** Haxball Proje Yöneticisi
**Neden:** İlk raporda `src/` klasörünün "depoda olmadığı" yazıyordu. Yanlıştı: kod **depoda** ve `Haxball Server.rar` arşivinin içinde duruyor. Bu, ilk turun 21 bulgusunun bir kısmını **geçersiz**, bir kısmını **doğrulanmış**, bir kısmını da **yeni** hale getiriyor.

---

## 1. Ne indirilmemişti?

GitHub `main` dalında **24 dosya** var (`git trees` → `truncated: false`). İlk turda bu 24 dosyanın 9'u okundu; okunmayanlar:

| Okunmayan | Neden kritik |
|---|---|
| `Haxball Server.rar` (7.9 MB) | **Gerçek kaynak kod burada**: `src/` (41 dosya + `match/` + `web/`), `tests/` (46 test), `scripts/run-tests.js`, `public/`, `config/`, `discord-bot/` |
| `veri.rar` (2.2 MB) | Repo kökünün tam kopyası + canlı DB |
| `node_modules.rar` (25.9 MB) | Bağımlılıklar |
| `database.sqlite`, `database_o.sqlite`, `*.db*` | Canlı veri |
| `package-lock.json`, `eslint.config.js`, `tsconfig.json` | Bağımlılık ve lint gerçeği |

**Doğrulama:** `Haxball Server.rar` gerçek bir RAR5 arşivi (sihirli bayt `Rar!\x1a\x07\x01\x00`) ve **511 dosya** çıkarıldı. Git deposunda `.gitignore:28` `maps/*.hbs` diyor ama `public/`, `config/`, `discord-bot/` **hiç ignore edilmemiş** — yani bunlar da yalnızca arşivde, çünkü repoya hiç commit edilmemişler.

---

## 2. Geçersiz olan bulgular (geri çekiliyor)

| İlk turda yazılan | Gerçek | Kaynak |
|---|---|---|
| "`src/` klasörü yok, sunucu boot etmiyor" | `src/` var: 41 dosya, 16.556 satır. Sunucu boot edebilir. | `Haxball Server.rar → src/` |
| "Test altyapısı sıfır, `npm test` `MODULE_NOT_FOUND`" | **46 test dosyası, 9.926 satır**, `scripts/run-tests.js` var ve `node --test` ile çalışıyor. | `scripts/run-tests.js:1-62`, `tests/` |
| "`src/bans.js` yok, anahtar formatı bilinmiyor" | `src/bans.js:10-13` — `deviceBanKey(auth)` = `sha256(auth)`, **auth tabanlı**. İsim anahtarı ban tarafında yok. | `src/bans.js:10` |
| "`src/mutes.js` yok" | `src/mutes.js:10-18` — `nameKey` ve `deviceKey` var, `isMuted` `{until, reason, mutedBy}` döner. | `src/mutes.js:38-53` |
| "`room.onPlayerLeave` hiç tanımlı değil" | **Tanımlı**: `src/match/index.js:1808-1830` → `noteLeave`, `clearState`, `reporter.onPlayerLeave`, `motionGuard.forgetPlayer`, dizilim temizliği. | `src/match/index.js:1808` |
| "`createJoinGate` her join'de yeniden kuruluyor, state kaybı" | Gate zaten **stateless**: kararı `room.getPlayerList()` + `slotGuard.canJoin` üretiyor, instance state yok. | `src/joinGate.js:37-80` |
| "`server.js:179` ban anahtarı çıplak isim" | Doğru değil: `:179` **mute** için `nameKey`, `:181` **ban** için `deviceBanKey` kullanıyor. Ban isim anahtarı taşımıyor. | `server.js:178-190` |
| "`joinGate` reddinde bildirim yok" | `src/joinGate.js:29-35` `rejectJoin` → `sendAnnouncement` + `kickPlayer` + `logPlayer('SLOT_KICK')`. | `src/joinGate.js:29` |

---

## 3. Doğrulanan bulgular (aynen duruyor)

| # | Bulgu | Kanıt |
|---|-------|-------|
| 1 | `.env`'de 7 canlı sır + `README.md:2`'de e-posta/şifre | `.env` 447 B, 10 değişken dolu; `README.md:2` |
| 2 | Port öldürme **iki yerde** birden | `server.js:93-99` **ve** `src/web/index.js:146-163` (`killPortOwner` → `SIGTERM`+`SIGKILL`+`taskkill /F /T`) |
| 3 | 400 ms timeout fail-closed → DB yavaşlarsa odaya kimse giremez | `server.js:29` `JOIN_CHECK_TIMEOUT_MS`, `src/joinGate.js:5` `CHECK_TIMEOUT_MS`, `:67` `permUncertain → rejectJoin('SYSTEM_ERROR')` |
| 4 | Join async, oyuncu kontrol bitene kadar serbest | `server.js:178` async IIFE |
| 5 | `uncaughtException` sonrası süreç yarı-canlı | `server.js:356-364` |
| 6 | Token doğrulaması `startWebServer` sonrasında | `server.js:106` vs `:108-114` |
| 7 | `initState` red yollarından önce | `server.js:160` |
| 8 | Ban sebebi oyuncuya aynen gidiyor | `server.js:214` |
| 9 | `auth`/`conn` loglara düz yazılıyor | `server.js:149, 162, 271, 275` |
| 10 | Şifre sohbet satırında isteniyor | `server.js:291` |
| 11 | `JWT_SECRET === ADMIN_JWT_SECRET` kontrolü yok | `server.js:40-57` |
| 12 | Shutdown interval/timer temizlemiyor, zorunlu tavan yok | `server.js:368-384` |
| 13 | `verifyLatestBackup` reddederse scheduler hiç başlamıyor | `server.js:74-86` |
| 14 | Üç araç betiği: sahte başarı / sessiz yıkım / tanımsız bağımlılık | `verify.js`, `remove_comments.js`, `strip_script.js` |

---

## 4. Yeni bulgular (ancak `src/` okununca görüldü)

### 🔴 Y1 — İki ayrı port-öldürme mekanizması
`src/web/index.js:146-163` içindeki `killPortOwner()` `EADDRINUSE` alınca devreye giriyor ve `SIGTERM` + `SIGKILL` + `taskkill /F /T` üçlüsünü sırayla uyguluyor. `server.js:93-99`'daki blokla birlikte **iki bağımsız** yabancı süreç öldürme yolu var. `startWebServer` `EADDRINUSE`'da exit etmiyor, öldürüp yeniden dinlemeye çalışıyor (`:165-190`).

### 🟠 Y2 — Slot limiti Haxball'ın `maxPlayers`'ı ile çelişiyor
`server.js:120` `maxPlayers: 15`, ama `src/slotGuard.js:5-8` `totalLimit: 13`, `normalLimit: 10`. Yani 15 kişilik odada gerçekte 13 kişi alınıyor; 2 slot hiç kullanılmıyor. `config/runtime.json` ile değiştirilebilir (`src/runtimeConfig.js`).

### 🟠 Y3 — `onPlayerJoin` iki dosyada, kim kazanıyor belirsiz
`server.js:138` `room.onPlayerJoin = function(player)` **doğrudan atama** yapıyor ve `initCommands(room)` (`:130`) ile `initMatchMechanics(room)` (`:129`) **ondan önce** çağrılıyor. `src/commands.js:1046` `room.onPlayerChat` atıyor; `src/match/index.js:1808` `room.onPlayerLeave` atıyor. `onPlayerJoin`'in `src/` tarafında da atanıp atanmadığı kontrol edilmeli — ikinci bir atama varsa `server.js`'teki sürüm sessizce silinir.

### 🟠 Y4 — `identity.handleDuplicateAuthKick` senkron ve `noteJoin`'e bağımlı
`src/identity.js` `getDuplicateAuthInfo` yalnız `_liveById` / `_idsByAuth` in-memory kümelerine bakıyor. `noteJoin` (`server.js:162`) sessiz `try/catch` içinde; `noteJoin` çökerse duplicate-auth koruması **hiç çalışmaz** ve log'a hiçbir şey düşmez. Ayrıca `auth=null` misafirlerde `coerceHash(null) = null` → küme anahtarı `null`, yani tüm misafirler aynı "auth" altında toplanır; `handleDuplicateAuthKick` bunu `duplicate: false` ile geçiyor (doğru), ama `_idsByConn` tarafında `conn=null` aynı davranışı gösteriyor mu doğrulanmalı.

### 🟠 Y5 — Ban anahtarı ile mute anahtarı farklı uzayda
`src/bans.js:10` `dev:` + `sha256(auth)`, `src/mutes.js:17` `dev:` + `sha256(auth)` — **aynı**. Ama `src/mutes.js:10` `name:` + isim ayrı bir uzay açıyor ve `server.js:186` mute'ta önce `nameKey` deniyor. Sonuç: aynı kişi isim değiştirince **mute atlatılabilir** (ban atlatılamaz). Bu asimetri belgelenmemiş.

### 🟡 Y6 — Test kapsamı boşlukları (46 test var ama)
Test olmayan modüller: `bans.js`, `mutes.js`, `identity.js` (58 KB, en büyük modül), `dbBackup.js`, `achievements.js`, `seasons.js`, `violations.js`, `fairplay.js`, `bonservis.js`, `runtimeConfig.js`, `roomContext.js`, `styles.js`, `cryptoUtil.js`. Yani **moderasyon ve kimlik katmanının hiç testi yok** — en kritik alan.

### 🟡 Y7 — `src/web/index.js:101` statik kök `public/` — doğru
`express.static(path.join(__dirname, '../../public'))` yalnızca `public/` klasörünü servis ediyor; `.env` ve `*.sqlite` erişilemez. Bu bir **doğru uygulama**, ilk turdaki "statik servis riski" endişesi geçersiz.

### 🟡 Y8 — `/api/health` var ama oda durumunu bilmiyor
`src/web/routes/stats.js:85` `router.get('/health')` mevcut. Ama odanın bağlı olup olmadığını (`activeRoom`) yansıtmıyor; Haxball bağlantısı koptuğunda da 200 dönme riski var.

---

## 5. Güncellenmiş öncelik listesi

| # | Aksiyon | Neden |
|---|---------|-------|
| 1 | `.env` sırlarını rotasyona sok + `README.md:2` sil | Değişmedi — tek gerçek acil iş |
| 2 | **İki** port-öldürme bloğunu kaldır (`server.js:93-99` + `src/web/index.js:146-163`) | Y1 — artık iki kat risk |
| 3 | `bans`/`mutes`/`identity` için test yaz | Y6 — en kritik katmanın hiç testi yok |
| 4 | 400 ms timeout'u cache ile yumuşat | Bulgu 3 — oda kilitlenmesi |
| 5 | `maxPlayers: 15` ile `totalLimit: 13` çelişkisini çöz | Y2 — 2 slot boşa gidiyor |
| 6 | `onPlayerJoin` atama sırasını doğrula | Y3 — handler sessizce ezilebilir |

---

## 6. Bu düzeltmenin sınırları

- Testler **çalıştırılamadı**: makinede `node` yok, `node_modules` arşivden çıkarılmadı. 46 testin gerçekten geçtiği doğrulanmadı.
- `Haxball Server.rar` içindeki `src/` sürümü ile GitHub'daki `main` arasındaki fark bilinmiyor — arşiv **commit edilmemiş** bir çalışma kopyası olabilir (arşivde `server.js` yok, yani kısmi).
- `.rar` içindeki `logs/`, `backups/` ve `Her_boku_bilen_discord_botu/` klasörleri incelenmedi.
- `identity.js` (58 KB) yalnızca `noteJoin`/`noteLeave`/`handleDuplicateAuthKick` bölümlerinden okundu.
