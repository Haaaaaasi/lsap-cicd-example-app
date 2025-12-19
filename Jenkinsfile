pipeline {
  agent any

  // 關鍵修正 1：告訴 Jenkins 不要自動 Checkout，避免與後面的步驟衝突
  options {
    skipDefaultCheckout()
  }

  environment {
    DISCORD_WEBHOOK = credentials('discord_webhook')
    YOUR_NAME = '潘俊諺'
    YOUR_STUDENT_ID = 'B11303140'
  }

  stages {
    stage('Clean Workspace') {
      steps {
        // 關鍵修正 2：確保工作區是乾淨的，刪除上一殘留的爛檔案
        // 這相當於 rm -rf *
        deleteDir() 
      }
    }

    stage('Checkout') {
      steps {
        script {
           // 關鍵修正 3：手動執行 Checkout
           // 因為我們關掉了 skipDefaultCheckout，這一步現在是唯一的 Git 操作
           checkout scm
           
           // 確認檔案真的有抓下來
           sh 'ls -la' 
        }
      }
    }

    stage('Static Analysis') {
      steps {
        script {
          // 再次檢查 package.json 是否存在 (雙重確認)
          sh 'ls -la package.json || echo "Still missing package.json?"'

          // 執行 Docker 任務
          // 注意：現在 Workspace 應該是乾淨且包含代碼的
          sh """
            docker run --rm -v \$(pwd):/app -w /app node:18-alpine \
            sh -c 'npm install && npm run lint'
          """
        }
      }
    }
  }

  post {
    always { 
      script {
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