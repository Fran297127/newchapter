pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "pointbreak29/newchapter:${BUILD_NUMBER}" // Ganti dockerhub_username dengan akun Docker Hub kamu
        SONARQUBE_SERVER = 'http://sonarqube:9000' // Samakan dengan config di Jenkins
        GOOGLE_CREDENTIALS = credentials('gcp-service-account-json')
        TELEGRAM_TOKEN = credentials('telegram-bot-token') // Ganti dengan token bot Telegram kamu
        TELEGRAM_CHAT_ID = credentials('telegram-chat-id') // Ganti dengan ID chat Telegram kamu
        DOCKER_HUB_CREDENTIALS = credentials('dockerhub-creds') // Ganti dengan ID kredensial Docker Hub di Jenkins

    }

    stages {
        stage('Clone') {
            steps {
                checkout scm
            }
        }

        stage('Unit Test') {
            steps {
                sh 'mvn clean test'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv("${SONARQUBE_SERVER}") {
                    sh 'mvn sonar:sonar'
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t $DOCKER_IMAGE ."
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'
                    sh "docker push $DOCKER_IMAGE"
                }
            }
        }

        stage('Deploy to Cloud Run') {
            steps {
                withCredentials([file(credentialsId: 'gcp-service-account-json', variable: 'GOOGLE_APPLICATION_CREDENTIALS')]) {
                    sh '''
                    gcloud auth activate-service-account --key-file=$GOOGLE_APPLICATION_CREDENTIALS
                    gcloud config set project <project-id>
                    gcloud run deploy newchapter \
                        --image docker.io/dockerhub_username/newchapter:${BUILD_NUMBER} \
                        --platform managed \
                        --region asia-southeast2 \
                        --allow-unauthenticated
                    '''
                }
            }
        }

        stage('Send Telegram Notification') {
            steps {
                script {
                    def status = currentBuild.currentResult
                    def message = "Build #${BUILD_NUMBER} - Status: ${status} - Repo: ${env.GIT_URL}"
                    sh """
                    curl -s -X POST https://api.telegram.org/bot${TELEGRAM_TOKEN}/sendMessage \\
                        -d chat_id=${TELEGRAM_CHAT_ID} \\
                        -d text="${message}"
                    """
                }
            }
        }
    }

    post {
        failure {
            script {
                def message = "Build #${BUILD_NUMBER} FAILED! Check Jenkins logs."
                sh """
                curl -s -X POST https://api.telegram.org/bot${TELEGRAM_TOKEN}/sendMessage \\
                    -d chat_id=${TELEGRAM_CHAT_ID} \\
                    -d text="${message}"
                """
            }
        }
        success {
            script {
                def message = "Build #${BUILD_NUMBER} SUCCESS! Deployed to Cloud Run."
                sh """
                curl -s -X POST https://api.telegram.org/bot${TELEGRAM_TOKEN}/sendMessage \\
                    -d chat_id=${TELEGRAM_CHAT_ID} \\
                    -d text="${message}"
                """
            }
        }
    }
}
