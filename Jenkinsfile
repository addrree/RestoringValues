pipeline {
  agent { label 'tool' }

  options {
    timestamps()
    timeout(time: 10, unit: 'MINUTES')
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
          test -f requirements.txt
          pip install -r requirements.txt
        '''
      }
    }

    stage('Smoke run (headless, no GUI)') {
      steps {
        sh '''#!/usr/bin/env bash
set -euo pipefail

. .venv/bin/activate
export WEBSOCKET_HOST=127.0.0.1

echo "==> Precheck listeners (before cleanup)"
ss -lntp || true
ss -lntp | egrep ':8092|:8093|:8094|:8095|:8050|:8051' || true

cleanup_port () {
  local p="$1"
  # 1) пробуем fuser (обычно есть)
  if command -v fuser >/dev/null 2>&1; then
    fuser -k "${p}/tcp" >/dev/null 2>&1 || true
  fi

  # 2) если есть lsof
  if command -v lsof >/dev/null 2>&1; then
    local pid
    pid="$(lsof -tiTCP:"${p}" -sTCP:LISTEN 2>/dev/null || true)"
    if [ -n "${pid}" ]; then
      kill -9 ${pid} 2>/dev/null || true
    fi
  fi
}

echo "==> Cleanup ports"
for p in 8092 8093 8094 8095 8050 8051; do
  cleanup_port "$p"
done

# на всякий — грохнем по имени процесса, если вдруг не поймали PID
pkill -f "server_web.py" 2>/dev/null || true
pkill -f "Simulator/simulator.py" 2>/dev/null || true
pkill -f "Reciever/reciever.py" 2>/dev/null || true
pkill -f "Business/business.py" 2>/dev/null || true
pkill -f "dash_app_prod.py" 2>/dev/null || true
pkill -f "dash_app_test.py" 2>/dev/null || true

echo "==> Listeners after cleanup"
ss -lntp | egrep ':8092|:8093|:8094|:8095|:8050|:8051' || true

# чистим старые файлы, чтобы проверка была честной
rm -f Reciever/*.csv Business/*.csv Business/data_out_*.csv Business/data_metrics_*.csv 2>/dev/null || true

# гарантированно прибьем всё в конце, даже если упадем
stop_all () {
  echo "==> Stopping processes"
  for f in simulator.pid reciever.pid business.pid; do
    if [ -f "$f" ]; then
      kill "$(cat "$f")" 2>/dev/null || true
    fi
  done
  pkill -f "server_web.py" 2>/dev/null || true
}
trap stop_all EXIT

echo "==> Start Simulator (it starts server_web.py 8092-8095)"
nohup python3 Simulator/simulator.py > simulator.log 2>&1 & echo $! > simulator.pid

echo "==> Wait WS ports 8092-8095"
for i in $(seq 1 60); do
  ok=0
  for p in 8092 8093 8094 8095; do
    if ss -lnt | awk '{print $4}' | grep -q ":$p$"; then ok=$((ok+1)); fi
  done
  if [ "$ok" -eq 4 ]; then
    echo "All WS ports are listening"
    break
  fi
  sleep 0.5
done

for p in 8092 8093 8094 8095; do
  if ! ss -lnt | awk '{print $4}' | grep -q ":$p$"; then
    echo "Port $p is NOT listening. Tail simulator.log:"
    tail -n 200 simulator.log || true
    exit 1
  fi
done

echo "==> Start Reciever + Business"
nohup python3 Reciever/reciever.py > reciever.log 2>&1 & echo $! > reciever.pid
nohup python3 Business/business.py  > business.log 2>&1 & echo $! > business.pid

echo "==> Let them work a bit"
sleep 25

echo "==> Check output CSV"
ls -la Reciever/*.csv >/dev/null 2>&1 || {
  echo "No CSV produced. Logs:"
  tail -n 200 simulator.log || true
  tail -n 200 reciever.log || true
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
