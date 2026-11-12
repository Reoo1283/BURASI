# BURASI — Haxball İncelemesi Devam Deposu

Bu depo, `Reoo1283/haxball-serverr` projesi için yürütülen incelemenin **hafızasıdır**.
Amaç: başka bir bilgisayarda bu depoyu hedef gösterdiğinizde, çalışmanın
**kaldığı yerden otomatik devam etmesi**.

---

## Yeni bilgisayarda: tek komutla devam

```powershell
gh repo clone Reoo1283/BURASI
cd BURASI
```

Sonra asistan sohbetine şunu yazın:

> `BURASI` deposunu klonladım. `DURUM.md`'yi oku, `sohbet/sohbet-gecmisi.txt` ile
> bağlamı tamamla ve `YAPILACAKLAR.md`'deki ilk açık maddeden devam et.

---

## Depo yapısı

| Klasör | İçerik |
|---|---|
| `DURUM.md` | **Buradan başla.** Nerede kaldık, ne bitti, ne açık. |
| `YAPILACAKLAR.md` | Öncelik sıralı açık işler, her biri sahipli ve dosya:satır referanslı |
| `KARARLAR.md` | Alınmış kararlar ve gerekçeleri — aynı tartışmayı tekrar açmamak için |
| `agents/` | 7 kişilik ajan ekibi + `agents.json` (sınırlar) |
| `raporlar/` | İnceleme, 1. tur özeti, düzeltme raporu, **test koşumu** |
| `sohbet/` | Konuşmanın tam metin dökümü |
| `kurulum/` | Adım adım kurulum kılavuzu |

---

## Kesin sınırlar (her oturumda geçerli)

| Kural | Değer |
|---|---|
| `haxball-serverr` deposuna yazma | **Yok** — yorum, review, PR, commit hiçbiri |
| Kod değiştirme | **Yok** — kullanıcı onayı olmadan |
| Rapor dili | Türkçe |
| Bulgu formatı | `dosya:satır` + önem + düzeltme |
| Sır değerleri | Yazdırılmaz; yalnızca değişken adı + konum |

Bu depo (`BURASI`) ise **okuma/yazma** açıktır — ilerleme buraya kaydedilir.

---

## Sır politikası

Bu depoda **hiçbir sır değeri yoktur ve olmayacaktır.** `sohbet/sohbet-gecmisi.txt`
içindeki tüm `.env` değerleri `<REDACTED>` olarak yazılmıştır. `.env`, `*.sqlite`,
`*.db` ve `.rar` dosyaları buraya **yüklenmez**.

Depoya bir şey eklemeden önce kontrol:

```powershell
# Gerçek sır deseni taraması — çıktı boş olmalı
Select-String -Path .\**\* -Pattern 'gh[pousr]_[A-Za-z0-9]{20,}|=[A-Za-z0-9+/]{24,}'
```

---

## İlerleme nasıl kaydedilir

Her oturum sonunda:

1. `DURUM.md` → "Son güncelleme" tarihini ve yeni durumu yaz
2. `YAPILACAKLAR.md` → biten maddeyi `[x]` yap, yeni çıkanı ekle
3. `KARARLAR.md` → yeni karar varsa ekle
4. `sohbet/` → yeni oturumun dökümünü `sohbet/oturum-YYYY-AA-GG.txt` olarak ekle

```powershell
git add -A
git commit -m "oturum: <ne yapildi>"
git push
```

---

## Proje künyesi

| | |
|---|---|
| İncelenen depo | `Reoo1283/haxball-serverr` (private) — **kaynak: Mediafire `Haxball Server.rar`** |
| Dal / commit | GitHub `main` @ `ad78869` · arşiv `.git` @ `1053c7a` (163 değişiklik izlenmemiş) |
| Dil | JavaScript (CommonJS), Node 20+ (test: v24.22.2) |
| Ölçek | `server.js` 386 satır · `src/` 72 dosya / 16.556 satır · `tests/` 47 dosya / 9.926 satır |
| Test durumu | **531 test / 531 geçti** — `raporlar/test-kosumu-2026-11-12.md` |
| Ajan ekibi | 7 ajan, `msitarzewski/agency-agents` formatında |
