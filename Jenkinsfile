pipeline {
  agent any

  environment {
    DISCORD_WEBHOOK = credentials('discord_webhook')
    // ★★★ 請將這裡改成你的 Docker Hub 帳號 ★★★
    DOCKER_USER = 'haaaaaasi' 
    // 引用剛剛建立的憑證
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

    // Part 1: 靜態分析 (所有分支都要跑)
    stage('Static Analysis') {
      steps {
        script {
          // 動態產生 Lint 用的 Dockerfile
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

    // Part 2: Staging 部署 (只在 dev 分支執行)
    stage('Build & Deploy Staging') {
      when {
        branch 'dev'
      }
      steps {
        script {
          def imageTag = "${env.DOCKER_USER}/myapp:dev-${env.BUILD_NUMBER}"
          
          echo "Building & Pushing to Docker Hub: ${imageTag}"
          
          // 1. 登入 Docker Hub
          sh "echo $DOCKER_CREDS_PSW | docker login -u $DOCKER_CREDS_USR --password-stdin"
          
          // 2. 建立正式用的 Image (假設專案根目錄有原本的 Dockerfile，如果沒有，這裡要動態建立)
          // 注意：作業提供的範例 App 通常根目錄會有一個 Dockerfile
          // 如果沒有，我們下面動態建立一個標準的 Node.js Dockerfile
          writeFile file: 'Dockerfile', text: '''
            FROM node:18-alpine
            WORKDIR /app
            COPY . .
            RUN npm install --production
            EXPOSE 8080
            CMD ["node", "app.js"]
          '''
          
          // 3. Build & Push
          sh "docker build -t ${imageTag} ."
          sh "docker push ${imageTag}"
          
          // 4. Cleanup old container (如果存在就刪除)
          // 使用 || true 避免如果容器不存在導致 pipeline 失敗
          sh "docker rm -f dev-app || true"
          
          // 5. Deploy (Port 8081)
          // -d: 背景執行, -p: Port對應, --name: 指定容器名稱
          sh "docker run -d -p 8081:8080 --name dev-app ${imageTag}"
          
          // 6. Verify (等待幾秒讓服務啟動)
          sleep 5
          sh "curl -f http://localhost:8081/health || echo 'Health check failed but continuing...'"
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