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

        stage('Run Container') {
            steps {
                sh '''
                  docker rm -f my-app-test || true
                  docker run -d --name my-app-test -p 3001:3000 $IMAGE_NAME
                  sleep 2
                  curl -f http://localhost:3001 || exit 1
                '''
            }
        }

        stage('Cleanup') {
            steps {
                sh 'docker rm -f my-app-test || true'
            }
        }
    }

    post {
        always {
            echo "Pipeline finished for branch: ${env.BRANCH_NAME}"
        }
    }
}
