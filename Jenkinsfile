
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        AWS_REGION = 'us-west-1'
        AWS_DEFAULT_REGION = 'us-west-1'
        AWS_ROLE_ARN = 'arn:aws:iam::060255765406:role/jenkins-eks-deployer'
        AWS_EC2_METADATA_DISABLED = 'true'

        EKS_CLUSTER = 'tc2-eks'
        ECR_REPOSITORY = 'hello-world'
        ECR_REGISTRY = '060255765406.dkr.ecr.us-west-1.amazonaws.com'

        K8S_NAMESPACE = 'default'
        HELM_RELEASE = 'hello-world'
        HELM_CHART = './k8s'
    }

    stages {

        stage('Checkout Source Code') {
            steps {
                checkout scm
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool 'SonarScanner'

                    withSonarQubeEnv('SonarQube') {
                        sh """
                            "${scannerHome}/bin/sonar-scanner" \
                            -Dsonar.projectKey=flask-eks-app \
                            -Dsonar.sources=. \
                            -Dsonar.exclusions=terraform/**,k8s/**,**/.terraform/**
                        """
                    }
                }
            }
        }

        stage('Trivy Filesystem Scan') {
            steps {
                sh '''
                    trivy fs \
                    --scanners vuln,secret \
                    --severity HIGH,CRITICAL \
                    --exit-code 0 \
                    .
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build \
                    -t "$ECR_REGISTRY/$ECR_REPOSITORY:$BUILD_NUMBER" .
                '''
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh '''
                    trivy image \
                    --scanners vuln \
                    --severity HIGH,CRITICAL \
                    --exit-code 0 \
                    "$ECR_REGISTRY/$ECR_REPOSITORY:$BUILD_NUMBER"
                '''
            }
        }

        stage('Push Docker Image to ECR') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'aws-oidc-token',
                        variable: 'AWS_WEB_IDENTITY_TOKEN_FILE'
                    )
                ]) {
                    sh '''
                        set -e

                        aws sts get-caller-identity

                        aws ecr get-login-password \
                        --region "$AWS_REGION" |
                        docker login \
                        --username AWS \
                        --password-stdin "$ECR_REGISTRY"

                        docker push \
                        "$ECR_REGISTRY/$ECR_REPOSITORY:$BUILD_NUMBER"
                    '''
                }
            }
        }

        stage('Deploy to Amazon EKS') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'aws-oidc-token',
                        variable: 'AWS_WEB_IDENTITY_TOKEN_FILE'
                    )
                ]) {
                    sh '''
                        set -e

                        export KUBECONFIG="$WORKSPACE/.kubeconfig"

                        aws eks update-kubeconfig \
                        --region "$AWS_REGION" \
                        --name "$EKS_CLUSTER"

                        helm upgrade --install \
                        "$HELM_RELEASE" "$HELM_CHART" \
                        --namespace "$K8S_NAMESPACE" \
                        --set-string image.repository="$ECR_REGISTRY/$ECR_REPOSITORY" \
                        --set-string image.tag="$BUILD_NUMBER" \
                        --wait \
                        --timeout 10m

                        kubectl get pods -n "$K8S_NAMESPACE"
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'Flask application deployment completed successfully!'
        }

        failure {
            echo 'Pipeline failed. Check Jenkins Console Output.'
        }

        always {
            sh 'rm -f "$WORKSPACE/.kubeconfig"'
        }
    }
}
