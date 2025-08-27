pipeline {
  agent any

  options { timestamps(); ansiColor('xterm') }

  parameters {
    // Обязательные для запуска
    string(name: 'SERVER_IP', defaultValue: '5.83.140.23', description: 'IP сервера = ID SSH credentials')
    string(name: 'PORT',      defaultValue: '1212',       description: 'Порт сервера')

    // Рекомендованные по умолчанию
    string(name: 'DEST_NAME',    defaultValue: 'prod',  description: 'Подкаталог деплоя: prod/dev/…')
    string(name: 'BRANCH',       defaultValue: 'master', description: 'Ветка для билда')
    string(name: 'REPO',         defaultValue: 'https://github.com/Total-War/total-prototypes.git', description: 'Git repo')
    string(name: 'DOTNET_CHANNEL', defaultValue: '9.0', description: '.NET channel')
    string(name: 'DEST_BASE',    defaultValue: '/opt/totalspace', description: 'База каталогов на сервере')

    // server_config.toml
    string(name: 'SERVER_NAME',   defaultValue: 'Total Space - economic war', description: 'hostname в конфиге')
    string(name: 'SERVER_DOMAIN', defaultValue: 'total-space.online',          description: 'для server_url (если нужно)')
    string(name: 'TICKRATE',      defaultValue: '30',  description: '[net] tickrate')
    booleanParam(name: 'LOBBYENABLED', defaultValue: true, description: '[game] lobbyenabled')
    string(name: 'AUTH_MODE',     defaultValue: '1',   description: '[auth] mode')
    booleanParam(name: 'ADVERTISE', defaultValue: true, description: '[hub] advertise')
    string(name: 'HUB_TAGS',      defaultValue: 'totalspace,${DEST_NAME},economy', description: '[hub] tags')
    booleanParam(name: 'WRITE_CFG_IF_MISSING', defaultValue: true, description: 'Создавать server_config.toml если отсутствует')
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
curl -fsSL https://dot.net/v1/dotnet-install.sh -o dotnet-install.sh
bash dotnet-install.sh --install-dir "$PWD/.dotnet" --channel "$DOTNET_CHANNEL"
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
rm -rf artifact; mkdir -p artifact
OUTDIR="$(find src/release -maxdepth 4 -type f -name Robust.Server -printf '%h\\n' -quit || true)"
if [ -n "$OUTDIR" ]; then
  rsync -a --delete --exclude=".git" "$OUTDIR"/ artifact/
else
  ZIP_FILE="$(ls -1 src/release/SS14.Server*_linux-x64.zip src/release/SS14.Server*.zip 2>/dev/null | head -n1 || true)"
  [ -z "$ZIP_FILE" ] && { echo "No build in src/release"; ls -la src/release || true; exit 1; }
  if command -v unzip >/dev/null 2>&1; then unzip -o "$ZIP_FILE" -d artifact/; else
    python3 - <<'PY'
import sys, zipfile; from pathlib import Path
z=Path(sys.argv[1]); d=Path(sys.argv[2]); d.mkdir(parents=True, exist_ok=True)
with zipfile.ZipFile(z,'r') as zf: zf.extractall(d)
PY
    "$ZIP_FILE" "artifact/"
  fi
  [ -f artifact/Robust.Server ] && chmod +x artifact/Robust.Server || true
fi
tar -C artifact -czf "ss14-server-${DEST_NAME}.tar.gz" .
'''
        archiveArtifacts artifacts: "ss14-server-${DEST_NAME}.tar.gz", fingerprint: true
      }
    }

    stage('Deploy via SSH') {
      steps {
        sshagent (credentials: [params.SERVER_IP]) { // ID кредов = сам IP
          sh '''#!/usr/bin/env bash
set -euo pipefail

DEST="${DEST_BASE}/${DEST_NAME}"
PORT="${PORT}"

ssh -o StrictHostKeyChecking=no "root@${SERVER_IP}" "mkdir -p ${DEST}"

# Бинарники без конфигов
rsync -a --delete --exclude 'server_config.toml' --exclude 'data/' \
  -e "ssh -o StrictHostKeyChecking=no" artifact/ "root@${SERVER_IP}:${DEST}/"

tmpdir="$(mktemp -d)"; cfg="$tmpdir/server_config.toml"
cat >"$cfg" <<EOF
[net]
tickrate = ${TICKRATE}
port = ${PORT}

[game]
hostname = "${SERVER_NAME} (${DEST_NAME})"
lobbyenabled = ${LOBBYENABLED}

[auth]
mode = ${AUTH_MODE}

[hub]
advertise = ${ADVERTISE}
tags = "${HUB_TAGS}"
server_url = "ss14://${SERVER_DOMAIN}:${PORT}"
EOF

if ${WRITE_CFG_IF_MISSING}; then
  if ! ssh -o StrictHostKeyChecking=no "root@${SERVER_IP}" "test -f ${DEST}/server_config.toml"; then
    scp -o StrictHostKeyChecking=no "$cfg" "root@${SERVER_IP}:${DEST}/server_config.toml"
  fi
fi

UNIT="ss14-${DEST_NAME}.service"
cat >"$tmpdir/${UNIT}" <<EOF
[Unit]
Description=SS14 ${DEST_NAME} server
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
