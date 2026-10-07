pipeline {
    agent any

    environment {
        AWS_REGION = 'ap-southeast-2'
        AWS_ACCOUNT_ID = '075810104493'
        ECR_REPO = 'boutique-frontend'
        EKS_CLUSTER = 'devops-project1'
        NAMESPACE = 'boutique'

        IMAGE_URI = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}"
        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Frontend Image') {
            steps {
                sh '''
                    echo "Building frontend image..."
                    docker build \
                      -t ${IMAGE_URI}:${IMAGE_TAG} \
                      ./src/frontend
                '''
            }
        }

        stage('Trivy Security Scan') {
            steps {
                sh '''
                    echo "Running Trivy security scan..."

                    trivy image \
                      --severity HIGH,CRITICAL \
                      --exit-code 1 \
                      ${IMAGE_URI}:${IMAGE_TAG}
                '''
            }
        }

        stage('Login to ECR') {
            steps {
                sh '''
                    echo "Logging into Amazon ECR..."

                    aws ecr get-login-password \
                      --region ${AWS_REGION} | \
                    docker login \
                      --username AWS \
                      --password-stdin \
                      ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
                '''
            }
        }

        stage('Push Image to ECR') {
            steps {
                sh '''
                    echo "Pushing image to ECR..."

                    docker push ${IMAGE_URI}:${IMAGE_TAG}
                '''
            }
        }

        stage('Deploy with Helm') {
            steps {
                sh '''
                    echo "Updating EKS kubeconfig..."

                    aws eks update-kubeconfig \
                      --region ${AWS_REGION} \
                      --name ${EKS_CLUSTER}

                    echo "Deploying with Helm..."

                    helm upgrade --install boutique ./helm-chart \
                      --namespace ${NAMESPACE} \
                      --create-namespace \
                      -f helm-aws-values.yaml \
                      --set frontend.image.repository=${IMAGE_URI} \
                      --set frontend.image.tag=${IMAGE_TAG}
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                    echo "Checking frontend rollout..."

                    kubectl rollout status \
                      deployment/frontend \
                      -n ${NAMESPACE} \
                      --timeout=5m

                    echo "Frontend image:"

                    kubectl get deployment frontend \
                      -n ${NAMESPACE} \
                      -o jsonpath='{.spec.template.spec.containers[0].image}'

                    echo ""

                    echo "Helm release:"

                    helm list -n ${NAMESPACE}
                '''
            }
        }
    }

    post {
        success {
            echo 'CI/CD pipeline completed successfully!'
        }

        failure {
            echo 'CI/CD pipeline failed. Check the stage logs.'
        }
    }
}
