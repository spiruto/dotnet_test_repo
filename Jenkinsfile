pipeline {
  agent any

  environment {
    DOTNET_SDK_IMAGE = 'mcr.microsoft.com/dotnet/sdk:9.0'
    COMPOSE          = 'docker compose'
    WEB_SERVICE      = 'web'
    HEALTH_URL       = 'http://localhost:80' // Nginx -> web:8080
  }

  options {
    timestamps()
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Restore / Build / Test (.NET 9)') {
      steps {
        // Run dotnet commands inside the SDK container, mounting your source
        sh """
          docker run --rm \
            -v "\$PWD/web:/src" \
            -w /src \
            ${DOTNET_SDK_IMAGE} \
            bash -lc '
              dotnet --info
              dotnet restore
              dotnet build -c Release --no-restore
              dotnet test -c Release --no-build --logger "trx;LogFileName=test.trx"
            '
        """
      }
      post {
        always {
          // Collect test results if you install the MSTest/JUnit plugins later
          archiveArtifacts artifacts: 'web/**/TestResults/*.trx', allowEmptyArchive: true
        }
      }
    }

    stage('Docker Build (web image)') {
      steps {
        sh 'docker build -t local-web:dev ./web'
      }
    }

    stage('Deploy (recreate only web)') {
      steps {
        // rebuild & restart only the web service; nginx and mssql keep running
        sh '${COMPOSE} up -d --no-deps --build ${WEB_SERVICE}'
      }
    }

    stage('Health check') {
      steps {
        // Simple 10x retry curl against Nginx (port from .env: NGINX_PORT=80)
        sh """
          n=0
          until [ \$n -ge 10 ]; do
            if curl -fsS ${HEALTH_URL} >/dev/null; then
              echo 'Healthy'
              exit 0
            fi
            n=\$((n+1))
            echo "Waiting for app... (\$n/10)"
            sleep 2
          done
          echo 'App did not become healthy in time' >&2
          exit 1
        """
      }
    }
  }

  post {
    success {
      echo '✅ Build, test, and local deploy completed.'
    }
    failure {
      echo '❌ Pipeline failed. Check logs above.'
    }
  }
}
