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
      steps {
        sh 'npm ci || npm install'
        sh 'npm run lint'
      }
    }
  }

  post {
    failure {
      sh """
      curl -X POST -H 'Content-type: application/json' \
      --data '{
        "content": "❌ Build Failed\\nName: ${YOUR_NAME}\\nStudent ID: ${YOUR_STUDENT_ID}\\nJob: ${JOB_NAME}\\nBuild: ${BUILD_NUMBER}\\nRepo: ${GIT_URL}\\nBranch: ${BRANCH_NAME}\\nStatus: ${currentBuild.currentResult}"
      }' ${DISCORD_WEBHOOK}
      """
    }
  }
}
