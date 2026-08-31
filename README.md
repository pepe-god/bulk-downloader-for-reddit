# Bulk Downloader for Reddit

[![PyPI Status](https://img.shields.io/pypi/status/bdfr?logo=PyPI)](https://pypi.python.org/pypi/bdfr)
[![PyPI version](https://img.shields.io/pypi/v/bdfr.svg?logo=PyPI)](https://pypi.python.org/pypi/bdfr)
[![PyPI downloads](https://img.shields.io/pypi/dm/bdfr?logo=PyPI)](https://pypi.python.org/pypi/bdfr)
[![AUR version](https://img.shields.io/aur/version/python-bdfr?logo=Arch%20Linux)](https://aur.archlinux.org/packages/python-bdfr)
[![Code style: black](https://img.shields.io/badge/code%20style-black-000000.svg?logo=Python)](https://github.com/psf/black)

Bulk Downloader for Reddit (BDFR) downloads submissions or submission data from Reddit. It can archive data, crawl Reddit for research, or download media through an extensive command-line interface. See the [list of supported sources](#list-of-currently-supported-sources) below.

## Installation

BDFR requires **Python 3.14 or above**.

```bash
python3 -m pip install bdfr --upgrade
```

or via [pipx](https://pypa.github.io/pipx):

```bash
python3 -m pipx install bdfr
```

- **Update**: re-run the install command, or `pipx upgrade bdfr`
- **Version**: `bdfr --version`
- **Shell completions**: `bdfr completions`

**AUR** (Arch Linux): [python-bdfr](https://aur.archlinux.org/packages/python-bdfr) / [python-bdfr-git](https://aur.archlinux.org/packages/python-bdfr-git)

## Usage

Three modes are available:

- `bdfr download` — downloads the media linked in submissions (images, videos, etc.)
- `bdfr archive` — saves submission data (post details, comments, stats) as JSON, XML, or YAML
- `bdfr clone` — performs both download and archive in one pass

```bash
bdfr download ./output --subreddit Python -L 10
bdfr download ./output --user me --saved --authenticate -L 25
bdfr archive ./output --subreddit all --format yaml -L 500
```

Options can also be loaded from a YAML file:

```bash
bdfr download ./output --opts my_opts.yaml
```

## Options

For a full list of options, run `bdfr download --help`, `bdfr archive --help`, or `bdfr clone --help`.

| Option | Description |
|--------|-------------|
| `-s, --subreddit` | Subreddit source (repeatable, CSV supported) |
| `-u, --user` | User source; use `--user me` with `--authenticate` |
| `-l, --link` | Direct submission URL or ID (repeatable) |
| `-m, --multireddit` | Multireddit source (repeatable) |
| `-L, --limit` | Max submissions per source (default: Reddit max ~1000) |
| `-S, --sort` | Sort: `hot`, `new`, `top`, `rising`, `controversial`, `relevance` |
| `-t, --time` | Time filter: `all`, `hour`, `day`, `week`, `month`, `year` |
| `--search` | Search term (requires `--subreddit` or `--multireddit`) |
| `--saved` | Use saved posts (requires `--authenticate --user me`) |
| `--upvoted` | Use upvoted posts (requires `--authenticate --user me`) |
| `--submitted` | Use user's submissions |
| `--authenticate` | Use authenticated Reddit session |
| `--config` | Path to configuration file |
| `--opts` | Path to YAML options file |
| `--file-scheme` | File naming pattern (default: `{REDDITOR}_{TITLE}_{POSTID}`) |
| `--folder-scheme` | Folder naming pattern (default: `{SUBREDDIT}`) |
| `--skip` | Skip file extensions (repeatable) |
| `--skip-domain` | Skip domains (repeatable) |
| `--skip-subreddit` | Skip subreddits (repeatable) |
| `--min-score` / `--max-score` | Upvote score filter |
| `--min-score-ratio` / `--max-score-ratio` | Upvote ratio filter |
| `--exclude-id` | Skip submission IDs (repeatable) |
| `--include-id-file` | Include only IDs from file(s) |
| `--ignore-user` | Ignore users (repeatable) |
| `--disable-module` | Disable downloader modules (repeatable) |
| `--filename-restriction-scheme` | Force `windows` or `linux` filename rules |
| `--max-wait-time` | Max retry wait in seconds (default: 120) |
| `--no-dupes` | Skip duplicate downloads (MD5 hash) |
| `--make-hard-links` | Hard-link duplicates instead of re-downloading |
| `--search-existing` | Hash existing files for dedup/hard-link |
| `-f, --format` | Archive format: `json`, `xml`, `yaml` |
| `--all-comments` | Download all user comments |
| `--comment-context` | Download parent submission for comments |
| `--log` | Custom log file path |
| `-v, --verbose` | Increase verbosity (repeatable) |

Available name scheme keys: `DATE`, `FLAIR`, `POSTID`, `REDDITOR`, `SUBREDDIT`, `TITLE`, `UPVOTES`. Always include `{POSTID}` for uniqueness.

## Authentication

BDFR uses OAuth2 to connect to Reddit. Authentication is only required for private data (saved posts, upvoted posts, private multireddits). On first use, BDFR opens a Reddit authorization URL — grant only the requested permissions. The token is saved for subsequent runs.

## List of currently supported sources

- Direct links
- Delay for Reddit
- Erome
- Gfycat
- Gif Delivery Network
- Imgur
- Reddit Galleries
- Reddit Text Posts
- Reddit Videos
- Redgifs
- Vidble
- YouTube (and any [YT-DLP supported site](https://github.com/yt-dlp/yt-dlp/blob/master/supportedsites.md))

## Contributing

See [CONTRIBUTING](docs/CONTRIBUTING.md) for development setup and guidelines. Please follow the [Code of Conduct](docs/CODE_OF_CONDUCT.md) when interacting with the project.
