
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        skipDefaultCheckout(true)
    }

    environment {
        // AWS configuration
        AWS_REGION = 'us-west-1'
        AWS_DEFAULT_REGION = 'us-west-1'
        AWS_EC2_METADATA_DISABLED = 'true'

        AWS_ROLE_ARN = 'arn:aws:iam::060255765406:role/jenkins-oidc-deployer'

        // ECR configuration
        ECR_REPOSITORY = 'hello-world'
        ECR_REGISTRY = '060255765406.dkr.ecr.us-west-1.amazonaws.com'

        // EKS and Helm configuration
        EKS_CLUSTER = 'tc2-eks'
        K8S_NAMESPACE = 'default'
        HELM_RELEASE = 'hello-world'
        HELM_CHART = './k8s'

        // Security scanning: 0 = report, 1 = block on findings
        TRIVY_EXIT_CODE = '0'
    }

    stages {

        stage('Checkout GitHub Repository') {
            steps {
                deleteDir()

                git branch: 'main',
                    url: 'https://github.com/skysamuels-art/eks-project.git'

                sh '''
                    set -eu
                    echo "GitHub checkout successful."
                    git log -1 --oneline
                '''
            }
        }

        stage('Verify Project Files and Tools') {
            steps {
                sh '''
                    set -eu

                    command -v git
                    command -v sonar-scanner
                    command -v trivy
                    command -v docker
                    command -v aws
                    command -v helm
                    command -v kubectl

                    if [ ! -f Dockerfile ]; then
                        echo "ERROR: Dockerfile not found in repository root."
                        exit 1
                    fi

                    if [ ! -f "$HELM_CHART/Chart.yaml" ]; then
                        echo "ERROR: Helm chart not found at $HELM_CHART"
                        echo "Available chart files:"
                        find . -name Chart.yaml
                        exit 1
                    fi

                    echo "Required files and tools are present."
                '''
            }
        }

        stage('SonarQube Code Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                        set -eu

                        echo "Starting SonarQube analysis..."

                        sonar-scanner \
                            -Dsonar.projectKey=flask-eks-app \
                            -Dsonar.sources=. \
                            -Dsonar.exclusions=terraform/**,k8s/**,**/.terraform/**,**/__pycache__/**

                        echo "SonarQube analysis submitted."
                    '''
                }
            }
        }

        stage('Trivy Filesystem Scan') {
            steps {
                sh '''
                    set -eu

                    echo "Scanning source code and dependencies..."

                    trivy fs \
                        --scanners vuln,secret \
                        --severity HIGH,CRITICAL \
                        --exit-code "$TRIVY_EXIT_CODE" \
                        --skip-dirs .git \
                        --skip-dirs .terraform \
                        .

                    echo "Filesystem scan completed."
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    set -eu

                    echo "Building Flask Docker image..."

                    docker build \
                        -t "$ECR_REGISTRY/$ECR_REPOSITORY:$BUILD_NUMBER" \
                        .

                    echo "Docker build successful."
                '''
            }
        }

        stage('Trivy Docker Image Scan') {
            steps {
                sh '''
                    set -eu

                    echo "Scanning Docker image..."

                    trivy image \
                        --scanners vuln \
                        --severity HIGH,CRITICAL \
                        --exit-code "$TRIVY_EXIT_CODE" \
                        "$ECR_REGISTRY/$ECR_REPOSITORY:$BUILD_NUMBER"

                    echo "Container image scan completed."
                '''
            }
        }

        stage('OIDC Authentication and ECR Push') {
            steps {
                withCredentials([
                    string(
                        credentialsId: 'aws-oidc-token',
                        variable: 'OIDC_TOKEN'
                    )
                ]) {
                    sh '''#!/usr/bin/env bash
set +x
set -euo pipefail
umask 077

TOKEN_FILE=$(mktemp)
DOCKER_CONFIG=$(mktemp -d)

trap 'rm -f "$TOKEN_FILE"; rm -rf "$DOCKER_CONFIG"' EXIT

printf '%s' "$OIDC_TOKEN" > "$TOKEN_FILE"

export AWS_WEB_IDENTITY_TOKEN_FILE="$TOKEN_FILE"
export DOCKER_CONFIG

unset AWS_ACCESS_KEY_ID
unset AWS_SECRET_ACCESS_KEY
unset AWS_SESSION_TOKEN
unset AWS_PROFILE

echo "Authenticating to AWS using OpenID Connect..."

aws sts get-caller-identity

echo "Logging into Amazon ECR..."

aws ecr get-login-password \
    --region "$AWS_REGION" |
    docker login \
        --username AWS \
        --password-stdin "$ECR_REGISTRY"

echo "Pushing Docker image to ECR..."

docker push \
    "$ECR_REGISTRY/$ECR_REPOSITORY:$BUILD_NUMBER"

echo "Docker image pushed successfully."
'''
                }
            }
        }

        stage('Deploy to Amazon EKS') {
            steps {
                withCredentials([
                    string(
                        credentialsId: 'aws-oidc-token',
                        variable: 'OIDC_TOKEN'
                    )
                ]) {
                    sh '''#!/usr/bin/env bash
set +x
set -euo pipefail
umask 077

TOKEN_FILE=$(mktemp)
trap 'rm -f "$TOKEN_FILE"' EXIT

printf '%s' "$OIDC_TOKEN" > "$TOKEN_FILE"

export AWS_WEB_IDENTITY_TOKEN_FILE="$TOKEN_FILE"
export KUBECONFIG="$WORKSPACE/.kubeconfig"

unset AWS_ACCESS_KEY_ID
unset AWS_SECRET_ACCESS_KEY
unset AWS_SESSION_TOKEN
unset AWS_PROFILE

echo "Connecting to Amazon EKS..."

aws eks update-kubeconfig \
    --region "$AWS_REGION" \
    --name "$EKS_CLUSTER"

echo "Deploying Flask application with Helm..."

helm upgrade --install \
    "$HELM_RELEASE" "$HELM_CHART" \
    --namespace "$K8S_NAMESPACE" \
    --set-string image.repository="$ECR_REGISTRY/$ECR_REPOSITORY" \
    --set-string image.tag="$BUILD_NUMBER" \
    --wait \
    --timeout 10m

echo "Checking Kubernetes deployment..."

kubectl get deployments,pods \
    -n "$K8S_NAMESPACE"

echo "EKS deployment completed."
'''
                }
            }
        }
    }

    post {
        success {
            echo 'SUCCESS: CI/CD pipeline completed!'
            echo 'GitHub, SonarQube, Trivy, Docker, ECR and EKS stages succeeded.'
        }

        failure {
            echo 'FAILED: Check Jenkins Console Output for the failing stage.'
        }

        always {
            sh 'rm -f "$WORKSPACE/.kubeconfig"'
        }
    }
}
