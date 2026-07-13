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

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t $IMAGE_NAME .'
            }
        }
    }

    post {
        always {
            echo "Pipeline finished for branch: ${env.BRANCH_NAME}"
        }
    }
}
