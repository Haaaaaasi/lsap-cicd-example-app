pipeline {
  agent any

  environment {
    DISCORD_WEBHOOK = credentials('discord_webhook')
    YOUR_NAME = '潘俊諺'
    YOUR_STUDENT_ID = 'B11303140'
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Static Analysis') {
      steps {
        script {
          // 1. 動態建立一個臨時的 Dockerfile
          // 我們使用 COPY . . 把 Jenkins 當前的程式碼複製進映像檔
          // 這樣就避開了 Volume 掛載路徑不一致的問題
          writeFile file: 'Dockerfile.lint', text: '''
            FROM node:18-alpine
            WORKDIR /app
            COPY . .
            RUN npm install
            CMD ["npm", "run", "lint"]
          '''

          echo "Building Lint Image..."
          // 2. 建置映像檔 (這會把程式碼送給 Docker Daemon)
          sh 'docker build -t lint-image -f Dockerfile.lint .'

          echo "Running Lint Check..."
          // 3. 執行檢查
          sh 'docker run --rm lint-image'

          // 4. 清理映像檔
          sh 'docker rmi lint-image'
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