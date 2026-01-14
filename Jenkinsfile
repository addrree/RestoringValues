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
        sh '''#!/bin/bash
          set -e
          . .venv/bin/activate

          # clean previous outputs (optional)
          rm -f Reciever/*.csv Business/*.csv || true

          # start 3 services in background
          python3 Simulator/simulator.py > simulator.log 2>&1 &
          SIM_PID=$!
          python3 Reciever/reciever.py > reciever.log 2>&1 &
          REC_PID=$!
          python3 Business/business.py > business.log 2>&1 &
          BUS_PID=$!

          # let it work a bit
          sleep 25

          echo "==> Check processes"
          ps -p $SIM_PID -o pid= >/dev/null || (echo "simulator died"; tail -n 50 simulator.log; exit 1)
          ps -p $REC_PID -o pid= >/dev/null || (echo "reciever died"; tail -n 50 reciever.log; exit 1)
          ps -p $BUS_PID -o pid= >/dev/null || (echo "business died"; tail -n 50 business.log; exit 1)

          echo "==> Check outputs (CSV exist)"
          ls -la Reciever/*.csv >/dev/null 2>&1 || (echo "No receiver CSV"; ls -la Reciever || true; exit 1)
          ls -la Business/*.csv >/dev/null 2>&1 || (echo "No business CSV"; ls -la Business || true; exit 1)

          echo "==> Stop services"
          kill $SIM_PID $REC_PID $BUS_PID || true
          sleep 2
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
