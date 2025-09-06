pipeline {
  agent any

  environment {
    SRC_ROOT     = '/workspace/razorApp'        // project root mounted into Jenkins
    APP_DIR      = "${SRC_ROOT}/web"            // your csproj lives here
    IMAGE_TAG    = 'local-web:dev'
    COMPOSE_FILE = "${SRC_ROOT}/compose.yaml"   // your compose at root
    HEALTH_URL   = 'http://localhost:80'
  }

  options { timestamps() }

  stages {
    stage('Verify layout') {
      steps {
        sh 'ls -la ${SRC_ROOT}'
        sh 'ls -la ${APP_DIR}'
        sh 'test -f ${COMPOSE_FILE} || (echo "compose.yaml not found at ${COMPOSE_FILE}" >&2; exit 1)'
        sh 'test -f ${APP_DIR}/razorApp.csproj || (echo "razorApp.csproj not found in ${APP_DIR}" >&2; exit 1)'
        sh 'dotnet --info'
      }
    }

    stage('Restore / Build / Test (.NET 9)') {
      steps {
        dir("${APP_DIR}") {
          sh '''
            set -e
            dotnet restore razorApp.csproj
            dotnet build -c Release --no-restore razorApp.csproj
            # keep || true until you add tests
            dotnet test  -c Release --no-build --logger "trx;LogFileName=test.trx" || true
          '''
        }
      }
      post {
        always { archiveArtifacts artifacts: 'web/**/TestResults/*.trx', allowEmptyArchive: true }
      }
    }

    stage('Docker Build (web image)') {
      steps {
        sh 'docker build -t ${IMAGE_TAG} -f ${APP_DIR}/Dockerfile ${SRC_ROOT}'
      }
    }

    stage('Deploy (recreate only web)') {
      steps {
        sh 'docker compose -f ${COMPOSE_FILE} up -d --no-deps --build web'
      }
    }

    stage('Health check') {
      steps {
        sh '''
          n=0
          until [ $n -ge 10 ]; do
            if curl -fsS '${HEALTH_URL}' >/dev/null; then
              echo 'Healthy'; exit 0
            fi
            n=$((n+1)); echo "Waiting for app... ($n/10)"; sleep 2
          done
          echo 'App did not become healthy in time' >&2
          exit 1
        '''
      }
    }
  }

  post {
    success { echo '✅ Local build, image build, and deploy done.' }
    failure { echo '❌ Pipeline failed — check logs above.' }
  }
}
