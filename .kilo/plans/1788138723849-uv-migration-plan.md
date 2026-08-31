# Plan: bdfr'i uv altyapısına geçir + bağımlılıkları ve Python sürümünü güncelle

## Bağlam (Context)
Proje `bulk-downloader-for-reddit` (bdfr), standart `setuptools` ile derleniyor. `pyproject.toml`
var ama `uv.lock` yok ve ortamda `uv` kurulu değil. Hedef:
1. Bağımlılıkları en son sürümlere yükseltmek
2. Ana (minimum) Python sürümünü `>=3.11`'e çıkarmak
3. Projeyi `uv` altyapısına geçirmek (uv.lock + `.python-version`)

Onaylanan kararlar:
- Hedef Python: **`>=3.11`** (3.9 ve 3.10 desteği düşer)
- Sabitleme: **`pyproject.toml`'de `>=` minimumları koru, tam sürümler `uv.lock` ile kilitlenir** (idiomatik uv yaklaşımı)
- Bağımlılık stratejisi: `>=` alt sınırları olduğu için `uv lock` zaten son uyumlu sürümü çözer; `--upgrade` ile güncel tutulur.

## Adımlar

### 1. uv'yi kur ve Python 3.11'i hazırla
- `curl -LsSf https://astral.sh/uv/install.sh | sh` (installer betiği `astral.sh`'e 301 ile yönlendiriyor, çalışır).
- PATH'e ekle: `export PATH="$HOME/.local/bin:$PATH"` (veya kurulumun gösterdiği yol).
- `uv python install 3.11` ile 3.11'i indir.
- Proje köküne `.python-version` dosyası oluştur, içine `3.11` yaz.

### 2. `pyproject.toml` güncellemeleri
- `requires-python = ">=3.11"` yap.
- `classifiers` içinden `"Programming Language :: Python :: 3.9"` ve `"... 3.10"` satırlarını çıkar;
  yerine `"... 3.11"`, `"... 3.12"`, `"... 3.13"` ekle.
- `[tool.isort]` altındaki `py_version = 39` değerini `311` yap.
- `dependencies` ve `dev` (optional) listelerindeki `>=` alt sınırlarını **olduğu gibi bırak** (uv.lock çözüme bakar).
  İsteğe bağlı: alt sınırları güncel amaçlı son bilinen sürümlere yükselt (zorunlu değil).

### 3. uv.lock üret
- `uv lock --upgrade` çalıştır. Bu, tüm bağımlılıkları (ve dev extras'ları) en son uyumlu
  sürümlere çözüp `uv.lock` dosyasını oluşturur.
- Not: `praw`, `yt-dlp`, `requests`, `beautifulsoup4`, `click` gibi paketlerde yeni major
  sürümler API değişiklikleri getirebilir. `uv.lock` sonrası testlerle doğrulanacak.

### 4. Ortamı kur ve bağımlılıkları yükle
- `uv sync --all-extras` ile sanal ortamı oluştur ve tüm bağımlılıkları (dev dahil) yükle.
  (bdfr setuptools ile dinamik `bdfr.__version__` sürümü kullandığından editable build yapılacak.)

### 5. Doğrulama (testler)
- Çevrimdışı testleri çalıştır: `uv run pytest tests -m "not online and not reddit and not authenticated"`.
- Format/lint kontrolü: `uv run tox -e format_check` (veya `uv run isort --check bdfr tests`
  ve `uv run black --check bdfr tests`).
- Eğer yeni bağımlılık sürümleri nedeniyle kırılım olursa:
  - Ya ilgili bağımlılığa üst sınır ekle (örn. `praw>=7.2.0,<8`),
  - ya da kırılan kodu yeni API'ye göre düzelt.
  - Testler yeşile dönene kadar yinele.

### 6. Dokümantasyon / konfigürasyon güncellemeleri (hafif)
- `README.md` satır 19: "Python version 3.9 or above" → "3.11 or above" güncelle.
- `README.md` "Source code"/CONTRIBUTING bölümüne kısa bir "Geliştirme için uv" notu ekle
  (`uv sync --all-extras` ve `uv run pytest`).
- İsteğe bağlı: `.pre-commit-config.yaml` rev değerlerini (black/isort/flake8) güncel stabil sürümlere çek.

## Riskler
- `yt-dlp`/`praw` gibi hızlı değişen kütüphanelerin son major sürümleri kodla uyumsuz olabilir
  → adım 5'te testlerle yakalanır, üst sınır veya kod düzeltmesi ile çözülür.
- `uv` kurulumu ağ erişimi gerektirir (PyPI ve astral.sh erişilebilir doğrulandı).
- `bdfr.__version__` dinamik sürümü setuptools build sırasında gerektiğinden, editable kurulumda
  paket kaynak dizini (örn. `bdfr/__init__.py`) sürümü tanımlamalı; mevcut yapı bunu karşılıyor.

## Doğrulama (Definition of Done)
- `uv.lock` oluşturuldu ve commit'lenebilir.
- `.python-version` = 3.11 mevcut.
- `requires-python >=3.11` ve classifier'lar güncel.
- `uv sync --all-extras` hatasız tamamlanıyor.
- İlgili pytest testleri (offline) ve format kontrolleri yeşil.

## Açık sorular
- Commit/push yapılması istenmiyor (açıkça istenmedikçe commit atılmamalı). Uygulama sonrası
  kullanıcı commit isterse ayrıca yapılır.

## Notlar
- Bu plan yalnızca bağımlılık/Python/uv geçişini kapsar; upstream kaynak kodu güncellemesi (git pull) kapsam dışıdır.
