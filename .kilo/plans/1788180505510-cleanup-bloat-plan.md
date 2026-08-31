# Plan: Proje temizliği (bloat / eski / gereksiz kısımlar)

## Context
BDFR deposunda pre-commit zaten kaldırıldı. Kalan "eski / gereksiz / bozuk" parçalar:
- `publish.yml` hâlâ twine + `python -m build` + `TWINE_*` secret kullanıyor → yeni uv tabanlı publish akışıyla çelişiyor.
- `mdl` (Ruby markdownlint) ekstra-dilli bir araç; `.mdlrc`, `.markdown_style.rb`, `tox.ini` ve `formatting_check.yml` içinde referanslı; `test.yml` zaten `**/*.md` dosyalarını yok sayıyor.
- `scripts/tests/*.bats` + üç `bats-*` git submodule'i **boş/uninitialized** ve CI'da hiç çalıştırılmıyor.
- `scripts/` içindeki yardımcı dev scriptleri (log ayrıştırıcılar, `unsaveposts.py`) artık istenmiyor.
- `tox.ini` hâlâ `mdl` çağırıyor → mdl kaldırılınca sadeleştirilmeli.

Tüm 9 Python bağımlılığı gerçekten kullanılıyor; `completion.py`, `archive_entry`, `devscripts/` (CI test konfigürasyonu) **korunur**.

## Hedef kapsam (kullanıcı onayladı)
1. `publish.yml` → `uv build` + `uv publish` akışına çevir.
2. `mdl` aracını tamamen kaldır.
3. `bats` submodule'leri + `.bats` shell testlerini sil.
4. `tox.ini`'yi sadeleştir (mdl çıksın, ruff+black kalsın).
5. `scripts/` yardımcı scriptlerini sil.

## Uygulama adımları

### A. mdl aracını kaldır
- Sil: `.mdlrc`, `.markdown_style.rb`.
- `tox.ini`: `[testenv:format_check]` bloğundan `allowlist_externals = mdl` ve `mdl README.md docs/` satırlarını çıkar (yalnız `ruff check` + `black --check` kalsın).
- `.github/workflows/formatting_check.yml`: `sudo gem install mdl` adımını sil.
- `.github/workflows/test.yml`: iki `paths-ignore` listesinden `"-.markdown_style.rb"` ve `"-.mdlrc"` girişlerini çıkar.
- `docs/CONTRIBUTING.md`: markdownlint (mdl) başlıklı maddeyi (yaklaşık satır 91) ve varsa `scripts/`/mdl atıflarını çıkar.

### B. bats submodule + .bats testlerini kaldır
- `git submodule deinit -f scripts/tests/bats scripts/tests/test_helper/bats-assert scripts/tests/test_helper/bats-support` (uninitialized ise yalnızca dizin+`.gitmodules` silme yeterli; `.git/config` içindeki submodule girişlerini de temizle).
- Sil: `.gitmodules`.
- Sil: `scripts/tests/` altındaki `test_extract_failed_ids.bats`, `test_extract_successful_ids.bats`, `bats/`, `test_helper/`, `example_logfiles/` (bats testlerine ait örnek loglar), `README.md`.

### C. scripts/ yardımcı scriptlerini sil
- Sil: `scripts/extract_failed_ids.sh`, `scripts/extract_failed_ids.ps1`, `scripts/extract_successful_ids.sh`, `scripts/extract_successful_ids.ps1`, `scripts/print_summary.sh`, `scripts/print_summary.ps1`, `scripts/unsaveposts.py`, `scripts/README.md`.
- Sonuçta `scripts/` dizini tamamen boşalacak → `scripts/` dizinini de kaldır.
- Doğrula: README.md / docs içinde `scripts/` (devscripts hariç) referansı kalmadı (grep ile).

### D. tox.ini sadeleştir
- B kapsamında zaten mdl çıkarıldı; `format` env değişmez (`ruff check --fix` + `black`). `[testenv:format_check]` yalnız `ruff check` + `black --check` çalıştırır.

### E. publish.yml → uv'ye çevir
Yeni içerik (eski twine akışının yerine):
```yaml
name: Upload Python Package
on:
  release:
    types: [created]
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Install uv
        uses: astral-sh/setup-uv@v3   # en güncel major tag doğrulanmalı
      - name: Set up Python
        run: uv python install 3.14
      - name: Build and publish
        env:
          UV_PUBLISH_TOKEN: ${{ secrets.UV_PUBLISH_TOKEN }}
        run: |
          uv build --no-sources
          uv publish
      - name: Upload dist folder
        uses: actions/upload-artifact@v3
        with:
          name: dist
          path: dist/
```
- **Breaking secret change:** `TWINE_USERNAME`/`TWINE_PASSWORD` → `UV_PUBLISH_TOKEN`. Repo ayarlarında bu secret'ın tanımlanması gerekir (uygulayıcı not düşsün / kullanıcıya iletilsin).
- `astral-sh/setup-uv` action tag'ini (v3/v4/v5) yayın anında doğrula.

## Korunacaklar (değişmez)
- `bdfr/` Python kaynağı, 9 bağımlılık, `completion.py`, `archive_entry/`, `devscripts/configure.*` (CI test konfigürasyonu), `tox.ini` (sadeleştirilmiş haliyle), `protect_master.yml`, `README.md` (scripts referansı yok), `docs/CODE_OF_CONDUCT.md`, `_config.yml` (kapsam dışı).

## Riskler
- `publish.yml` secret adı değişimi: yayın otomasyonu eski `TWINE_*` secret'larına bağlıysa yeni `UV_PUBLISH_TOKEN` tanımlanana dek kırılır.
- `scripts/` silme: başka bir yerde (ör. dış README/wiki) bu scriptlere bağlanılıyorsa referans kopar; repo içi referans bulunamadı.
- `astral-sh/setup-uv` tag: mevcut değilse workflow hatası; en güncel major ile değiştir.

## Doğrulama
- `git status` → silinen dosyalar: `.mdlrc`, `.markdown_style.rb`, `.gitmodules`, `scripts/` (tamamı), `tox.ini`/`publish.yml`/`formatting_check.yml`/`test.yml`/`CONTRIBUTING.md` değişiklikleri.
- `uv run ruff check bdfr tests` temiz çıkmalı (mdl'a bağlı bir şey kalmamalı).
- `uv run pytest -m "not online and not reddit and not authenticated"` → script silme Python testlerini etkilemez (scriptler import edilmiyor).
- `tox -e format_check` (varsa) yalnız ruff+black çalıştırmalı, `mdl` hatası vermemeli.
- `git submodule status` → submodule kalmamalı (boş çıktı).
