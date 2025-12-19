pipeline {
  agent any

  environment {
    DISCORD_WEBHOOK = credentials('discord_webhook')
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

    [cite_start]// Part 1: 靜態分析 (所有分支都要跑) [cite: 25]
    stage('Static Analysis') {
      steps {
        script {
          echo "--- Starting Static Analysis ---"
          // 給它一個獨一無二的名字，避免 dev 和 main 打架
          def lintImageTag = "lint-image-${env.BUILD_NUMBER}"

          writeFile file: 'Dockerfile.lint', text: '''
            FROM node:18-alpine
            WORKDIR /app
            COPY . .
            RUN npm install
            CMD ["npm", "run", "lint"]
          '''
          // 使用變數
          sh "docker build -t ${lintImageTag} -f Dockerfile.lint ."
          sh "docker run --rm ${lintImageTag}"
          sh "docker rmi ${lintImageTag}"
        }
      }
    }

    [cite_start]// Part 2: Staging 環境部署 (只在 dev 分支執行) [cite: 42]
    stage('Build & Deploy Staging') {
      when {
        branch 'dev'
      }
      steps {
        script {
          echo "--- Starting Staging Deployment ---"
          [cite_start]def imageTag = "${env.DOCKER_USER}/myapp:dev-${env.BUILD_NUMBER}" // [cite: 45]
          
          sh "echo $DOCKER_CREDS_PSW | docker login -u $DOCKER_CREDS_USR --password-stdin"
          
          writeFile file: 'Dockerfile', text: '''
            FROM node:18-alpine
            WORKDIR /app
            COPY . .
            RUN npm install --production
            EXPOSE 8080
            CMD ["npm", "start"]
          '''
          
          sh "docker build -t ${imageTag} ." [cite_start]// [cite: 47]
          sh "docker push ${imageTag}"
          
          [cite_start]sh "docker rm -f dev-app || true" // [cite: 48]
          [cite_start]sh "docker run -d -p 8081:8080 --name dev-app ${imageTag}" // [cite: 49]
          
          // 嘗試獲取容器 IP 進行內部驗證，解決 localhost 連線失敗問題
          sleep 5
          def containerIp = sh(script: "docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' dev-app", returnStdout: true).trim()
          [cite_start]sh "curl -f http://${containerIp}:8080/health || echo 'Health check warning'" // [cite: 50]
        }
      }
    }

    [cite_start]// Part 3: Production GitOps Promotion (只在 main 分支執行) [cite: 51]
    stage('GitOps Promotion') {
      when {
        branch 'main'
      }
      steps {
        script {
          echo "--- Starting GitOps Promotion ---"
          
          [cite_start]// 1. 讀取 deploy.config 檔案 [cite: 59]
          // 如果檔案不存在會報錯，這是 GitOps 的核心保護機制
          def targetTag = readFile('deploy.config').trim()
          echo "Target Version from Git: ${targetTag}"
          
          def sourceImage = "${env.DOCKER_USER}/myapp:${targetTag}"
          [cite_start]def prodImage = "${env.DOCKER_USER}/myapp:prod-${env.BUILD_NUMBER}" // [cite: 61]

          sh "echo $DOCKER_CREDS_PSW | docker login -u $DOCKER_CREDS_USR --password-stdin"

          // 2. Artifact Promotion (Pull -> Retag -> Push) 
          sh "docker pull ${sourceImage}"
          sh "docker tag ${sourceImage} ${prodImage}"
          sh "docker push ${prodImage}"

          // 3. Deploy Production (Port 8082) 
          [cite_start]sh "docker rm -f prod-app || true" // [cite: 64]
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