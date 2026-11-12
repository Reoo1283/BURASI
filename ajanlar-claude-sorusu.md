# Ajanlar Claude'a Göre Mi Yapıldı? — Açıklama ve Çözüm

**Kısa cevap: Hayır.** Ajanlar `msitarzewski/agency-agents` formatında yapıldı ve bu format
**Claude'a özel değil**. Herhangi bir model veya araç aynı `.md` dosyalarını okuyabilir.
Ama "Claude" kelimesi geçiyor — nedeni şu:

---

## 1. Nereden çıktı bu izlenim

Upstream deposunun README'si kendi ajanlarını şu araçlara **kurduğunu** söylüyor:

> "installs it into **Claude Code**, Cursor, Codex, Gemini, Osaurus, and more"

Yani "Claude Code" listede **ilk sırada** geçiyor, ama listede **beş araç** var. Bir asistan
README'yi okuyup "Claude'a göre yapılmış" demiş olabilir. Teknik olarak:

| Gerçek | Ayrıntı |
|---|---|
| Format | Düz Markdown + YAML frontmatter |
| Claude'a özel sözdizimi | **Yok** — `CLAUDE.md`, `.claude/`, `@import` gibi hiçbir şey kullanılmadı |
| Kullanılan frontmatter | `name`, `description`, `color`, `emoji`, `vibe` — hepsi jenerik |
| Model bağımlılığı | **Yok** — dosyalar prompt metni, hangi model okursa aynı çalışır |

---

## 2. Kanıt: bizim ajanlar upstream'e birebir uyuyor

Upstream `CONTRIBUTING.md:87-180` "Agent File Structure" 9 bölüm istiyor. Denetim sonucu:

```
9/9  engineering/haxball-oda-cekirdek-muhendisi.md
9/9  engineering/haxball-surec-dayanikliligi-muhendisi.md
9/9  engineering/haxball-teknik-denetci.md
9/9  project-management/haxball-proje-yoneticisi.md
9/9  security/haxball-ban-mute-kimlik-muhendisi.md
9/9  security/haxball-sir-yapilandirma-muhendisi.md
9/9  testing/haxball-test-kanit-muhendisi.md
```

9 bölümün tamamı: Identity & Memory · Core Mission · Critical Rules · Technical Deliverables ·
Workflow Process · Communication Style · Learning & Memory · Success Metrics · Advanced Capabilities

Frontmatter örneği (`project-management/haxball-proje-yoneticisi.md:2-6`):

```yaml
name: Haxball Proje Yöneticisi
description: Reoo1283/haxball-serverr için ajan ekibini yöneten, ...
color: "#0EA5E9"
emoji: 🧭
vibe: İşi doğru kişiye verir, çıktıyı satır numarasıyla kabul eder, ...
```

`agents.json` bunu makine-okunur hale getiriyor ve **hangi upstream dosyasından türetildiğini**
kaydediyor:

```json
"derivedFrom": [
  "project-management/project-manager-senior.md",
  "engineering/engineering-code-reviewer.md"
]
```

Yani içerik **sıfırdan bu projeye yazıldı**, upstream'den kopyalanmadı — ölçüldü:
8 kelimelik shingle örtüşmesi en yüksek **%0.79** (upstream FAIL eşiği %40).

---

## 3. Çözüm: hangi araçta kullanacaksan o okur

Bu dosyalar için "Claude sürümü" ya da "başka sürüm" diye bir şey yok. Kullanım şekli:

### 3.1 Bu asistanla (AbacusAI Bot)
Ajanları **`delegate_task`** ile çalıştırırsınız. Her seferinde ajan dosyasının içeriğini
göreve yapıştırın:

> `haxball-agents/security/haxball-ban-mute-kimlik-muhendisi.md` dosyasını oku.
> Bu ajanın kimliğine bürün ve şu görevi yap: `<görev>`. Sınırlar: `agents.json` →
> `githubWrites: false`, `codeChangesWithoutApproval: false`.

Zaten 1. turda tam olarak böyle yapıldı — 5 ajana `delegate_task` verildi, 76 bulgu geldi.

### 3.2 Claude Code / Cursor / Codex / Gemini CLI
Kopyala, tanıt, kullan:

```powershell
# Claude Code
Copy-Item .\haxball-agents\*.md "$env:USERPROFILE\.claude\agents\" -Recurse

# Cursor
Copy-Item .\haxball-agents\* "$env:USERPROFILE\.cursor\rules\" -Recurse
```

Her aracın kendi klasörü var, ama **dosya formatı aynı kalıyor** — değiştirmek gerekmiyor.

### 3.3 Hiçbir araç olmadan
Dosyayı açıp bir yapay zekâ sohbetine yapıştırın. Frontmatter'ı (`---` arası) silmeniz yeterli.

---

## 4. İsterseniz "araç-bağımsız" hale getirelim

Şu an ajanlar **İngilizce gövde + Türkçe frontmatter** karışımı kullanıyor (upstream şablonu
İngilizce olduğu için). Bu bir sorun değil ama iki iyileştirme mümkün:

| İyileştirme | Ne kazandırır |
|---|---|
| Gövdeyi tamamen Türkçe yap | Türkçe çalışan modellerde daha tutarlı çıktı |
| `services:` alanı ekle | Upstream `CONTRIBUTING.md:98-101` opsiyonel alan; hangi dış servise ihtiyaç duyduğu makine-okunur olur |
| `tools:` alanı ekle | Hangi araçlara izni olduğu açıkça yazılır (şu an `agents.json`'da) |

**Ama bunların hiçbiri "Claude sorunu"nu çözmez — çünkü ortada çözülecek bir sorun yok.**

---

## 5. Özet

| Soru | Cevap |
|---|---|
| Ajanlar Claude'a mı özel? | **Hayır.** Düz Markdown + YAML, hiçbir modele bağlı değil. |
| Neden öyle denildi? | Upstream README'sinde Claude Code ilk sırada listelendiği için. |
| Değiştirmem gerekir mi? | **Hayır.** Olduğu gibi çalışıyor. |
| Bu asistanla nasıl kullanılır? | `delegate_task` + ajan dosyasını göreve yapıştır. 1. turda böyle yapıldı. |
