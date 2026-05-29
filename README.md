# Настройка sing-box VPN на Ubuntu 24.04 через V2Ray/base64/VLESS подписку

## 0. Назначение гайда

Этот гайд описывает развёртывание sing-box VPN с нуля на Ubuntu 24.04.

Сценарий:

```text
Ubuntu 24.04
VPN-провайдер: BlancVPN или аналогичный
Формат подписки: V2Ray-style subscription
Подписка закодирована в base64
После декодирования внутри список vless:// ссылок
sing-box: 1.13.x
Основной режим: TUN, весь трафик через VPN
Дополнительный режим: proxy/rules, только выбранные домены через VPN
```

Приватные данные в гайде заменены на плейсхолдеры:

```text
<SUBSCRIPTION_URL>
<UUID>
<NODE_NAME>
<SERVER>
<PORT>
<SNI>
<HOST>
<PATH>
```

---

## 1. Как будет работать итоговая схема

Структура файлов:

```text
/etc/sing-box/config.json                    # активный конфиг sing-box
/etc/sing-box/mode                           # tun или proxy
/etc/sing-box/subscription/url.txt           # subscription URL
/etc/sing-box/subscription/raw.txt           # скачанная base64-подписка
/etc/sing-box/subscription/decoded.txt       # декодированные vless:// ссылки
/etc/sing-box/subscription/node-name.txt     # выбранная нода
/etc/sing-box/rules/proxy-domains.txt        # домены для proxy/rules mode
/etc/sing-box/backups/                       # backup конфигов
/usr/local/lib/singbox/generate-config.py    # генератор sing-box config
/usr/local/sbin/vpn-core                     # управляющий скрипт
/usr/local/bin/vpn-*                         # короткие команды
```

Режимы:

```text
TUN mode:
  весь трафик системы -> VPN

Proxy/rules mode:
  только AI/dev/Docker/GitHub/заданные домены -> VPN
  остальное -> напрямую
  локальный proxy: 127.0.0.1:2080
```

---

## 2. Установка sing-box на Ubuntu 24.04

```bash
sudo apt update
sudo apt install -y curl ca-certificates jq python3 coreutils iproute2 nftables dnsutils

sudo mkdir -p /etc/apt/keyrings

sudo curl -fsSL https://sing-box.app/gpg.key \
  -o /etc/apt/keyrings/sagernet.asc

sudo chmod a+r /etc/apt/keyrings/sagernet.asc

cat <<'EOF' | sudo tee /etc/apt/sources.list.d/sagernet.sources >/dev/null
Types: deb
URIs: https://deb.sagernet.org/
Suites: *
Components: *
Enabled: yes
Signed-By: /etc/apt/keyrings/sagernet.asc
EOF

sudo apt update
sudo apt install -y sing-box
```

Проверка:

```bash
sing-box version
which sing-box
systemctl status sing-box --no-pager
```

Пока конфиг не готов — остановить сервис:

```bash
sudo systemctl stop sing-box
sudo systemctl disable sing-box
```

---

## 3. Создание структуры папок

```bash
sudo mkdir -p \
  /etc/sing-box/subscription \
  /etc/sing-box/backups \
  /etc/sing-box/rules \
  /usr/local/lib/singbox

sudo chmod 700 /etc/sing-box
sudo chmod 700 /etc/sing-box/subscription
sudo chmod 700 /etc/sing-box/backups
sudo chmod 755 /etc/sing-box/rules
sudo chmod 755 /usr/local/lib/singbox
```

---

## 4. Сохранение subscription URL

Не вставляй subscription URL в публичные чаты, GitHub, Notion, логи. Это приватный токен.

```bash
sudo install -m 600 -o root -g root /dev/null /etc/sing-box/subscription/url.txt
sudo nano /etc/sing-box/subscription/url.txt
```

Вставить внутрь файла:

```text
<SUBSCRIPTION_URL>
```

Сохранить:

```text
Ctrl + O
Enter
Ctrl + X
```

Проверка без вывода ссылки:

```bash
sudo test -s /etc/sing-box/subscription/url.txt && echo "OK: subscription URL сохранён"
sudo ls -lah /etc/sing-box/subscription/url.txt
```

---

## 5. Скачать и декодировать подписку

```bash
sudo curl -fsSL "$(sudo cat /etc/sing-box/subscription/url.txt)" \
  -o /etc/sing-box/subscription/raw.txt

sudo chmod 600 /etc/sing-box/subscription/raw.txt
```

Проверить первые символы:

```bash
sudo head -c 20 /etc/sing-box/subscription/raw.txt
echo
```

Если видишь что-то вроде:

```text
dmxlc3M6Ly...
```

