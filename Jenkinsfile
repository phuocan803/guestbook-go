pipeline {
    agent any

    tools {
        go 'go1.20'
    }

    environment {
        AWS_REGION            = 'ap-southeast-1'
        EKS_CLUSTER_NAME      = 'prod-guestbook-cluster'
        DOCKERHUB_CREDENTIALS = 'docker-registry-credentials'
        DOCKER_IMAGE          = 'phuocan803/guestbook'
        GITHUB_CREDENTIALS    = 'github-credentials'
        AWS_CREDENTIALS_ID    = 'aws-credentials'
    }

    stages {
        stage('Source'){
            steps {
                echo "Checking out repo"
                git url: 'https://github.com/phuocan803/guestbook-go.git', branch: 'master',
                credentialsId: "${GITHUB_CREDENTIALS}"
            }
        }
        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool 'sonar-scanner'
                    withSonarQubeEnv('sonar-server') {
                        sh "${scannerHome}/bin/sonar-scanner"
                    }
                }
            }
        }
        stage('Build'){
            steps {
                script {
                    echo "Starting docker build"
                    sh "docker build -t ${DOCKER_IMAGE}:v3.${env.BUILD_ID} ."
                    sh "docker tag ${DOCKER_IMAGE}:v3.${env.BUILD_ID} ${DOCKER_IMAGE}:latest"
                    echo "Docker built successfully"
                }
            }
        }
        stage('Trivy Image Scan'){
            steps {
                script {
                    echo "Starting Trivy scan"
                    sh "trivy image ${DOCKER_IMAGE}:v3.${env.BUILD_ID}"
                    echo "Trivy scan completed"
                }
            }
        }
        stage('Push Image'){
            steps {
                echo "Pushing to docker hub"
                script {
                    docker.withRegistry('https://registry.hub.docker.com', DOCKERHUB_CREDENTIALS) {
                        docker.image("${DOCKER_IMAGE}:v3.${env.BUILD_ID}").push()
                        docker.image("${DOCKER_IMAGE}:latest").push()
                    }
                }
                echo "Done"
            }
        }
        stage('Deploy to Amazon EKS'){
            steps {
                script {
                    sh "sed -i 's|image: .*|image: ${DOCKER_IMAGE}:v3.${env.BUILD_ID}|' guestbook-controller.yaml"
                    
                    withCredentials([[
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: "${AWS_CREDENTIALS_ID}"
                    ]]) {
                        sh """
                            aws eks --region ${AWS_REGION} update-kubeconfig --name ${EKS_CLUSTER_NAME}
                            kubectl apply -f redis-master-controller.yaml
                            kubectl apply -f redis-master-service.yaml
                            kubectl apply -f redis-replica-controller.yaml
                            kubectl apply -f redis-replica-service.yaml
                            kubectl apply -f guestbook-controller.yaml
                            kubectl apply -f guestbook-service.yaml
                            kubectl rollout status deployment/guestbook --timeout=300s
                        """
                    }
                    echo "Deployment to EKS completed successfully"
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}
