pipeline {
    agent any
    environment {
        // Helm will use these base paths
        DOCKER_REGISTRY = 'docker.io'
        DOCKER_USER = 'rpadallery' // Using the user part of the registry path
        DOCKER_CREDENTIALS_ID = 'rpadallery'
        GIT_CREDENTIALS_ID = 'PaleHeraldry'
        IMAGE_TAG = "${BUILD_NUMBER}"
        HELM_CHART_PATH = './charts'

        // Full image paths for easy reference
        MOVIE_IMAGE_BASE = "${DOCKER_USER}/movie-service"
        CAST_IMAGE_BASE = "${DOCKER_USER}/cast-service"
    }

    stages {
        stage('Build Docker Images') {
            steps {
                script {
                    // Build images with the unique BUILD_NUMBER tag
                    docker.build("${MOVIE_IMAGE_BASE}:${IMAGE_TAG}", "./movie-service")
                    docker.build("${CAST_IMAGE_BASE}:${IMAGE_TAG}", "./cast-service")
                }
            }
        }

        stage('Tag and Push Docker Images') {
            steps {
                script {
                    docker.withRegistry("https://${DOCKER_REGISTRY}", "${DOCKER_CREDENTIALS_ID}") {
                        // 1. Push unique tags
                        docker.image("${MOVIE_IMAGE_BASE}:${IMAGE_TAG}").push()
                        docker.image("${CAST_IMAGE_BASE}:${IMAGE_TAG}").push()

                        // 2. Tag and Push 'latest'
                        docker.image("${MOVIE_IMAGE_BASE}:${IMAGE_TAG}").tag("${MOVIE_IMAGE_BASE}:latest")
                        docker.image("${CAST_IMAGE_BASE}:${IMAGE_TAG}").tag("${CAST_IMAGE_BASE}:latest")
                        docker.image("${MOVIE_IMAGE_BASE}:latest").push()
                        docker.image("${CAST_IMAGE_BASE}:latest").push()
                    }
                }
            }
        }

        stage('Deploy to Dev') {
            when {
                // Deploys to Dev for any branch that is not 'master' or 'develop'
                not {
                    anyOf {
                        branch 'master'
                        branch 'develop'
                    }
                }
            }
            steps {
                sh """
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

        stage('Deploy to QA') {
            when {
                branch 'develop'
            }
            steps {
                sh """
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

        stage('Deploy to Staging') {
            when {
                // This will execute immediately after QA if on the 'develop' branch
                branch 'develop'
            }
            steps {
                // Typically you'd add a manual approval input here, but keeping it for 'develop' branch flow
                sh """
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

        stage('Deploy to Production') {
            when {
                // Assuming 'master' is the production branch (or 'main' based on your log)
                branch 'master'
            }
            steps {
                input(message: 'Deploy to Production?', ok: 'Deploy')
                sh """
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

    post {
        always {
            cleanWs() // Always clean workspace
        }
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}