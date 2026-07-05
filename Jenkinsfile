pipeline {
    agent any

    environment {
        AWS_DEFAULT_REGION    = 'il-central-1'
        AWS_ACCOUNT_ID        = '564430720198'
        ECR_REGISTRY          = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com"
        EKS_CLUSTER_NAME      = 'devops-task-manager'
        AWS_ACCESS_KEY_ID     = credentials('aws-access-key-id')
        AWS_SECRET_ACCESS_KEY = credentials('aws-secret-access-key')
        IMAGE_TAG             = "v${env.BUILD_NUMBER}"
    }

    stages {
        stage('Build images') {
            steps {
                sh """
                    docker build -t ${ECR_REGISTRY}/devops-task-manager-backend:${IMAGE_TAG} ./backend
                    docker build -t ${ECR_REGISTRY}/devops-task-manager-frontend:${IMAGE_TAG} ./frontend
                """
            }
        }

        stage('Push to ECR') {
            steps {
                sh """
                    aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}
                    docker push ${ECR_REGISTRY}/devops-task-manager-backend:${IMAGE_TAG}
                    docker push ${ECR_REGISTRY}/devops-task-manager-frontend:${IMAGE_TAG}
                """
            }
        }

        stage('Deploy to EKS') {
            steps {
                sh """
                    aws eks update-kubeconfig --region ${AWS_DEFAULT_REGION} --name ${EKS_CLUSTER_NAME}
                    kubectl set image deployment/backend backend=${ECR_REGISTRY}/devops-task-manager-backend:${IMAGE_TAG}
                    kubectl set image deployment/frontend frontend=${ECR_REGISTRY}/devops-task-manager-frontend:${IMAGE_TAG}
                    kubectl rollout status deployment/backend --timeout=120s
                    kubectl rollout status deployment/frontend --timeout=120s
                """
            }
        }
    }
}