это base64.

Декодировать:

```bash
sudo base64 -d /etc/sing-box/subscription/raw.txt \
  | sudo tee /etc/sing-box/subscription/decoded.txt >/dev/null

sudo chmod 600 /etc/sing-box/subscription/decoded.txt
```

Проверить количество VLESS-нод:

```bash
sudo grep -c '^vless://' /etc/sing-box/subscription/decoded.txt
```

---

## 6. Посмотреть список серверов без UUID

```bash
sudo grep '^vless://' /etc/sing-box/subscription/decoded.txt \
  | sed 's/.*#//' \
  | python3 -c 'import sys, urllib.parse; [print(urllib.parse.unquote(line.strip())) for line in sys.stdin]' \
  | nl -w3 -s' | '
```

Найти Германию/Берлин:

```bash
sudo grep '^vless://' /etc/sing-box/subscription/decoded.txt \
  | sed 's/.*#//' \
  | python3 -c 'import sys, urllib.parse; [print(urllib.parse.unquote(line.strip())) for line in sys.stdin]' \
  | grep -i 'берлин\|berlin\|germany\|германия'
```

Проверить тип транспорта безопасно:

```bash
sudo python3 - <<'PY'
from urllib.parse import urlsplit, parse_qs, unquote

with open("/etc/sing-box/subscription/decoded.txt", "r", errors="ignore") as f:
    links = [x.strip() for x in f if x.startswith("vless://")]

for i, link in enumerate(links, 1):
    p = urlsplit(link)
    name = unquote(p.fragment or "")
    if any(x in name.lower() for x in ["берлин", "berlin", "germany", "германия"]):
        q = parse_qs(p.query)
        def get(k, default=""):
            return unquote(q.get(k, [default])[-1])
        print(f"{i:03d} | {name} | type={get('type','tcp')} | security={get('security','')} | alpn={get('alpn','')}")
PY
```

Для первого запуска лучше брать:

```text
type=tcp | security=tls
```

или:

```text
type=ws | security=tls
```

`xhttp/splithttp` для первого запуска лучше не брать.

---

## 7. Зафиксировать одну ноду

Пример:

```bash
echo '🇩🇪 Берлин, Германия, Extra' | sudo tee /etc/sing-box/subscription/node-name.txt >/dev/null
sudo chmod 600 /etc/sing-box/subscription/node-name.txt
```

Проверка:

```bash
sudo cat /etc/sing-box/subscription/node-name.txt
```

---

## 8. Создать большой список proxy-доменов для AI/dev/Docker

