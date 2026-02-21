@Library('shared') _

pipeline {
    agent { label 'Node' }

    environment {
        SONAR_HOME = tool "sonar"
    }

    parameters {
        string(name: 'FRONTEND_DOCKER_TAG', defaultValue: '', description: 'Docker tag for frontend')
        string(name: 'BACKEND_DOCKER_TAG', defaultValue: '', description: 'Docker tag for backend')
    }

    stages {

        stage("Validate Parameters") {
            steps {
                script {
                    if (params.FRONTEND_DOCKER_TAG == '' || params.BACKEND_DOCKER_TAG == '') {
                        error("FRONTEND_DOCKER_TAG and BACKEND_DOCKER_TAG must be provided.")
                    }
                }
            }
        }

        stage("Workspace cleanup") {
            steps {
                cleanWs()
            }
        }

        stage("Git: Code Checkout") {
            steps {
                git branch: 'main', url: 'https://github.com/Shaik282/Wanderlust-Mega-Project-devops.git'
            }
        }

        stage("Trivy: Filesystem scan") {
            steps {
                // Run Trivy via Docker to avoid "trivy: not found"
                sh '''
                    docker run --rm -v $PWD:/app aquasec/trivy fs /app
                '''
            }
        }

        stage("OWASP: Dependency check") {
            steps {
                sh '''
                    mkdir -p reports/owasp
                    dependency-check.sh --project "wanderlust" --scan /app --format "ALL" --out reports/owasp
                '''
                archiveArtifacts artifacts: 'reports/owasp/*', allowEmptyArchive: true
            }
        }

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

        stage("SonarQube: Code Quality Gates") {
            steps {
                waitForQualityGate abortPipeline: true
            }
        }

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

        stage("Docker: Push to DockerHub") {
            steps {
                sh "docker push madhar11/wanderlust-backend:${params.BACKEND_DOCKER_TAG}"
                sh "docker push madhar11/wanderlust-frontend:${params.FRONTEND_DOCKER_TAG}"
            }
        }

    }

    post {
        success {
            archiveArtifacts artifacts: '*.xml', followSymlinks: false
            build job: "Wanderlust-CD", parameters: [
                string(name: 'FRONTEND_DOCKER_TAG', value: "${params.FRONTEND_DOCKER_TAG}"),
                string(name: 'BACKEND_DOCKER_TAG', value: "${params.BACKEND_DOCKER_TAG}")
            ]
        }
    }
}
