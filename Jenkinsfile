pipeline {
    agent any

    environment {
        IMAGE_NAME = "cicd-stackly:${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Test') {
            steps {
                sh '''
                    docker run --rm \
                      -v "$WORKSPACE:/app" \
                      -w /app \
                      python:3.12-slim \
                      sh -c "pip install --no-cache-dir -r requirements.txt && python -m unittest discover -s tests -v"
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t ${IMAGE_NAME} .'
            }
        }

        stage('Trivy Security Scan') {
            steps {
                sh '''
                    mkdir -p trivy-reports
                    trivy image --format json \
                      --output trivy-reports/trivy-${BUILD_NUMBER}.json \
                      ${IMAGE_NAME}
                '''
            }
        }

        stage('Deploy') {
            steps {
                sh 'docker compose up -d --build'
            }
        }

        stage('Health Check') {
            steps {
                sh '''
                    sleep 5
                    curl -f http://localhost:5000/health
                '''
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'trivy-reports/*.json',
                             allowEmptyArchive: true
        }
    }
}