```bash
sudo tee /etc/sing-box/rules/proxy-domains.txt >/dev/null <<'EOF'
# AI / LLM
openai.com
api.openai.com
chatgpt.com
chat.openai.com
oaistatic.com
oaiusercontent.com
files.oaiusercontent.com
auth0.openai.com
platform.openai.com
cdn.openai.com
sora.com

anthropic.com
api.anthropic.com
claude.ai
console.anthropic.com

google.com
googleapis.com
generativelanguage.googleapis.com
aistudio.google.com
makersuite.google.com
gemini.google.com
bard.google.com
gstatic.com
googleusercontent.com
deepmind.google
notebooklm.google
notebooklm.google.com

mistral.ai
api.mistral.ai
console.mistral.ai

cohere.com
api.cohere.ai
dashboard.cohere.com

perplexity.ai
api.perplexity.ai

groq.com
api.groq.com
console.groq.com

together.ai
api.together.xyz
together.xyz

fireworks.ai
api.fireworks.ai

replicate.com
api.replicate.com
replicate.delivery

openrouter.ai
api.openrouter.ai

huggingface.co
hf.co
cdn-lfs.huggingface.co
huggingfaceusercontent.com
spaces.huggingface.co

stability.ai
api.stability.ai
platform.stability.ai

elevenlabs.io
api.elevenlabs.io
elevenlabs.com

assemblyai.com
api.assemblyai.com

deepgram.com
api.deepgram.com

cartesia.ai
api.cartesia.ai

fal.ai
fal.run

runpod.io
api.runpod.ai
runpod.ai

modal.com
modal.run

baseten.co
api.baseten.co

lambda.ai
cloud.lambda.ai

character.ai
poe.com
you.com
phind.com

# AI IDE / agents
cursor.com
cursor.sh
api2.cursor.sh
cursor-cdn.com

windsurf.com
codeium.com
codeiumdata.com

replit.com
replit.dev
replitusercontent.com

bolt.new
stackblitz.com
stackblitz.io

v0.dev
vercel.ai
lovable.dev

# GitHub / Copilot
github.com
api.github.com
gist.github.com
raw.githubusercontent.com
githubusercontent.com
githubassets.com
githubcopilot.com
api.githubcopilot.com
api.individual.githubcopilot.com
api.business.githubcopilot.com
api.enterprise.githubcopilot.com
copilot-proxy.githubusercontent.com
copilot-telemetry.githubusercontent.com
copilot-ide.githubusercontent.com
github.io
objects.githubusercontent.com
objects-origin.githubusercontent.com
release-assets.githubusercontent.com
codeload.github.com

gitlab.com
gitlab-static.net
bitbucket.org
atlassian.com
atlassian.net

# Docker / containers
docker.com
docker.io
hub.docker.com
registry-1.docker.io
auth.docker.io
production.cloudflare.docker.com
desktop.docker.com
download.docker.com
docs.docker.com
docker.elastic.co

ghcr.io
pkg-containers.githubusercontent.com

quay.io
cdn.quay.io

gcr.io
pkg.dev
k8s.gcr.io
registry.k8s.io

mcr.microsoft.com
azurecr.io

public.ecr.aws
ecr.aws
amazonaws.com

nvcr.io
nvidia.com
developer.download.nvidia.com

# Package managers
npmjs.com
registry.npmjs.org
npmjs.org
nodejs.org
yarnpkg.com
pnpm.io

pypi.org
files.pythonhosted.org
pythonhosted.org
python.org

anaconda.org
conda.io
repo.anaconda.com
conda.anaconda.org

rubygems.org
packagist.org
getcomposer.org

crates.io
static.crates.io
rust-lang.org
rustup.rs

golang.org
go.dev
proxy.golang.org
sum.golang.org
pkg.go.dev

maven.org
maven.apache.org
repo.maven.apache.org
repo1.maven.org
gradle.org
plugins.gradle.org

nuget.org
api.nuget.org

pub.dev
storage.googleapis.com
deno.land
jsr.io
bun.sh

# Cloud / deploy
vercel.com
vercel.app
netlify.com
netlify.app
cloudflare.com
workers.dev
pages.dev
cloudflareworkers.com
fly.io
fly.dev
render.com
onrender.com
railway.app
railway.com
heroku.com
herokuapp.com
digitalocean.com
linode.com
vultr.com
aws.amazon.com
console.aws.amazon.com
cloudfront.net
azure.com
portal.azure.com
azureedge.net
microsoft.com
microsoftonline.com
live.com
cloud.google.com
console.cloud.google.com
supabase.com
supabase.co
firebase.google.com
firebaseio.com
firebasedatabase.app
planetscale.com
neon.tech
upstash.com
mongodb.com
mongodb.net
redis.com
redis.io

# IDE / tools
visualstudio.com
vscode.dev
marketplace.visualstudio.com
vscode-unpkg.net
vscode-cdn.net
jetbrains.com
plugins.jetbrains.com

sentry.io
ingest.sentry.io
posthog.com
app.posthog.com
segment.io
segment.com

# Search / browser automation
bing.com
duckduckgo.com
brave.com
search.brave.com
serpapi.com
api.serpapi.com
browserless.io
browserbase.com
api.browserbase.com
scrapingbee.com
brightdata.com

# Work tools
notion.so
notion.com
notion.site
slack.com
slack-edge.com
discord.com
discordapp.com
linear.app
figma.com
figmausercontent.com
miro.com
airtable.com
zapier.com
make.com
n8n.io

# CDN / files
fastly.net
fastly.com
jsdelivr.net
unpkg.com
cdnjs.cloudflare.com
cloudflareinsights.com
dropbox.com
dropboxapi.com
dropboxusercontent.com
box.com
boxcdn.net

# IP tests
ipinfo.io
ifconfig.me
ifconfig.co
icanhazip.com
ident.me
EOF

sudo chmod 600 /etc/sing-box/rules/proxy-domains.txt
sudo chown root:root /etc/sing-box/rules/proxy-domains.txt
```

---

## 9. Создать генератор sing-box config

Генератор учитывает sing-box 1.13.x:

```text
новый DNS format
без legacy sniff в inbound
route.default_domain_resolver есть
sniff сделан через route action
proxy-domain list читается из /etc/sing-box/rules/proxy-domains.txt
```

