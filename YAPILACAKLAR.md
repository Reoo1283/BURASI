# YAPILACAKLAR — Öncelik sıralı açık işler

Her madde: ne, nerede, neden, kim yapabilir, nasıl doğrulanır.
Bitince `[x]` yapın ve `DURUM.md`'ye işleyin.

---

## 🔴 P0 — Sadece kullanıcı yapabilir

### [ ] 1. Sırları rotasyona sok
**Nerede:** `.env` (10 değişken), `README.md:2`
**Neden:** Hepsi commit edilmiş durumda; depoyu gören herkes kullanabilir.
**Sıra:** `HAXBALL_TOKEN` → `ADMIN_JWT_SECRET` → `JWT_SECRET` → `ADMIN_PASSWORD` →
`BOT_API_KEY` → `DISCORD_BOT_TOKEN` → `ROOM_PASSWORD`
**Doğrulama:** Eski token ile Haxball bağlantısı kurulamıyor olmalı; `README.md:2` boş.
**Sonra:** `git rm --cached .env` → `.gitignore`'a `.env` ekle → `git filter-repo` ile geçmiş temizliği.

---

## 🔴 P1 — Kod düzeltmesi (onay gerektirir)

### [x] 2. İki port-öldürme bloğunu kaldır
**Bitti:** 12 Kasım 2026, oturum 3 · `server.js:90-94` yorum bloğu, `src/web/index.js` `killPortOwner` silindi;
`EADDRINUSE` → `logError('web.listen')` + `process.exit(1)`. Kalan açıklama:
**Nerede:** `server.js:93-99` **ve** `src/web/index.js:146-163` (`killPortOwner`)
**Neden:** Sahiplik kontrolü olmadan yabancı süreç öldürüyor. `startWebServer`
`EADDRINUSE`'da exit etmiyor, öldürüp yeniden deniyor (`:165-190`).
**Düzeltme:** `EADDRINUSE` → `logError` + `process.exit(1)`. `killPortOwner` tamamen sil.
**Doğrulama:** Port meşgulken başlat → süreç temiz çıkmalı, başka süreç ölmemeli.

### [x] 3. Moderasyon katmanına test yaz
**Bitti:** `tests/moderation.test.js` — **22 test** (bans 7, mutes 6, identity 9). Kalan açıklama:
**Nerede:** `src/bans.js`, `src/mutes.js`, `src/identity.js` — **hiç test yok**
**Neden:** En kritik katman (ban/mute/kimlik) tamamen kapsam dışı.
**Yazılacak testler:**
- `bans.test.js` — süreli/süresiz ban, süre dolumu, `COLLATE NOCASE`, `removeLinkedBans` escape
- `mutes.test.js` — `nameKey` vs `deviceKey` asimetrisi (isim değiştirince mute atlanıyor mu?)
- `identity.test.js` — `noteJoin`/`noteLeave` simetrisi, `auth=null` misafirler, duplicate-auth kick
**Doğrulama:** `npm test` → 49+ dosya, hepsi yeşil.

### [x] 4. 400 ms fail-closed timeout'u yumuşat
**Bitti:** `src/joinGate.js` — `_permCache` (TTL 30 sn, max 500, **yalnız pozitif** yetki). Kalan açıklama:
**Nerede:** `src/joinGate.js:5` `CHECK_TIMEOUT_MS`, `:67` `permUncertain → rejectJoin('SYSTEM_ERROR')`
**Neden:** DB 400 ms'den yavaş yanıt verirse odaya kimse giremez (fail-frozen).
**Düzeltme:** Son bilinen yetkiyi cache'le (TTL 30 sn); timeout'ta cache'i kullan,
yalnız cache de yoksa reddet. Reddi `SYSTEM_ERROR` yerine "tekrar deneyin" yap.
**Doğrulama:** DB'yi 1 sn geciktir → mevcut oyuncular girebilmeli.

