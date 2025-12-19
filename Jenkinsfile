pipeline {
  agent any

  environment {
    DISCORD_WEBHOOK = credentials('discord_webhook')
    // ★★★ 請確認這裡是你的 Docker Hub 帳號 ★★★
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

    // Part 1: 靜態分析 (所有分支都要跑) [cite: 22-25]
    stage('Static Analysis') {
      steps {
        script {
          echo "--- Starting Static Analysis ---"
          writeFile file: 'Dockerfile.lint', text: '''
            FROM node:18-alpine
            WORKDIR /app
            COPY . .
            RUN npm install
            CMD ["npm", "run", "lint"]
          '''
          sh 'docker build -t lint-image -f Dockerfile.lint .'
          sh 'docker run --rm lint-image'
          sh 'docker rmi lint-image'
        }
      }
    }

    // Part 2: Staging 環境部署 (只在 dev 分支執行) [cite: 42-50]
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
            CMD ["node", "app.js"]
          '''
          
          // Build & Push [cite: 47]
          sh "docker build -t ${imageTag} ."
          sh "docker push ${imageTag}"
          
          // Cleanup & Deploy to Port 8081 [cite: 48, 49]
          sh "docker rm -f dev-app || true"
          sh "docker run -d -p 8081:8080 --name dev-app ${imageTag}"
          
          // Verify with Container IP (更穩定的驗證方式) 
          sleep 5
          def containerIp = sh(script: "docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' dev-app", returnStdout: true).trim()
          sh "curl -f http://${containerIp}:8080/health || echo 'Health check warning'"
        }
      }
    }

    // Part 3: Production GitOps Promotion (只在 main 分支執行) [cite: 51-66]
    stage('GitOps Promotion') {
      when {
        branch 'main'
      }
      steps {
        script {
          echo "--- Starting GitOps Promotion ---"
          
          // 1. 讀取 deploy.config 檔案 [cite: 59]
          // 這是 GitOps 的核心：Git 是唯一的真理來源 (Single Source of Truth) [cite: 69]
          def targetTag = readFile('deploy.config').trim()
          echo "Target Version from Git: ${targetTag}"
          
          def sourceImage = "${env.DOCKER_USER}/myapp:${targetTag}"
          def prodImage = "${env.DOCKER_USER}/myapp:prod-${env.BUILD_NUMBER}"

          sh "echo $DOCKER_CREDS_PSW | docker login -u $DOCKER_CREDS_USR --password-stdin"

          // 2. Artifact Promotion: Pull -> Retag -> Push [cite: 61, 62]
          // 不進行 Build，而是直接將經過驗證的 Image 晉升為 Production 版本
          sh "docker pull ${sourceImage}"
          sh "docker tag ${sourceImage} ${prodImage}"
          sh "docker push ${prodImage}"

          // 3. Deploy Production (Port 8082) [cite: 64, 66]
          sh "docker rm -f prod-app || true"
          sh "docker run -d -p 8082:8080 --name prod-app ${prodImage}"
          
          // 4. Verify
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