pipeline {
    agent any

    environment {
        // 確保你的 Jenkins 憑證庫中有 id 為 'discord_webhook' 的 Secret Text
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
            // 在這個階段啟動 Node.js 的 Docker 容器
            agent {
                docker { 
                    image 'node:18-alpine' 
                    // 為了確保容器能存取 workspace，通常 Jenkins 會自動 mount，
                    // 但使用 alpine 有時要注意 user 權限，這裡先維持預設。
                    args '-u root:root' 
                }
            }
            steps {
                // 使用 npm install 比較保險，因為 npm ci 需要 package-lock.json 嚴格匹配
                sh 'npm install'
                sh 'npm run lint'
            }
        }
    }

    post {
        failure {
            script {
                // 將訊息內容定義為變數，比較容易閱讀與除錯
                // 注意：這裡所有的變數都加上了 env. 前綴
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
                // 為了避免換行符號在 shell 中造成 JSON 格式錯誤，我們手動將換行取代為 \n
                // 這是發送多行訊息到 Discord/Slack 的常見技巧
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