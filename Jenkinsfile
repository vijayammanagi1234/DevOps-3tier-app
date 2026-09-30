pipeline {
    agent any

    environment {
        JAVA_HOME = '/usr/lib/jvm/java-21-openjdk-amd64'
        PATH = "${JAVA_HOME}/bin:${env.PATH}"

        AWS_ACCOUNT_ID = '072672872821'
        AWS_REGION     = 'ap-south-1'
        ECR_REGISTRY   = "${env.AWS_ACCOUNT_ID}.dkr.ecr.${env.AWS_REGION}.amazonaws.com"
        IMAGE_TAG      = "build-${env.BUILD_NUMBER}"
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
                    echo "JAVA_HOME=$JAVA_HOME"
                    java -version
                    mvn -version
                '''
            }
        }

        stage('Build Backend') {
            steps {
                dir('back-end-app') {
                    sh 'mvn clean package -DskipTests'
                }
            }
        }

        stage('Build Docker Images') {
            steps {
                sh 'docker build -t backend-app ./back-end-app'
                sh 'docker build -t frontend-app ./front-end-app'
            }
        }

        stage('Push to ECR') {
            steps {
                sh '''
                    export AWS_EC2_METADATA_DISABLED=true
                    aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}
                    docker tag backend-app ${ECR_REGISTRY}/backend-app:latest
                    docker tag backend-app ${ECR_REGISTRY}/backend-app:${IMAGE_TAG}
                    docker push ${ECR_REGISTRY}/backend-app:latest
                    docker push ${ECR_REGISTRY}/backend-app:${IMAGE_TAG}
                    docker tag frontend-app ${ECR_REGISTRY}/frontend-app:latest
                    docker tag frontend-app ${ECR_REGISTRY}/frontend-app:${IMAGE_TAG}
                    docker push ${ECR_REGISTRY}/frontend-app:latest
                    docker push ${ECR_REGISTRY}/frontend-app:${IMAGE_TAG}
                '''
            }
        }

        stage('Deploy Containers') {
            steps {
                sh 'docker compose down || true'
                sh 'docker compose up -d --build'
            }
        }
        
        stage('Cleanup Local Images') {
            steps {
                sh 'docker image prune -f || true'
            }
        }

        stage('Update GitOps Manifests') {
            steps {
                withCredentials([string(credentialsId: 'github-token-creds', variable: 'GITHUB_TOKEN')]) {
                    sh '''
                        git config --global user.email "jenkins@devops.com"
                        git config --global user.name "Jenkins CI"
                        
                        git remote set-url origin https://muthunsuman:${GITHUB_TOKEN}@github.com/muthunsuman/DevOps-end-to-end-project.git
                        git pull origin main
                        
                        sed -i 's|image: 433985779049.dkr.ecr.ap-southeast-2.amazonaws.com/backend-app:.*|image: 433985779049.dkr.ecr.ap-southeast-2.amazonaws.com/backend-app:build-${BUILD_NUMBER}|g' kubernetes/dev/backend.yaml
                        sed -i 's|image: 433985779049.dkr.ecr.ap-southeast-2.amazonaws.com/frontend-app:.*|image: 433985779049.dkr.ecr.ap-southeast-2.amazonaws.com/frontend-app:build-${BUILD_NUMBER}|g' kubernetes/dev/fronttend.yaml
                        
                        git add kubernetes/dev/backend.yaml kubernetes/dev/fronttend.yaml
                        git commit -m "CI: Update image tags to ${IMAGE_TAG}"
                        git push origin main
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