```bash
sudo tee /usr/local/lib/singbox/generate-config.py >/dev/null <<'PY'
#!/usr/bin/env python3
import json
import os
import re
import sys
from pathlib import Path
from urllib.parse import urlsplit, parse_qs, unquote

DECODED = Path("/etc/sing-box/subscription/decoded.txt")
NODE_FILE = Path("/etc/sing-box/subscription/node-name.txt")
MODE_FILE = Path("/etc/sing-box/mode")
CONFIG = Path("/etc/sing-box/config.json")
PROXY_DOMAINS_FILE = Path("/etc/sing-box/rules/proxy-domains.txt")

def die(message):
    print(f"ERROR: {message}", file=sys.stderr)
    sys.exit(1)

def get_q(qs, key, default=""):
    return unquote(qs.get(key, [default])[-1])

def split_alpn(value):
    if not value:
        return []
    return [x.strip() for x in re.split(r"[,;]", value) if x.strip()]

def parse_vless(link):
    p = urlsplit(link.strip())
    if p.scheme != "vless":
        die("not a vless link")

    qs = parse_qs(p.query, keep_blank_values=True)

    node = {
        "uuid": unquote(p.username or ""),
        "server": p.hostname or "",
        "port": p.port or 443,
        "name": unquote(p.fragment or ""),
        "query": qs
    }

    if not node["uuid"]:
        die("UUID not found")
    if not node["server"]:
        die("server not found")

    return node

def find_node():
    if not DECODED.exists():
        die("/etc/sing-box/subscription/decoded.txt not found")

    if not NODE_FILE.exists():
        die("/etc/sing-box/subscription/node-name.txt not found")

    wanted = NODE_FILE.read_text(errors="ignore").strip()
    if not wanted:
        die("node-name.txt is empty")

    links = [
        line.strip()
        for line in DECODED.read_text(errors="ignore").splitlines()
        if line.strip().startswith("vless://")
    ]

    exact = []
    partial = []

    for link in links:
        node = parse_vless(link)
        name = node["name"]

        if name == wanted:
            exact.append(link)
        elif wanted.lower() in name.lower():
            partial.append(link)

    if len(exact) == 1:
        return parse_vless(exact[0])

    if len(exact) > 1:
        die("multiple exact nodes found")

    if len(partial) == 1:
        return parse_vless(partial[0])

    if len(partial) > 1:
        print("Several similar nodes found:", file=sys.stderr)
        for link in partial:
            print("- " + parse_vless(link)["name"], file=sys.stderr)
        die("please specify exact NODE_NAME")

    die("selected node not found in subscription")

def make_outbound(node):
    q = node["query"]

    transport_type = get_q(q, "type", "tcp").lower()
    security = get_q(q, "security", "").lower()

    if transport_type in ["xhttp", "splithttp"]:
        die("xhttp/splithttp node is not supported by this generator. Choose tcp+tls or ws+tls.")

    outbound = {
        "type": "vless",
        "tag": "proxy",
        "server": node["server"],
        "server_port": int(node["port"]),
        "uuid": node["uuid"],
        "packet_encoding": "xudp"
    }

    flow = get_q(q, "flow", "")
    if flow:
        outbound["flow"] = flow

    if security == "tls":
        tls = {
            "enabled": True
        }

        sni = (
            get_q(q, "sni", "") or
            get_q(q, "serverName", "") or
            get_q(q, "peer", "") or
            get_q(q, "host", "")
        )

        if sni:
            tls["server_name"] = sni

        alpn = split_alpn(get_q(q, "alpn", ""))
        if alpn:
            tls["alpn"] = alpn

        fp = get_q(q, "fp", "") or get_q(q, "fingerprint", "")
        if fp:
            tls["utls"] = {
                "enabled": True,
                "fingerprint": fp
            }

        outbound["tls"] = tls

    elif security in ["", "none"]:
        pass
    else:
        die(f"unsupported security={security}")

    if transport_type in ["tcp", "raw"]:
        pass

    elif transport_type in ["ws", "websocket"]:
        path = get_q(q, "path", "/") or "/"
        host = get_q(q, "host", "")

        transport = {
            "type": "ws",
            "path": path
        }

        if host:
            transport["headers"] = {
                "Host": host
            }

        outbound["transport"] = transport

    elif transport_type in ["http", "h2"]:
        path = get_q(q, "path", "")
        host = get_q(q, "host", "")

        transport = {
            "type": "http",
            "path": path
        }

        if host:
            transport["host"] = [host]

        outbound["transport"] = transport

    elif transport_type == "grpc":
        service_name = get_q(q, "serviceName", "") or get_q(q, "path", "").strip("/")
        outbound["transport"] = {
            "type": "grpc",
            "service_name": service_name
        }

    else:
        die(f"unsupported transport type={transport_type}")

    return outbound

def dns_config_tun():
    return {
        "servers": [
            {
                "type": "local",
                "tag": "local"
            },
            {
                "type": "https",
                "tag": "remote",
                "server": "1.1.1.1",
                "server_port": 443,
                "path": "/dns-query",
                "detour": "proxy",
                "tls": {
                    "enabled": True,
                    "server_name": "cloudflare-dns.com"
                }
            }
        ],
        "final": "remote"
    }

def dns_config_proxy():
    return {
        "servers": [
            {
                "type": "local",
                "tag": "local"
            },
            {
                "type": "https",
                "tag": "remote",
                "server": "1.1.1.1",
                "server_port": 443,
                "path": "/dns-query",
                "detour": "proxy",
                "tls": {
                    "enabled": True,
                    "server_name": "cloudflare-dns.com"
                }
            }
        ],
        "final": "local"
    }

def load_proxy_domains():
    if not PROXY_DOMAINS_FILE.exists():
        return [
            "openai.com",
            "chatgpt.com",
            "github.com",
            "docker.io",
            "ipinfo.io"
        ]

    domains = []
    for line in PROXY_DOMAINS_FILE.read_text(errors="ignore").splitlines():
        line = line.strip()
        if not line:
            continue
        if line.startswith("#"):
            continue
        domains.append(line)

    seen = set()
    result = []
    for d in domains:
        if d not in seen:
            seen.add(d)
            result.append(d)

    return result

def make_tun_config(outbound):
    return {
        "log": {
            "level": "info",
            "timestamp": True
        },
        "dns": dns_config_tun(),
        "inbounds": [
            {
                "type": "tun",
                "tag": "tun-in",
                "address": [
                    "172.19.0.1/30"
                ],
                "mtu": 9000,
                "auto_route": True,
                "strict_route": True,
                "auto_redirect": True,
                "stack": "mixed"
            }
        ],
        "outbounds": [
            outbound,
            {
                "type": "direct",
                "tag": "direct"
            }
        ],
        "route": {
            "auto_detect_interface": True,
            "default_domain_resolver": {
                "server": "local"
            },
            "rules": [
                {
                    "action": "sniff"
                },
                {
                    "protocol": "dns",
                    "action": "hijack-dns"
                },
                {
                    "ip_is_private": True,
                    "action": "route",
                    "outbound": "direct"
                }
            ],
            "final": "proxy"
        }
    }

def make_proxy_config(outbound):
    return {
        "log": {
            "level": "info",
            "timestamp": True
        },
        "dns": dns_config_proxy(),
        "inbounds": [
            {
                "type": "mixed",
                "tag": "mixed-in",
                "listen": "127.0.0.1",
                "listen_port": 2080
            }
        ],
        "outbounds": [
            outbound,
            {
                "type": "direct",
                "tag": "direct"
            }
        ],
        "route": {
            "auto_detect_interface": True,
            "default_domain_resolver": {
                "server": "local"
            },
            "rules": [
                {
                    "action": "sniff"
                },
                {
                    "domain_suffix": load_proxy_domains(),
                    "action": "route",
                    "outbound": "proxy"
                }
            ],
            "final": "direct"
        }
    }

def main():
    mode = "tun"
    if MODE_FILE.exists():
        mode = MODE_FILE.read_text(errors="ignore").strip() or "tun"

    if mode not in ["tun", "proxy"]:
        die("/etc/sing-box/mode must be tun or proxy")

    node = find_node()
    outbound = make_outbound(node)

    if mode == "tun":
        config = make_tun_config(outbound)
    else:
        config = make_proxy_config(outbound)

    CONFIG.write_text(json.dumps(config, ensure_ascii=False, indent=2) + "\n")
    os.chmod(CONFIG, 0o600)

    q = node["query"]
    print("OK: config generated")
    print(f"mode={mode}")
    print(f"node={node['name']}")
    print(f"type={get_q(q, 'type', 'tcp')}")
    print(f"security={get_q(q, 'security', '')}")
    print(f"alpn={get_q(q, 'alpn', '')}")

if __name__ == "__main__":
    main()
PY

sudo chmod 700 /usr/local/lib/singbox/generate-config.py
sudo chown root:root /usr/local/lib/singbox/generate-config.py
```

