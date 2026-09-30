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
                sh '''
                    echo "Deploying image: ${IMAGE_NAME}"
                    IMAGE_NAME=${IMAGE_NAME} docker compose up -d
                '''
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
      success {
        sh '''
            echo "Marking successful image as stable..."
            docker tag ${IMAGE_NAME} cicd-stackly:stable
            echo "Stable image is now: ${IMAGE_NAME}"
        '''
    }

    failure {
        sh '''
            echo "Pipeline failed. Checking for stable image..."

            if docker image inspect cicd-stackly:stable >/dev/null 2>&1; then
                echo "Rolling back to stable image: cicd-stackly:stable"

                IMAGE_NAME=cicd-stackly:stable docker compose up -d

                echo "Waiting for rollback deployment..."

                for i in $(seq 1 12); do
                    STATUS=$(docker inspect --format='{{.State.Health.Status}}' cicd-stackly-app 2>/dev/null || true)

                    echo "Rollback health status: $STATUS"

                    if [ "$STATUS" = "healthy" ]; then
                        echo "Rollback successful. Application is healthy."
                        exit 0
                    fi

                    sleep 5
                done

                echo "Rollback failed: application did not become healthy."
                exit 1
            else
                echo "No stable image available. Rollback cannot be performed."
            fi
        '''
    }

    always {
        archiveArtifacts artifacts: 'trivy-reports/*.json',
                         allowEmptyArchive: true
    }
  }
}
