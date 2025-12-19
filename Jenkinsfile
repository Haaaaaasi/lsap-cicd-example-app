pipeline {
  agent any

  environment {
    DISCORD_WEBHOOK = credentials('discord_webhook')
    YOUR_NAME = '潘俊諺'
    YOUR_STUDENT_ID = 'B11303140'
  }

  stages {
    // 1. 加回 Checkout 階段
    // 之前因為 API 限制報錯，但為了要有 package.json，這一步是必須的。
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Static Analysis') {
      steps {
        script {
          // 2. 為了 Debug，再次列出檔案，確認這次有抓到 package.json
          sh 'ls -la'
          
          // 3. 執行 Docker 任務
          // 這裡的 $(pwd) 會對應到剛剛 checkout 下來的程式碼目錄
          sh """
            docker run --rm -v \$(pwd):/app -w /app node:18-alpine \
            sh -c 'npm install && npm run lint'
          """
        }
      }
    }
  }

  post {
    // 無論成功或失敗都發送通知
    always { 
      script {
        // 嘗試抓取 Git URL，如果環境變數是 null，使用預設值 (避免 Discord 顯示 null)
        def repoUrl = env.GIT_URL ?: 'https://github.com/Haaaaaasi/lsap-cicd-example-app.git'
        
        def errorMsg = """
${currentBuild.currentResult == 'SUCCESS' ? '✅ Build Success' : '❌ Build Failed'}
Name: ${env.YOUR_NAME}
Student ID: ${env.YOUR_STUDENT_ID}
Job: ${env.JOB_NAME}
Build: ${env.BUILD_NUMBER}
Repo: ${repoUrl}
Branch: ${env.BRANCH_NAME}
Status: ${currentBuild.currentResult}
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