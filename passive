#!/usr/bin/env bash
# webreconroutine-passive: minimalist passive recon for wildcard scopes (*.target.com)
# Uses only tools shipped with Linux Mint: bash, curl, grep, sed, awk, sort, xargs, getent
# No direct traffic to the target (no brute force, no port scan, no dir fuzzing).
# Usage: ./webreconroutine-passive.sh [-r] target.com
#   -r  resolve discovered subdomains in parallel with the system DNS resolver (optional)
# Tuning: WB_TIMEOUT=150 WB_LIMIT=500000 ./webreconroutine-passive.sh target.com
set -u

resolve=0
[ "${1:-}" = "-r" ] && { resolve=1; shift; }
d=${1:-}
[ -z "$d" ] && { echo "Usage: $0 [-r] target.com"; exit 1; }
d=${d#\*.}; d=${d,,}

WB_TIMEOUT=${WB_TIMEOUT:-75}     # max seconds for the Wayback query
WB_LIMIT=${WB_LIMIT:-200000}     # max number of URLs fetched from Wayback

out="recon_$d"; mkdir -p "$out"; cd "$out" || exit 1
dre=${d//./\\.}
# get URL [max-time]  -> no retries, short timeout: one slow source must not stall the script
get() { curl -s --connect-timeout 8 --max-time "${2:-45}" -A "Mozilla/5.0" "$1"; }

echo "[1/3] Querying sources in parallel..."
mkdir -p .tmp
( s=$SECONDS
  get "https://crt.sh/?q=%25.$d&output=json" 60 \
    | grep -o '"name_value":"[^"]*"' | cut -d'"' -f4 | sed 's/\\n/\n/g' > .tmp/crt
  echo "crt     $((SECONDS-s))s" >> .tmp/times ) &
( s=$SECONDS
  get "https://api.certspotter.com/v1/issuances?domain=$d&include_subdomains=true&expand=dns_names" \
    | grep -o '"dns_names":\[[^]]*\]' | grep -o '"[^"]*"' | tr -d '"' | grep -v '^dns_names$' > .tmp/spotter
  echo "spotter $((SECONDS-s))s" >> .tmp/times ) &
( s=$SECONDS
  get "https://api.hackertarget.com/hostsearch/?q=$d" 30 | cut -d, -f1 > .tmp/ht
  echo "ht      $((SECONDS-s))s" >> .tmp/times ) &
( s=$SECONDS
  get "https://otx.alienvault.com/api/v1/indicators/domain/$d/passive_dns" \
    | grep -o '"hostname": *"[^"]*"' | cut -d'"' -f4 > .tmp/otx
  echo "otx     $((SECONDS-s))s" >> .tmp/times ) &
( s=$SECONDS
  get "http://web.archive.org/cdx/search/cdx?url=*.$d&fl=original&collapse=urlkey&output=text&limit=$WB_LIMIT" "$WB_TIMEOUT" \
    | sort -u > wayback_urls.txt
  echo "wayback $((SECONDS-s))s" >> .tmp/times ) &
wait

# Extract hosts from Wayback URLs, merge everything, clean up
sed -E 's|^[a-zA-Z]+://||; s|[:/?#].*||' wayback_urls.txt > .tmp/wb_hosts
cat .tmp/crt .tmp/spotter .tmp/ht .tmp/otx .tmp/wb_hosts 2>/dev/null \
  | tr 'A-Z' 'a-z' | sed 's/^\*\.//; s/^\.//' \
  | grep -E "^[a-z0-9._-]+$" | grep -E "(^|\.)$dre$" | sort -u > subdomains.txt
echo "    source timings:"; sort .tmp/times | sed 's/^/      /'
rm -rf .tmp
echo "    -> $(wc -l < subdomains.txt) subdomains, $(wc -l < wayback_urls.txt) wayback URLs"

echo "[2/3] URL analysis"
grep -E '\?[^=]+=' wayback_urls.txt | sed -E 's/.*\?//' | tr '&' '\n' | cut -d= -f1 \
  | sort | uniq -c | sort -rn | awk '{print $2}' > params.txt &
grep -iE '\.js(\?|$)' wayback_urls.txt > js_files.txt &
grep -iE '\.(php|aspx?|jsp|do|action|cgi)(\?|$)' wayback_urls.txt > dynamic_pages.txt &
grep -iE '\.(json|xml|yml|yaml|conf|config|env|ini|log|bak|old|sql|zip|tar|gz|7z|pem|key|pdf|xls|xlsx|csv|txt)(\?|$)' \
  wayback_urls.txt > interesting_files.txt &
grep -iE '/(admin|login|api|v[0-9]+|graphql|swagger|debug|internal|upload|backup|test|dev|staging)' \
  wayback_urls.txt > interesting_paths.txt &
wait
printf "    params:%s js:%s dynamic:%s files:%s paths:%s\n" \
  "$(wc -l < params.txt)" "$(wc -l < js_files.txt)" "$(wc -l < dynamic_pages.txt)" \
  "$(wc -l < interesting_files.txt)" "$(wc -l < interesting_paths.txt)"

echo "[3/3] DNS resolution"
if [ "$resolve" -eq 1 ]; then
  xargs -a subdomains.txt -P 30 -I{} sh -c \
    'ip=$(getent ahostsv4 "$1" | awk "NR==1{print \$1}"); [ -n "$ip" ] && echo "$1 $ip"' _ {} \
    | sort -u > resolved.txt
  echo "    -> $(wc -l < resolved.txt) hosts resolved (resolved.txt)"
else
  echo "    skipped (enable with -r)"
fi
echo "Done. Output: $(pwd)"
