// Jenkinsfile (Declarative Pipeline)
pipeline {
    // Exécuter le pipeline sur n'importe quel agent disponible
    agent any

    // Définition des variables d'environnement
    environment {
        // Paramètres du Registre Docker
        DOCKER_REGISTRY = 'docker.io'
        DOCKER_USER = 'rpadallery'              // Votre nom d'utilisateur Docker Hub
        DOCKER_CREDENTIALS_ID = 'DOCKER_HUB_PASS' // This is the ID of the Secret Text credential
        
        // Identifiants Git (pour SCM, si nécessaire)
        GIT_CREDENTIALS_ID = 'PaleHeraldry'
        
        // Tag d'image unique basé sur le numéro de build de Jenkins
        IMAGE_TAG = "${BUILD_NUMBER}"
        
        // Chemin vers vos charts Helm
        HELM_CHART_PATH = './charts'

        // Chemins d'image complets
        MOVIE_IMAGE_BASE = "${DOCKER_USER}/movie-service"
        CAST_IMAGE_BASE = "${DOCKER_USER}/cast-service"
    }

    stages {
        stage('Test Application (Docker Compose)') {
            steps {
                sh 'docker compose -f docker-compose.yml up --build -d'
                sh 'sleep 10'
                sh 'curl -f http://localhost:8081/api/v1/movies/'
                sh 'docker compose -f docker-compose.yml down'
            }
        }

        stage('Build Docker Images') {
            steps {
                script {
                    echo "Building ${MOVIE_IMAGE_BASE}:${IMAGE_TAG}..."
                    docker.build("${MOVIE_IMAGE_BASE}:${IMAGE_TAG}", "./movie-service")

                    echo "Building ${CAST_IMAGE_BASE}:${IMAGE_TAG}..."
                    docker.build("${CAST_IMAGE_BASE}:${IMAGE_TAG}", "./cast-service")
                }
            }
        }

        stage('Tag and Push Docker Images') {
            steps {
                script {
                    withCredentials([string(credentialsId: env.DOCKER_CREDENTIALS_ID, variable: 'DOCKER_PASSWORD')]) {
                        sh "echo \$DOCKER_PASSWORD | docker login -u ${env.DOCKER_USER} --password-stdin ${env.DOCKER_REGISTRY}"

                        sh "docker tag ${env.MOVIE_IMAGE_BASE}:${env.IMAGE_TAG} ${env.MOVIE_IMAGE_BASE}:${env.IMAGE_TAG}"
                        sh "docker tag ${env.CAST_IMAGE_BASE}:${env.IMAGE_TAG} ${env.CAST_IMAGE_BASE}:${env.IMAGE_TAG}"

                        sh "docker push ${env.MOVIE_IMAGE_BASE}:${env.IMAGE_TAG}"
                        sh "docker push ${env.CAST_IMAGE_BASE}:${env.IMAGE_TAG}"

                        sh "docker tag ${env.MOVIE_IMAGE_BASE}:${env.IMAGE_TAG} ${env.MOVIE_IMAGE_BASE}:latest"
                        sh "docker tag ${env.CAST_IMAGE_BASE}:${env.IMAGE_TAG} ${env.CAST_IMAGE_BASE}:latest"

                        sh "docker push ${env.MOVIE_IMAGE_BASE}:latest"
                        sh "docker push ${env.CAST_IMAGE_BASE}:latest"

                        sh "docker logout ${env.DOCKER_REGISTRY}"
                    }
                }
            }
        }

        stage('Deploy to Dev') {
            when {
                expression { env.BRANCH_NAME == 'dev' }
            }
            steps {
                withCredentials([file(credentialsId: 'config', variable: 'KUBECONFIG_FILE')]) {
                    sh """
                    echo "Deploying to DEV environment..."
                    export KUBECONFIG=\${KUBECONFIG_FILE}
                    helm upgrade --install fastapiapp-dev ${HELM_CHART_PATH} \\
                        --namespace dev \\
                        --create-namespace \\
                        --set movie.image.repository=${MOVIE_IMAGE_BASE} \\
                        --set cast.image.repository=${CAST_IMAGE_BASE} \\
                        --set image.tag=${IMAGE_TAG} \\
                        --set replicaCount=1
                    """
                }
            }
        }

        stage('Deploy to QA') {
            when {
                expression { env.BRANCH_NAME == 'qa' }
            }
            steps {
                withCredentials([file(credentialsId: 'config', variable: 'KUBECONFIG_FILE')]) {
                    sh """
                    echo "Deploying to QA environment..."
                    export KUBECONFIG=\${KUBECONFIG_FILE}
                    helm upgrade --install fastapiapp-qa ${HELM_CHART_PATH} \\
                        --namespace qa \\
                        --create-namespace \\
                        --set movie.image.repository=${MOVIE_IMAGE_BASE} \\
                        --set cast.image.repository=${CAST_IMAGE_BASE} \\
                        --set image.tag=${IMAGE_TAG} \\
                        --set replicaCount=2
                    """
                }
            }
        }

        stage('Deploy to Staging') {
            when {
                expression { env.BRANCH_NAME == 'staging' }
            }
            steps {
                input(message: 'QA successful. Proceed to STAGING?', ok: 'Deploy')
                withCredentials([file(credentialsId: 'config', variable: 'KUBECONFIG_FILE')]) {
                    sh """
                    echo "Deploying to STAGING environment..."
                    export KUBECONFIG=\${KUBECONFIG_FILE}
                    helm upgrade --install fastapiapp-staging ${HELM_CHART_PATH} \\
                        --namespace staging \\
                        --create-namespace \\
                        --set movie.image.repository=${MOVIE_IMAGE_BASE} \\
                        --set cast.image.repository=${CAST_IMAGE_BASE} \\
                        --set image.tag=${IMAGE_TAG} \\
                        --set replicaCount=2
                    """
                }
            }
        }

        stage('Deploy to Production') {
            when {
                expression { env.BRANCH_NAME == 'main' }
            }
            steps {
                input(message: 'STAGING successful. Deploy to PRODUCTION?', ok: 'Deploy')
                withCredentials([file(credentialsId: 'config', variable: 'KUBECONFIG_FILE')]) {
                    sh """
                    echo "Deploying to PRODUCTION environment..."
                    export KUBECONFIG=\${KUBECONFIG_FILE}
                    helm upgrade --install fastapiapp-prod ${HELM_CHART_PATH} \\
                        --namespace prod \\
                        --create-namespace \\
                        --set movie.image.repository=${MOVIE_IMAGE_BASE} \\
                        --set cast.image.repository=${CAST_IMAGE_BASE} \\
                        --set image.tag=${IMAGE_TAG} \\
                        --set replicaCount=3
                    """
                }
            }
        }
    } // <-- This closing brace for 'stages' was missing!

    post {
        always {
            cleanWs()
        }
        success {
            echo '✅ Pipeline completed successfully!'
        }
        failure {
            echo '❌ Pipeline failed! Check the console output for errors.'
        }
    }
}