### [x] 5. `maxPlayers` ile `totalLimit` çelişkisini çöz
**Bitti:** `src/slotGuard.js` → `limitsForRoom()`; `server.js:111` artık `roomLimits.maxPlayers` (env `ROOM_MAX_PLAYERS`). Kalan açıklama:
**Nerede:** `server.js:120` (`maxPlayers: 15`) vs `src/slotGuard.js:5-8` (`totalLimit: 13`, `normalLimit: 10`)
**Neden:** 15 kişilik odada gerçekte 13 kişi alınıyor, 2 slot hiç kullanılmıyor.
**Düzeltme:** `config/runtime.json` ile ikisini tek kaynaktan türet, ya da `maxPlayers`'ı 13 yap.
**Doğrulama:** Oda 13. oyuncuda değil 15. oyuncuda dolmalı (veya limit 13'e çekilmeli).

### [x] 6. `onPlayerJoin` atama sırasını doğrula
**Bitti (doğrulama):** `src/match/index.js:1951-1988` accessor ile sarmalıyor; `server.js:138` son atama → **ezilme yok**.
**Kalan risk:** motor `destroy()` edilirse accessor eski motora bağlı kalabilir — ayrı madde 6b. Kalan açıklama:
**Nerede:** `server.js:138` (doğrudan atama) vs `src/commands.js:1046` (`onPlayerChat`),
`src/match/index.js:1808` (`onPlayerLeave`)
**Neden:** `initCommands` (`:130`) ve `initMatchMechanics` (`:129`) `onPlayerJoin`'den **önce**
çağrılıyor. `src/` tarafında ikinci bir `onPlayerJoin` ataması varsa `server.js`'teki sürüm
sessizce silinir — kim kazanıyor belirsiz.
**Doğrulama:** `grep -rn "onPlayerJoin" src/ server.js` → kaç atama var, hangisi son?

---

## 🟠 P2 — Doğrulama ve hijyen

### [x] 7. Testleri gerçekten çalıştır
**Komut:** `node scripts/run-tests.js` · **Bitti:** 12 Kasım 2026, oturum 2
**Sonuç:** **531 test, 531 geçti, 0 düştü**, 66 sn. `tsc --noEmit` temiz.
**Kanıt:** `raporlar/test-kosumu-2026-11-12.md`
**Yan bulgu:** `eslint .` 1.283 hata veriyor; 924'ü `hbx-ed-main-map_uygulama/dist/`
build çıktısından (`eslint.config.js` ignores listesine `**/dist/**` eklenmeli).
**Ders:** Arşivdeki `node_modules` Windows derlemesi → `invalid ELF header`; native
modüller (`sqlite3`, `bcrypt`) her makinede yeniden derlenmeli, arşivlenmemeli.

### [x] 7b. `eslint.config.js`'e `**/dist/**` ekle
**Bitti:** 1.283 → **28** hata (`dist`, `backups`, `Her_boku_bilen_discord_botu`, `hbx-ed-main-map_uygulama` ignore edildi).
**Kalan 28:** `src/match/index.js` 14 · `src/identity.js` 5 · testler 6 · `public/js/views/identity.js` 2 · `src/commands.js` 1 (hepsi kozmetik).
**Neden:** 924 sahte hata gerçek 359 hatayı gizliyor.
**Doğrulama:** `npx eslint .` → hata sayısı ~359'a düşmeli.

### [x] 8. `src/` sürümü ile GitHub `main` arasındaki farkı çıkar
**Bitti:** GitHub `main` 24 dosya ↔ Mediafire kökü: 10 ortak dosya **birebir aynı**; arşiv `.git`'e göre **27 dosya, +2.897/−554 satır commit edilmemiş**. Detay: `raporlar/duzeltme-turu-2026-11-12.md` §4.
**Sonuç:** Çalışan sürüm yalnızca arşivde; GitHub `main` eski bir anlık görüntü.
**Neden:** Arşivdeki `src/` commit edilmemiş bir çalışma kopyası olabilir.
**Komut:** İkisini ayrı klonla, `diff -r` çalıştır, farkı raporla.

### [x] 9. `identity.js` tam oku
**Bitti:** 1.363 satır, 11 bulgu (I1–I11). En önemlisi **I7**: ban eşleştirmesi `dev:` + 68 karakter varsayımına bağlı. Detay: aynı rapor §5.
**Neden:** 58 KB, en büyük modül; yalnız 3 fonksiyonu incelendi.

### [x] 10. Git geçmişini tara
**Bitti:** Arşiv `.git` **1 commit** (`1053c7a`), `.env`/`*.sqlite`/`*.rar` hiç commit edilmemiş. GitHub `haxball-serverr` **1 commit**, `.env` ve `README.md` takipte.
**Sonuç:** `filter-repo` kapsamı 1 commit; rotasyon yine de ilk iş.
**Neden:** Sırların kaç commit geriye gittiği bilinmiyor — rotasyon sonrası
`filter-repo` kapsamı buna bağlı.

---

## 🟡 P3 — İyileştirme

### [x] 11. Üç araç betiğini düzelt
**Bitti:** `verify.js` gerçek doğrulama yapıyor (441 dosya, sorun varsa `exit 1`); `remove_comments.js` varsayılan **kuru çalışma** (`--write --yes` gerekir); `strip_script.js` bağımlılıksız, `remove_comments.js`'e devrediyor. Kalan açıklama:
- `verify.js:21-31` — her koşulda `exit 0`, gerçek doğrulama yok
- `remove_comments.js:19-31` — tüm dizini sessizce yeniden yazıyor
- `strip_script.js:1` — tanımsız bağımlılık `strip-comments`

### [x] 12. `/api/health` oda durumunu yansıtsın
**Bitti:** `{status, db, room, phase, players}` döner; `db && room` değilse **503**. Kalan açıklama:
**Nerede:** `src/web/routes/stats.js:85`
**Neden:** Haxball bağlantısı koptuğunda da 200 dönüyor.
**Düzeltme:** `{db, room, phase}` döndür; oda yoksa 503.

### [x] 13. Ban sebebini nötrleştir
**Bitti:** Oyuncu nötr mesaj + `banId` referansı görür; gerçek sebep log/panelde kalır (`server.js:225-233`, `src/bans.js:61`). Kalan açıklama:
**Nerede:** `server.js:214`
**Neden:** İç not oyuncuya aynen gidiyor. `banId` ver, sebebi panelde göster.

### [x] 14. `auth`/`conn` loglara düz yazılmasın
**Bitti:** `shortHash()` = ilk 6 karakter + `…` + sha256 ilk 8; `[DupName]` logu maskeli. Kalan açıklama:
**Nerede:** `server.js:149, 162, 271, 275`
**Düzeltme:** İlk 6 karakter + hash.

### [x] 15. Şifreyi sohbet satırından çıkar
**Bitti:** `!giris <sifre>` daveti kaldırıldı; "Admin girişi web panelinden yapılır" metni geldi. Kalan açıklama:
**Nerede:** `server.js:291` (`!giris <sifre>`)
**Neden:** Ekran görüntüsü/log riski + kimlik avı zemini. DM veya web paneline taşı.

---

### [ ] 6b. `match.destroy()` sonrası `onPlayerJoin` accessor'ı geri alınsın
**Nerede:** `src/match/index.js:1988` (`destroy()`)
**Neden:** Motor yeniden kurulursa accessor `_origJoin` üzerinden eski motora bağlı kalabilir.
**Doğrulama:** destroy → init → iki kez `onPlayerJoin` ata, son atanan kazanmalı.

### [ ] 16. `identity.js` I7 — ban eşleştirmesini anahtar formatından bağımsız yap
**Nerede:** `src/identity.js:359-366`
**Neden:** `dev:` + 68 karakter varsayımı bozulursa ban'lar **sessizce** tanınmaz.
**Düzeltme:** `bans.js`'e `isDeviceKey(k)` yardımcısı koy, `identity` onu kullansın.

### [ ] 17. Kalan 28 eslint hatasını temizle
**Nerede:** `src/match/index.js` 14 · `src/identity.js` 5 · `tests/*` 6 · `public/js/views/identity.js` 2 · `src/commands.js` 1
**Neden:** Hepsi `no-unused-vars`/`prefer-const`; ölü kod mu eksik mi olduğu dosya dosya karar gerektiriyor.

### [x] 18. Açılışta sır sağlık kontrolü
**Nerede:** `server.js:50-70`
**Neden:** `JWT_SECRET`/`ADMIN_JWT_SECRET`/`BOT_API_KEY` **aynı** olsa da kod açılıyor; yalnız "tanımlı mı" bakılıyor.
**Yapılacak:** (a) üç sırrın birbirinden farklı olduğunu doğrula, (b) min uzunluk kontrolü, (c) `logs/`'a sır yazılmadığını doğrulayan test, (d) `verify.js`'e `.env` takip kontrolü.
**Bitti (12 Kasım 2026):** `server.js` — 6 değişken için min uzunluk + örnek değer reddi; `JWT_SECRET`/`ADMIN_JWT_SECRET`/`BOT_API_KEY` aynıysa **açılış reddedilir**. `ADMIN_PASSWORD` kısaysa yalnız uyarı (eski davranış korundu). Sır değeri hiçbir koşulda basılmaz.
**Test:** `tests/secretCheck.test.js` — 7 test, 7/7 geçti (sunucu çalıştırılmadan yalnız sır bloğu değerlendirilir).
**Rehber:** `kurulum/SIR-ROTASYONU.md` §3 ve §8.


### [x] 19. Ajan raporu bulguları (C1–C17) — kısmi: C1, C2 bitti

**C2 BİTTİ (12 Kasım 2026):** `src/match/index.js` → `createMatchEngine` kapsamında `prevJoinDescriptor`; `initMatchMechanics` kurmadan önceki tanımı saklar, `destroy()` `Object.defineProperty` ile **aynen geri koyar** (tanım yoksa `delete`). Önceden accessor kalıcı kalıyor ve `_origJoin` eski motora bağlı kalıyordu → motor yeniden kurulunca aynı join **iki kez** işleniyordu.
**Kanıt:** `tests/match.index.test.js` +2 test (30/30). Regresyon testleri eski kodla **2 kırmızı**, yeni kodla yeşil doğrulandı. Tam paket **569/569**.
**C5 BİTTİ (12 Kasım 2026) — sabit `sleep` temizliği:** 9 test dosyasında **21 sabit bekleme** koşul-bekleyen desene çevrildi (`settle`, `waitDbRow`, `waitRow`, `waitAnnounce`, inline `while (deadline)`). Kalan 3 sabit bekleme **bilinçli** ve gerekçesi yorumda: yokluğu doğrulayan testler (own-goal avatar, ısınma istatistiği, R4 nick yazımı) — `waitFor` "olmamasını" bekleyemez, hemen döner.
**Kanıt:** 3 tam paket tekrarı **569/569** (70–79 sn). Tekrarlarda çıkan 2 gerçek kırılganlık bulundu ve düzeltildi (own-goal `ownGoals` DB yazımı, `setPlayerHelper` `isHelper` DB yazımı).
**Bonus:** fairplay dedup (`${dbId}|censor`, 60 sn) testler arasında sızıyordu — `commands.js` fairplay'i modül düzeyinde aldığı için DI ile izole edilemiyor. Censor testleri benzersiz `dbId` kullanıyor + davranış 2 yeni testle kilitlendi.

**C1 BİTTİ (12 Kasım 2026):** `src/bans.js` → `isDeviceKey()`, `parseDeviceKey()`, `DEVICE_KEY_PREFIX`, `DEVICE_KEY_HEX_LEN`. `src/identity.js:355-375` → format varsayımı kalktı, beklenmeyen uzunluk `logInfo('identity.ban-key-unrecognized')` ile raporlanıyor (sır basılmıyor).
**Kanıt:** `tests/moderation.test.js` +5 test (27/27). Regresyon testi **eski kodla kırmızı, yeni kodla yeşil** doğrulandı. Tam paket **565/565**.
**Bonus:** `src/identity.js` S1/S6 döngülerinde tanımsız `ah` (runtime `ReferenceError`) ve kullanılmayan bağlama düzeltildi; eslint 28 → 25.
**Kalan:**
**Kaynak:** `raporlar/ajan-raporu-2026-11-12.md`
**En kritik 5:** C1 (ban anahtarı), C2 (accessor geri alma), C4 (sır rotasyonu), C7 (skor yedeği), C3 (sightings büyümesi).
**Kural:** Her biri ayrı onay ister; C4 yalnız kullanıcı yapabilir.

---

## Bitiş kontrolü

Depoya her commit'ten önce:

```powershell
Select-String -Path .\**\* -Pattern 'gh[pousr]_[A-Za-z0-9]{20,}|=[A-Za-z0-9+/]{24,}'
```

Çıktı **boş** olmalı. Sır sızdıysa `git reset` + geçmiş temizliği.
