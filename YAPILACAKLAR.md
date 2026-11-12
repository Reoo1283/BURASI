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

### [ ] 2. İki port-öldürme bloğunu kaldır
**Nerede:** `server.js:93-99` **ve** `src/web/index.js:146-163` (`killPortOwner`)
**Neden:** Sahiplik kontrolü olmadan yabancı süreç öldürüyor. `startWebServer`
`EADDRINUSE`'da exit etmiyor, öldürüp yeniden deniyor (`:165-190`).
**Düzeltme:** `EADDRINUSE` → `logError` + `process.exit(1)`. `killPortOwner` tamamen sil.
**Doğrulama:** Port meşgulken başlat → süreç temiz çıkmalı, başka süreç ölmemeli.

### [ ] 3. Moderasyon katmanına test yaz
**Nerede:** `src/bans.js`, `src/mutes.js`, `src/identity.js` — **hiç test yok**
**Neden:** En kritik katman (ban/mute/kimlik) tamamen kapsam dışı.
**Yazılacak testler:**
- `bans.test.js` — süreli/süresiz ban, süre dolumu, `COLLATE NOCASE`, `removeLinkedBans` escape
- `mutes.test.js` — `nameKey` vs `deviceKey` asimetrisi (isim değiştirince mute atlanıyor mu?)
- `identity.test.js` — `noteJoin`/`noteLeave` simetrisi, `auth=null` misafirler, duplicate-auth kick
**Doğrulama:** `npm test` → 49+ dosya, hepsi yeşil.

### [ ] 4. 400 ms fail-closed timeout'u yumuşat
**Nerede:** `src/joinGate.js:5` `CHECK_TIMEOUT_MS`, `:67` `permUncertain → rejectJoin('SYSTEM_ERROR')`
**Neden:** DB 400 ms'den yavaş yanıt verirse odaya kimse giremez (fail-frozen).
**Düzeltme:** Son bilinen yetkiyi cache'le (TTL 30 sn); timeout'ta cache'i kullan,
yalnız cache de yoksa reddet. Reddi `SYSTEM_ERROR` yerine "tekrar deneyin" yap.
**Doğrulama:** DB'yi 1 sn geciktir → mevcut oyuncular girebilmeli.

### [ ] 5. `maxPlayers` ile `totalLimit` çelişkisini çöz
**Nerede:** `server.js:120` (`maxPlayers: 15`) vs `src/slotGuard.js:5-8` (`totalLimit: 13`, `normalLimit: 10`)
**Neden:** 15 kişilik odada gerçekte 13 kişi alınıyor, 2 slot hiç kullanılmıyor.
**Düzeltme:** `config/runtime.json` ile ikisini tek kaynaktan türet, ya da `maxPlayers`'ı 13 yap.
**Doğrulama:** Oda 13. oyuncuda değil 15. oyuncuda dolmalı (veya limit 13'e çekilmeli).

### [ ] 6. `onPlayerJoin` atama sırasını doğrula
**Nerede:** `server.js:138` (doğrudan atama) vs `src/commands.js:1046` (`onPlayerChat`),
`src/match/index.js:1808` (`onPlayerLeave`)
**Neden:** `initCommands` (`:130`) ve `initMatchMechanics` (`:129`) `onPlayerJoin`'den **önce**
çağrılıyor. `src/` tarafında ikinci bir `onPlayerJoin` ataması varsa `server.js`'teki sürüm
sessizce silinir — kim kazanıyor belirsiz.
**Doğrulama:** `grep -rn "onPlayerJoin" src/ server.js` → kaç atama var, hangisi son?

---

## 🟠 P2 — Doğrulama ve hijyen

### [ ] 7. Testleri gerçekten çalıştır
**Komut:** `npm ci && npm test && npm run lint && npm run typecheck`
**Neden:** 46 testin geçtiğine dair hiç kanıt yok.
**Çıktı:** `raporlar/test-kosumu-YYYY-AA-GG.md`

### [ ] 8. `src/` sürümü ile GitHub `main` arasındaki farkı çıkar
**Neden:** Arşivdeki `src/` commit edilmemiş bir çalışma kopyası olabilir.
**Komut:** İkisini ayrı klonla, `diff -r` çalıştır, farkı raporla.

### [ ] 9. `identity.js` tam oku
**Neden:** 58 KB, en büyük modül; yalnız 3 fonksiyonu incelendi.

### [ ] 10. Git geçmişini tara
**Neden:** Sırların kaç commit geriye gittiği bilinmiyor — rotasyon sonrası
`filter-repo` kapsamı buna bağlı.

---

## 🟡 P3 — İyileştirme

### [ ] 11. Üç araç betiğini düzelt
- `verify.js:21-31` — her koşulda `exit 0`, gerçek doğrulama yok
- `remove_comments.js:19-31` — tüm dizini sessizce yeniden yazıyor
- `strip_script.js:1` — tanımsız bağımlılık `strip-comments`

### [ ] 12. `/api/health` oda durumunu yansıtsın
**Nerede:** `src/web/routes/stats.js:85`
**Neden:** Haxball bağlantısı koptuğunda da 200 dönüyor.
**Düzeltme:** `{db, room, phase}` döndür; oda yoksa 503.

### [ ] 13. Ban sebebini nötrleştir
**Nerede:** `server.js:214`
**Neden:** İç not oyuncuya aynen gidiyor. `banId` ver, sebebi panelde göster.

### [ ] 14. `auth`/`conn` loglara düz yazılmasın
**Nerede:** `server.js:149, 162, 271, 275`
**Düzeltme:** İlk 6 karakter + hash.

### [ ] 15. Şifreyi sohbet satırından çıkar
**Nerede:** `server.js:291` (`!giris <sifre>`)
**Neden:** Ekran görüntüsü/log riski + kimlik avı zemini. DM veya web paneline taşı.

---

## Bitiş kontrolü

Depoya her commit'ten önce:

```powershell
Select-String -Path .\**\* -Pattern 'gh[pousr]_[A-Za-z0-9]{20,}|=[A-Za-z0-9+/]{24,}'
```

Çıktı **boş** olmalı. Sır sızdıysa `git reset` + geçmiş temizliği.
