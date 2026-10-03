pipeline {
    agent any

    environment {
        JAVA_HOME = '/usr/lib/jvm/java-21-openjdk-amd64'
        PATH = "${JAVA_HOME}/bin:${env.PATH}"

        AWS_ACCOUNT_ID = '072672872821'
        AWS_REGION     = 'ap-south-1'

        ECR_REGISTRY   = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"

        BACKEND_ECR    = "${ECR_REGISTRY}/back-end-ecr"
        FRONTEND_ECR   = "${ECR_REGISTRY}/front-end-ecr"

        IMAGE_TAG      = "build-${BUILD_NUMBER}"
    }

    triggers {
        githubPush()
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/vijayammanagi1234/DevOps-3tier-app.git'
            }
        }

        stage('Java Check') {
            steps {
                sh '''
                    set -e

                    echo "===== JAVA ====="
                    echo "JAVA_HOME=$JAVA_HOME"
                    java -version

                    echo "===== MAVEN ====="
                    mvn -version

                    echo "===== AWS CLI ====="
                    aws --version
                '''
            }
        }

        stage('AWS Identity Check') {
            steps {
                sh '''
                    set -e

                    echo "===== AWS REGION ====="
                    echo "$AWS_REGION"

                    echo "===== AWS ACCOUNT ====="
                    aws sts get-caller-identity
                '''
            }
        }

        stage('Build Backend') {
            steps {
                dir('back-end-app') {
                    sh '''
                        set -e
                        mvn clean package -DskipTests
                    '''
                }
            }
        }

        stage('Build Docker Images') {
            steps {
                sh '''
                    set -e

                    docker build -t backend-app ./back-end-app
                    docker build -t frontend-app ./front-end-app

                    echo "===== DOCKER IMAGES ====="
                    docker images | grep -E 'backend-app|frontend-app' || true
                '''
            }
        }

        stage('Push to AWS ECR') {
            steps {
                sh '''
                    set -e

                    echo "===== AWS ACCOUNT ====="
                    aws sts get-caller-identity

                    echo "===== ECR LOGIN ====="

                    aws ecr get-login-password \
                        --region "$AWS_REGION" | \
                    docker login \
                        --username AWS \
                        --password-stdin "$ECR_REGISTRY"

                    echo "===== BACKEND TAG ====="

                    docker tag backend-app \
                        "$BACKEND_ECR:latest"

                    docker tag backend-app \
                        "$BACKEND_ECR:$IMAGE_TAG"

                    echo "===== BACKEND PUSH ====="

                    docker push "$BACKEND_ECR:latest"
                    docker push "$BACKEND_ECR:$IMAGE_TAG"

                    echo "===== FRONTEND TAG ====="

                    docker tag frontend-app \
                        "$FRONTEND_ECR:latest"

                    docker tag frontend-app \
                        "$FRONTEND_ECR:$IMAGE_TAG"

                    echo "===== FRONTEND PUSH ====="

                    docker push "$FRONTEND_ECR:latest"
                    docker push "$FRONTEND_ECR:$IMAGE_TAG"

                    echo "===== ECR PUSH SUCCESS ====="
                '''
            }
        }

        stage('Deploy Containers') {
            steps {
                sh '''
                    set -e

                    docker compose down || true
                    docker compose up -d --build

                    echo "===== CONTAINERS ====="
                    docker compose ps
                '''
            }
        }

        stage('Cleanup Local Images') {
            steps {
                sh '''
                    docker image prune -f || true
                '''
            }
        }

        stage('Update GitOps Manifests') {
            steps {
                withCredentials([
                    string(
                        credentialsId: 'github-token-creds',
                        variable: 'GITHUB_TOKEN'
                    )
                ]) {

                    sh '''
                        set -e

                        echo "===== GIT CONFIG ====="

                        git config user.email "jenkins@devops.com"
                        git config user.name "Jenkins CI"

                        echo "===== GITOPS REPOSITORY ====="

                        git remote set-url origin \
                        "https://muthunsuman:${GITHUB_TOKEN}@github.com/muthunsuman/DevOps-end-to-end-project.git"

                        git pull origin main

                        echo "===== UPDATE BACKEND IMAGE ====="

                        sed -i \
                        "s|image: .*back-end-ecr:.*|image: ${BACKEND_ECR}:${IMAGE_TAG}|g" \
                        kubernetes/dev/backend.yaml

                        echo "===== UPDATE FRONTEND IMAGE ====="

                        sed -i \
                        "s|image: .*front-end-ecr:.*|image: ${FRONTEND_ECR}:${IMAGE_TAG}|g" \
                        kubernetes/dev/frontend.yaml

                        echo "===== VERIFY MANIFESTS ====="

                        grep -n "image:" kubernetes/dev/backend.yaml
                        grep -n "image:" kubernetes/dev/frontend.yaml

                        echo "===== GIT COMMIT ====="

                        git add \
                            kubernetes/dev/backend.yaml \
                            kubernetes/dev/frontend.yaml

                        git commit \
                            -m "CI: Update image tags to ${IMAGE_TAG}" \
                            || echo "No changes to commit"

                        echo "===== GIT PUSH ====="

                        git push origin main

                        echo "===== GITOPS UPDATE SUCCESS ====="
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline Build #${env.BUILD_NUMBER} - Deployment Successful!"
        }

        failure {
            echo "Pipeline Build #${env.BUILD_NUMBER} - Deployment Failed!"
        }
    }
}
