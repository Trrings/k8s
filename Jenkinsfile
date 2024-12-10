def DOCKER_REGISTRY = 'trrings-registry'
def GITHUB_REPO_URL = 'https://github.com/Trrings'

pipeline {
    agent any

    tools {
        maven 'Maven3'
        jdk 'JDK17'
    }

    environment {
        GITHUB_CRED = credentials('github-cred')
        DOCKER_CRED = credentials('docker-cred')
    }

    stages {
        stage('Checkout Common') {
            steps {
                dir('trrings-common') {
                    git branch: 'main',
                        url: "${GITHUB_REPO_URL}/trrings-common.git",
                        credentialsId: 'github-cred'
                }
            }
        }

        stage('Build & Install Common') {
            steps {
                dir('trrings-common') {
                    sh 'mvn clean install'
                }
            }
        }

        stage('Build Services') {
            parallel {
                stage('Config Server') {
                    steps {
                        dir('config-server') {
                            git branch: 'main',
                                url: "${GITHUB_REPO_URL}/config-server.git",
                                credentialsId: 'github-cred'
                            sh 'mvn clean package -DskipTests'
                        }
                    }
                }
                stage('Discovery Server') {
                    steps {
                        dir('discovery-server') {
                            git branch: 'main',
                                url: "${GITHUB_REPO_URL}/discovery-server.git",
                                credentialsId: 'github-cred'
                            sh 'mvn clean package -DskipTests'
                        }
                    }
                }
                stage('API Gateway') {
                    steps {
                        dir('api-gateway') {
                            git branch: 'main',
                                url: "${GITHUB_REPO_URL}/api-gateway.git",
                                credentialsId: 'github-cred'
                            sh 'mvn clean package -DskipTests'
                        }
                    }
                }
                stage('User Service') {
                    steps {
                        dir('user-service') {
                            git branch: 'main',
                                url: "${GITHUB_REPO_URL}/user-service.git",
                                credentialsId: 'github-cred'
                            sh 'mvn clean package -DskipTests'
                        }
                    }
                }
                stage('Post Service') {
                    steps {
                        dir('post-service') {
                            git branch: 'main',
                                url: "${GITHUB_REPO_URL}/post-service.git",
                                credentialsId: 'github-cred'
                            sh 'mvn clean package -DskipTests'
                        }
                    }
                }
            }
        }

        stage('Build Docker Images') {
            steps {
                script {
                    def services = ['config-server', 'discovery-server', 'api-gateway', 'user-service', 'post-service']
                    services.each { service ->
                        dir(service) {
                            sh """
                                docker build -t ${DOCKER_REGISTRY}/${service}:${BUILD_NUMBER} .
                                docker tag ${DOCKER_REGISTRY}/${service}:${BUILD_NUMBER} ${DOCKER_REGISTRY}/${service}:latest
                            """
                        }
                    }
                }
            }
        }

        stage('Push Docker Images') {
            steps {
                script {
                    sh "docker login -u ${DOCKER_CRED_USR} -p ${DOCKER_CRED_PSW} ${DOCKER_REGISTRY}"
                    def services = ['config-server', 'discovery-server', 'api-gateway', 'user-service', 'post-service']
                    services.each { service ->
                        sh """
                            docker push ${DOCKER_REGISTRY}/${service}:${BUILD_NUMBER}
                            docker push ${DOCKER_REGISTRY}/${service}:latest
                        """
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    sh """
                        export DOCKER_REGISTRY=${DOCKER_REGISTRY}
                        export VERSION=${BUILD_NUMBER}
                        
                        kubectl apply -f k8s/config-server.yaml
                        sleep 30
                        
                        kubectl apply -f k8s/discovery-server.yaml
                        sleep 30
                        
                        kubectl apply -f k8s/services.yaml
                        sleep 30
                        
                        kubectl apply -f k8s/api-gateway.yaml
                    """
                }
            }
        }
    }

    post {
        success {
            echo 'Deployment successful!'
        }
        failure {
            echo 'Deployment failed!'
        }
        always {
            // Clean workspace
            cleanWs()
        }
    }
}
