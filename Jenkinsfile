pipeline {
    agent any

    environment {
        AWS_REGION = 'ap-south-1'
        AWS_ACCOUNT_ID = '913436626979'

        ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"

        ECR_AUTH = "${ECR_REGISTRY}/streaming-auth"
        ECR_ADMIN = "${ECR_REGISTRY}/streaming-admin"
        ECR_CHAT = "${ECR_REGISTRY}/streaming-chat"
        ECR_FRONTEND = "${ECR_REGISTRY}/streaming-frontend"
        ECR_STREAMING = "${ECR_REGISTRY}/streaming-service"
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/karan11619/StreamingApp.git'
            }
        }

        stage('Build Docker Images') {
            steps {

                echo 'Preparing authentication Dockerfile for bcrypt...'

                sh '''
                    sed -i 's/RUN npm install --production/RUN apk add --no-cache python3 make g++ \\&\\& npm install --production/' backend/authService/Dockerfile
                '''

                echo 'Building authentication service...'
                sh '''
                    docker build \
                        -t streaming-auth:1.0.0 \
                        backend/authService
                '''

                echo 'Building admin service...'
                sh '''
                    docker build \
                        -t streaming-admin:1.0.0 \
                        -f backend/adminService/Dockerfile \
                        backend
                '''

                echo 'Building chat service...'
                sh '''
                    docker build \
                        -t streaming-chat:1.0.0 \
                        -f backend/chatService/Dockerfile \
                        backend
                '''

                echo 'Building frontend...'
                sh '''
                    docker build \
                        -t streaming-frontend:1.0.2 \
                        frontend
                '''

                echo 'Building streaming service...'
                sh '''
                    docker build \
                        -t streaming-service:1.0.2 \
                        -f backend/streamingService/Dockerfile \
                        backend
                '''
            }
        }

        stage('Login to ECR') {
            steps {

                withCredentials([
                    string(
                        credentialsId: 'aws-access-key-id',
                        variable: 'AWS_ACCESS_KEY_ID'
                    ),
                    string(
                        credentialsId: 'aws-secret-access-key',
                        variable: 'AWS_SECRET_ACCESS_KEY'
                    )
                ]) {

                    sh '''
                        set +x

                        export AWS_DEFAULT_REGION="$AWS_REGION"

                        echo "Verifying AWS identity..."
                        aws sts get-caller-identity

                        echo "Logging in to Amazon ECR..."

                        aws ecr get-login-password \
                            --region "$AWS_REGION" |
                            docker login \
                            --username AWS \
                            --password-stdin "$ECR_REGISTRY"
                    '''
                }
            }
        }

        stage('Tag and Push Images') {
            steps {

                echo 'Tagging images...'

                sh 'docker tag streaming-auth:1.0.0 "$ECR_AUTH:1.0.0"'
                sh 'docker tag streaming-admin:1.0.0 "$ECR_ADMIN:1.0.0"'
                sh 'docker tag streaming-chat:1.0.0 "$ECR_CHAT:1.0.0"'
                sh 'docker tag streaming-frontend:1.0.2 "$ECR_FRONTEND:1.0.2"'
                sh 'docker tag streaming-service:1.0.2 "$ECR_STREAMING:1.0.2"'

                echo 'Pushing authentication service...'
                sh 'docker push "$ECR_AUTH:1.0.0"'

                echo 'Pushing admin service...'
                sh 'docker push "$ECR_ADMIN:1.0.0"'

                echo 'Pushing chat service...'
                sh 'docker push "$ECR_CHAT:1.0.0"'

                echo 'Pushing frontend...'
                sh 'docker push "$ECR_FRONTEND:1.0.2"'

                echo 'Pushing streaming service...'
                sh 'docker push "$ECR_STREAMING:1.0.2"'
            }
        }

        stage('Deploy to EKS') {
            steps {

                withCredentials([
                    string(
                        credentialsId: 'aws-access-key-id',
                        variable: 'AWS_ACCESS_KEY_ID'
                    ),
                    string(
                        credentialsId: 'aws-secret-access-key',
                        variable: 'AWS_SECRET_ACCESS_KEY'
                    )
                ]) {

                    sh '''
                        set +x

                        export AWS_DEFAULT_REGION="$AWS_REGION"

                        echo "Updating EKS kubeconfig..."

                        aws eks update-kubeconfig \
                            --region "$AWS_REGION" \
                            --name streamingapp-cluster

                        echo "Deploying Helm release..."

                        helm upgrade streamingapp ./helm \
                            --install \
                            -n streamingapp \
                            --set image.repository="$ECR_STREAMING" \
                            --set image.tag="1.0.2" \
                            --set auth.image.repository="$ECR_AUTH" \
                            --set auth.image.tag="1.0.0" \
                            --set admin.image.repository="$ECR_ADMIN" \
                            --set admin.image.tag="1.0.0" \
                            --set chat.image.repository="$ECR_CHAT" \
                            --set chat.image.tag="1.0.0" \
                            --set frontend.image.repository="$ECR_FRONTEND" \
                            --set frontend.image.tag="1.0.2"

                        echo "Waiting for auth deployment..."

                        kubectl rollout status deployment/auth \
                            -n streamingapp \
                            --timeout=180s

                        echo "Waiting for admin deployment..."

                        kubectl rollout status deployment/admin \
                            -n streamingapp \
                            --timeout=180s

                        echo "Waiting for chat deployment..."

                        kubectl rollout status deployment/chat \
                            -n streamingapp \
                            --timeout=180s

                        echo "Waiting for frontend deployment..."

                        kubectl rollout status deployment/frontend \
                            -n streamingapp \
                            --timeout=180s

                        echo "Waiting for streaming deployment..."

                        kubectl rollout status deployment/streaming \
                            -n streamingapp \
                            --timeout=180s

                        echo "All deployments rolled out successfully."

                        echo "Current application pods:"
                        kubectl get pods -n streamingapp

                        echo "Current deployments:"
                        kubectl get deployments -n streamingapp
                    '''
                }
            }
        }
    }

    post {

        success {
            echo '========================================'
            echo 'StreamingApp CI/CD pipeline SUCCESS'
            echo '========================================'
        }

        failure {
            echo '========================================'
            echo 'StreamingApp CI/CD pipeline FAILED'
            echo '========================================'
        }
    }
}