---

## 10. Сгенерировать первый TUN config

```bash
echo "tun" | sudo tee /etc/sing-box/mode >/dev/null
sudo chmod 600 /etc/sing-box/mode

sudo /usr/local/lib/singbox/generate-config.py
sudo sing-box check -c /etc/sing-box/config.json
```

Если вывод пустой или без `FATAL` — конфиг валидный.

Не выводи `/etc/sing-box/config.json` в чат: там UUID, сервер и SNI.

---

## 11. Создать команды управления VPN

```bash
sudo tee /usr/local/sbin/vpn-core >/dev/null <<'BASH'
#!/usr/bin/env bash
set -euo pipefail

CONFIG="/etc/sing-box/config.json"
MODE="/etc/sing-box/mode"
GEN="/usr/local/lib/singbox/generate-config.py"
BACKUPS="/etc/sing-box/backups"

cmd="$(basename "$0")"

need_root() {
  if [[ "${EUID}" -ne 0 ]]; then
    echo "ERROR: запусти через sudo" >&2
    exit 1
  fi
}

backup_config() {
  mkdir -p "$BACKUPS"
  if [[ -f "$CONFIG" ]]; then
    cp -a "$CONFIG" "$BACKUPS/config.$(date +%Y%m%d-%H%M%S).json"
    chmod 600 "$BACKUPS"/config.*.json 2>/dev/null || true
  fi
}

case "$cmd" in
  vpn-on)
    need_root
    systemctl start sing-box
    systemctl status sing-box --no-pager --full
    ;;

  vpn-off)
    need_root
    systemctl stop sing-box
    echo "OK: VPN выключен"
    ;;

  vpn-restart)
    need_root
    systemctl restart sing-box
    systemctl status sing-box --no-pager --full
    ;;

  vpn-status)
    systemctl status sing-box --no-pager --full
    ;;

  vpn-logs)
    journalctl -u sing-box --output cat -e -n 120
    ;;

  vpn-test)
    need_root
    sing-box check -c "$CONFIG"
    echo "OK: config валидный"
    ;;

  vpn-update)
    need_root
    backup_config

    URL_FILE="/etc/sing-box/subscription/url.txt"
    RAW="/etc/sing-box/subscription/raw.txt"
    DECODED="/etc/sing-box/subscription/decoded.txt"
    TMP_RAW="$(mktemp)"

    if [[ ! -s "$URL_FILE" ]]; then
      echo "ERROR: нет subscription URL в $URL_FILE" >&2
      exit 1
    fi

    echo "INFO: пробую скачать свежую подписку..."

    if curl -fsSL --retry 2 --retry-delay 5 \
      -A "sing-box-subscription-updater/1.0" \
      "$(cat "$URL_FILE")" \
      -o "$TMP_RAW"; then

      mv "$TMP_RAW" "$RAW"
      chmod 600 "$RAW"
      echo "OK: свежая подписка скачана"

    else
      rm -f "$TMP_RAW"
      echo "WARN: не удалось скачать свежую подписку. Возможно, rate limit 429."
      echo "WARN: использую уже сохранённый raw.txt"

      if [[ ! -s "$RAW" ]]; then
        echo "ERROR: raw.txt отсутствует, нечего использовать как fallback" >&2
        exit 1
      fi
    fi

    if base64 -d "$RAW" > "$DECODED"; then
      chmod 600 "$DECODED"
    else
      echo "ERROR: не удалось декодировать base64 из raw.txt" >&2
      exit 1
    fi

    "$GEN"
    sing-box check -c "$CONFIG"

    if systemctl is-active --quiet sing-box; then
      systemctl restart sing-box
      echo "OK: config пересобран, sing-box перезапущен"
    else
      echo "OK: config пересобран. sing-box сейчас выключен"
    fi
    ;;

  vpn-check-ip)
    curl -4fsS https://ipinfo.io/json || curl -4fsS https://ifconfig.co/json
    echo
    ;;

  vpn-mode-tun)
    need_root
    backup_config
    echo "tun" > "$MODE"
    chmod 600 "$MODE"
    "$GEN"
    sing-box check -c "$CONFIG"
    systemctl restart sing-box
    echo "OK: включён TUN mode"
    ;;

  vpn-mode-proxy)
    need_root
    backup_config
    echo "proxy" > "$MODE"
    chmod 600 "$MODE"
    "$GEN"
    sing-box check -c "$CONFIG"
    systemctl restart sing-box
    echo "OK: включён proxy/rules mode"
    echo "Локальный proxy: 127.0.0.1:2080"
    ;;

  *)
    echo "Unknown command: $cmd" >&2
    exit 1
    ;;
esac
BASH

sudo chmod 755 /usr/local/sbin/vpn-core
sudo chown root:root /usr/local/sbin/vpn-core
```

