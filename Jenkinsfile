pipeline {
  agent { label 'tool' }

  options {
    timestamps()
    timeout(time: 10, unit: 'MINUTES')
    disableConcurrentBuilds()   // <-- ВАЖНО: не даём двум билдам гонять порты одновременно
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

        # --- ЖЁСТКАЯ ЗАЧИСТКА ДО СТАРТА ---
        # прибиваем всё, что могло остаться от прошлых запусков (даже если PID-файлов нет)
        pkill -f "server_web.py" 2>/dev/null || true
        pkill -f "Simulator/simulator.py" 2>/dev/null || true
        pkill -f "Reciever/reciever.py" 2>/dev/null || true
        pkill -f "Business/business.py" 2>/dev/null || true

        # если есть fuser — вообще идеально, убивает по порту
        command -v fuser >/dev/null 2>&1 && sudo fuser -k 8092/tcp 8093/tcp 8094/tcp 8095/tcp 2>/dev/null || true

        echo "==> Ports before start:"
        ss -lntp | egrep ':8092|:8093|:8094|:8095|:8050|:8051' || true

        cleanup() {
        echo "==> cleanup"
        # убиваем процесс-группы (ВАЖНО: минус перед PID)
        [ -f business.pid ]  && kill -- -$(cat business.pid) 2>/dev/null || true
        [ -f reciever.pid ]  && kill -- -$(cat reciever.pid) 2>/dev/null || true
        [ -f simulator.pid ] && kill -- -$(cat simulator.pid) 2>/dev/null || true

        pkill -f "server_web.py" 2>/dev/null || true
        }
        trap cleanup EXIT

        rm -f Reciever/*.csv Business/*.csv Business/data_out_*.csv Business/data_metrics_*.csv || true

        # --- 1) стартуем Simulator в НОВОЙ process group ---
        # setsid создаёт новую сессию/группу → можно убить всё дочернее одним kill -- -PID
        setsid nohup python3 Simulator/simulator.py > simulator.log 2>&1 & echo $! > simulator.pid

        # --- 2) ждём порты ---
        echo "Waiting for WS ports 8092-8095..."
        for i in $(seq 1 80); do
        ok=0
        for p in 8092 8093 8094 8095; do
            if ss -lnt | awk '{print $4}' | grep -q ":$p$"; then ok=$((ok+1)); fi
        done
        [ "$ok" -eq 4 ] && break
        sleep 0.25
        done

        for p in 8092 8093 8094 8095; do
        if ! ss -lnt | awk '{print $4}' | grep -q ":$p$"; then
            echo "Port $p NOT listening. simulator.log:"
            tail -n 200 simulator.log || true
            echo "Current listeners:"
            ss -lntp | egrep ':8092|:8093|:8094|:8095' || true
            exit 1
        fi
        done

        # --- 3) Reciever + Business тоже в process groups ---
        setsid nohup python3 Reciever/reciever.py > reciever.log 2>&1 & echo $! > reciever.pid
        setsid nohup python3 Business/business.py > business.log 2>&1 & echo $! > business.pid

        sleep 25

        ls -la Reciever/*.csv >/dev/null 2>&1 || {
        echo "No CSV. Logs:"
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
