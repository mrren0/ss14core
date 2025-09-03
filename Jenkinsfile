pipeline {
	agent any
  options { timestamps(); ansiColor('xterm'); durabilityHint('PERFORMANCE_OPTIMIZED'); timeout(time: 90, unit: 'MINUTES') }

  parameters {
                string(name: 'BRANCH', defaultValue: 'master', description: 'Ветка деплоя')
    string(name: 'REPO', defaultValue: 'https://github.com/thunder-ss14/corporate-war.git', description: 'Git repo (https или ssh)')
    string(name: 'SERVER_IP', defaultValue: '88.99.104.199', description: 'IP адрес сервера')
    string(name: 'SSH_CREDENTIALS_ID', defaultValue: '88.99.104.199', description: 'ID SSH credentials в Jenkins')
    string(name: 'PORT', defaultValue: '1212', description: 'Порт сервера')
    string(name: 'CONFIG_REPO', defaultValue: 'https://github.com/mrren0/ss14core.git', description: 'Опциональный git репозиторий с server_config.toml')
    string(name: 'CONFIG_PATH', defaultValue: 'configs/server_config.toml', description: 'Путь к server_config.toml в CONFIG_REPO')

    credentials(name: 'SSH_CRED',
      description: 'ID=host (IP/DNS), Username+Key',
      defaultValue: '88.99.104.199',
      credentialType: 'com.cloudbees.jenkins.plugins.sshcredentials.impl.BasicSSHUserPrivateKey')
  }

  environment {
		DOTNET_CLI_TELEMETRY_OPTOUT = '1'
    GIT_TERMINAL_PROMPT = '0'
    MSBUILDDISABLENODEREUSE = '1'
    DOTNET_CLI_HOME = "${WORKSPACE}/.dotnet_home"
    MSBUILDDEBUGPATH = "${WORKSPACE}/msbuild-logs"
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
mkdir -p .dotnet "$DOTNET_CLI_HOME" "$MSBUILDDEBUGPATH"
curl -fsSL --retry 8 --retry-all-errors --retry-delay 2 -o dotnet-install.sh https://dot.net/v1/dotnet-install.sh
bash dotnet-install.sh --install-dir "$PWD/.dotnet" --channel 9.0
export PATH="$PWD/.dotnet:$PATH"
dotnet --info
'''
      }
    }

    stage('Restore (retry)') {
			steps {
				sh '''#!/usr/bin/env bash
set -Eeuo pipefail
export PATH="$PWD/.dotnet:$PATH"
cd src
dotnet nuget locals all --clear || true
for i in 1 2 3; do
  stdbuf -oL -eL dotnet restore --no-cache && s=0 && break || s=$?
  echo "restore retry $i failed with $s"; sleep $((5*i))
done
[ "${s:-0}" -eq 0 ]
'''
      }
    }

    stage('Build & Package') {
			steps {
				retry(2) {
					sh '''#!/usr/bin/env bash
set -Eeuo pipefail
export PATH="$PWD/.dotnet:$PATH"
cd src
( while true; do echo "[keepalive] $(date -Iseconds) build alive"; sleep 55; done ) & KA=$!
trap 'kill $KA 2>/dev/null || true; dotnet build-server shutdown || true' EXIT
stdbuf -oL -eL dotnet build Content.Packaging --configuration Release -v minimal -m:1 -p:UseSharedCompilation=false
stdbuf -oL -eL dotnet run --project Content.Packaging server --hybrid-acz --platform linux-x64
'''
        }
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

    stage('Ensure .NET 9 runtime on target') {
			steps {
        withEnv(["HOST_HOST=${params.SSH_CRED}"]) {
  				withCredentials([sshUserPrivateKey(credentialsId: params.SSH_CREDENTIALS_ID, keyFileVariable: 'SSH_KEY', usernameVariable: 'SSH_USER')]) {
  					sh '''#!/usr/bin/env bash
set -Eeuo pipefail
SSH_OPTS="-o StrictHostKeyChecking=no -o ServerAliveInterval=30 -o ServerAliveCountMax=120 -i \"$SSH_KEY\""
HOST="$SSH_USER@$HOST_HOST"

if ssh $SSH_OPTS "$HOST" "dotnet --list-runtimes 2>/dev/null | grep -q '^Microsoft.NETCore.App 9\\.'"; then
  echo "dotnet 9 runtime: OK"
else
  echo "dotnet 9 runtime: INSTALL"
  ssh $SSH_OPTS "$HOST" /bin/bash -lc '
    set -Eeuo pipefail
    if ! dpkg -s packages-microsoft-prod >/dev/null 2>&1; then
      . /etc/os-release
      UBU="${VERSION_ID:-22.04}"
      URL="https://packages.microsoft.com/config/ubuntu/${UBU}/packages-microsoft-prod.deb"
      curl -fsSL "$URL" -o /tmp/ms.deb || curl -fsSL "https://packages.microsoft.com/config/ubuntu/22.04/packages-microsoft-prod.deb" -o /tmp/ms.deb
      sudo dpkg -i /tmp/ms.deb
      rm -f /tmp/ms.deb
    fi
    sudo apt-get update -y
    sudo apt-get install -y dotnet-runtime-9.0
  '
  ssh $SSH_OPTS "$HOST" "dotnet --list-runtimes | grep -q '^Microsoft.NETCore.App 9\\.'"
  echo "dotnet 9 runtime: INSTALLED"
fi
'''
          }
        }
      }
    }

    stage('Deploy via SSH') {
			steps {
        withEnv(["HOST_HOST=${params.SSH_CRED}"]) {
  				withCredentials([sshUserPrivateKey(credentialsId: params.SSH_CREDENTIALS_ID, keyFileVariable: 'SSH_KEY', usernameVariable: 'SSH_USER')]) {
  					sh '''#!/usr/bin/env bash
set -Eeuo pipefail
  SSH_OPTS="-o StrictHostKeyChecking=no -o ServerAliveInterval=30 -o ServerAliveCountMax=120 -i \"$SSH_KEY\""
  HOST="$SSH_USER@$HOST_HOST"

repo_path="$(printf '%s' "${REPO}" | sed -E 's#(git@github.com:|https://github.com/)([^/]+/[^/.]+)(\\.git)?#\\2#')"
owner="${repo_path%%/*}"; repo="${repo_path##*/}"
safe_branch="$(printf '%s' "${BRANCH}" | tr '/ ' '_' | tr -cd 'A-Za-z0-9._-')"
DEST="/opt/${owner}/${repo}/${safe_branch}"
UNIT="ss14-${safe_branch}.service"

ssh $SSH_OPTS "$HOST" "sudo mkdir -p '${DEST}/logs' && sudo chown -R ${SSH_USER}:${SSH_USER} '${DEST}'"

# firewall
if ssh $SSH_OPTS "$HOST" "command -v ufw >/dev/null 2>&1"; then
  ssh $SSH_OPTS "$HOST" "sudo ufw allow ${PORT}/tcp || true; sudo ufw allow ${PORT}/udp || true"
elif ssh $SSH_OPTS "$HOST" "command -v firewall-cmd >/dev/null 2>&1"; then
  ssh $SSH_OPTS "$HOST" "sudo firewall-cmd --permanent --add-port=${PORT}/tcp || true; sudo firewall-cmd --permanent --add-port=${PORT}/udp || true; sudo firewall-cmd --reload || true"
else
  ssh $SSH_OPTS "$HOST" "sudo iptables -I INPUT -p tcp --dport ${PORT} -j ACCEPT || true; sudo iptables -I INPUT -p udp --dport ${PORT} -j ACCEPT || true"
fi

  # upload binaries
  rsync -a --delete --exclude 'server_config.toml' --exclude 'data/' -e "ssh $SSH_OPTS" artifact/ "$HOST:${DEST}/"

  # upload server_config
  if [ -n "$CONFIG_REPO" ]; then
    rm -rf cfgrepo
    git clone --depth 1 "$CONFIG_REPO" cfgrepo
    scp $SSH_OPTS "cfgrepo/$CONFIG_PATH" "$HOST:${DEST}/server_config.toml"
  else
    scp $SSH_OPTS configs/server_config.toml "$HOST:${DEST}/server_config.toml"
  fi

  # systemd unit
  tmpdir="$(mktemp -d)"
  unit_local="$tmpdir/${UNIT}"
cat >"$unit_local" <<EOF
[Unit]
Description=SS14 ${safe_branch} server
After=network-online.target
Wants=network-online.target

[Service]
User=${SSH_USER}
Group=${SSH_USER}
WorkingDirectory=${DEST}
ExecStart=${DEST}/Robust.Server
Restart=always
RestartSec=3
Environment=DOTNET_TieredPGO=1
Environment=DOTNET_TC_QuickJitForLoops=1
Environment=ROBUST_NUMERICS_AVX=true

[Install]
WantedBy=multi-user.target
EOF

scp $SSH_OPTS "$unit_local" "$HOST:/tmp/${UNIT}"
ssh $SSH_OPTS "$HOST" "sudo mv /tmp/${UNIT} /etc/systemd/system/${UNIT} && sudo systemctl daemon-reload && sudo systemctl enable ${UNIT} || true && sudo systemctl restart ${UNIT}"

# check
ssh $SSH_OPTS "$HOST" /bin/bash -lc '
  set -e
  sleep 30
  # вывод подключений
  echo "connect_ip: ss14://'"${SERVER_IP}"':'"${PORT}"'"

  # дать серверу ожить


  # сокет поднят?
  for i in {1..20}; do ss -lntup | grep -q ":'"${PORT}"'\\b" && ok=1 && break || sleep 1; done
  [ "${ok:-}" = "1" ] || { sudo journalctl -u '"${UNIT}"' -n 200 --no-pager; exit 1; }

  which jq >/dev/null 2>&1 || { sudo apt-get update -y && sudo apt-get install -y jq; }

  # /info summary (ждём до 60с)
  INFO_JSON=""
  for i in {1..30}; do
    INFO_JSON="$(curl -s http://127.0.0.1:'"${PORT}"'/info || true)"
    [ -n "$INFO_JSON" ] && break
    sleep 5
  done

  echo "INFO_JSON $INFO_JSON"

  DESC_OUT="$(printf '%s' "$INFO_JSON" | jq -r '.desc // empty' 2>/dev/null || true)"
  ENGINE_OUT="$(printf '%s' "$INFO_JSON" | jq -r '.build.engine_version // empty' 2>/dev/null || true)"
  BUILD_OUT="$(printf '%s' "$INFO_JSON" | jq -r '.build.version // empty' 2>/dev/null || true)"
  ACZ_OUT="$(printf '%s' "$INFO_JSON" | jq -r '.build.acz // empty' 2>/dev/null || true)"

  echo "desc: ${DESC_OUT}"
  [ -n "$ENGINE_OUT" ] && echo "engine: ${ENGINE_OUT}"
  [ -n "$BUILD_OUT" ] && echo "build: ${BUILD_OUT}"
  [ -n "$ACZ_OUT" ] && echo "acz: ${ACZ_OUT}"

  echo "connect_ip: ss14://'"${SERVER_IP}"':'"${PORT}"'"

  echo "--- server_config.toml (masked) ---"
  sed "s/^[[:space:]]*pg_password[[:space:]]*=.*/pg_password = \"***\"/" '"${DEST}"'/server_config.toml || true


'
'''
          }
        }
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: 'msbuild-logs/**', allowEmptyArchive: true
    }
    success { echo '✅ Deploy completed.' }
    failure { echo '❌ Deploy failed.' }
  }
}
