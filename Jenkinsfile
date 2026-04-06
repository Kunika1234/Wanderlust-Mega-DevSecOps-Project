pipeline {
    agent any   // ✅ FIXED (no more Node error)

    environment {
        SONAR_HOME = tool "Sonar"
        DOCKER_CREDENTIALS = 'dockerhub'   // 🔐 credentials ID
    }

    parameters {
        string(name: 'FRONTEND_DOCKER_TAG', defaultValue: 'latest', description: 'Frontend Docker Tag')
        string(name: 'BACKEND_DOCKER_TAG', defaultValue: 'latest', description: 'Backend Docker Tag')
    }

    stages {

        stage("Workspace cleanup") {
            steps {
                cleanWs()
            }
        }

        stage('Git: Code Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/DevMadhup/Wanderlust-Mega-Project.git'
            }
        }

        stage("Trivy: Filesystem scan") {
            steps {
                sh "trivy fs . || true"
            }
        }

        stage("OWASP: Dependency check") {
            steps {
                sh "dependency-check.sh --scan . --format XML || true"
            }
        }

        stage("SonarQube: Code Analysis") {
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

        stage("SonarQube: Quality Gate") {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Exporting environment variables') {
            parallel {

                stage("Backend env setup") {
                    steps {
                        dir("Automations") {
                            sh "bash updatebackendnew.sh"
                        }
                    }
                }

                stage("Frontend env setup") {
                    steps {
                        dir("Automations") {
                            sh "bash updatefrontendnew.sh"
                        }
                    }
                }
            }
        }

        stage("Docker: Build Images") {
            steps {
                script {
                    dir('backend') {
                        sh "docker build -t madhupdevops/wanderlust-backend-beta:${params.BACKEND_DOCKER_TAG} ."
                    }

                    dir('frontend') {
                        sh "docker build -t madhupdevops/wanderlust-frontend-beta:${params.FRONTEND_DOCKER_TAG} ."
                    }
                }
            }
        }

        stage("Docker: Login & Push") {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: "${DOCKER_CREDENTIALS}",
                    usernameVariable: 'USER',
                    passwordVariable: 'PASS'
                )]) {

                    sh """
                    echo $PASS | docker login -u $USER --password-stdin
                    docker push madhupdevops/wanderlust-backend-beta:${params.BACKEND_DOCKER_TAG}
                    docker push madhupdevops/wanderlust-frontend-beta:${params.FRONTEND_DOCKER_TAG}
                    """
                }
            }
        }
    }

    post {
        success {
            archiveArtifacts artifacts: '**/*.xml', followSymlinks: false

            build job: "Wanderlust-CD", parameters: [
                string(name: 'FRONTEND_DOCKER_TAG', value: "${params.FRONTEND_DOCKER_TAG}"),
                string(name: 'BACKEND_DOCKER_TAG', value: "${params.BACKEND_DOCKER_TAG}")
            ]
        }

        failure {
            echo "❌ Pipeline Failed"
        }
    }
}
