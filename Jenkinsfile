pipeline {
    agent { label 'ec2-dev' }

    environment {
        DOCKERHUB_CREDENTIALS_ID = 'dockerhub-credentials'
        BRANCH_NAME = 'dev'
        IMAGE_TAG   = "dev-${BUILD_NUMBER}"
        DOCKERHUB_USER = 'zeesha345'
    }

    options { 
        skipStagesAfterUnstable()
    }

    stages {

        stage('Checkout Source') { 
            steps { 
                checkout scm
            } 
        }

        stage('Docker Hub Login') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: "${DOCKERHUB_CREDENTIALS_ID}",
                    usernameVariable: 'DOCKERHUB_USER',
                    passwordVariable: 'DOCKERHUB_PASS'
                )]) {
                    sh 'echo $DOCKERHUB_PASS | docker login -u $DOCKERHUB_USER --password-stdin'
                    script { env.DOCKERHUB_USER = "${DOCKERHUB_USER}" }
                }
            }
        }

        stage('Build & Tag Images') {
            steps {
                script {
                    env.BACKEND_TAG_DH = "${DOCKERHUB_USER}/three-tier-app-backend:${IMAGE_TAG}"
                    env.FRONTEND_TAG_DH = "${DOCKERHUB_USER}/three-tier-app-frontend:${IMAGE_TAG}"
                }
                sh '''
                    docker build -t ${BACKEND_TAG_DH} ./backend
                    docker build -t ${FRONTEND_TAG_DH} ./frontend
                '''
            }
        }

        stage('Push Images to Docker Hub') {
            steps {
                sh '''
                    docker push ${BACKEND_TAG_DH}
                    docker push ${FRONTEND_TAG_DH}
                '''
            }
        }

        stage('Prepare .env') {
            steps {
                writeFile(
                    file: '.env',
                    text: """BACKEND_IMAGE=${BACKEND_TAG_DH}
FRONTEND_IMAGE=${FRONTEND_TAG_DH}
ENV=dev
BACKEND_PORT=5003
FRONTEND_PORT=3003
"""
                )
            }
        }

        stage('Deploy Dev') {
            steps {
                sh '''
                    docker-compose -f docker-compose.yml --env-file .env down
                    docker-compose -f docker-compose.yml --env-file .env pull
                    docker-compose -f docker-compose.yml --env-file .env up -d --remove-orphans
                '''
            }
        }

        stage('Cleanup') {
            steps {
                sh 'docker rmi ${BACKEND_TAG_DH} ${FRONTEND_TAG_DH} || true'
            }
        }
    }

    post {
        success { echo "✅ Dev deployed: ${IMAGE_TAG}" }
        failure { echo "❌ Dev deployment failed." }
    }
}
