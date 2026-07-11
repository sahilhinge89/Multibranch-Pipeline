pipeline {
    agent any

    environment {
        IMAGE_NAME = "my-app:${env.BRANCH_NAME}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }


      
    post {
        always {
            echo "Pipeline finished for branch: ${env.BRANCH_NAME}"
        }
    }
}
