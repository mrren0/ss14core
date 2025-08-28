pipeline {
  agent any
  options { timestamps(); ansiColor('xterm'); durabilityHint('PERFORMANCE_OPTIMIZED') }

  parameters {
    string(name: 'BRANCH', defaultValue: 'master', description: 'Ветка деплоя')
    string(name: 'REPO', defaultValue: 'https://github.com/thunder-ss14/corporate-war.git', description: 'Git repo (https или ssh)')
    string(name: 'SERVER_IP', defaultValue: '5.83.140.23', description: 'IP сервера = ID SSH credentials')
    string(name: 'PORT', defaultValue: '1212', description: 'Порт сервера')

    // server_config.toml
    string(name: 'SERVER_NAME', defaultValue: 'TRAIN TDM 3000 TICKETS NO RULES 24/7', description: 'Имя сервера')
    string(name: 'SERVER_DESC', defaultValue: 'DEATH MATCH', description: 'Описание (можно пустое)')
    string(name: 'SERVER_DOMAIN', defaultValue: 'total-space.online', description: 'Домен (для server_url)')
    string(name: 'TICKRATE', defaultValue: '60', description: '[net] tickrate')
    booleanParam(name: 'LOBBYENABLED', defaultValue: true, description: '[game] lobbyenabled')
    string(name: 'AUTH_MODE', defaultValue: '1', description: '[auth] mode')
    string(name: 'HUB_TAGS', defaultValue: 'totalspace,economy', description: '[hub] tags')

    string(name: 'MAX_PLAYERS', defaultValue: '64', description: '[game] maxplayers')
    string(name: 'SOFT_MAX_PLAYERS', defaultValue: '64', description: '[game] soft_max_players (можно = MAX_PLAYERS)')

    choice(name: 'DB_ENGINE', choices: ['sqlite','postgres'], description: '[database] engine')
    string(name: 'PG_HOST', defaultValue: 'localhost', description: 'Postgres host')
    string(name: 'PG_PORT', defaultValue: '5432', description: 'Postgres port')
    string(name: 'PG_DB',   defaultValue: 'ss14', description: 'Postgres database')
    string(name: 'PG_USER', defaultValue: '', description: 'Postgres user')
    string(name: 'PG_PASS', defaultValue: '', description: 'Postgres password (или оставь пустым)')

    booleanParam(name: 'FORCE_CONFIG', defaultValue: true, description: 'Перезаписывать server_config.toml при деплое')
  }

  environment {
    DOTNET_CLI_TELEMETRY_OPTOUT = '1'
    GIT_TERMINAL_PROMPT = '0'
  }

  stages {
    stage('Checkout + Submodules') {
      steps {
        withCredentials([string(credentialsId: 'token-github', variable: 'GITHUB_TOKEN')]) {
          sh '''#!/usr/bin/env bash
set -euo pipefail
rm -rf src
cat > askpass.sh <<'EOS'
#!/bin/sh
case "$1" in
  *Username*) echo "x-access-token" ;;
  *Password*) echo "$GITHUB_TOKEN" ;;
esac
EOS
chmod +x askpass.sh
GIT_ASKPASS="$PWD/askpass.sh" GIT_ASKPASS_REQUIRE=force git clone --depth 1 -b "$BRANCH" "$REPO" src
GIT_ASKPASS="$PWD/askpass.sh" GIT_ASKPASS_REQUIRE=force git -C src -c url.https://github.com/.insteadof=git@github.com: submodule update --init --recursive
git -C src config --local safe.directory "$(pwd)/src" || true
'''
        }
      }
    }

    stage('.NET SDK (local)') {
      steps {
        sh '''#!/usr/bin/env bash
set -euo pipefail
mkdir -p .dotnet
curl -fsSL --retry 8 --retry-all-errors --retry-delay 2 -o dotnet-install.sh https://dot.net/v1/dotnet-install.sh
bash dotnet-install.sh --install-dir "$PWD/.dotnet" --channel 9.0
export PATH="$PWD/.dotnet:$PATH"
dotnet --info
'''
      }
    }

    stage('Build & Package') {
      steps {
        sh '''#!/usr/bin/env bash
set -Eeuo pipefail
export PATH="$PWD/.dotnet:$PATH"
cd src
( while true; do echo "[keepalive] $(date -Iseconds) build alive"; sleep 55; done ) & KA=$!
trap 'kill $KA 2>/dev/null || true' EXIT
stdbuf -oL -eL dotnet build Content.Packaging --configuration Release -v minimal
stdbuf -oL -eL dotnet run --project Content.Packaging server --hybrid-acz --platform linux-x64
'''
      }
    }

    stage('Archive artifact') {
      steps {
        sh '''#!/usr/bin/env bash
set -euo pipefail
rm -rf artifact
mkdir -p artifact
OUTDIR="$(find src/release -maxdepth 4 -type f -name Robust.Server -printf '%h\\n' -quit || true)"
if [ -n "$OUTDIR" ]; then
  rsync -a --delete --exclude=".git" "$OUTDIR"/ artifact/
else
  ZIP_FILE="$(ls -1 src/release/SS14.Server*_linux-x64.zip src/release/SS14.Server*.zip 2>/dev/null | head -n1 || true)"
  if [ -z "$ZIP_FILE" ]; then
    echo "❌ Нет сборки в src/release"
    ls -la src/release || true
    exit 1
  fi
  unzip -o "$ZIP_FILE" -d artifact/
  [ -f artifact/Robust.Server ] && chmod +x artifact/Robust.Server || true
fi
tar -C artifact -czf "ss14-server-${BRANCH}.tar.gz" .
'''
        archiveArtifacts artifacts: "ss14-server-${BRANCH}.tar.gz", fingerprint: true
      }
    }

    stage('Deploy via SSH') {
      steps {
        sshagent (credentials: [params.SERVER_IP]) {
          sh '''#!/usr/bin/env bash
set -Eeuo pipefail

SSH_OPTS="-o StrictHostKeyChecking=no -o ServerAliveInterval=30 -o ServerAliveCountMax=120"
esc() { printf '%s' "$1" | sed -e 's/\\\\/\\\\\\\\/g' -e 's/"/\\\\\\"/g'; }

repo_path="$(printf '%s' "${REPO}" | sed -E 's#(git@github.com:|https://github.com/)([^/]+/[^/.]+)(\\.git)?#\\2#')"
owner="${repo_path%%/*}"; repo="${repo_path##*/}"
safe_branch="$(printf '%s' "${BRANCH}" | tr '/ ' '_' | tr -cd 'A-Za-z0-9._-')"

DEST="/opt/${owner}/${repo}/${safe_branch}"
PORT="${PORT}"

ssh $SSH_OPTS "root@${SERVER_IP}" "mkdir -p \"${DEST}\""

# бинарники
rsync -a --delete \
  --exclude 'server_config.toml' \
  --exclude 'data/' \
  -e "ssh $SSH_OPTS" artifact/ "root@${SERVER_IP}:${DEST}/"

# firewall (только PORT)
if ssh $SSH_OPTS "root@${SERVER_IP}" "command -v ufw >/dev/null 2>&1"; then
  ssh $SSH_OPTS "root@${SERVER_IP}" "ufw allow ${PORT}/tcp || true; ufw allow ${PORT}/udp || true"
elif ssh $SSH_OPTS "root@${SERVER_IP}" "command -v firewall-cmd >/dev/null 2>&1"; then
  ssh $SSH_OPTS "root@${SERVER_IP}" "firewall-cmd --permanent --add-port=${PORT}/tcp || true; firewall-cmd --permanent --add-port=${PORT}/udp || true; firewall-cmd --reload || true"
else
  ssh $SSH_OPTS "root@${SERVER_IP}" "iptables -I INPUT -p tcp --dport ${PORT} -j ACCEPT || true; iptables -I INPUT -p udp --dport ${PORT} -j ACCEPT || true"
fi

# server_config.toml (порт и статус-порт в одном числе)
tmpdir="$(mktemp -d)"; cfg="$tmpdir/server_config.toml"
SERVER_NAME_E="$(esc "$SERVER_NAME")"; SERVER_DESC_E="$(esc "$SERVER_DESC")"; HUB_TAGS_E="$(esc "$HUB_TAGS")"

cat >"$cfg" <<EOF
[net]
tickrate = ${TICKRATE}
port = ${PORT}

[game]
hostname = "${SERVER_NAME_E}"
description = "${SERVER_DESC_E}"
lobbyenabled = ${LOBBYENABLED}
maxplayers = ${MAX_PLAYERS}
soft_max_players = ${SOFT_MAX_PLAYERS}

[auth]
mode = ${AUTH_MODE}

[hub]
advertise = true
tags = "${HUB_TAGS_E}"
desc = "${SERVER_DESC_E}"
EOF

[ -n "${SERVER_DOMAIN:-}" ] && echo "server_url = \"ss14://${SERVER_DOMAIN}:${PORT}\"" >> "$cfg"

# статус на том же порту
cat >>"$cfg" <<EOF
[status]
bind = "*:${PORT}"
EOF

# БД
if [ "${DB_ENGINE}" = "postgres" ]; then
  PG_HOST_E="$(esc "$PG_HOST")"; PG_DB_E="$(esc "$PG_DB")"; PG_USER_E="$(esc "$PG_USER")"; PG_PASS_E="$(esc "$PG_PASS")"
  cat >>"$cfg" <<EOF
[database]
engine = "postgres"
pg_host = "${PG_HOST_E}"
pg_port = ${PG_PORT}
pg_database = "${PG_DB_E}"
pg_username = "${PG_USER_E}"
pg_password = "${PG_PASS_E}"
# SQLite (шаблон, отключён):
# engine = "sqlite"
# sqlite_dbpath = "preferences.db"
EOF
else
  cat >>"$cfg" <<'EOF'
[database]
engine = "sqlite"
sqlite_dbpath = "preferences.db"
# Postgres (шаблон, отключён):
# engine = "postgres"
# pg_host = "localhost"
# pg_port = 5432
# pg_database = "ss14"
# pg_username = "user"
# pg_password = "pass"
EOF
fi

# запись конфига
if [ "${FORCE_CONFIG}" = "true" ]; then
  scp $SSH_OPTS "$cfg" "root@${SERVER_IP}:${DEST}/server_config.toml"
else
  if ! ssh $SSH_OPTS "root@${SERVER_IP}" "test -f ${DEST}/server_config.toml"; then
    scp $SSH_OPTS "$cfg" "root@${SERVER_IP}:${DEST}/server_config.toml"
  fi
fi

# systemd unit БЕЗ --cvar BasePort, читаем только конфиг
UNIT="ss14-${safe_branch}.service"
cat >"$tmpdir/${UNIT}" <<EOF
[Unit]
Description=SS14 ${safe_branch} server
After=network-online.target
Wants=network-online.target

[Service]
WorkingDirectory=${DEST}
ExecStart=${DEST}/Robust.Server --cfg ${DEST}/server_config.toml
Restart=always
RestartSec=3
Environment=DOTNET_TieredPGO=1
Environment=DOTNET_TC_QuickJitForLoops=1
Environment=ROBUST_NUMERICS_AVX=true

[Install]
WantedBy=multi-user.target
EOF

scp $SSH_OPTS "$tmpdir/${UNIT}" "root@${SERVER_IP}:/etc/systemd/system/${UNIT}"
ssh $SSH_OPTS "root@${SERVER_IP}" "systemctl daemon-reload && systemctl enable ${UNIT} || true && systemctl restart ${UNIT}"

# проверка: слушается ли нужный порт, иначе вывод лога и ошибка
ssh $SSH_OPTS "root@${SERVER_IP}" "sleep 2; ss -lntup | grep -q ':${PORT}\\b' || { journalctl -u ${UNIT} -n 200 --no-pager; exit 1; } && systemctl status ${UNIT} --no-pager || true"
'''
        }
      }
    }
  }

  post {
    success { echo '✅ Deploy completed.' }
    failure { echo '❌ Deploy failed.' }
  }
}
