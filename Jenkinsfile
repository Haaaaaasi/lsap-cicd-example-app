pipeline {
  agent any

  environment {
    DISCORD_WEBHOOK = credentials('discord_webhook')
    // ★★★ 請將這裡改成你的 Docker Hub 帳號 (不是 GitHub 帳號) ★★★
    DOCKER_USER = 'haaaaaasi' 
    // 引用剛剛建立的憑證 (ID 必須是 docker-hub-credentials)
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

    // Part 1: 靜態分析 (使用 Docker Build 方式，解決路徑問題)
    stage('Static Analysis') {
      steps {
        script {
          echo "--- Starting Static Analysis ---"
          // 動態產生 Lint 用的 Dockerfile
          writeFile file: 'Dockerfile.lint', text: '''
            FROM node:18-alpine
            WORKDIR /app
            COPY . .
            RUN npm install
            CMD ["npm", "run", "lint"]
          '''
          // 建置並執行檢查
          sh 'docker build -t lint-image -f Dockerfile.lint .'
          sh 'docker run --rm lint-image'
          sh 'docker rmi lint-image'
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
          
          // 1. 登入 Docker Hub
          sh "echo $DOCKER_CREDS_PSW | docker login -u $DOCKER_CREDS_USR --password-stdin"
          
          // 2. 建立部署用的 Dockerfile (如果專案根目錄沒有，手動產生一個)
          writeFile file: 'Dockerfile', text: '''
            FROM node:18-alpine
            WORKDIR /app
            COPY . .
            RUN npm install --production
            EXPOSE 8080
            CMD ["node", "app.js"]
          '''
          
          // 3. Build & Push Image
          sh "docker build -t ${imageTag} ."
          sh "docker push ${imageTag}"
          
          // 4. 清理舊容器 (如果存在)
          // || true 代表如果容器不存在也不要報錯
          sh "docker rm -f dev-app || true"
          
          // 5. 部署容器 (對應 Host Port 8081)
          sh "docker run -d -p 8081:8080 --name dev-app ${imageTag}"
          
          // 6. 驗證服務
          sleep 5
          sh "curl -f http://localhost:8081/health || echo 'Health check warning'"
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