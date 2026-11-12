# haxball-agents — Haxball Sunucusu Ajan Ekibi

Bu klasör, **Reoo1283/haxball-serverr** projesi için `msitarzewski/agency-agents` formatında üretilmiş 7 ajanı içerir. Ajanlar upstream şablonuna (frontmatter + Identity/Mission/Rules/Deliverables/Workflow/Metrics) birebir uyar, ama içerik bu projeye özeldir — kopyala-yapıştır "re-skin" değil.

## Ekip

| Ajan | Rol | Alan | Dosya |
|------|-----|------|-------|
| **Haxball Proje Yöneticisi** | Müdür | Görev dağıtımı, bulgu takibi, sahibine rapor | `project-management/haxball-proje-yoneticisi.md` |
| **Haxball Teknik Denetçi** | İnceleme lideri | Kod incelemesi, rapor birleştirme, önem derecesi | `engineering/haxball-teknik-denetci.md` |
| **Haxball Oda Çekirdek Mühendisi** | Çekirdek | `onPlayerJoin`, room lifecycle, playerState, join gate | `engineering/haxball-oda-cekirdek-muhendisi.md` |
| **Haxball Ban Mute Kimlik Mühendisi** | Güvenlik | Ban/mute anahtarları, duplicate auth, kimlik sızıntısı | `security/haxball-ban-mute-kimlik-muhendisi.md` |
| **Haxball Süreç Dayanıklılığı Mühendisi** | SRE | Boot, crash policy, shutdown, yedek, gözlemlenebilirlik | `engineering/haxball-surec-dayanikliligi-muhendisi.md` |
| **Haxball Sır ve Yapılandırma Mühendisi** | Güvenlik | `.env`, git hijyeni, rotasyon, `.env.example` sözleşmesi | `security/haxball-sir-yapilandirma-muhendisi.md` |
| **Haxball Test ve Kanıt Mühendisi** | QA | Test harness, regresyon testleri, kanıt | `testing/haxball-test-kanit-muhendisi.md` |

Makine okunur liste: `agents.json`.

## Upstream uyumu

- **Kaynak:** `msitarzewski/agency-agents` @ `8ae61646` (2026-11-09)
- **Şablon:** `CONTRIBUTING.md` → "Agent File Structure" (`name`, `description`, `color`, `emoji`, `vibe` frontmatter; Identity & Memory, Core Mission, Critical Rules, Technical Deliverables, Workflow Process, Communication Style, Learning & Memory, Success Metrics, Advanced Capabilities bölümleri)
- **Renkler:** `scripts/convert.sh` içindeki `resolve_opencode_color()` haritasındaki adlar veya `#RRGGBB` — hepsi hex kullanıyor
- **Doğrulama:** `scripts/lint-agents.sh` kuralları yerel olarak uygulandı → 0 hata, 0 uyarı
- **Özgünlük:** `scripts/check-agent-originality.sh` mantığıyla (8 kelimelik shingle örtüşmesi) ölçüldü → en yüksek çakışma **%0.79** (upstream FAIL eşiği %40, WARN %20)

## Sınırlar

`agents.json → boundaries`:

- `githubWrites: false` — hiçbir ajan GitHub'a yorum/PR/review yazmaz
- `codeChangesWithoutApproval: false` — kod değişikliği yalnızca sahibin açık onayıyla
- `reportLanguage: tr` — raporlar Türkçe

## Kurulum notu

Upstream `scripts/install.sh` yalnızca kendi `AGENT_DIRS` listesindeki bölümleri kurar; `haxball-agents` yeni bir bölüm olduğu için:

1. Upstream deposunu klonla,
2. `haxball-agents/` klasörünü kök dizine kopyala,
3. `scripts/convert.sh` ve `scripts/lint-agents.sh` içindeki `AGENT_DIRS` dizisine `haxball-agents` ekle,
4. `divisions.json`'a `"haxball-agents": { "label": "Haxball Server", "icon": "Gamepad2", "color": "#3B82F6" }` ekle,
5. `./scripts/convert.sh --tool claude-code` (veya kullandığın araç) → `integrations/claude-code/agents/*.md`,
6. `./scripts/install.sh --agent haxball-teknik-denetci`.

Bu adımlar uygulanmadan da dosyalar doğrudan `~/.claude/agents/` (veya ilgili araç dizinine) elle kopyalanabilir — frontmatter her iki biçimle uyumlu.

## İş bölümü kuralı

Tek görev, tek sahip. Çakışma müdürün hatasıdır:

| Alan | Sahip |
|------|-------|
| `server.js:116-345`, `src/playerState`, `src/joinGate` | Oda Çekirdek |
| `server.js:135-224`, `src/bans`, `src/mutes`, `src/identity` | Ban Mute Kimlik |
| `server.js:41-114`, `server.js:347-384`, `src/dbBackup`, `src/logger` | Süreç Dayanıklılığı |
| `.env`, `.gitignore`, `README.md`, `package.json`, arşivler | Sır ve Yapılandırma |
| `scripts/run-tests.js`, `tests/**` | Test ve Kanıt |
| Rapor birleştirme, önem sıralaması | Teknik Denetçi |
| Görev dağıtımı, durum takibi | Proje Yöneticisi |
