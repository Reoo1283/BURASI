# Haxball Ajan Ekibi — Yeni Bilgisayara Otomatik Kurulum

Bu paket, `Reoo1283/haxball-serverr` deposu için kurulmuş **7 kişilik ajan ekibini** ve
tüm inceleme raporlarını taşır. Yeni bilgisayarda tek komutla yerine oturur.

---

## 1. Paket içeriği

```
haxball-agents/
├── README.md                                   # ekip tanıtımı
├── agents.json                                 # sınırlar: githubWrites=false, reportLanguage=tr
├── tur-1-ozet.md                               # 1. tur analiz özeti (76 bulgu → 21 benzersiz)
├── duzeltme-raporu.md                          # RAR açıldıktan sonraki düzeltme (8 geri çekildi)
├── project-management/haxball-proje-yoneticisi.md
├── engineering/haxball-teknik-denetci.md
├── engineering/haxball-oda-cekirdek-muhendisi.md
├── engineering/haxball-surec-dayanikliligi-muhendisi.md
├── security/haxball-ban-mute-kimlik-muhendisi.md
├── security/haxball-sir-yapilandirma-muhendisi.md
└── testing/haxball-test-kanit-muhendisi.md

haxball-serverr-inceleme.md                     # ilk dosya:satır incelemesi (13 bulgu)
```

---

## 2. Yeni bilgisayarda kurulum

### 2.1 Ön koşul

Node.js **20 veya üzeri** (projede `node --test`, `structuredClone`, `crypto.randomUUID` kullanılıyor).

```powershell
node -v      # v20+ olmalı
npm -v
```

Node yoksa: <https://nodejs.org/en/download> → LTS kurulumu, sonra terminali kapat-aç.

### 2.2 Ajan paketini yerleştir

Bu klasörü (`haxball-agents/` + `haxball-serverr-inceleme.md`) şuraya kopyalayın:

```
C:\Users\<kullanici>\.abacusai-bot\session-home\
```

Yani sonuç şöyle olmalı:

```
C:\Users\<kullanici>\.abacusai-bot\session-home\haxball-agents\README.md
C:\Users\<kullanici>\.abacusai-bot\session-home\haxball-serverr-inceleme.md
```

**PowerShell ile tek satır:**
```powershell
$dst = "$env:USERPROFILE\.abacusai-bot\session-home"
New-Item -ItemType Directory -Force -Path $dst | Out-Null
Copy-Item -Recurse -Force .\haxball-agents $dst
Copy-Item -Force .\haxball-serverr-inceleme.md $dst
```

### 2.3 Depoyu klonla

```powershell
mkdir C:\dev -Force | Out-Null
cd C:\dev
gh repo clone Reoo1283/haxball-serverr
# veya: git clone https://github.com/Reoo1283/haxball-serverr.git
```

### 2.4 Bağımlılıkları kur

```powershell
cd C:\dev\haxball-serverr
npm ci
```

> `npm ci` kilitli sürümleri (`package-lock.json`) kurar. `npm install` değil — `ci` kullanın.

### 2.5 `.env` dosyası

Depoda `.env` **commit edilmiş** durumda ve içinde canlı sırlar var. Yeni makinede:

1. `.env.example`'ı `.env` olarak kopyalayın.
2. **Yeni** değerler üretin — eski değerleri kullanmayın (hepsi repoda açıkta kaldı, yanmış sayılır).
3. `JWT_SECRET` ile `ADMIN_JWT_SECRET` **birbirinden farklı** olmalı.

```powershell
Copy-Item .env.example .env
node -e "const c=require('crypto');['JWT_SECRET','ADMIN_JWT_SECRET','BOT_API_KEY','ROOM_PASSWORD'].forEach(k=>console.log(k+'='+c.randomBytes(32).toString('hex')))"
```

Çıktıyı `.env` içine yapıştırın. `HAXBALL_TOKEN` ve `DISCORD_BOT_TOKEN` Haxball/Discord
panellerinden yeniden alınmalı.

### 2.6 Doğrulama

```powershell
npm test          # 46 test dosyası, node --test ile
npm run lint      # eslint
npm run typecheck # tsc --noEmit
```

Üçü de geçerse kurulum tamamdır.

---

## 3. İlk iş: 1. turu tekrar çalıştır

Kurulumdan sonra ajanlara söylenecek tek cümle:

> `haxball-agents/agents.json` sınırlarını oku. 1. turda üretilen 21 bulguyu
> `tur-1-ozet.md`'den al, `duzeltme-raporu.md`'deki 8 geri çekilmiş bulguyu düş,
> kalan bulguları **artık gerçekten mevcut olan** `src/` kodu üzerinde doğrula.
> GitHub'a yazma, kod değiştirme.

Bunun nedeni: 1. tur `src/` klasörü görülmeden yapıldı. Artık kod elimizde olacak,
yani bulguların gerçekliği dosya:satır ile **kanıtlanabilir** hale geliyor.

---

## 4. Kesin sınırlar (her turda geçerli)

| Kural | Değer |
|---|---|
| GitHub'a yazma | **Yok** — yorum, review, PR, commit hiçbiri |
| Kod değiştirme | **Yok** — onay alınmadan |
| Rapor dili | Türkçe |
| Bulgu formatı | `dosya:satır` + önem + düzeltme |
| Sır değerleri | Yazdırılmaz; yalnızca değişken adı + konum |

Kaynak: `haxball-agents/agents.json`

---

## 5. Taşıma notu

Eski makinede `C:\Users\reo\AppData\Local\Temp\hs-full\` altındaki 151 MB'lık
indirilmiş depo **silindi**. Oradaki içerik yeniden klonlamayla geri gelir;
kalıcı olarak kaybolan bir şey yok.
