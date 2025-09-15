pipeline {
    agent any
    parameters {
        string(name: 'FRONTEND_TAG', defaultValue: 'latest', description: 'Docker tag for frontend')
        string(name: 'BACKEND_TAG', defaultValue: 'latest', description: 'Docker tag for backend')
    }
    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-credentials') // Set up in Jenkins
        SONARQUBE_SERVER = 'SonarQube' // Set up in Jenkins
    }
    stages {
        stage('Validate Parameters') {
            steps {
                script {
                    if (!params.FRONTEND_TAG?.trim()) {
                        error("FRONTEND_TAG parameter is required.")
                    }
                    if (!params.BACKEND_TAG?.trim()) {
                        error("BACKEND_TAG parameter is required.")
                    }
                }
            }
        }
        stage('Checkout') {
            steps {
                git url: 'https://github.com/Roshanx96/wanderlust.git', branch: 'main'
            }
        }
        stage('Security Scans') {
            parallel {
                stage('Trivy Scan') {
                    steps {
                        sh 'trivy fs . --exit-code 1 --severity HIGH,CRITICAL || true'
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
                SONAR_TOKEN = credentials('SonarQube') // Set up in Jenkins
            }
            steps {
                // Ensure sonar-scanner is installed in user directory and update PATH
                                sh '''
                                        export SONAR_SCANNER_HOME="$HOME/.sonar-scanner"
                                        if ! [ -x "$SONAR_SCANNER_HOME/bin/sonar-scanner" ]; then
                                            wget -O sonar-scanner-cli-5.0.1.3006-linux.zip https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-5.0.1.3006-linux.zip
                                            rm -rf sonar-scanner-5.0.1.3006-linux
                                            unzip -o sonar-scanner-cli-5.0.1.3006-linux.zip
                                            mv sonar-scanner-5.0.1.3006-linux "$SONAR_SCANNER_HOME"
                                        fi
                                        export PATH="$SONAR_SCANNER_HOME/bin:$PATH"
                                '''
                // Add sonar-scanner to PATH for the analysis step
                withEnv(["PATH=$HOME/.sonar-scanner/bin:$PATH"]) {
                    withSonarQubeEnv("${SONARQUBE_SERVER}") {
                        sh 'sonar-scanner -Dsonar.projectKey=wanderlust -Dsonar.sources=. -Dsonar.host.url=$SONAR_HOST_URL -Dsonar.login=$SONAR_TOKEN'
                    }
                }
            }
        }
        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        stage('Build & Push Docker Images') {
            steps {
                script {
                    // Build and push frontend
                    sh """
                        docker build -t roshanx/wanderlust-frontend:${params.FRONTEND_TAG} ./frontend
                        echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin
                        docker push roshanx/wanderlust-frontend:${params.FRONTEND_TAG}
                    """
                    // Build and push backend
                    sh """
                        docker build -t roshanx/wanderlust-backend:${params.BACKEND_TAG} ./backend
                        echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin
                        docker push roshanx/wanderlust-backend:${params.BACKEND_TAG}
                    """
                }
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
            echo 'CI pipeline failed.'
        }
        success {
            echo 'CI pipeline completed successfully.'
        }
    }
}