pipeline {
    agent any

    environment {
        DOCKERHUB_CREDENTIALS_ID = 'dockerhub_credentials'
        DOCKERHUB_USER = 'zeesha345'
        IMAGE_TAG = "${env.BRANCH_NAME}-${BUILD_NUMBER}"

        // Assign dynamic ports only for dev
        MONGO_PORT     = "${env.BRANCH_NAME == 'dev' ? "27${BUILD_NUMBER}" : "27017"}"
        BACKEND_PORT   = "${env.BRANCH_NAME == 'dev' ? "50${BUILD_NUMBER}" : "5000"}"
        FRONTEND_PORT  = "${env.BRANCH_NAME == 'dev' ? "30${BUILD_NUMBER}" : "3000"}"
    }

    stages {

        stage('Checkout Source') {
            steps { checkout scm }
        }

        stage('Docker Hub Login') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: "${DOCKERHUB_CREDENTIALS_ID}",
                    usernameVariable: 'DH_USER',
                    passwordVariable: 'DH_PASS'
                )]) {
                    sh 'echo $DH_PASS | docker login -u $DH_USER --password-stdin'
                }
            }
        }

        stage('Build & Tag Images') {
            steps {
                script {
                    env.FRONTEND_TAG_DH = "${DOCKERHUB_USER}/three-tier-app-frontend:${IMAGE_TAG}"
                    env.BACKEND_TAG_DH  = "${DOCKERHUB_USER}/three-tier-app-backend:${IMAGE_TAG}"

                    sh """
                        docker build -t ${BACKEND_TAG_DH} ./backend
                        docker build -t ${FRONTEND_TAG_DH} ./frontend
                    """
                }
            }
        }

        stage('Push Images to Docker Hub') {
            steps {
                sh """
                    docker push ${BACKEND_TAG_DH}
                    docker push ${FRONTEND_TAG_DH}
                """
            }
        }

        stage('Prepare .env for Compose') {
            steps {
                script {
                    writeFile file: '.env', text: """
BACKEND_IMAGE=${BACKEND_TAG_DH}
FRONTEND_IMAGE=${FRONTEND_TAG_DH}
ENV=${env.BRANCH_NAME}

MONGO_PORT=${MONGO_PORT}
BACKEND_PORT=${BACKEND_PORT}
FRONTEND_PORT=${FRONTEND_PORT}
"""
                }
            }
        }

        stage('Approval for Staging / Prod Deploy') {
            when {
                anyOf { branch 'stg'; branch 'prod' }
            }
            steps {
                input message: "Deploy to ${env.BRANCH_NAME} environment?", ok: "Yes, Deploy"
            }
        }

        stage('Clean Previous Containers (Port Conflicts Fix)') {
            steps {
                sh """
                    # Remove containers using dev ports
                    docker ps -aq --filter "publish=${MONGO_PORT}" | xargs -r docker rm -f
                    docker ps -aq --filter "publish=${BACKEND_PORT}" | xargs -r docker rm -f
                    docker ps -aq --filter "publish=${FRONTEND_PORT}" | xargs -r docker rm -f

                    # Bring compose down
                    docker-compose --env-file .env down --remove-orphans || true
                """
            }
        }

        stage('Deploy Environment') {
            steps {
                sh """
                    docker-compose --env-file .env pull
                    docker-compose --env-file .env up -d --remove-orphans
                """
            }
        }

        stage('Cleanup Local Images') {
            steps {
                sh "docker rmi ${BACKEND_TAG_DH} ${FRONTEND_TAG_DH} || true"
            }
        }
    }

    post {
        success {
            echo "🚀 Successfully deployed ${env.BRANCH_NAME} environment using Docker Hub images!"
        }
        failure {
            echo "❌ Deployment failed for ${env.BRANCH_NAME}. Check logs."
        }
    }
}
