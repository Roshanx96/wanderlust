pipeline {
    agent any
    parameters {
        string(name: 'FRONTEND_TAG', defaultValue: 'latest', description: 'Docker tag for frontend')
        string(name: 'BACKEND_TAG', defaultValue: 'latest', description: 'Docker tag for backend')
    }
    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-credentials') // Set up in Jenkins
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
                script {
                    /* 1. LOAD THE TOOL
                       We use the specific name 'OWASP-Dependency-Check' that you configured
                       in Manage Jenkins > Tools. This sets the path to the executable.
                    */
                    def depCheckHome = tool 'OWASP-Dependency-Check'
                    
                    /* 2. RUN THE SCAN
                       - --project flask-app: Sets the name of the project in the report.
                       - --scan .: Scans the current directory.
                       - --format ALL: Generates XML (required for Jenkins Sidebar) AND HTML (for download).
                       - (Removed --noupdate): Allows the tool to download the vulnerability database.
                    */
                    sh "${depCheckHome}/bin/dependency-check.sh --project flask-app --scan . --format ALL"
                }
            }
            post {
                always {
                    /* 3. PUBLISH TO SIDEBAR
                       This reads the 'dependency-check-report.xml' file generated above
                       and creates the "Dependency-Check" link/graph on the Jenkins sidebar.
                    */
                    dependencyCheckPublisher pattern: 'dependency-check-report.xml'
                    
                    /* 4. ARCHIVE ARTIFACTS
                       This saves the 'dependency-check-report.html' file so you can 
                       download it manually from the "Build Artifacts" section.
                    */
                    archiveArtifacts allowEmptyArchive: true, artifacts: 'dependency-check-report.html'
                }
               }
             }
           }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    /* 1. TOOL DEFINITION
                       We manually define the scanner home because the 'tools {}' block 
                       doesn't work with some plugin versions. 
                       'SonarScanner' must match the Name in Manage Jenkins > Tools.
                    */
                    def scannerHome = tool name: 'SonarScanner', type: 'hudson.plugins.sonar.SonarRunnerInstallation'
                    
                    /* 2. SERVER CONNECTION
                       withSonarQubeEnv('SonarQube'):
                       - 'SonarQube' matches the Name in Manage Jenkins > System.
                       - This automatically injects:
                            $SONAR_HOST_URL (Server URL)
                            $SONAR_AUTH_TOKEN (Login Token)
                       - You DO NOT need to pass -Dsonar.host.url manually.
                    */
                    withSonarQubeEnv('SonarQube') {
                        
                        /* 3. THE SCAN COMMAND
                           -Dsonar.projectKey:  Unique ID for this project in SonarQube (Required).
                           -Dsonar.projectName: Friendly name displayed on the dashboard (Optional).
                           -Dsonar.sources:     Where the code is located. '.' means current directory (Required).
                        */
                        sh """
                        ${scannerHome}/bin/sonar-scanner \
                        -Dsonar.projectKey=wanderlust \
                        -Dsonar.projectName='wanderlust Application' \
                        -Dsonar.sources=.
                        """
                    }
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
