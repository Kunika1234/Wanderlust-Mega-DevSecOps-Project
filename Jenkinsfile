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
                git branch: 'main', url: 'https://github.com/DevMadhup/Wanderlust-Mega-Project.git'
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install || true'
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                sh 'dependency-check.sh --scan . --format XML || true'
            }
        }

        stage('Trivy Filesystem Scan') {
            steps {
                sh 'trivy fs . || true'
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
                script {
                    dir('backend') {
                        sh "docker build -t $USER/wanderlust-backend-beta:${params.BACKEND_DOCKER_TAG} ."
                    }
                    dir('frontend') {
                        sh "docker build -t $USER/wanderlust-frontend-beta:${params.FRONTEND_DOCKER_TAG} ."
                    }
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh """
                trivy image $USER/wanderlust-backend-beta:${params.BACKEND_DOCKER_TAG} || true
                trivy image $USER/wanderlust-frontend-beta:${params.FRONTEND_DOCKER_TAG} || true
                """
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
                    echo $PASS | docker login -u $USER --password-stdin

                    docker push $USER/wanderlust-backend-beta:${BACKEND_DOCKER_TAG}
                    docker push $USER/wanderlust-frontend-beta:${FRONTEND_DOCKER_TAG}
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "✅ FULL DEVSECOPS PIPELINE SUCCESS 🚀"
        }
        failure {
            echo "❌ Pipeline Failed"
        }
    }
}
