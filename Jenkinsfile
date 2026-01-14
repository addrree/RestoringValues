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

    VENV=".venv/bin/activate"
    . "$VENV"

    # 1) стартуем reciever (он должен поднять WS/порты)
    nohup python3 Reciever/reciever.py > reciever.log 2>&1 & echo $! > reciever.pid

    # 2) ждём, пока порт начнёт слушать
    echo "Waiting for 8092 to be ready..."
    for i in $(seq 1 30); do
    if ss -lnt | awk '{print $4}' | grep -q ":8092$"; then
        echo "8092 is listening"
        break
    fi
    sleep 1
    done

    if ! ss -lnt | awk '{print $4}' | grep -q ":8092$"; then
    echo "Port 8092 is still not listening. Logs:"
    tail -n 200 reciever.log || true
    exit 1
    fi

    # 3) запускаем simulator и business
    nohup python3 Simulator/simulator.py > simulator.log 2>&1 & echo $! > simulator.pid
    nohup python3 Business/business.py   > business.log 2>&1 & echo $! > business.pid

    # 4) ждём чуть-чуть
    sleep 25

    # 5) проверка: должны появиться csv от reciever
    ls -la Reciever/*.csv || (echo "No CSV produced. Logs:"; tail -n 200 reciever.log; tail -n 200 simulator.log; exit 1)

    # 6) стопаем всё
    kill $(cat simulator.pid) 2>/dev/null || true
    kill $(cat business.pid) 2>/dev/null || true
    kill $(cat reciever.pid) 2>/dev/null || true

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
