# Ajan Ekibi Raporu — 12 Kasım 2026

**Ne:** 7 ajanın kendi alanında bulduğu eksikler, kritik hatalar ve çözüm planları.
**Yöntem:** Her ajan kendi sahasını `src/` üzerinde **dosya:satır** ile taradı. Kod **değiştirilmedi**.
**Önceki tur:** `raporlar/duzeltme-turu-2026-11-12.md` (15 madde kapandı, 553→**560** test geçiyor).

> **Sınır:** GitHub'a hiçbir yorum/review/PR gönderilmedi. Sır değeri yazdırılmadı.

---

## 0. Yönetici özeti — en kritik 5 iş

| # | İş | Sahip ajan | Etki | Efor |
|---|---|---|---|---|
| **C1** | Ban anahtarı formatına bağımlılığı kaldır (`dev:` + 68 karakter varsayımı) | Ban Mute Kimlik | Ban'lar **sessizce** tanınmıyor | ~1 saat |
| **C2** | `match.destroy()` sonrası `onPlayerJoin` accessor'ını geri al | Oda Çekirdek | Yeniden kurulumda **çift sarmalama** | ~30 dk |
| **C3** | `identity_sightings` büyümesini sınırla | Ban Mute Kimlik | DB **sınırsız büyüme** | ~2 saat |
| **C4** | Sır rotasyonu (yalnız kullanıcı) + `README.md` temizliği | Sır ve Yapılandırma | Sırlar **yanmış** durumda | 30 dk |
| **C5** | Test paketinde kalan sabit beklemeler | Test ve Kanıt | Kırmızı testler **yanlış alarm** | ~2 saat |

---

## 1. Haxball Ban Mute Kimlik Mühendisi

### C1 — 🔴 Ban eşleştirmesi anahtar formatına bağlı
**Yer:** `src/identity.js:359-366`
**Sorun:** Kod `banKey.startsWith('dev:') && banKey.length === 68` varsayıyor. `dev:` + 64 hex = 68 karakter. Format değişirse (kısa hash, farklı prefix, `auth:` anahtarı) ban **sessizce tanınmaz** — hata vermez, log da yazmaz.
**Çözüm:**
1. `src/bans.js`'e `isDeviceKey(k)` yardımcısı ekle: `/^dev:[0-9a-f]{64}$/`.
2. `identity.js` bu yardımcıyı kullansın, `startsWith`+`length` ikilisi kalksın.
3. Tanınmayan anahtar için `logInfo('identity.ban-key-unrecognized', ...)` yaz (bir kez, saatte en fazla).
**Test:** `tests/moderation.test.js`'e "bilinmeyen anahtar formatı loglanır ve ban uygulanır" testi.
**Kabul:** `dev:` prefix'i olmayan geçerli bir ban kaydı da yakalanıyor.

