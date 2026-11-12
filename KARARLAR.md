# KARARLAR — Alınmış kararlar ve gerekçeleri

Aynı tartışmayı tekrar açmamak için. Yeni karar eklerken tarih ve gerekçe yazın.

---

### K1 — Klasör git deposu değildi, GitHub API'ye geçildi
**Tarih:** Oturum 1 · **Karar:** `gh`/`git` kurulu değildi, `GH_TOKEN` vardı → REST API.
**Sonuç:** 9 depo tarandı.

### K2 — PR yok, doğrudan `main` incelendi
**Gerekçe:** 9 depoda 0 açık + 0 kapalı PR, yalnız `main` dalı var.
**Sonuç:** İnceleme `main` @ `ad78869` üzerinden yapıldı.

### K3 — Ajanlar upstream şablonuna birebir uyar, içerik projeye özel olur
**Gerekçe:** `msitarzewski/agency-agents` formatı zorunluydu ama kopyala-yapıştır
re-skin reddedildi.
**Kanıt:** Yerel lint 0 hata, 0 uyarı; upstream ile özgünlük örtüşmesi **%0.79**
(upstream FAIL eşiği %40). Upstream ref `8ae61646`.

### K4 — Tek görev = tek sahip
**Gerekçe:** `server.js` satır aralıkları ajanlara bölündü. Aynı satıra iki ajan
bakarsa çakışma **müdürün hatası** sayılır.
**Sonuç:** 76 ham bulgu tekilleştirilerek 21'e indi; "kaç ajan" sütunu önem kanıtı oldu.

### K5 — Sınırlar `agents.json`'a yazıldı, sözlü bırakılmadı
**Değerler:** `githubWrites: false`, `codeChangesWithoutApproval: false`,
`reportLanguage: "tr"`.
**Gerekçe:** Kullanıcı sınırı birkaç kez tekrarladı; makine-okunur olması gerekiyordu.

### K6 — Renkler hex olarak yazıldı
**Gerekçe:** Upstream `resolve_opencode_color()` haritası hex bekliyor.

### K7 — `src/` yok bulgusu geri çekildi
**Tarih:** Oturum 1, düzeltme turu
**Gerekçe:** `Haxball Server.rar` açıldı → `src/` (41 dosya) ve `tests/` (46 dosya) **vardı**.
**Etki:** 21 bulgunun 8'i geçersiz, 14'ü doğrulandı, 8 yeni bulgu çıktı.
**Ders:** "Depoda yok" demeden önce arşivleri aç.

### K8 — Sır değerleri hiçbir yere yazılmaz, dökümde bile
**Gerekçe:** Kullanıcı birkaç kez istedi. `sohbet-gecmisi.txt` üretilirken tüm
`.env` değerleri `<REDACTED>` yapıldı; token/JWT/şifre desenleri tarandı → 0 sızıntı.
**Yan not:** İlk redaksiyon `JWT_SECRET === ADMIN_JWT_SECRET` gibi **karşılaştırma**
satırlarını da yanlışlıkla maskelemişti; düzeltildi. Regex'i dar tutun.

### K9 — `haxball-serverr` deposu salt-okunur
**Gerekçe:** Kullanıcı GitHub'a hiçbir şey yazılmamasını istedi.
**İstisna:** `BURASI` deposu okuma/yazma açık — ilerleme oraya kaydedilir.

### K10 — `BURASI` deposu hafıza deposu olarak kuruldu
**Tarih:** Oturum 1 (taşınma)
**Gerekçe:** Bilgisayar değişiyor, sohbet bağlamı taşınmıyor. Tek `git clone` ile
tüm geçmiş, ajanlar ve açık işler geri gelsin.
**Yapı:** `DURUM.md` (nerede kaldık) + `YAPILACAKLAR.md` (açık işler) +
`KARARLAR.md` (bu dosya) + `agents/` + `raporlar/` + `sohbet/` + `kurulum/`.
**Kural:** Depoya sır, `.env`, `*.sqlite`, `*.db`, `.rar` **girmez**.

### K11 — Testler gerçekten çalıştırıldı, 531/531 geçti
**Tarih:** Oturum 2 · **Karar:** "46 test" iddiası yerine ölçülen değer yazılır: **531 test / 531 geçti / 0 düştü**.
**Kanıt:** `raporlar/test-kosumu-2026-11-12.md` (Node v24.22.2, 66 sn).
**Yan sonuç:** `tsc --noEmit` temiz; `eslint .` 1.283 hata (924'ü `dist/` build çıktısı).

### K12 — Arşivdeki `node_modules` taşınmaz
**Gerekçe:** RAR içindeki `sqlite3`/`bcrypt` Windows derlemesi (`invalid ELF header`), 41 test dosyası yanlış yere düştü. `node_modules/.bin` betikleri izinsiz paketlenmiş (`node-gyp-build: Permission denied`).
**Kural:** Kaynak taşınır, bağımlılık `package-lock.json`'dan kurulur; native modüller makinede derlenir.

### K13 — Kaynak artık Mediafire arşivi, GitHub `main` değil
**Gerekçe:** Kullanıcı `haxball-serverr` ZIP indirmesinden vazgeçti; `Haxball Server.rar` (RAR5, 504 MB açılım) tam kopyayı içeriyor: `src/`, `tests/`, `scripts/`, `config/`, `discord-bot/`, `backups/`, `.git`.
**Etki:** `src/` artık iç içe arşivde değil, doğrudan kökte. İnceleme bu kopya üzerinden yürür.
