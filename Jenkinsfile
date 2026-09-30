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
                    python3 -m pip install --break-system-packages --no-cache-dir -r requirements.txt
                    python3 -m unittest discover -s tests -v
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
                    echo "Waiting for application to become healthy..."

                    for i in $(seq 1 12); do
                        STATUS=$(docker inspect --format='{{.State.Health.Status}}' cicd-stackly-app 2>/dev/null || true)

                        echo "Health status: $STATUS"

                        if [ "$STATUS" = "healthy" ]; then
                            echo "Application is healthy!"
                            exit 0
                        fi

                        sleep 5
                    done

                    echo "Application did not become healthy"
                    exit 1
                '''
            }
        }
              stage('Image Cleanup') {
                steps {
                 sh '''
                    echo "Cleaning old application images..."

                    IMAGES=$(docker images "cicd-stackly" \
                        --format "{{.Repository}}:{{.Tag}}" \
                        | grep -E '^cicd-stackly:[0-9]+$' \
                        | sort -t: -k2,2n)

                    COUNT=$(echo "$IMAGES" | grep -c . || true)

                    if [ "$COUNT" -gt 3 ]; then
                        REMOVE_COUNT=$((COUNT - 3))

                        echo "Removing $REMOVE_COUNT old image(s)..."

                        echo "$IMAGES" | head -n "$REMOVE_COUNT" | while read IMAGE; do
                            echo "Removing old image: $IMAGE"
                            docker rmi "$IMAGE" || true
                        done
                    else
                        echo "No old images need cleanup."
                    fi

                    echo "Image cleanup completed."
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