### C3 — 🟠 `identity_sightings` sınırsız büyüyor
**Yer:** `src/identity.js:226` (`recordSighting`, her join'de çağrılıyor), `:732` (purge **günde bir**, 90 gün)
**Sorun:** Yoğun odada günde binlerce satır; purge günde bir kez → tepe anında şişkinlik. SQLite yazma kilitleri uzar (bu, `joinGate` 400 ms sorununun da kaynağıydı).
**Çözüm:**
1. `recordSighting` upsert olsun: `(authHash, day)` tekilleştir, gün içinde tekrar join **UPDATE** etsin.
2. Purge'u saatlik yap (`PURGE_EVERY_MS = 3600_000`), `dbRun` sonrası `VACUUM` **yapma** (pahalı) — `PRAGMA auto_vacuum=INCREMENTAL` + `incremental_vacuum`.
3. Tablo boyutu için `/api/health`'e `sightingsRows` ekle.
**Test:** aynı oyuncu 100 kez join → 1 satır.

### C6 — 🟡 Mute anahtarı isme bağlı
**Yer:** `src/mutes.js:10` (`nameKey`)
**Sorun:** Oyuncu adını değiştirince mute atlanır. Cihaz anahtarı (`deviceKey`) var ama join yolunda isim anahtarı da kullanılıyor.
**Çözüm:** Join'de `authHash` varsa **yalnız** `deviceKey` kullan; isim anahtarı yalnız kimliksiz oyuncular için yedek olsun.
**Test:** `moderation.test.js` zaten bunu kilitliyor ("isim değiştirince anahtar değişir") — testin adı "isim değiştirse de mute sürer" olarak **değiştirilmeli** (davranış değişikliği onay ister).

---

## 2. Haxball Oda Çekirdek Mühendisi

### C2 — 🔴 `destroy()` accessor'ı geri almıyor
**Yer:** `src/match/index.js:1951-1988` (accessor), `destroy()` içinde temizlik **yok**
**Sorun:** `Object.defineProperty(room, 'onPlayerJoin', {...})` bir kez kuruluyor; `_origJoin` eski motorun fonksiyonuna bağlı kalıyor. Motor yeniden kurulursa **çift sarmalama** olur (aynı join iki kez işlenir → çift log, çift slot sayımı).
**Çözüm:**
```js
// initMatchMechanics içinde sakla:
const prevDescriptor = Object.getOwnPropertyDescriptor(room, 'onPlayerJoin');
// destroy() içinde geri koy:
if (prevDescriptor) Object.defineProperty(room, 'onPlayerJoin', prevDescriptor);
else delete room.onPlayerJoin;
```
**Test:** `destroy()` → `initMatchMechanics()` → join → işleyici **bir kez** çalışmalı.

### C7 — 🟠 `retryGetScores` 5 denemede toplam 1 sn bekliyor
**Yer:** `src/match/index.js:1605-1619`
**Sorun:** Skor yoksa 0+50+150+300+500 ms sonra `null` dönüyor. Yavaş makinede maç sonucu **kaybolur** (ELO yazılmaz, `reporter.abort()` çağrılır).
**Çözüm:** `lastScores` yedeğini **önce** dene (`retryGetScores() || lastScores` zaten var ama `lastScores` yalnız `onTeamGoal`'da güncelleniyor). `onGameStart` ve her `onTeamGoal`'da güncelle; ayrıca son denemeyi 800 ms'ye çıkar.
**Test:** `getScores` ilk 4 denemede `null`, 5'te değer dönerse sonuç yazılmalı.

### C8 — 🟡 `onGameStop` promise'i switch tamamlanmadan çözülüyor
**Yer:** `src/match/index.js:1775-1795`
**Sorun:** `requestSwitch` yalnız 250 ms'lik timer kurar; `onGameStop` promise'i timer çalışmadan çözer. Testler bunu sabit `sleep` ile telafi ediyordu (bu turda `waitFor` ile düzeltildi).
**Çözüm:** `requestSwitch` bir promise döndürsün, `applySwitch` içinde `resolve` edilsin; `onGameStop` onu `await` etsin.
**Kazanç:** Hem üretimde sıra garantisi hem testlerde sabit bekleme ihtiyacı kalkar.

---

## 3. Haxball Süreç Dayanıklılığı Mühendisi

### C9 — 🟠 `/api/health` `players` alanı `null` dönebilir
**Yer:** `src/web/routes/stats.js` (bu turda eklendi), `src/match/index.js` `getPlayerList`
**Sorun:** Oda hazır değilse `null` döner ama `room:true` olabilir → izleme sistemi "oda var, oyuncu yok" sanır.
**Çözüm:** `players:null` iken `room:false` yap; `phase`'i `'booting'` olarak raporla.

### C10 — 🟡 `logs/` büyümesi sınırsız
**Yer:** `src/logger.js`
**Sorun:** `app.log`, `players.log`, `auth.log`, `errors.log` için rotasyon yok. Aylık gigabaytlara çıkabilir, disk dolar.
**Çözüm:** Günlük rotasyon (`app-YYYY-MM-DD.log`), 14 gün saklama, açılışta eski dosyaları sil. Boyut > 100 MB ise aynı gün içinde de döndür.

### C11 — 🟡 Yedek doğrulaması yok
**Yer:** `scripts/` yedek betikleri, `BACKUP_RETENTION_DAYS`
**Sorun:** Yedek alınıyor ama **geri yüklenebilir olduğu test edilmiyor**.
**Çözüm:** Haftalık `backup-verify` işi: yedeği geçici dizine aç, `PRAGMA integrity_check` çalıştır, tablo sayılarını karşılaştır, sonucu `logInfo('backup.verify', ...)`.

---

## 4. Haxball Sır ve Yapılandırma Mühendisi

### C4 — 🔴 Sırlar yanmış durumda
**Yer:** `.env` (git'te), `github-upload/README.md`
**Durum:** Bu turda **açılışta sır sağlık kontrolü** eklendi (`server.js`, madde 18). Ama rotasyon yapılmadı.
**Çözüm:** `kurulum/SIR-ROTASYONU.md` — 6 değer değişecek (3'ü sır değil: `OWNER_NAME`, rol ve kanal kimlikleri). Discord **kapsam dışı** (kullanıcı kararı).
**Kalan risk:** `JWT_SECRET` sızarsa herkes her kullanıcı olarak giriş yapabilir; `ADMIN_JWT_SECRET` sızarsa panelde admin olur.

### C12 — 🟠 `.env.example` ile kod arasında sözleşme yok
**Yer:** `.env.example` (24 satır), `server.js:53-105`
**Sorun:** Kod 6 değişkeni zorunlu sayıyor ama `.env.example` hangisinin zorunlu olduğunu işaretlemiyor; yeni geliştirici eksik başlatıyor.
**Çözüm:** `.env.example`'a `# ZORUNLU:` / `# OPSIYONEL:` işaretleri; `verify.js` `.env.example`'daki her anahtarın kodda okunduğunu doğrulasın.

### C13 — 🟡 Sırlar `logs/`'a sızabilir
**Yer:** `src/logger.js`, `server.js` (bu turda `shortHash` eklendi)
**Sorun:** `shortHash` yalnız `[DupName]` logunda kullanılıyor; başka bir yerde `console.log(auth)` eklenirse düz yazılır.
**Çözüm:** `logger.js`'e `redactSecrets(text)` ekle: `HAXBALL_TOKEN`/`JWT_SECRET` değerlerini `process.env`'den okuyup çıktıda `<REDACTED>` ile değiştir. Tüm `log*` fonksiyonlarından geçir.

---

## 5. Haxball Test ve Kanıt Mühendisi

### C5 — 🟠 Test paketinde sabit beklemeler
**Durum:** Bu turda **4 test** sağlamlaştırıldı:
| Test | Önce | Sonra |
|---|---|---|
| `match.index` — onGameStop skor yok | `sleep(1800)` | `await onGameStop` + `waitFor(mode==='warmup')` |
| `match.index` — warmup 4+ oyuncu | `sleep(5600)` | `waitFor(mode==='match', 8000)` |
| `slotWatch` — periyodik çalışma | `sleep(65)` | `while (runs<2)` (en fazla 3 sn) |
| `vip.commands` — çekiliş sonucu | `sleep(5600)` | `while (!/ÇEKİLİŞ SONUCU/)` (en fazla 12 sn) |

**Sonuç:** **560 test / 560 geçti / 0 düştü.**
**Kalan:** Pakette hâlâ ~20 sabit `sleep` var (en uzunu 5,6 sn). Hepsi `waitFor` desenine çevrilmeli.
**Çözüm:** `tests/helpers/waitFor.js` ortak yardımcı; `node --test` süresi ~%30 kısalır.

### C14 — 🟡 Moderasyon testleri yalnız birim düzeyinde
**Yer:** `tests/moderation.test.js` (22 test, bu turda eklendi)
**Sorun:** `bans`/`mutes`/`identity` fonksiyonları test ediliyor ama **join yolundan uçtan uca** ban/mute uygulanması test edilmiyor.
**Çözüm:** `tests/moderation.e2e.test.js`: mock room + gerçek `joinGate` → banlı oyuncu join → `kickPlayer` çağrıldı mı, `[Ban]` logu yazıldı mı.

### C15 — 🟡 Kapsam ölçümü yok
**Sorun:** Hangi dosyaların test edilmediği bilinmiyor. `src/match/index.js` (2.400 satır) en riskli dosya.
**Çözüm:** `node --test --experimental-test-coverage` çıktısını `raporlar/kapsam-<tarih>.md`'ye yaz; %60 altındaki dosyaları listele.

---

## 6. Haxball Teknik Denetçi — kod kalitesi

### C16 — 🟡 `src/match/index.js` tek dosyada 2.400 satır
**Sorun:** Isınma, dizilim, maç, skor, rebalance, altın gol, replay hepsi aynı closure'da. Değişiklik yapmak riskli; 14 eslint hatası da burada.
**Çözüm (aşamalı):** `match/warmup.js`, `match/dizilim.js` (var), `match/scoring.js`, `match/lifecycle.js` olarak ayır. İlk adım: saf fonksiyonları (`decideNextModeAfterMatch`, `retryGetScores`) dışa taşı — davranış değişmez, test edilebilir olur.

### C17 — 🟢 Kalan 28 eslint hatası
`src/match/index.js` 14 · `src/identity.js` 5 · testler 6 · `public/js/views/identity.js` 2 · `src/commands.js` 1. Hepsi `no-unused-vars`/`prefer-const`. Dosya dosya "ölü kod mu, eksik mi" kararı gerekiyor.

---

## 7. Uygulama sırası (öneri)

| Sıra | İş | Neden bu sıra |
|---|---|---|
| 1 | **C1** ban anahtarı | En yüksek sessiz hata riski, ~1 saat |
| 2 | **C2** accessor geri alma | Yeniden kurulumda çift işleme, ~30 dk |
| 3 | **C4** sır rotasyonu | Yalnız kullanıcı; kod tarafı hazır |
| 4 | **C7** skor yedeği | ELO kaybı riski |
| 5 | **C3** sightings büyümesi | DB sağlığı |
| 6 | **C5** kalan sleep'ler | Test güvenilirliği |
| 7 | C6, C8, C9-C17 | Orta/düşük |

---

## 8. Bu turda yapılanlar (kanıt)

| İş | Sonuç |
|---|---|
| Ajanların alan taraması | 17 bulgu (C1-C17), hepsi dosya:satır ile |
| Test sağlamlaştırma | 4 test `waitFor` desenine çevrildi |
| **Tam test paketi** | **560 test / 560 geçti / 0 düştü** |
| `tsc --noEmit` | temiz |
| `eslint` (değişen testler) | temiz |

**Kod değişikliği yapılmadı** — bu rapor yalnız bulgu ve plan içerir. Uygulama için madde madde onay gerekiyor.
