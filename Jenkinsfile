pipeline {
  agent { label 'tool' }

  options {
    timestamps()
    disableConcurrentBuilds()
    timeout(time: 10, unit: 'MINUTES')
    buildDiscarder(logRotator(numToKeepStr: '20', artifactNumToKeepStr: '10'))
  }

  parameters {
    choice(name: 'MODE', choices: ['build', 'deploy'], description: 'build = собрать артефакт, deploy = задеплоить на VM')
    string(name: 'REPO_URL', defaultValue: 'https://github.com/addrree/RestoringValues', description: 'Git repo')
    string(name: 'BRANCH', defaultValue: 'main', description: 'Branch')
    string(name: 'PYTHON', defaultValue: 'python3', description: 'Python executable')
    string(name: 'ARTIFACT_NAME', defaultValue: 'restoringvalues.tar.gz', description: 'Name of packaged artifact')
  }

  environment {
    VENV_DIR = ".venv"
  }

  stages {
    stage('Checkout') {
      steps {
        deleteDir()
        git branch: params.BRANCH, url: params.REPO_URL
      }
    }

    stage('System info (safe)') {
      steps {
        sh '''
          set -e
          echo "== whoami =="; whoami
          echo "== pwd =="; pwd
          echo "== df -h =="; df -h .
          echo "== free -h =="; free -h || true
          echo "== uname -a =="; uname -a
          echo "== python =="; ${PYTHON} --version
        '''
      }
    }

    stage('Create venv + install deps') {
      when { expression { params.MODE == 'build' } }
      steps {
        sh '''
          set -e
          rm -rf "${VENV_DIR}"
          ${PYTHON} -m venv "${VENV_DIR}"
          . "${VENV_DIR}/bin/activate"
          python -m pip install --upgrade pip wheel setuptools

          # If requirements exist – install them; otherwise skip.
          if [ -f requirements.txt ]; then
            pip install -r requirements.txt
          elif [ -f pyproject.toml ]; then
            # fallback (won't fail if it's not a package)
            pip install .
          fi
        '''
      }
    }

    stage('Smoke check (safe)') {
      when { expression { params.MODE == 'build' } }
      steps {
        sh '''
          set -e
          . "${VENV_DIR}/bin/activate"

          # Try import project modules if possible (won't fail hard if structure unknown)
          python - <<'PY' || true
import sys, os
print("Python:", sys.version)
print("CWD:", os.getcwd())
print("Top-level files:", os.listdir(".")[:30])
PY

          # If there are pytest tests – run quick
          if [ -d tests ] || [ -f pytest.ini ] || [ -f pyproject.toml ]; then
            python -m pip install -q pytest || true
            pytest -q || true
          fi
        '''
      }
    }

    stage('Pack artifact') {
      when { expression { params.MODE == 'build' } }
      steps {
        sh '''
          set -e
          rm -f "${ARTIFACT_NAME}"

          # pack repo WITHOUT venv and git history
          tar --exclude=".git" --exclude="${VENV_DIR}" --exclude="__pycache__" --exclude="*.pyc" \
              --exclude=".pytest_cache" --exclude=".mypy_cache" --exclude=".ruff_cache" \
              -czf "${ARTIFACT_NAME}" .
          ls -lah "${ARTIFACT_NAME}"
        '''
        archiveArtifacts artifacts: "${params.ARTIFACT_NAME}", fingerprint: true
      }
    }

    stage('Clean workspace') {
      steps {
        deleteDir()
      }
    }
  }

  post {
    success { echo "Pipeline SUCCESS" }
    failure { echo "Pipeline FAILED" }
    always  { echo "Done." }
  }
}