Создать короткие команды:

```bash
for cmd in \
  vpn-on \
  vpn-off \
  vpn-restart \
  vpn-status \
  vpn-logs \
  vpn-test \
  vpn-update \
  vpn-check-ip \
  vpn-mode-tun \
  vpn-mode-proxy
do
  sudo ln -sf /usr/local/sbin/vpn-core "/usr/local/bin/$cmd"
done
```

Проверка:

```bash
ls -lah /usr/local/sbin/vpn-core
vpn-status || true
sudo vpn-test
```

---

## 12. Запуск TUN mode

```bash
sudo vpn-mode-tun
```

Проверка:

```bash
vpn-status
ip link show | grep -i tun
vpn-check-ip
journalctl -u sing-box --output cat -e -n 40
```

Ожидаемо:

```text
Active: active (running)
tun0 поднят
IP: Berlin / DE или страна выбранной ноды
в логах: inbound/tun и outbound/vless[proxy]
```

---

## 13. Включить автозапуск

```bash
sudo systemctl enable sing-box
systemctl is-enabled sing-box
```

Ожидаемо:

```text
enabled
```

Проверка после перезагрузки:

```bash
sudo reboot
```

После входа:

```bash
systemctl is-enabled sing-box
vpn-status
ip link show | grep -i tun
vpn-check-ip
```

