# Test Koşumu — 12 Kasım 2026

**Oturum:** 2 (yeni makine) · **Durum:** ✅ İlk gerçek test kanıtı
**Kapatılan madde:** `YAPILACAKLAR.md` → 7 (Testleri gerçekten çalıştır)

---

## 1. Neden bu koşum farklı

Önceki oturumda 46 testin geçtiğine dair **kanıt yoktu**: eski makinede `node` yoktu,
`node_modules` arşiv içindeydi. Bu oturumda kaynak Mediafire arşivinden açıldı ve
bağımlılıklar bu makinede (Linux, Node v24.22.2) yeniden kuruldu.

| | |
|---|---|
| Kaynak | `mediafire.com/file/lb7z45teodeosgx/Haxball+Server.rar` (RAR5, 131.107.391 bayt) |
| Açılan içerik | 12.675 dosya / 1.701 klasör / 504 MB |
| Çalışma dizini | arşivden çıkan `Haxball Server/` kökü |
| Node | v24.22.2 |
| Komut | `node scripts/run-tests.js` (`package.json:6` → `npm test` ile aynı) |

---

## 2. Sonuç

```
ℹ tests 531
ℹ pass 531
ℹ fail 0
ℹ cancelled 0
ℹ skipped 0
ℹ duration_ms 66015
exit code: 0
```

**531 test, 531 geçti, 0 düştü.** 47 test dosyası + 1 yardımcı (`tests/webTestUtil.js`).

---

## 3. İlk denemede düşen 41 dosya — kod hatası değil

İlk koşumda 41 test dosyası `test failed` verdi. Kök neden:

```
Error: node_modules/sqlite3/build/Release/node_sqlite3.node: invalid ELF header
code: 'ERR_DLOPEN_FAILED'
```

Arşivdeki `node_modules/sqlite3` **Windows** için derlenmiş bir native modül taşıyor
(`.node` dosyası ELF değil, PE). Aynı sorun `bcrypt` için de geçerliydi.
`npm rebuild` arşiv içinde `node-gyp-build: Permission denied` ile düştü
(`node_modules/.bin` betikleri çalıştırma izni olmadan paketlenmiş).

**Çözüm:** Temiz bir dizinde `sqlite3@6.0.1` + `bcrypt@6.0.0` kaynaktan derlendi,
sonra uygulamanın bağımlılıkları yeniden kuruldu:

```bash
npm install --no-audit --no-fund
npm install-scripts approve sqlite3
npm install-scripts approve bcrypt
npm rebuild sqlite3 bcrypt
```

Sonra tüm testler geçti. Yani **kaynak kodda bir test kırığı yok**; kırık olan
arşive konmuş `node_modules`.

> **Ders:** `node_modules` arşivlenip taşınmamalı. `package-lock.json` yeterli;
> native modüller platforma bağlıdır.

---

## 4. Lint ve typecheck

| Komut | Sonuç |
|---|---|
| `npx tsc --noEmit` | ✅ temiz (0 hata) |
| `npx eslint .` | ❌ **1.283 hata** |

Lint hatalarının dağılımı:

| Dosya | Hata |
|---|---|
| `hbx-ed-main-map_uygulama/dist/assets/index-DU6q1hYM.js` | 924 |
| `src/match/index.js` | 14 |
| `src/identity.js` | 11 |
| `verify.js` | 5 |
| `src/match/dizilim.js` | 4 |
| `tests/dizilim.test.js` | 3 |
| `public/js/views/identity.js` | 2 |
| `remove_comments.js` | 2 |
| `strip_script.js` | 2 |
| `Her_boku_bilen_discord_botu/**` | 3 |

**Yorum:** 1.283 hatanın 924'ü (`%72`) build çıktısı olan `dist/` dosyasından geliyor —
`eslint.config.js` içindeki `ignores` listesine `**/dist/**` eklenmeli. Gerçek kaynakta
kalan ~359 hata çoğunlukla `no-unused-vars` ve `prefer-const`. `verify.js`'in 5 hatası
`duzeltme-raporu.md` bulgu 14 ile aynı dosyayı işaret ediyor (kullanılmayan
`modified` / `newContent` değişkenleri) — dosya zaten kaldırılacaklar listesinde.

`eslint .` çıkış kodu `0` döndü çünkü hatalar `tail`'e boru hattına girdi; komut
doğrudan çalıştırıldığında `1` döner. Bu, `verify.js`'in "her koşulda exit 0" davranışıyla
karıştırılmamalı — burada sorun yalnızca boru hattı.

---

## 5. Kapsam dışı kalanlar

- `npm run bot` (`discord-bot/index.js`) çalıştırılmadı — gerçek Discord token'ı ister.
- `npm start` (`server.js`) çalıştırılmadı — gerçek Haxball token'ı ve canlı oda gerektirir.
- Canlı veritabanı (`database.sqlite`) testlerde kullanılmadı; testler kendi geçici
  SQLite dosyalarını `/tmp` altında açıp kapatıyor.
- Arşivdeki `.git` (commit `1053c7a`, origin `haxball-server`) incelenmedi; çalışma
  ağacında 163 değişiklik görünüyor → `YAPILACAKLAR.md` madde 8 hâlâ açık.

---

## 6. Sonuç ve sıradaki adım

`DURUM.md`'deki "46 test çalıştırılmadı, kanıt yok" uyarısı **kapatıldı**: testler
geçiyor ve kanıt bu dosyada.

Sırada kalan doğrulama: `duzeltme-raporu.md`'deki 21 bulgunun `src/` üzerinde
dosya:satır ile teyidi (özellikle `src/web/index.js:146-163`, `src/slotGuard.js:5`,
`src/joinGate.js:67`, `src/mutes.js:10`). Bu, `YAPILACAKLAR.md` madde 2–6'nın
önkoşulu.

**Sınırlar korundu:** GitHub'a hiçbir yorum/review/PR gönderilmedi, incelenen kodda
hiçbir dosya değiştirilmedi. Sır değeri yazdırılmadı.
