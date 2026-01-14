pipeline {
  agent { label 'tool' }

  options {
    timestamps()
    timeout(time: 10, unit: 'MINUTES')
    disableConcurrentBuilds()
  }

  environment {
    PORTS = "8092 8093 8094 8095"
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Build wheel/sdist') {
      steps {
        sh '''#!/usr/bin/env bash
          set -eux
          python3 -V
          rm -rf .venv .venv_test dist build *.egg-info run_output
          python3 -m venv .venv
          . .venv/bin/activate
          python -m pip install -U pip
          python -m pip install build
          python -m build
          ls -la dist
        '''
      }
    }

    stage('Install wheel into clean venv') {
      steps {
        sh '''#!/usr/bin/env bash
          set -eux
          python3 -m venv .venv_test
          . .venv_test/bin/activate
          python -m pip install -U pip
          python -m pip install dist/*.whl
          python -c "import restoringvalues; print(restoringvalues.__version__)"
          restoringvalues-run --help >/dev/null
        '''
      }
    }

    stage('Smoke run (25s, no GUI)') {
      steps {
        sh '''#!/usr/bin/env bash
set -euo pipefail
. .venv_test/bin/activate

mkdir -p run_output
rm -f Reciever/*.csv Business/*.csv Business/data_out_*.csv Business/data_metrics_*.csv || true

start_bg() {
  local name="$1"; shift
  setsid nohup "$@" > "run_output/${name}.log" 2>&1 & echo $! > "run_output/${name}.pid"
  echo "Started $name pid=$(cat run_output/${name}.pid)"
}

stop_group() {
  local name="$1"
  if [ -f "run_output/${name}.pid" ]; then
    local pid
    pid="$(cat run_output/${name}.pid)"
    echo "Stopping $name group (pid=$pid)"
    kill -- "-$pid" >/dev/null 2>&1 || true
  fi
}

wait_ports() {
  for i in $(seq 1 60); do
    ok=0
    for p in $PORTS; do
      if ss -lnt | awk '{print $4}' | grep -q ":$p$"; then ok=$((ok+1)); fi
    done
    if [ "$ok" -eq 4 ]; then
      return 0
    fi
    sleep 0.5
  done
  return 1
}

# Стартуем как раньше (файлами) — это самый совместимый способ
start_bg simulator python3 Simulator/simulator.py

if ! wait_ports; then
  echo "Ports not ready. Logs:"
  tail -n 200 run_output/simulator.log || true
  stop_group simulator
  exit 1
fi

start_bg reciever python3 Reciever/reciever.py
start_bg business python3 Business/business.py

sleep 25

if ! ls -la Reciever/*.csv >/dev/null 2>&1; then
  echo "No CSV produced. Logs:"
  tail -n 200 run_output/reciever.log || true
  tail -n 200 run_output/simulator.log || true
  tail -n 200 run_output/business.log || true
  stop_group business
  stop_group reciever
  stop_group simulator
  exit 1
fi

cp -a Reciever/*.csv run_output/ 2>/dev/null || true
cp -a Business/*.csv run_output/ 2>/dev/null || true

stop_group business
stop_group reciever
stop_group simulator
'''
      }
    }

    stage('Archive artifacts') {
      steps {
        archiveArtifacts artifacts: 'dist/*, run_output/*', fingerprint: true
      }
    }
  }

  post {
    always {
      cleanWs()
    }
  }
}
