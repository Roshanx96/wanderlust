pipeline {
    agent any

    parameters {
        string(name: 'FRONTEND_TAG', description: 'Frontend Docker Image Tag', defaultValue: 'latest')
        string(name: 'BACKEND_TAG', description: 'Backend Docker Image Tag', defaultValue: 'latest')
    }

    environment {
        GITHUB_CREDENTIALS = 'Github-cred'
        DOCKER_CREDENTIALS = 'Dockerhub-cred'
        DOCKERHUB_USERNAME = 'roshanx' // Your DockerHub username
        GITHUB_REPO = 'https://github.com/Roshanx96/wanderlust-mega-project.git'
    }

    stages {
        stage('Validate Parameters') {
            steps {
                script {
                    if (!params.FRONTEND_TAG?.trim() || !params.BACKEND_TAG?.trim()) {
                        error("Docker image tags cannot be empty!")
                    }
                }
            }
        }

        stage('Checkout Code') {
            steps {
                git branch: 'main', credentialsId: "${env.GITHUB_CREDENTIALS}", url: "${env.GITHUB_REPO}"
            }
        }

        stage('Security Scans') {
            parallel {
                stage('Trivy Filesystem Scan') {
                    steps {
                        sh 'trivy fs --exit-code 1 --severity CRITICAL --no-progress . || true'
                    }
                }
                stage('OWASP Dependency Check') {
                    steps {
                        sh 'dependency-check.sh --project wanderlust --scan . || true'
                    }
                }
            }
        }

        stage('SonarQube Analysis') {
            environment {
                SONARQUBE_SERVER = 'SonarQube' // Jenkins SonarQube server name
            }
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh 'sonar-scanner'
                }
            }
        }

        stage('Wait for SonarQube Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Update .env Files') {
            steps {
                sh '''
                    chmod +x Automations/updatefrontendnew.sh
                    ./Automations/updatefrontendnew.sh

                    chmod +x Automations/updatebackendnew.sh
                    ./Automations/updatebackendnew.sh
                '''
            }
        }

        stage('Docker Login') {
            steps {
                withCredentials([usernamePassword(credentialsId: "${env.DOCKER_CREDENTIALS}", usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                    '''
                }
            }
        }

        stage('Build & Push Frontend Image') {
            steps {
                sh '''
                    docker build -t $DOCKERHUB_USERNAME/wanderlust-frontend-beta:${FRONTEND_TAG} ./frontend
                    docker push $DOCKERHUB_USERNAME/wanderlust-frontend-beta:${FRONTEND_TAG}
                '''
            }
        }

        stage('Build & Push Backend Image') {
            steps {
                sh '''
                    docker build -t $DOCKERHUB_USERNAME/wanderlust-backend-beta:${BACKEND_TAG} ./backend
                    docker push $DOCKERHUB_USERNAME/wanderlust-backend-beta:${BACKEND_TAG}
                '''
            }
        }

        stage('Trigger CD Pipeline') {
            steps {
                build job: 'wanderlust-cd', parameters: [
                    string(name: 'FRONTEND_TAG', value: params.FRONTEND_TAG),
                    string(name: 'BACKEND_TAG', value: params.BACKEND_TAG)
                ]
            }
        }
    }

    post {
        failure {
                 echo "The Wanderlust CI pipeline has failed. Please check Jenkins."
        }
    }
}
