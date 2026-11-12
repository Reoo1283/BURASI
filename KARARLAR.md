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

### K14 — P1/P2/P3 maddeleri uygulandı (kullanıcı onayıyla)
**Tarih:** Oturum 3 · **Karar:** Kullanıcı "yapmadığın neler kaldı onları tamamen yap" dedi → `codeChangesWithoutApproval: false` sınırı bu tur için onaylandı.
**Sonuç:** 15 madde kapandı, +294/−156 satır, 16 dosya + 1 yeni test. Test 531 → **553/553**.
**Sınır korundu:** Değişiklikler `haxball-serverr` deposuna **gönderilmedi**; yalnız `raporlar/duzeltme-turu-2026-11-12.md` içinde diff olarak duruyor.

### K15 — Yetki önbelleğinde yalnız POZİTİF sonuç tutulur
**Gerekçe:** İlk sürüm "yetki yok" sonucunu da 30 sn önbelleğe alıyordu; DB kesintisinde gerçek adminler normal oyuncu sayılacaktı. Mevcut `joinGate.test.js` bunu yakaladı.
**Kural:** Negatif sonuç önbelleğe alınmaz; belirsizlik + önbellek yok = fail-closed (reddet).

### K16 — `maxPlayers` tek kaynaktan türetilir
**Gerekçe:** `server.js` sabit `15` ile `slotGuard.totalLimit` çelişiyordu (2 slot boşa gidiyordu).
**Kural:** `limitsForRoom()` tek kaynak; `config/runtime.json → slots` esas alınır, `ROOM_MAX_PLAYERS` yalnız üst sınırı yükseltebilir.

### K17 — Ajanlar `~/.claude/agents/`'a kopyalanır, format değiştirilmez
**Tarih:** Oturum 3 · **Karar:** Kullanıcı "format aynı kalıyor değiştirmeye gerek kalmıyor" dedi.
**Uygulama:** 7 dosya düz `cp` ile kopyalandı; `diff` 7/7 birebir, frontmatter 7/7 geçerli.
**Sınır:** Bu dizin Claude Code'un; AbacusAI Bot'ta ayrı ajan süreci yoktur.

### K18 — Ban anahtarı biçimi tek yerden okunur
**Tarih:** Oturum 3 · **Karar:** `dev:` + 64 hex varsayımı kodun hiçbir yerinde tekrarlanmaz; `bans.parseDeviceKey()` tek kaynak.
**Gerekçe:** Biçim değişince ban **sessizce** tanınmıyordu (hata yok, log yok).
**Davranış:** Geçerli her `dev:<hex>` anahtar kullanılır; beklenmeyen uzunluk kullanılır **ama** `logInfo('identity.ban-key-unrecognized')` ile raporlanır. `dev:` dışı anahtarlar cihaz anahtarı sayılmaz.

### K19 — Sarmalanan room alanları destroy() ile geri alınır
**Tarih:** Oturum 3 · **Karar:** Bir motorda `Object.defineProperty` ile sarmalanan her room alanı için önceki tanım saklanır ve `destroy()` aynen geri koyar (tanım yoksa siler).
**Gerekçe:** Sarmalayıcı kalıcı olursa `_origJoin` eski motora bağlı kalır; yeni motor kurulunca aynı olay iki kez işlenir (çift log, çift mevki uygulaması). Hata görünmez, sadece davranış bozulur.
**Uygulama:** `createMatchEngine` → `prevJoinDescriptor`; `destroy()` → `defineProperty(prev)` / `delete`.

### K20 — Testlerde sabit `sleep` yerine koşul bekleme
**Tarih:** Oturum 3 · **Karar:** Asenkron iş bitişini bekleyen testler sabit süre yerine koşul bekler (`settle`, `waitDbRow`, `waitAnnounce`).
**İstisna:** Test **yokluğu** doğruluyorsa (avatar çizilmemesi, istatistik yazılmaması, yanlış kolona yazılmaması) kısa sabit bekleme korunur ve gerekçesi yorumda yazılır; `waitFor` yokluğu bekleyemez, hemen döner ve test hiçbir şey ölçmez.
**Gerekçe:** Sabit süre makine hızına bağlı; yüklü makinede yanlış kırmızı, hızlı makinede boşa geçen süre.
