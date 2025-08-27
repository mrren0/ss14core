pipeline {
  agent any

  options { timestamps(); ansiColor('xterm') }

  parameters {
    choice(name: 'ENV', choices: ['prod', 'dev'], description: 'Куда деплоим')
    string(name: 'BRANCH', defaultValue: 'master', description: 'Ветка: master→prod, dev→dev')

    // новое
    string(name: 'SERVER_IP', defaultValue: '5.83.140.23', description: 'IP сервера = ID SSH credentials')
    string(name: 'PORT', defaultValue: '', description: 'Явный порт. Если пусто — берётся PROD/DEV порт по ENV')
  }

  environment {
    REPO  = 'https://github.com/Total-War/total-prototypes.git'
    // DEST_BASE удалён. Путь формируем из REPO/BRANCH
    PROD_PORT = '1212'
    DEV_PORT  = '1213'
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
# устойчивее к сбоям CDN
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
set -euo pipefail
export PATH="$PWD/.dotnet:$PATH"
cd src
dotnet build Content.Packaging --configuration Release
dotnet run --project Content.Packaging server --hybrid-acz --platform linux-x64
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
  if command -v unzip >/dev/null 2>&1; then
    unzip -o "$ZIP_FILE" -d artifact/
  else
    python3 - <<'PY'
import sys, zipfile
from pathlib import Path
z=Path(sys.argv[1]); d=Path(sys.argv[2]); d.mkdir(parents=True, exist_ok=True)
with zipfile.ZipFile(z,'r') as zf: zf.extractall(d)
PY
    "$ZIP_FILE" "artifact/"
  fi
  [ -f artifact/Robust.Server ] && chmod +x artifact/Robust.Server || true
fi
tar -C artifact -czf "ss14-server-${ENV}.tar.gz" .
'''
        archiveArtifacts artifacts: "ss14-server-${ENV}.tar.gz", fingerprint: true
      }
    }

    stage('Deploy via SSH') {
      steps {
        script {
          // порт: явный > из ENV
          env.CHOSEN_PORT = params.PORT?.trim() ? params.PORT.trim() : (params.ENV == 'prod' ? env.PROD_PORT : env.DEV_PORT)
          env.ADVERTISE   = (params.ENV == 'prod') ? 'true' : 'false'
        }

        // ID кредов == SERVER_IP
        sshagent (credentials: [params.SERVER_IP]) {
          sh '''#!/usr/bin/env bash
set -euo pipefail

# --- Разбор REPO: github.com/<owner>/<repo>(.git) ---
repo_path="$(printf '%s' "${REPO}" \
  | sed -E 's#(git@github.com:|https://github.com/)([^/]+/[^/.]+)(\\.git)?#\\2#')"
owner="${repo_path%%/*}"
repo="${repo_path##*/}"

# --- Безопасное имя ветки для пути и имени юнита ---
safe_branch="$(printf '%s' "${BRANCH}" | tr '/ ' '_' | tr -cd 'A-Za-z0-9._-')"

DEST="/opt/${owner}/${repo}/${safe_branch}"
PORT="${CHOSEN_PORT}"
ADVERTISE="${ADVERTISE}"

# 1) Подготовка директории на сервере
ssh -o StrictHostKeyChecking=no "root@${SERVER_IP}" "mkdir -p \"${DEST}\""

# 2) Заливка бинарей (конфиг и data/ не трогаем)
rsync -a --delete \
  --exclude 'server_config.toml' \
  --exclude 'data/' \
  -e "ssh -o StrictHostKeyChecking=no" artifact/ "root@${SERVER_IP}:${DEST}/"

# 3) Открыть порт в фаерволе (ufw / firewalld / iptables)
if ssh -o StrictHostKeyChecking=no "root@${SERVER_IP}" "command -v ufw >/dev/null 2>&1"; then
  ssh -o StrictHostKeyChecking=no "root@${SERVER_IP}" "ufw allow ${PORT}/tcp || true; ufw allow ${PORT}/udp || true"
elif ssh -o StrictHostKeyChecking=no "root@${SERVER_IP}" "command -v firewall-cmd >/dev/null 2>&1"; then
  ssh -o StrictHostKeyChecking=no "root@${SERVER_IP}" "firewall-cmd --permanent --add-port=${PORT}/tcp || true; firewall-cmd --permanent --add-port=${PORT}/udp || true; firewall-cmd --reload || true"
else
  ssh -o StrictHostKeyChecking=no "root@${SERVER_IP}" "iptables -I INPUT -p tcp --dport ${PORT} -j ACCEPT || true; iptables -I INPUT -p udp --dport ${PORT} -j ACCEPT || true"
fi

# 4) Сформировать server_config.toml (если его нет)
tmpdir="$(mktemp -d)"
cat >"$tmpdir/server_config.toml" <<EOF
[net]
tickrate = 30
port = ${PORT}

[game]
hostname = "Total Space - economic war (${ENV})"
lobbyenabled = true

[auth]
mode = 1

[hub]
advertise = ${ADVERTISE}
tags = "totalspace,${ENV},economy"
EOF
if [ "${ENV}" = "prod" ]; then
  echo "server_url = \\"ss14://total-space.online:${PORT}\\"" >> "$tmpdir/server_config.toml"
fi

if ! ssh -o StrictHostKeyChecking=no "root@${SERVER_IP}" "test -f ${DEST}/server_config.toml"; then
  scp -o StrictHostKeyChecking=no "$tmpdir/server_config.toml" "root@${SERVER_IP}:${DEST}/server_config.toml"
fi

# 5) systemd unit → сервер
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

scp -o StrictHostKeyChecking=no "$tmpdir/${UNIT}" "root@${SERVER_IP}:/etc/systemd/system/${UNIT}"
ssh -o StrictHostKeyChecking=no "root@${SERVER_IP}" "systemctl daemon-reload && systemctl enable ${UNIT} || true && systemctl restart ${UNIT} && systemctl status --no-pager ${UNIT} || true"
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