---

## 14. Проверка proxy/rules mode

Переключиться:

```bash
sudo vpn-mode-proxy
```

Проверить системный IP и proxy IP:

```bash
echo "=== SYSTEM IP ==="
curl -4fsS https://ipinfo.io/json

echo
echo "=== PROXY IP ==="
curl -x socks5h://127.0.0.1:2080 -4fsS https://ipinfo.io/json
echo
```

Ожидаемо:

```text
SYSTEM IP -> обычный IP сервера
PROXY IP  -> IP выбранной VPN-ноды
```

Проверить AI/dev-сервисы:

```bash
curl -I -x socks5h://127.0.0.1:2080 https://api.openai.com
curl -I -x socks5h://127.0.0.1:2080 https://api.anthropic.com
curl -I -x socks5h://127.0.0.1:2080 https://github.com
curl -I -x socks5h://127.0.0.1:2080 https://registry-1.docker.io/v2/
```

Нормальные ответы:

```text
OpenAI: 421 / 401 / 403 / другой HTTP-ответ — нормально
Anthropic: 404 — нормально
GitHub: 200 — нормально
Docker Registry: 401 — нормально, потому что без авторизации
```

Главное, чтобы не было:

```text
timeout
connection refused
could not resolve host
```

Вернуться в TUN:

```bash
sudo vpn-mode-tun
sleep 2
vpn-check-ip
```

---

## 15. Как добавлять домены в proxy/rules mode

Редактировать:

```bash
sudo nano /etc/sing-box/rules/proxy-domains.txt
```

Добавить домен, например:

```text
example.com
api.example.com
```

Применить:

```bash
sudo vpn-mode-proxy
```

или, если основной режим TUN:

```bash
sudo vpn-mode-tun
```

---

## 16. Обновление подписки

Ручное обновление:

```bash
sudo vpn-update
```

Что делает команда:

```text
1. делает backup /etc/sing-box/config.json
2. скачивает свежую подписку
3. если провайдер отдал 429 — использует старый raw.txt
4. декодирует base64
5. ищет выбранный NODE_NAME
6. пересобирает config.json
7. проверяет sing-box check
8. перезапускает sing-box, если он активен
```

Часто не дёргай `vpn-update`, иначе можно получить `429 Too Many Requests`.

---

## 17. Автообновление подписки через systemd timer

```bash
cat <<'EOF' | sudo tee /etc/systemd/system/sing-box-subscription-update.service >/dev/null
[Unit]
Description=Update sing-box subscription and regenerate config
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/vpn-update
EOF

cat <<'EOF' | sudo tee /etc/systemd/system/sing-box-subscription-update.timer >/dev/null
[Unit]
Description=Daily sing-box subscription update

[Timer]
OnCalendar=*-*-* 04:20:00
Persistent=true

[Install]
WantedBy=timers.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now sing-box-subscription-update.timer
```

Проверка:

```bash
systemctl list-timers | grep sing-box
```

---

## 18. Основные команды

```bash
sudo vpn-on          # включить VPN
sudo vpn-off         # выключить VPN
sudo vpn-restart     # перезапустить VPN
vpn-status           # статус
vpn-logs             # логи
sudo vpn-test        # проверить config.json
sudo vpn-update      # обновить подписку и пересобрать config
vpn-check-ip         # проверить внешний IP
sudo vpn-mode-tun    # весь трафик через VPN
sudo vpn-mode-proxy  # только домены из списка через VPN
```

---

## 19. Диагностика

### 19.1 Проверить сервис

```bash
systemctl status sing-box --no-pager --full
journalctl -u sing-box --output cat -e -n 120
```

### 19.2 Проверить конфиг

```bash
sudo sing-box check -c /etc/sing-box/config.json
```

Безопасный вывод ошибок:

