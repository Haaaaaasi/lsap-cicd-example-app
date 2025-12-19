pipeline {
  agent any

  environment {
    // 確保這裡的 ID 對應到你 Jenkins 裡設定的 ID
    DISCORD_WEBHOOK = credentials('discord_webhook')
    YOUR_NAME = '潘俊諺'
    YOUR_STUDENT_ID = 'B11303140'
  }

  stages {
    // 1. 標準 Checkout
    // 在新建立的 Job 中，這一步會正常運作
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    // 2. 靜態分析 (Part 1)
    stage('Static Analysis') {
      steps {
        script {
          // Debug: 確保 package.json 存在
          sh 'ls -la'
          
          // 使用 Docker 執行 Lint
          // 因為我們已經修復了 Docker 權限，這裡會成功下載 image 並執行
          sh """
            docker run --rm -v \$(pwd):/app -w /app node:18-alpine \
            sh -c 'npm install && npm run lint'
          """
        }
      }
    }
  }

  // 3. 通知 (Part 1 ChatOps)
  post {
    always { 
      script {
        // 取得 Git URL，如果為 null 則使用預設值
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
        // 處理 JSON 換行問題
        def jsonContent = errorMsg.replace("\n", "\\n")

        // 發送 Discord 通知
        sh """
        curl -X POST -H "Content-type: application/json" \
        --data '{"content": "${jsonContent}"}' \
        \$DISCORD_WEBHOOK
        """
      }
    }
  }
}