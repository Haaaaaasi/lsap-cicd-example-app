pipeline {
  agent any

  environment {
    DISCORD_WEBHOOK = credentials('discord_webhook')
    YOUR_NAME = '潘俊諺'
    YOUR_STUDENT_ID = 'B11303140'
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Static Analysis') {
      // 移除 agent { docker ... } 因為你的 Jenkins 不支援
      steps {
        script {
          // 手動啟動一個 node 容器
          // -v $(pwd):/app : 把 Jenkins 目前的工作目錄掛載到容器內的 /app
          // -w /app : 設定容器的工作目錄為 /app
          // node:18-alpine : 使用的映像檔
          // sh -c '...' : 在容器內執行的指令
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
        def errorMsg = """
❌ Build Failed
Name: ${env.YOUR_NAME}
Student ID: ${env.YOUR_STUDENT_ID}
Job: ${env.JOB_NAME}
Build: ${env.BUILD_NUMBER}
Repo: ${env.GIT_URL}
Branch: ${env.BRANCH_NAME}
Status: ${currentBuild.currentResult}
"""
        def jsonContent = errorMsg.replace("\n", "\\n")

        sh """
        curl -X POST -H "Content-type: application/json" \
        --data '{"content": "${jsonContent}"}' \
        ${env.DISCORD_WEBHOOK}
        """
      }
    }
  }
}