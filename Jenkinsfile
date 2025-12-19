pipeline {
  agent any

  environment {
    DISCORD_WEBHOOK = credentials('discord_webhook')
    // 請確認這裡是你的 Docker Hub 帳號
    DOCKER_USER = 'haaaaaasi' 
    DOCKER_CREDS = credentials('docker-hub-credentials')
    
    YOUR_NAME = '潘俊諺'
    YOUR_STUDENT_ID = 'B11303140'
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    // Part 1: 靜態分析
    stage('Static Analysis') {
      steps {
        script {
          echo "--- Starting Static Analysis ---"
          def lintImageTag = "lint-image-${env.BUILD_NUMBER}"

          writeFile file: 'Dockerfile.lint', text: '''
            FROM node:18-alpine
            WORKDIR /app
            COPY . .
            RUN npm install
            CMD ["npm", "run", "lint"]
          '''
          
          sh "docker build -t ${lintImageTag} -f Dockerfile.lint ."
          sh "docker run --rm ${lintImageTag}"
          sh "docker rmi ${lintImageTag}"
        }
      }
    }

    // Part 2: Staging 環境部署 (只在 dev 分支執行)
    stage('Build & Deploy Staging') {
      when {
        branch 'dev'
      }
      steps {
        script {
          echo "--- Starting Staging Deployment ---"
          def imageTag = "${env.DOCKER_USER}/myapp:dev-${env.BUILD_NUMBER}"
          
          sh "echo $DOCKER_CREDS_PSW | docker login -u $DOCKER_CREDS_USR --password-stdin"
          
          writeFile file: 'Dockerfile', text: '''
            FROM node:18-alpine
            WORKDIR /app
            COPY . .
            RUN npm install --production
            EXPOSE 8080
            CMD ["npm", "start"]
          '''
          
          sh "docker build -t ${imageTag} ."
          sh "docker push ${imageTag}"
          
          sh "docker rm -f dev-app || true"
          sh "docker run -d -p 8081:8080 --name dev-app ${imageTag}"
          
          sleep 5
          def containerIp = sh(script: "docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' dev-app", returnStdout: true).trim()
          sh "curl -f http://${containerIp}:8080/health || echo 'Health check warning'"
        }
      }
    }

    // Part 3: Production GitOps Promotion (只在 main 分支執行)
    stage('GitOps Promotion') {
      when {
        branch 'main'
      }
      steps {
        script {
          echo "--- Starting GitOps Promotion ---"
          
          def targetTag = readFile('deploy.config').trim()
          echo "Target Version from Git: ${targetTag}"
          
          def sourceImage = "${env.DOCKER_USER}/myapp:${targetTag}"
          def prodImage = "${env.DOCKER_USER}/myapp:prod-${env.BUILD_NUMBER}"

          sh "echo $DOCKER_CREDS_PSW | docker login -u $DOCKER_CREDS_USR --password-stdin"

          sh "docker pull ${sourceImage}"
          sh "docker tag ${sourceImage} ${prodImage}"
          sh "docker push ${prodImage}"

          sh "docker rm -f prod-app || true"
          sh "docker run -d -p 8082:8080 --name prod-app ${prodImage}"
          
          sleep 5
          def containerIp = sh(script: "docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' prod-app", returnStdout: true).trim()
          sh "curl -f http://${containerIp}:8080/health || echo 'Health check warning'"
        }
      }
    }
  }

  post {
    always { 
      script {
        def repoUrl = env.GIT_URL ?: 'https://github.com/Haaaaaasi/lsap-cicd-example-app.git'
        def statusEmoji = currentBuild.currentResult == 'SUCCESS' ? '✅' : '❌'
        
        def errorMsg = """
${statusEmoji} Build ${currentBuild.currentResult}
Name: ${env.YOUR_NAME}
Student ID: ${env.YOUR_STUDENT_ID}
Job: ${env.JOB_NAME}
Build: ${env.BUILD_NUMBER}
Repo: ${repoUrl}
Branch: ${env.BRANCH_NAME}
"""
        def jsonContent = errorMsg.replace("\n", "\\n")

        sh """
        curl -X POST -H "Content-type: application/json" \
        --data '{"content": "${jsonContent}"}' \
        \$DISCORD_WEBHOOK
        """
      }
    }
  }
}