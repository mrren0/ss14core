pipeline {
  agent any

  options {
    timestamps()
    ansiColor('xterm')
  }

  parameters {
    choice(name: 'ENV', choices: ['prod', 'dev'], description: 'Куда деплоим')
    string(name: 'BRANCH', defaultValue: 'master', description: 'Ветка: master→prod, dev→dev')
  }

  environment {
    REPO = 'https://github.com/Total-War/total-prototypes.git'
    DEST_BASE = '/opt/totalspace'
    PROD_PORT = '1212'
    DEV_PORT  = '1213'
    SERVER_IP = '5.83.140.23'   // статично
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

# askpass: username = x-access-token, password = PAT
cat > askpass.sh <<'EOS'
#!/bin/sh
case "$1" in
  *Username*) echo "x-access-token" ;;
  *Password*) echo "$GITHUB_TOKEN" ;;
esac
EOS
chmod +x askpass.sh

# Клонирование через HTTPS без подсветки токена в URL
GIT_ASKPASS="$PWD/askpass.sh" GIT_ASKPASS_REQUIRE=force \
  git clone --depth 1 -b "$BRANCH" "$REPO" src

# Сабмодули: временно переписываем ssh→https для github
GIT_ASKPASS="$PWD/askpass.sh" GIT_ASKPASS_REQUIRE=force \
  git -C src -c url.https://github.com/.insteadof=git@github.com: \
  submodule update --init --recursive

# На всякий случай отмечаем как safe (иногда нужно в Jenkins)
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
curl -fsSL https://dot.net/v1/dotnet-install.sh -o dotnet-install.sh
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

# 1) Пробуем найти распакованный билд
OUTDIR="$(find src/release -maxdepth 4 -type f -name Robust.Server -printf '%h\\n' -quit || true)"
if [ -n "$OUTDIR" ]; then
  echo "✔ Найден распакованный билд: $OUTDIR"
  rsync -a --delete --exclude=".git" "$OUTDIR"/ artifact/
else
  # 2) Иначе ищем ZIP и распаковываем
  ZIP_FILE="$(ls -1 src/release/SS14.Server*_linux-x64.zip src/release/SS14.Server*.zip 2>/dev/null | head -n1 || true)"
  if [ -z "$ZIP_FILE" ]; then
    echo "❌ Не найден ни распакованный билд, ни ZIP в src/release/"
    echo "Содержимое src/release:"
    ls -la src/release || true
    exit 1
  fi
  echo "✔ Найден ZIP: $ZIP_FILE"

  if command -v unzip >/dev/null 2>&1; then
    unzip -o "$ZIP_FILE" -d artifact/
  else
    python3 - <<'PY'
import sys, zipfile
from pathlib import Path
z = Path(sys.argv[1]); d = Path(sys.argv[2])
d.mkdir(parents=True, exist_ok=True)
with zipfile.ZipFile(z, 'r') as zf: zf.extractall(d)
print("Extracted", z, "->", d)
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
        sshagent (credentials: ['5.83.140.23']) {
          sh '''#!/usr/bin/env bash
set -euo pipefail

DEST="${DEST_BASE}/${ENV}"
if [ "$ENV" = "prod" ]; then
  PORT="$PROD_PORT"; ADVERTISE=true
else
  PORT="$DEV_PORT"; ADVERTISE=false
fi

# 1) Подготовка на сервере
ssh -o StrictHostKeyChecking=no "root@${SERVER_IP}" "mkdir -p ${DEST_BASE}/prod ${DEST_BASE}/dev"

# 2) Заливка бинарей (конфиг и data/ не трогаем)
rsync -a --delete \
  --exclude 'server_config.toml' \
  --exclude 'data/' \
  -e "ssh -o StrictHostKeyChecking=no" artifact/ "root@${SERVER_IP}:${DEST}/"

# 3) Локально собрать server_config.toml и отправить ТОЛЬКО если его нет
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

# 4) systemd unit локально → сервер
UNIT="ss14-${ENV}.service"
cat >"$tmpdir/${UNIT}" <<EOF
[Unit]
Description=SS14 ${ENV} server
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
