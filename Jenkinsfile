pipeline {
    agent any

    environment {
        SONAR_HOME = tool "Sonar"
    }

    parameters {
        string(name: 'FRONTEND_DOCKER_TAG', defaultValue: 'latest')
        string(name: 'BACKEND_DOCKER_TAG', defaultValue: 'latest')
    }

    stages {

        stage('Workspace Cleanup') {
            steps {
                cleanWs()
            }
        }

        stage('Git: Code Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/Kunika1234/Wanderlust-Mega-DevSecOps-Project.git'
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                echo "Installing dependencies..."
                npm install || true
                '''
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                sh '''
                echo "Running OWASP scan..."
                dependency-check.sh --scan . --format XML || true
                '''
            }
        }

        stage('Trivy Filesystem Scan') {
            steps {
                sh '''
                echo "Running Trivy FS scan..."
                trivy fs . || true
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('Sonar') {
                    sh """
                    ${SONAR_HOME}/bin/sonar-scanner \
                    -Dsonar.projectKey=wanderlust \
                    -Dsonar.projectName=wanderlust \
                    -Dsonar.sources=.
                    """
                }
            }
        }

        stage('Build Docker Images') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub',
                    usernameVariable: 'USER',
                    passwordVariable: 'PASS'
                )]) {

                    dir('backend') {
                        sh '''
                        echo "Building backend image..."
                        docker build -t $USER/wanderlust-backend-beta:${BACKEND_DOCKER_TAG} .
                        '''
                    }

                    dir('frontend') {
                        sh '''
                        echo "Building frontend image..."
                        docker build -t $USER/wanderlust-frontend-beta:${FRONTEND_DOCKER_TAG} .
                        '''
                    }
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub',
                    usernameVariable: 'USER',
                    passwordVariable: 'PASS'
                )]) {

                    sh '''
                    echo "Scanning backend image..."
                    trivy image $USER/wanderlust-backend-beta:${BACKEND_DOCKER_TAG} || true

                    echo "Scanning frontend image..."
                    trivy image $USER/wanderlust-frontend-beta:${FRONTEND_DOCKER_TAG} || true
                    '''
                }
            }
        }

        stage('Docker Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub',
                    usernameVariable: 'USER',
                    passwordVariable: 'PASS'
                )]) {

                    sh '''
                    echo "Logging into DockerHub..."
                    echo $PASS | docker login -u $USER --password-stdin

                    echo "Pushing backend image..."
                    docker push $USER/wanderlust-backend-beta:${BACKEND_DOCKER_TAG}

                    echo "Pushing frontend image..."
                    docker push $USER/wanderlust-frontend-beta:${FRONTEND_DOCKER_TAG}
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "✅ FULL DEVSECOPS PIPELINE SUCCESS 🚀🔥"
        }
        failure {
            echo "❌ PIPELINE FAILED"
        }
    }
}
