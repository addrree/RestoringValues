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
        sh '''#!/usr/bin/env bash
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

PORTS="8092 8093 8094 8095"

echo "==> Cleanup old outputs"
rm -f Reciever/*.csv Business/*.csv Business/data_out_*.csv Business/data_metrics_*.csv *.pid *.log || true

kill_by_ports() {
  for p in $PORTS; do
    # сначала пробуем без sudo
    fuser -k ${p}/tcp >/dev/null 2>&1 || true
    # потом — с sudo без запроса пароля (если разрешено)
    sudo -n fuser -k ${p}/tcp >/dev/null 2>&1 || true
  done
}

echo "==> Precheck listeners (before cleanup)"
ss -lntp | egrep ':8092|:8093|:8094|:8095' || true

echo "==> Cleanup ports"
kill_by_ports
sleep 1

echo "==> Listeners after cleanup"
ss -lntp | egrep ':8092|:8093|:8094|:8095' || true

start_bg() {
  # стартуем в отдельной сессии => PID == PGID, можно убить kill -- -PID
  # $1 = name, $2.. = command
  local name="$1"; shift
  setsid nohup "$@" > "${name}.log" 2>&1 & echo $! > "${name}.pid"
  echo "Started $name pid=$(cat ${name}.pid)"
}

stop_group() {
  local name="$1"
  if [ -f "${name}.pid" ]; then
    local pid
    pid="$(cat ${name}.pid)"
    echo "Stopping $name group (pid=$pid)"
    kill -- "-$pid" >/dev/null 2>&1 || true
  fi
}

echo "==> Start Simulator (it starts server_web.py 8092-8095)"
start_bg simulator python3 Simulator/simulator.py

echo "==> Wait WS ports 8092-8095"
for i in $(seq 1 60); do
  ok=0
  for p in $PORTS; do
    if ss -lnt | awk '{print $4}' | grep -q ":$p$"; then ok=$((ok+1)); fi
  done
  if [ "$ok" -eq 4 ]; then
    echo "All WS ports are listening"
    break
  fi
  sleep 0.5
done

for p in $PORTS; do
  if ! ss -lnt | awk '{print $4}' | grep -q ":$p$"; then
    echo "Port $p is NOT listening. Logs:"
    tail -n 200 simulator.log || true
    stop_group simulator
    exit 1
  fi
done

echo "==> Start Reciever + Business"
start_bg reciever python3 Reciever/reciever.py
start_bg business python3 Business/business.py

echo "==> Let them work a bit"
sleep 25

echo "==> Check output CSV"
ls -la Reciever/*.csv >/dev/null 2>&1 || {
  echo "No CSV produced. Logs:"
  tail -n 200 reciever.log || true
  tail -n 200 simulator.log || true
  tail -n 200 business.log || true
  stop_group business
  stop_group reciever
  stop_group simulator
  kill_by_ports
  exit 1
}

echo "Smoke run OK"

echo "==> Stopping processes"
stop_group business
stop_group reciever
stop_group simulator

echo "==> Final port cleanup"
kill_by_ports
'''
      }
    }

    stage('Pack artifact') {
      steps {
        sh '''#!/usr/bin/env bash
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
