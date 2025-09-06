pipeline {
  agent any

  environment {
    // Paths inside the Jenkins container (mounted by your compose)
    SRC_ROOT     = '/workspace/razorApp'
    APP_DIR      = "${SRC_ROOT}/web"
    COMPOSE_FILE = "${SRC_ROOT}/compose.yaml"

    // Images/URLs
    IMAGE_TAG    = 'local-web:dev'
    HEALTH_URL   = 'http://nginx'       // or 'http://web:8080' to hit Kestrel directly
  }

  options {
    timestamps()
  }

  stages {
    stage('Verify layout') {
      steps {
        sh """
          ls -la "${SRC_ROOT}"
          ls -la "${APP_DIR}"
          test -f "${COMPOSE_FILE}" || (echo "compose.yaml not found at ${COMPOSE_FILE}" >&2; exit 1)
          test -f "${APP_DIR}/razorApp.csproj" || (echo "razorApp.csproj not found in ${APP_DIR}" >&2; exit 1)
          dotnet --info
        """
      }
    }

    stage('Restore / Build / Test (.NET 9)') {
      steps {
        dir("${APP_DIR}") {
          sh """
            set -e
            dotnet restore razorApp.csproj
            dotnet build -c Release --no-restore razorApp.csproj
            # keep || true until you add tests, so the stage doesn't fail due to no tests
            dotnet test -c Release --no-build --logger "trx;LogFileName=test.trx" || true
          """
        }
      }
      post {
        always {
          // Archive test results if present; harmlessly skips otherwise
          archiveArtifacts artifacts: '${APP_DIR}/**/TestResults/*.trx', allowEmptyArchive: true
        }
      }
    }

    stage('Docker Build (web image)') {
      steps {
        sh 'docker build -t ${IMAGE_TAG} -f ${APP_DIR}/Dockerfile ${SRC_ROOT}'
      }
    }

    stage('Deploy (recreate only web)') {
      steps {
        sh """
          # Ensure the proxy is up (idempotent)
          docker compose -f "${COMPOSE_FILE}" up -d nginx

          # Recreate just the web app (no deps)
          docker compose -f "${COMPOSE_FILE}" up -d --no-deps --build web
        """
      }
    }

    stage('Health check') {
      steps {
        sh """
          n=0
          until [ \$n -ge 15 ]; do
            if curl -fsS "${HEALTH_URL}" >/dev/null; then
              echo 'Healthy'
              exit 0
            fi
            n=\$((n+1))
            echo "Waiting for app... (\$n/15)"
            sleep 2
          done

          echo 'App did not become healthy in time' >&2
          echo '--- docker compose ps ---'
          docker compose -f "${COMPOSE_FILE}" ps || true
          echo '--- nginx logs ---'
          docker logs nginx --tail=200 || true
          echo '--- web logs ---'
          docker logs web --tail=200 || true
          exit 1
        """
      }
    }
  }

  post {
    success {
      echo '✅ Local build, image build, and deploy done.'
    }
    failure {
      echo '❌ Pipeline failed — check logs above.'
    }
  }
}
