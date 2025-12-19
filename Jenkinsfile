pipeline {
  agent any

  environment {
    DISCORD_WEBHOOK = credentials('discord_webhook')
    YOUR_NAME = '潘俊諺'
    YOUR_STUDENT_ID = 'B11303140'
  }

  stages {
    // 1. 移除了 'Checkout' stage
    // Declarative Pipeline 預設會自動在 agent any 之後執行 checkout scm
    // 不需要手動再寫一次，這樣可以解決 "Could not determine exact tip revision" 錯誤

    stage('Static Analysis') {
      steps {
        script {
          // 2. 確保目錄存在 (有時候自動 checkout 會有權限問題，先印出來確認)
          sh 'ls -la'
          
          // 3. 使用手動 Docker 指令 (因為你的 Jenkins 沒裝 Docker Pipeline plugin)
          // 這裡加上 --entrypoint 確保不會因為 image 預設指令卡住
          sh """
            docker run --rm -v \$(pwd):/app -w /app node:18-alpine \
            sh -c 'npm install && npm run lint'
          """
        }
      }
    }
  }

  post {
    failure {
      script {
        // 嘗試抓取 Git URL，如果環境變數是 null，嘗試用 scm 設定抓取 (fallback)
        def repoUrl = env.GIT_URL ?: 'https://github.com/Haaaaaasi/lsap-cicd-example-app.git'
        
        def errorMsg = """
❌ Build Failed
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