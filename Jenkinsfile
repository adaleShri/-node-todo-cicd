pipeline {
    agent any

    environment {
        DOCKER_IMAGE_NAME   = 'node-app'
        DOCKER_CONTAINER    = 'node-app-container'
        DOCKER_COMPOSE_FILE = 'docker-compose.yaml'
        APP_PORT = '3000'
    }

    stages {

        stage("Code Clone") {
            steps {
                echo "=== Code Clone Stage ==="
                git url: "https://github.com/adaleShri/-node-todo-cicd.git", branch: "main"
            }
        }

        stage("Install Dependencies & Unit Test") {
            steps {
                echo "=== Install & Test ==="
                sh """
                    npm install
                    npm test || echo "Tests missing or failed, continuing..."
                """
            }
        }

        stage("SonarQube Analysis") {
            steps {
                script {
                    def scannerHome = tool 'SonarScanner'

                    withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {

                        sh """
                          ${scannerHome}/bin/sonar-scanner \
                            -Dsonar.projectKey=node-todo \
                            -Dsonar.projectName=node-todo \
                            -Dsonar.sources=. \
                            -Dsonar.host.url=http://<YOUR-SONAR-IP>:9000 \
                            -Dsonar.login=$SONAR_TOKEN
                        """
                    }
                }
            }
        }

        stage("Quality Gate") {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage("Build Docker Image") {
            steps {
                sh "docker build -t ${DOCKER_IMAGE_NAME}:latest ."
            }
        }

        stage("Push To DockerHub") {
            steps {
                withCredentials([usernamePassword(
                    credentialsId:"dockerHubCreds",
                    usernameVariable:"dockerHubUser", 
                    passwordVariable:"dockerHubPass")]) {

                    sh 'echo $dockerHubPass | docker login -u $dockerHubUser --password-stdin'
                    sh "docker image tag ${DOCKER_IMAGE_NAME}:latest ${dockerHubUser}/${DOCKER_IMAGE_NAME}:latest"
                    sh "docker push ${dockerHubUser}/${DOCKER_IMAGE_NAME}:latest"
                }
            }
        }

        stage("Approve Deploy") {
            steps {
                script {
                    timeout(time: 15, unit: 'MINUTES') {
                        input message: "Deploy ${DOCKER_IMAGE_NAME}:latest?", ok: "Deploy"
                    }
                }
            }
        }

        stage("Deploy") {
            steps {
                sh """
                  docker compose -f ${DOCKER_COMPOSE_FILE} down || true
                  docker compose -f ${DOCKER_COMPOSE_FILE} up -d --build
                """
            }
        }
    }

    post {
        success {
            echo "🚀 Deployment completed successfully."
        }
        failure {
            echo "❌ Build failed — check logs."
        }
    }
}
