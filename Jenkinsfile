pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
        disableConcurrentBuilds()
    }

    environment {
        AWS_REGION = 'us-west-1'
        ECR_REGISTRY = '060255765406.dkr.ecr.us-west-1.amazonaws.com'
        ECR_REPOSITORY = '060255765406.dkr.ecr.us-west-1.amazonaws.com/hello-world'
        EKS_CLUSTER = 'tc2-eks'
    }

    stages {
        stage('Download application') {
            steps {
                checkout scm
            }
        }

        stage('Check tools and chart') {
            steps {
                sh '''
                    docker version
                    aws --version
                    kubectl version --client
                    helm version
                    helm lint ./k8s
                '''
            }
        }

        stage('Build Docker image') {
            steps {
                sh '''
                    docker build -t "$ECR_REPOSITORY:jenkins-$BUILD_NUMBER" .
                '''
            }
        }

        stage('Sign in to ECR') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'tech2-aws'
                ]]) {
                    sh '''
                        set +x
                        aws sts get-caller-identity

                        ECR_PASSWORD=$(aws ecr get-login-password --region "$AWS_REGION")
                        printf '%s' "$ECR_PASSWORD" |
                          docker login --username AWS --password-stdin "$ECR_REGISTRY"
                        unset ECR_PASSWORD
                    '''
                }
            }
        }

        stage('Upload image to ECR') {
            steps {
                sh '''
                    docker push "$ECR_REPOSITORY:jenkins-$BUILD_NUMBER"
                '''
            }
        }

        stage('Deploy to EKS') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'tech2-aws'
                ]]) {
                    sh '''
                        aws eks update-kubeconfig \
                          --region "$AWS_REGION" \
                          --name "$EKS_CLUSTER"

                        helm upgrade --install hello-world ./k8s \
                          --namespace default \
                          --set-string image.repository="$ECR_REPOSITORY" \
                          --set-string image.tag="jenkins-$BUILD_NUMBER" \
                          --wait \
                          --timeout 10m

                        kubectl rollout status deployment/hello-world \
                          --namespace default \
                          --timeout=300s
                    '''
                }
            }
        }
    }
}