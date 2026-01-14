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
    echo "==> Ports before start"
    ss -lntp | egrep ':8092|:8093|:8094|:8095|:8050|:8051' || true

    # если хочешь строго валить билд когда занято:
    if ss -lntp | awk '{print $4}' | egrep -q ':(8092|8093|8094|8095|8050|8051)$'; then
    echo "Some required port is already in use. Who?"
    sudo ss -lntp | egrep ':8092|:8093|:8094|:8095|:8050|:8051' || true
    exit 1
    fi
    # чтобы Simulator подключался к локальному ws-server гарантированно
    export WEBSOCKET_HOST=127.0.0.1

    # на всякий — чистим старые csv от прошлых запусков
    rm -f Reciever/*.csv Business/*.csv Business/data_out_*.csv Business/data_metrics_*.csv || true

    # 1) стартуем Simulator (он поднимает server_web.py и слушает 8092-8095)
    nohup python3 Simulator/simulator.py > simulator.log 2>&1 & echo $! > simulator.pid

    # 2) ждём, пока порты 8092-8095 начнут слушаться
    echo "Waiting for WS ports 8092-8095 to be ready..."
    for i in $(seq 1 40); do
    ok=0
    for p in 8092 8093 8094 8095; do
        if ss -lnt | awk '{print $4}' | grep -q ":$p$"; then
        ok=$((ok+1))
        fi
    done
    if [ "$ok" -eq 4 ]; then
        echo "All ports are listening"
        break
    fi
    sleep 0.5
    done

    for p in 8092 8093 8094 8095; do
    if ! ss -lnt | awk '{print $4}' | grep -q ":$p$"; then
        echo "Port $p is NOT listening. Simulator logs:"
        tail -n 200 simulator.log || true
        kill $(cat simulator.pid) 2>/dev/null || true
        exit 1
    fi
    done

    # 3) стартуем Reciever и Business (они клиенты/обработчики)
    nohup python3 Reciever/reciever.py > reciever.log 2>&1 & echo $! > reciever.pid
    nohup python3 Business/business.py > business.log 2>&1 & echo $! > business.pid

    # 4) ждём чуть-чуть
    sleep 25

    # 5) проверка: должны появиться CSV от Reciever
    ls -la Reciever/*.csv >/dev/null 2>&1 || {
    echo "No CSV produced by Reciever. Logs:"
    tail -n 200 reciever.log || true
    tail -n 200 simulator.log || true
    tail -n 200 business.log || true
    kill $(cat business.pid) 2>/dev/null || true
    kill $(cat reciever.pid) 2>/dev/null || true
    kill $(cat simulator.pid) 2>/dev/null || true
    exit 1
    }

    # 6) стопаем всё
    kill $(cat business.pid) 2>/dev/null || true
    kill $(cat reciever.pid) 2>/dev/null || true
    kill $(cat simulator.pid) 2>/dev/null || true

    echo "Smoke run OK"
    '''
        }
    }

    stage('Pack artifact') {
      steps {
        sh '''#!/bin/bash
          set -e
          # pack code + requirements + logs + example outputs
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
