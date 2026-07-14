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
<<<<<<< HEAD
    }
=======

    

        
>>>>>>> 3b03adcbeb3bfc05de2c1bd9f68bca7f468d0613

    post {
        always {
            echo "Pipeline finished for branch: ${env.BRANCH_NAME}"
        }
    }
}
