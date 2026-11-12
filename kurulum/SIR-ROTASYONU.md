# Sır Rotasyonu — Adım Adım (12 Kasım 2026)

> **Neden:** `.env` dosyası ve `github-upload/README.md` GitHub deposuna **commit edilmiş**
> (depo geçmişi: 1 commit). Depoya erişimi olan herkes bu değerleri görmüş sayılır.
> Rotasyon = her sırrı **yenisiyle değiştir**, eskisini **iptal et**.
>
> **Kural:** Bu dosyada hiçbir sır değeri yoktur ve olmayacaktır. Yeni değerleri yalnız
> sunucudaki `.env` dosyasına yazın; sohbete, panele, ekran görüntüsüne, commit'e koymayın.

---

## 0. Önce: hangi sırlar, ne kadar kritik?

`.env` içindeki 10 değişkenin **kodda nasıl kullanıldığı** (dosya:satır doğrulandı):

| # | Değişken | Kodda kullanım | Sızarsa ne olur | Öncelik |
|---|---|---|---|---|
| 1 | `HAXBALL_TOKEN` | `server.js:113` | Başkası sizin adınıza **oda açar**; Haxball hesabınızla ilişkili | 🔴 P1 |
| 2 | `JWT_SECRET` | `src/auth.js:39` (kullanıcı oturumu) | Herkes **her kullanıcı olarak** giriş yapabilir (HS256 imza) | 🔴 P1 |
| 3 | `ADMIN_JWT_SECRET` | `src/web/helpers.js:6` (panel oturumu) | Herkes **admin panelinde** oturum açabilir | 🔴 P1 |
| 4 | `ADMIN_PASSWORD` | `src/web/routes/auth.js:18`, `src/commands.js:37` | Panel + `!adminol` ile **moderasyon yetkisi** | 🔴 P1 |
| 5 | `DISCORD_BOT_TOKEN` | `discord-bot/` | Botunuz adına **mesaj/kanal erişimi** | 🟠 P2 |
| 6 | `BOT_API_KEY` | `src/web/helpers.js` (`requireBotKey`) | Bot uçlarına **yetkisiz istek** | 🟠 P2 |
| 7 | `ROOM_PASSWORD` | `server.js:135` | Oda şifresi (şu an **boş**) — sızacak bir şey yok | 🟢 P3 |
| 8 | `OWNER_NAME` | 3 yerde | Sır değil, **kullanıcı adı** | ⚪ rotasyon gerekmez |
| 9 | `DISCORD_ADMIN_ROLE_ID` | 2 yerde | Discord **rol kimliği**, sır değil | ⚪ rotasyon gerekmez |
| 10 | `DISCORD_REPLAY_CHANNEL_ID` | 2 yerde | Discord **kanal kimliği**, sır değil | ⚪ rotasyon gerekmez |

**Sonuç:** Gerçekten değişmesi gereken **6 değer** var (P1 ×4, P2 ×2). Son 3'ü kimlik, sır değil.

---

## 1. Yedek al (5 dakika)

```bash
cd /yol/Haxball\ Server
cp .env .env.yedek-$(date +%Y%m%d-%H%M)      # geri dönüş için
chmod 600 .env .env.yedek-*                   # yalnız sahibi okuyabilsin
```

`.env.yedek-*` dosyasını **asla** commit etmeyin, paylaşmayın, sonra silin.

---

## 2. Yeni değerleri üret (kopyala-yapıştır)

### 2a. Üç JWT/API sırrı — tek komut, üç farklı değer

```bash
node -e "const c=require('crypto');for(let i=0;i<3;i++)console.log(c.randomBytes(48).toString('base64url'))"
```

Çıktıdaki **üç satırı sırayla** şuraya yazın:

```dotenv
JWT_SECRET=<1. satır>
ADMIN_JWT_SECRET=<2. satır>
BOT_API_KEY=<3. satır>
```

> ⚠️ **Üçü birbirinden FARKLI olmalı.** Aynı değeri iki değişkene yazarsanız
> `ADMIN_JWT_SECRET`'in ayrı tutulma amacı ortadan kalkar (kod bunu kontrol etmiyor, siz edin).
> Her satır 64 karakter olur; `server.js:53,65` yalnız "tanımlı mı" diye bakar, uzunluk şartı koymaz.