```bash
sudo sing-box check -c /etc/sing-box/config.json 2>&1 \
  | sed -E 's/[0-9a-fA-F-]{20,}/<PRIVATE>/g' \
  | sed -E 's/([a-zA-Z0-9_-]+\.)+[a-zA-Z]{2,}/<DOMAIN>/g'
```

### 19.3 Проверить TUN

```bash
ip link show | grep -i tun || true
ip route | head -n 30
lsmod | grep tun || sudo modprobe tun
```

### 19.4 Проверить IP

```bash
vpn-check-ip
curl -4fsS https://ifconfig.me && echo
```

### 19.5 Проверить proxy-mode

```bash
sudo vpn-mode-proxy

curl -x socks5h://127.0.0.1:2080 -4fsS https://ipinfo.io/json
curl -I -x socks5h://127.0.0.1:2080 https://github.com
```

### 19.6 Проверить, есть ли выбранная нода

```bash
sudo cat /etc/sing-box/subscription/node-name.txt

sudo grep '^vless://' /etc/sing-box/subscription/decoded.txt \
  | sed 's/.*#//' \
  | python3 -c 'import sys, urllib.parse; [print(urllib.parse.unquote(line.strip())) for line in sys.stdin]' \
  | grep -F "$(sudo cat /etc/sing-box/subscription/node-name.txt)"
```

---

## 20. Фикс типовых ошибок

### Ошибка 1

```text
legacy DNS servers is deprecated
```

Причина: старый формат DNS.

Фикс: использовать новый DNS format, как в генераторе из этого гайда.

---

### Ошибка 2

```text
outbound DNS rule item is deprecated
```

Причина: старые DNS rules вида:

```json
{
  "outbound": "any",
  "server": "local"
}
```

Фикс: убрать этот блок или заменить на новый action-формат.

---

### Ошибка 3

```text
legacy inbound fields are deprecated / removed
```

Причина: старое поле внутри inbound:

```json
"sniff": true
```

Фикс: убрать `sniff` из inbound и добавить в route:

```json
{
  "action": "sniff"
}
```

---

### Ошибка 4

```text
missing route.default_domain_resolver
```

Причина: outbound использует доменное имя сервера, но resolver явно не задан.

Фикс в route:

```json
"default_domain_resolver": {
  "server": "local"
}
```

---

### Ошибка 5

```text
curl: (22) The requested URL returned error: 429
```

Причина: слишком часто скачивали subscription URL.

Фикс: текущий `vpn-update` уже умеет fallback:

```text
если свежая подписка не скачалась -> использовать старый raw.txt
```

---

### Ошибка 6

```text
bash: /usr/local/bin/vpn-status: Permission denied
```

Причина: `/usr/local/sbin/vpn-core` был `700`.

Фикс:

```bash
sudo chmod 755 /usr/local/sbin/vpn-core
sudo chown root:root /usr/local/sbin/vpn-core
```

---

### Ошибка 7

Proxy-mode показывает обычный IP.

Причина: домен проверки не входит в proxy-domains.

Фикс:

```bash
sudo nano /etc/sing-box/rules/proxy-domains.txt
```

Добавить:

```text
ipinfo.io
ifconfig.me
ifconfig.co
```

Применить:

```bash
sudo vpn-mode-proxy
```

---

### Ошибка 8

Docker Registry отдаёт `401`.

Это нормально:

```text
HTTP/2 401
www-authenticate: Bearer realm=...
```

Это значит, что Docker Registry доступен, но запрос без авторизации.

---

### Ошибка 9

OpenAI отдаёт `421`.

Для проверки `curl -I https://api.openai.com` это не критично. Главное, что есть HTTP-ответ от Cloudflare/OpenAI, а не timeout.

---

## 21. Откат к backup

Показать backup:

```bash
sudo ls -lah /etc/sing-box/backups/
```

Откатиться на последний backup:

```bash
latest="$(sudo ls -1t /etc/sing-box/backups/config.*.json | head -n 1)"

sudo cp -a "$latest" /etc/sing-box/config.json
sudo chmod 600 /etc/sing-box/config.json
sudo sing-box check -c /etc/sing-box/config.json
sudo systemctl restart sing-box
```

---

## 22. Финальная проверка

```bash
sudo vpn-mode-tun
sleep 2

systemctl is-enabled sing-box
vpn-status
ip link show | grep -i tun
vpn-check-ip
sudo vpn-test
```

Ожидаемый финал:

```text
enabled
Active: active (running)
tun0 есть
IP: страна выбранной VPN-ноды
OK: config валидный
```

После этого сервер готов:

```text
TUN mode работает
proxy/rules mode работает
выбранная нода зафиксирована
vpn-update обновляет подписку
backup создаётся автоматически
автозапуск включён
```
