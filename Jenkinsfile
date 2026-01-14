pipeline {
  agent { label 'tool' }

  options {
    timestamps()
    timeout(time: 10, unit: 'MINUTES')
    disableConcurrentBuilds()
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Create venv + install deps') {
      steps {
        sh '''#!/bin/bash
          set -e
          python3 -V
          rm -rf .venv
          python3 -m venv .venv
          . .venv/bin/activate
          pip install -U pip wheel
          if [ -f requirements.txt ]; then
            pip install -r requirements.txt
          else
            echo "requirements.txt not found - add it please"
            exit 1
          fi
        '''
      }
    }

    stage('Smoke run (headless, no GUI)') {
      steps {
        sh '''#!/usr/bin/env bash
set -euo pipefail
. .venv/bin/activate
export WEBSOCKET_HOST=127.0.0.1

# ---- helpers ----
is_free() {
  local p="$1"
  ! ss -lnt "( sport = :$p )" | tail -n +2 | grep -q .
}

pick_ports() {
  # try default
  if is_free 8092 && is_free 8093 && is_free 8094 && is_free 8095; then
    echo "8092 8093 8094 8095"
    return
  fi

  # find a free range
  for base in 18092 19092 20092 21092 22092 23092; do
    p1=$base; p2=$((base+1)); p3=$((base+2)); p4=$((base+3))
    if is_free "$p1" && is_free "$p2" && is_free "$p3" && is_free "$p4"; then
      echo "$p1 $p2 $p3 $p4"
      return
    fi
  done

  echo "No free port range found" >&2
  exit 1
}

# ---- 0) hard cleanup of leftovers ----
pkill -f "server_web.py" 2>/dev/null || true
pkill -f "Simulator/simulator.py" 2>/dev/null || true
pkill -f "Reciever/reciever.py" 2>/dev/null || true
pkill -f "Business/business.py" 2>/dev/null || true

command -v fuser >/dev/null 2>&1 && sudo fuser -k 8092/tcp 8093/tcp 8094/tcp 8095/tcp 2>/dev/null || true

echo "==> listeners BEFORE:"
ss -lntp | egrep ':8092|:8093|:8094|:8095|:8050|:8051' || true

# ---- 1) choose ports ----
read P1 P2 P3 P4 <<< "$(pick_ports)"
echo "==> Using ports: $P1 $P2 $P3 $P4"

# ---- 2) patch ports in workspace code (no changes in GitHub) ----
python3 - <<PY
import re, pathlib

p1,p2,p3,p4 = map(int, "${P1} ${P2} ${P3} ${P4}".split())
files = [
  pathlib.Path("Simulator/simulator.py"),
  pathlib.Path("Reciever/reciever.py"),
]

for f in files:
  if not f.exists():
    continue
  s = f.read_text(encoding="utf-8", errors="ignore")

  # ports = [8092, 8093, 8094, 8095]
  s = re.sub(
    r"ports\\s*=\\s*\\[\\s*8092\\s*,\\s*8093\\s*,\\s*8094\\s*,\\s*8095\\s*\\]",
    f"ports = [{p1}, {p2}, {p3}, {p4}]",
    s
  )

  # any literal list [8092, 8093, 8094, 8095]
  s = re.sub(
    r"\\[\\s*8092\\s*,\\s*8093\\s*,\\s*8094\\s*,\\s*8095\\s*\\]",
    f"[{p1}, {p2}, {p3}, {p4}]",
    s
  )

  f.write_text(s, encoding="utf-8")

print("patched ports ok")
PY

# ---- 3) clean outputs ----
rm -f Reciever/*.csv Business/*.csv Business/data_out_*.csv Business/data_metrics_*.csv || true

cleanup() {
  echo "==> cleanup"
  [ -f business.pid ]  && kill -- -$(cat business.pid) 2>/dev/null || true
  [ -f reciever.pid ]  && kill -- -$(cat reciever.pid) 2>/dev/null || true
  [ -f simulator.pid ] && kill -- -$(cat simulator.pid) 2>/dev/null || true
  pkill -f "server_web.py" 2>/dev/null || true
}
trap cleanup EXIT

# ---- 4) start simulator (creates server_web and listens ports) ----
setsid nohup python3 Simulator/simulator.py > simulator.log 2>&1 & echo $! > simulator.pid

echo "Waiting WS ports..."
for i in $(seq 1 120); do
  ok=0
  for p in "$P1" "$P2" "$P3" "$P4"; do
    ss -lnt | awk '{print $4}' | grep -q ":$p$" && ok=$((ok+1)) || true
  done
  [ "$ok" -eq 4 ] && break
  sleep 0.25
done

for p in "$P1" "$P2" "$P3" "$P4"; do
  if ! ss -lnt | awk '{print $4}' | grep -q ":$p$"; then
    echo "Port $p NOT listening. simulator.log:"
    tail -n 200 simulator.log || true
    echo "==> listeners NOW:"
    ss -lntp | egrep ":$P1|:$P2|:$P3|:$P4" || true
    exit 1
  fi
done

# ---- 5) start reciever + business ----
setsid nohup python3 Reciever/reciever.py > reciever.log 2>&1 & echo $! > reciever.pid
setsid nohup python3 Business/business.py  > business.log 2>&1 & echo $! > business.pid

sleep 25

ls -la Reciever/*.csv >/dev/null 2>&1 || {
  echo "No CSV produced. Logs:"
  tail -n 200 reciever.log || true
  tail -n 200 simulator.log || true
  tail -n 200 business.log || true
  exit 1
}

echo "Smoke run OK"
'''
      }
    }

    stage('Pack artifact') {
      steps {
        sh '''#!/bin/bash
          set -e
          tar -czf artifact-restoringvalues.tgz \
            Simulator Reciever Business GUI \
            requirements.txt \
            *.log || true
        '''
        archiveArtifacts artifacts: 'artifact-restoringvalues.tgz, *.log', fingerprint: true
      }
    }
  }

  post {
    always {
      echo "Done."
      cleanWs()
    }
  }
}
