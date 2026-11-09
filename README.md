# webreconroutine-passive

A minimalist **passive** recon script for wildcard scopes (`*.target.com`), written for bug bounty work.
It uses only tools that ship with Linux Mint (and most Debian/Ubuntu systems): `bash`, `curl`, `grep`, `sed`, `awk`, `sort`, `xargs`, `getent`. Nothing to install.

It never sends traffic to the target itself: no brute forcing, no port scanning, no directory fuzzing. Everything comes from third-party public data sources.

## Usage

```bash
chmod +x webreconroutine-passive.sh

./webreconroutine.sh target.com        # fully passive
./webreconroutine.sh -r target.com     # also resolve subdomains via your DNS resolver
```

`*.target.com` is accepted too; the leading `*.` is stripped automatically.
Results are written to `recon_<target>/`.

### Tuning

```bash
WB_TIMEOUT=150 WB_LIMIT=500000 ./webreconroutine-passive.sh target.com
```

| Variable     | Default  | Meaning                                  |
|--------------|----------|------------------------------------------|
| `WB_TIMEOUT` | `75`     | Max seconds for the Wayback Machine query |
| `WB_LIMIT`   | `200000` | Max number of URLs pulled from Wayback    |

## What it does

1. **Subdomain discovery** from these sources, queried in parallel:
   - [crt.sh](https://crt.sh) and [Cert Spotter](https://sslmate.com/certspotter/) (Certificate Transparency logs)
   - [HackerTarget](https://hackertarget.com/) and [AlienVault OTX](https://otx.alienvault.com/) (passive DNS)
   - [Wayback Machine CDX API](https://archive.org/help/wayback_api.php) (hosts found in archived URLs)
2. **URL analysis** on the archived Wayback URLs: parameters, JS files, dynamic pages, interesting files and paths.
3. **Optional DNS resolution** (`-r`), 30 lookups in parallel.

Each source's runtime is printed at the end of step 1, so you can see which one is slow or timing out.

## Output files

| File                     | Content                                                        |
|--------------------------|----------------------------------------------------------------|
| `subdomains.txt`         | Unique in-scope subdomains                                     |
| `wayback_urls.txt`       | Archived URLs for the target                                   |
| `params.txt`             | Parameter names, sorted by frequency                           |
| `js_files.txt`           | JavaScript file URLs                                           |
| `dynamic_pages.txt`      | `.php`, `.asp(x)`, `.jsp`, `.do`, `.action`, `.cgi` URLs       |
| `interesting_files.txt`  | `.env`, `.bak`, `.sql`, `.zip`, `.json`, `.log`, `.pem`, ...   |
| `interesting_paths.txt`  | URLs containing `admin`, `login`, `api`, `graphql`, `swagger`... |
| `resolved.txt`           | `host ip` pairs (only with `-r`)                               |

## Notes and limitations

- `-r` sends DNS queries to your configured resolver (not to the target's servers). Skip it if you need strictly zero interaction.
- Free APIs are rate limited and sometimes slow or empty. crt.sh in particular can time out; just re-run.
- Results from passive sources can include hosts that are out of scope or no longer exist. Always verify against the program's scope before testing anything.
- Retries are disabled and timeouts are short on purpose, so one slow source does not stall the whole run.

## Disclaimer

For authorized security testing and bug bounty programs only. Make sure you have permission and stay within the program's scope and rules. You are responsible for how you use this tool.