### 2b. `ADMIN_PASSWORD` — panel + `!adminol` şifresi

En az **16 karakter**, büyük/küçük harf + rakam + işaret karışık. Üretmek için:

```bash
node -e "const c=require('crypto');const s='ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz23456789!@#%*-_';console.log(Array.from(c.randomBytes(20),b=>s[b%s.length]).join(''))"
```

Çıktıyı **bir parola yöneticisine** kaydedin (sunucudaki `.env`'e ek olarak). Bu şifre:
- `src/web/routes/auth.js:18` → admin paneline giriş
- `src/commands.js:37` → oyunda `!adminol <sifre>` ile moderatör yetkisi

> `server.js:61` 8 karakterden kısaysa uyarı basar; sadece uyarıdır, engellemez. 16+ kullanın.

### 2c. `HAXBALL_TOKEN` — yalnız Haxball'dan alınır

1. Tarayıcıda **https://www.haxball.com/headlesstoken** adresini açın (Haxball hesabınızla giriş yapmış olun).
2. Sayfadaki **"Generate new token"** düğmesine basın.
3. Çıkan uzun token'ı kopyalayıp `.env`'e yazın: `HAXBALL_TOKEN=<yeni token>`

> Bu adım **eski token'ı geçersiz kılar** — yeni token üretildiği an eskisi çalışmaz.
> Yani eski token sızdıysa, bu tek adım sorunu çözer. Yeni token en az 20 karakter olmalı
> (`server.js:114` kontrolü).

### 2d. `DISCORD_BOT_TOKEN` — Discord Developer Portal

1. **https://discord.com/developers/applications** → botunuzu seçin.
2. **Bot** sekmesi → **Token** → **"Reset Token"** → onaylayın.
3. Yeni token'ı kopyalayıp `.env`'e yazın.

> Reset'ten sonra eski token **anında** ölür. Bot yeniden başlatılana kadar çevrimdışı kalır — normaldir.

---

## 3. `.env`'i güncelle ve doğrula

```bash
cd /yol/Haxball\ Server
nano .env          # 6 değeri değiştir: HAXBALL_TOKEN, JWT_SECRET, ADMIN_JWT_SECRET,
                   #                      ADMIN_PASSWORD, BOT_API_KEY, DISCORD_BOT_TOKEN
chmod 600 .env

# Değerleri EKRANA BASMADAN yalnız "tanımlı mı + uzunluk" kontrolü:
node -e "
require('dotenv').config();
const need={HAXBALL_TOKEN:20,JWT_SECRET:40,ADMIN_JWT_SECRET:40,ADMIN_PASSWORD:16,BOT_API_KEY:40,DISCORD_BOT_TOKEN:50};
for(const [k,min] of Object.entries(need)){
  const v=process.env[k]||'';
  console.log(k.padEnd(20), v ? (v.length>=min?'OK ('+v.length+')':'KISA ('+v.length+', en az '+min+')') : 'EKSIK');
}
const s=new Set([process.env.JWT_SECRET,process.env.ADMIN_JWT_SECRET,process.env.BOT_API_KEY]);
console.log('uc sir birbirinden farkli mi:', s.size===3?'EVET':'HAYIR - DUZELT');
"
```

Beklenen çıktı: altı satır **OK**, son satır **EVET**.

---

## 4. Yeniden başlat ve gerçekten çalıştığını gör

```bash
# 1) Süreci yeniden başlat (systemd kullanıyorsanız):
sudo systemctl restart haxball
sudo systemctl status haxball --no-pager | head -5

# 2) Sağlık kontrolü — yeni /api/health (bu turda eklendi):
curl -s http://127.0.0.1:3000/api/health | head -c 400
# Beklenen: {"status":"ok","db":true,"room":true,"phase":"warmup",...}
```

Sonra elle deneyin:

| Kontrol | Nasıl | Beklenen |
|---|---|---|
| Oda açıldı mı | Haxball'da oda adını arayın | Oda görünür, giriş yapılabilir |
| Panel şifresi | `/panel` → yeni `ADMIN_PASSWORD` | Giriş başarılı, **eski şifre reddedilir** |
| Oyun içi yetki | Oyunda `!adminol <yeni sifre>` | "Moderator yetkisi kazandiniz!" |
| Kullanıcı oturumu | Bir test hesabıyla giriş | Giriş başarılı (yeni `JWT_SECRET` ile imzalandı) |
| Discord botu | Kanaldan bir slash komutu | Yanıt gelir |

---

## 5. Sızmayı kaynağında kapat

Rotasyon **değerleri** değiştirir; **dosyanın GitHub'da durmasını** değiştirmez. İkisi ayrı iş:

```bash
cd /yol/Haxball\ Server

# .env zaten .gitignore'da (satır 5: .env*), ama geçmişte commit edilmişse takipten çıkarın:
git rm --cached .env
git commit -m "sir dosyasini takipten cikar"

# README.md'deki düz metin şifreyi SİLİN (değeri buraya yazmayın, sadece satırı kaldırın)
# sonra:
git add README.md && git commit -m "README'den kimlik bilgisini kaldir"
```

> **Not:** Bu iki commit **geçmişi temizlemez**. Depo geçmişi 1 commit olduğu için temizlik ucuz:
> ```bash
> pip install git-filter-repo
> git filter-repo --path .env --invert-paths          # .env'i tüm geçmişten sil
> git filter-repo --replace-text <(echo 'literal:<ESKI_SIFRE>==><REDACTED>')  # README'deki değeri maskele
> git push --force --all && git push --force --tags
> ```
> Depo **private** ise bu adımı atlayabilirsiniz — ama değerleri yine de değiştirin, çünkü
> depoya erişimi olan herkes (ve yedekler, fork'lar, CI logları) görmüş olabilir.
> ⚠️ `filter-repo` çalıştırmadan önce depoyu **ayrı bir yere klonlayın**; geri dönüşü yoktur.

---

## 6. Rotasyon sonrası temizlik

- [ ] `.env.yedek-*` dosyasını silin (`shred -u .env.yedek-*`)
- [ ] Eski `ADMIN_PASSWORD`'ü kaydettiğiniz her yerden (notlar, tarayıcı, telefon) silin
- [ ] Haxball ve Discord'da **aktif oturumları** kontrol edin (şüpheli cihaz varsa çıkış yapın)
- [ ] `logs/auth.log` ve `logs/errors.log`'da eski değer geçiyor mu bakın:
      `grep -c "eski-degerin-ilk-4-karakteri" logs/*.log` → 0 olmalı; değilse log dosyalarını döndürün
- [ ] Panelden **tüm kullanıcıları** bir kez çıkışa zorlayın (JWT_SECRET değişti, 12 saat içinde kendiliğinden olur)

---

## 7. Doğrulama: gerçekten bitti mi?

```bash
# 1) Eski değerler artık çalışmıyor mu?
#    - Panelde eski ADMIN_PASSWORD -> "Admin şifresi yanlış!" (401)
#    - Oyunda !adminol <eski sifre> -> "Sifre yanlis!"
#    - Eski JWT ile giriş denemesi -> 401

# 2) Kod tarafı temiz mi?
node verify.js        # sır deseni taraması: "sorun yok" demeli
```

| Durum | Anlamı |
|---|---|
| Altı değer de yeni, eski değerler reddediliyor | ✅ Rotasyon **tamam** |
| Panel açılıyor ama eski şifre de çalışıyor | ❌ Süreç yeniden başlatılmadı — `systemctl restart` |
| `room:false` dönüyor | ❌ `HAXBALL_TOKEN` yanlış/eksik — `server.js:114` kontrolü |
| Discord botu çevrimdışı | ❌ `DISCORD_BOT_TOKEN` yanlış — portal'dan tekrar kopyalayın |

---

## 8. Bir daha olmaması için (kod tarafı, onayınızla)

| Öneri | Nerede | Fayda |
|---|---|---|
| Açılışta sır uzunluğu + benzersizlik kontrolü | `server.js:50-70` | Kısa/aynı sırla açılmayı engeller |
| `.env` yüklenmezse **açık hata** | `server.js` | Sessiz `undefined` sırları önler |
| `logs/`'a sır yazılmadığını doğrulayan test | `tests/` | Regresyon kilidi |
| `verify.js`'e `.env` takip kontrolü | `verify.js` | Commit öncesi yakalar |

Bunlar `YAPILACAKLAR.md`'ye **madde 18** olarak eklendi; kod değişikliği olduğu için onayınızı bekliyor.

---

*Bu dosyada hiçbir sır değeri yoktur. Tüm değerler `<...>` yer tutucudur.*
