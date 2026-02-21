@Library('shared') _

pipeline {
    agent { label 'Node' }

    environment {
        SONAR_HOME = tool "sonar" // SonarQube installation path
    }

    parameters {
        string(name: 'FRONTEND_DOCKER_TAG', defaultValue: '', description: 'Docker tag for frontend')
        string(name: 'BACKEND_DOCKER_TAG', defaultValue: '', description: 'Docker tag for backend')
    }

    stages {

        // Stage 1: Validate that required parameters are provided
        stage("Validate Parameters") {
            steps {
                script {
                    if (params.FRONTEND_DOCKER_TAG == '' || params.BACKEND_DOCKER_TAG == '') {
                        error("FRONTEND_DOCKER_TAG and BACKEND_DOCKER_TAG must be provided.")
                    }
                }
            }
        }

        // Stage 2: Clean workspace to avoid conflicts
        stage("Workspace cleanup") {
            steps {
                cleanWs()
            }
        }

        // Stage 3: Checkout code from GitHub
        stage("Git: Code Checkout") {
            steps {
                git branch: 'main', url: 'https://github.com/Shaik282/Wanderlust-Mega-Project-devops.git'
            }
        }

        // Stage 4: Trivy scan using Docker (avoids 'trivy: not found')
        stage("Trivy: Filesystem scan") {
            steps {
                sh '''
                    docker run --rm -v $PWD:/app aquasec/trivy fs /app
                '''
            }
        }

        // Stage 5: OWASP Dependency Check
        stage("OWASP: Dependency check") {
            steps {
                sh '''
                    mkdir -p reports/owasp
                    dependency-check.sh --project "wanderlust" --scan /app --format "ALL" --out reports/owasp
                '''
                archiveArtifacts artifacts: 'reports/owasp/*', allowEmptyArchive: true
            }
        }

        // Stage 6: SonarQube code analysis
        stage("SonarQube: Code Analysis") {
            steps {
                withSonarQubeEnv('Sonar') {
                    sh '''
                        sonar-scanner \
                        -Dsonar.projectKey=wanderlust \
                        -Dsonar.sources=. \
                        -Dsonar.host.url=$SONAR_HOST_URL \
                        -Dsonar.login=$SONAR_AUTH_TOKEN
                    '''
                }
            }
        }

        // Stage 7: SonarQube quality gates
        stage("SonarQube: Code Quality Gates") {
            steps {
                waitForQualityGate abortPipeline: true
            }
        }

        // Stage 8: Export environment variables in parallel for frontend/backend
        stage("Exporting environment variables") {
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

        // Stage 9: Build Docker images for backend and frontend
        stage("Docker: Build Images") {
            steps {
                dir('backend') {
                    sh "docker build -t madhar11/wanderlust-backend:${params.BACKEND_DOCKER_TAG} ."
                }
                dir('frontend') {
                    sh "docker build -t madhar11/wanderlust-frontend:${params.FRONTEND_DOCKER_TAG} ."
                }
            }
        }

        // Stage 10: Push Docker images to DockerHub
        stage("Docker: Push to DockerHub") {
            steps {
                sh "docker push madhar11/wanderlust-backend:${params.BACKEND_DOCKER_TAG}"
                sh "docker push madhar11/wanderlust-frontend:${params.FRONTEND_DOCKER_TAG}"
            }
        }

    }

    post {
        success {
            // Archive XML reports if any
            archiveArtifacts artifacts: '*.xml', followSymlinks: false

            // Trigger CD pipeline after successful build
            build job: "Wanderlust-CD", parameters: [
                string(name: 'FRONTEND_DOCKER_TAG', value: "${params.FRONTEND_DOCKER_TAG}"),
                string(name: 'BACKEND_DOCKER_TAG', value: "${params.BACKEND_DOCKER_TAG}")
            ]
        }
    }
}
