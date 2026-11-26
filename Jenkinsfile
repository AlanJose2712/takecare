pipeline {
  agent any
  triggers { pollSCM('H/2 * * * *') }
  options { timeout(time: 45, unit: 'MINUTES') }
  environment {
    IMAGE = 'alanjose2929/takecare'
    TAG   = "${BUILD_NUMBER}"
  }
  stages {
    stage('Checkout') {
      steps {
        checkout scm
        script {
          def author = sh(script: 'git log -1 --pretty=%an', returnStdout: true).trim()
          env.SKIP = (author == 'jenkins-bot') ? 'true' : 'false'
        }
      }
    }
    stage('SAST - Semgrep') {
      when { expression { env.SKIP == 'false' } }
      steps {
        sh '''
          docker run --rm -v "$WORKSPACE:/src" semgrep/semgrep \
            semgrep scan --config p/python --config p/django --exclude static /src 2>&1 | tee semgrep.txt || true
        '''
      }
    }
    stage('Dependency Scan - Trivy') {
      when { expression { env.SKIP == 'false' } }
      steps {
        sh 'trivy fs --scanners vuln,secret --severity HIGH,CRITICAL --skip-version-check . | tee trivy-fs.txt'
      }
    }
    stage('Build Image') {
      when { expression { env.SKIP == 'false' } }
      steps { sh 'docker build -t $IMAGE:$TAG .' }
    }
    stage('Image Scan - Trivy') {
      when { expression { env.SKIP == 'false' } }
      steps { sh 'trivy image --severity CRITICAL --skip-version-check $IMAGE:$TAG | tee trivy-image.txt' }
    }
    stage('Push to Docker Hub') {
      when { expression { env.SKIP == 'false' } }
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
          sh '''
            echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin
            docker push $IMAGE:$TAG
          '''
        }
      }
    }
    stage('DAST - ZAP') {
      when { expression { env.SKIP == 'false' } }
      steps {
        catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
          timeout(time: 12, unit: 'MINUTES') {
            sh '''
              docker rm -f tc-dast tc-dast-db || true
              docker network rm tc-dast-net || true
              docker network create tc-dast-net
              docker run -d --name tc-dast-db --network tc-dast-net \
                -e MYSQL_ROOT_PASSWORD=dastpass -e MYSQL_DATABASE=takecare mysql:8.0
              sleep 30
              docker run -d --name tc-dast --network tc-dast-net -p 8003:8000 \
                -e DB_HOST=tc-dast-db -e DB_PASSWORD=dastpass -e DEBUG=False $IMAGE:$TAG
              sleep 25
              mkdir -p zap-out && chmod 777 zap-out
              docker run --rm --cpus=1 --network host -v $WORKSPACE/zap-out:/zap/wrk zaproxy/zap-stable \
                zap-baseline.py -t http://localhost:8003 -m 1 -T 5 -I -r zap.html
            '''
          }
        }
      }
    }
    stage('Update Manifest (GitOps)') {
      when { expression { env.SKIP == 'false' } }
      steps {
        withCredentials([usernamePassword(credentialsId: 'github-creds', usernameVariable: 'GH_USER', passwordVariable: 'GH_TOKEN')]) {
          sh '''
            sed -i "s|image: alanjose2929/takecare:.*|image: $IMAGE:$TAG|" k8s/deployment.yaml
            git config user.name jenkins-bot
            git config user.email jenkins-bot@example.com
            git commit -am "Deploy $TAG"
            git push https://$GH_USER:$GH_TOKEN@github.com/AlanJose2712/takecare.git HEAD:main
          '''
        }
      }
    }
  }
  post {
    always {
      sh '''
        docker rm -f tc-dast tc-dast-db || true
        docker network rm tc-dast-net || true
      '''
      archiveArtifacts artifacts: 'semgrep.txt, trivy-*.txt, zap-out/**', allowEmptyArchive: true
    }
  }
}
