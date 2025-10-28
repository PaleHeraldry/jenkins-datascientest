// Jenkinsfile (Declarative Pipeline)
pipeline {
    // Exécuter le pipeline sur n'importe quel agent disponible
    agent any

    // Définition des variables d'environnement
    environment {
        // Paramètres du Registre Docker
        DOCKER_REGISTRY = 'docker.io'
        DOCKER_USER = 'rpadallery'              // Votre nom d'utilisateur Docker Hub
        DOCKER_CREDENTIALS_ID = 'rpadallery'    // ID de vos credentials Docker Hub stockés dans Jenkins
        
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
                // Utiliser Docker Compose pour exécuter des tests d'intégration/unitaires
                // Le service 'movie_service' et 'cast_service' doivent avoir une commande de test dans leurs Dockerfiles.
                // OU, si vous utilisez un service de test séparé dans docker-compose :
                sh 'docker compose -f docker-compose.yml up --build -d' // Démarrer les services
                
                // Exécuter les tests. Cette étape est critique et doit renvoyer 0 si les tests réussissent.
                // NOTE: La commande de test exacte dépend de votre projet. C'est un exemple.
                sh 'sleep 10' 
                
                // Use curl to hit the Nginx service's health endpoint (port 8081 mapped to Jenkins agent)
                // Assuming port 8081 is exposed on the Jenkins agent.
                sh 'curl -f http://localhost:8081/api/v1/movies/'
                
                // Arrêter et supprimer les conteneurs
                sh 'docker compose -f docker-compose.yml down'
            }
        }

        // ------------------------------------------------------------------------------------------------------------------

        stage('Build Docker Images') {
            steps {
                script {
                    // Les étapes de build doivent être dans un bloc script pour utiliser l'objet 'docker'
                    echo "Building ${MOVIE_IMAGE_BASE}:${IMAGE_TAG}..."
                    docker.build("${MOVIE_IMAGE_BASE}:${IMAGE_TAG}", "./movie-service")
                    
                    echo "Building ${CAST_IMAGE_BASE}:${IMAGE_TAG}..."
                    docker.build("${CAST_IMAGE_BASE}:${IMAGE_TAG}", "./cast-service")
                }
            }
        }

        // ------------------------------------------------------------------------------------------------------------------

        stage('Tag and Push Docker Images') {
            steps {
                script {
                    // Se connecter au registre Docker Hub en utilisant les credentials
                    docker.withRegistry("https://${DOCKER_REGISTRY}", "${DOCKER_CREDENTIALS_ID}") {
                        // Pousser les images avec le tag unique (pour l'historique)
                        docker.image("${MOVIE_IMAGE_BASE}:${IMAGE_TAG}").push()
                        docker.image("${CAST_IMAGE_BASE}:${IMAGE_TAG}").push()

                        // Retagguer l'image unique avec 'latest' et pousser (pour la commodité du déploiement)
                        docker.image("${MOVIE_IMAGE_BASE}:${IMAGE_TAG}").tag("${MOVIE_IMAGE_BASE}:latest")
                        docker.image("${CAST_IMAGE_BASE}:${IMAGE_TAG}").tag("${CAST_IMAGE_BASE}:latest")
                        
                        docker.image("${MOVIE_IMAGE_BASE}:latest").push()
                        docker.image("${CAST_IMAGE_BASE}:latest").push()
                        
                        echo "Docker images pushed successfully with tags: ${IMAGE_TAG} and latest."
                    }
                }
            }
        }

        // ------------------------------------------------------------------------------------------------------------------

        stage('Deploy to Dev') {
            // Déploiement par défaut pour toutes les branches de fonctionnalité/pull requests
            when {
                // S'exécute si la branche n'est ni 'master' (ou main) ni 'develop'
                not {
                    anyOf {
                        branch 'master'
                        branch 'develop'
                        branch 'main' // Ajout de 'main' par précaution moderne
                    }
                }
            }
            steps {
                sh """
                echo "Deploying to DEV environment..."
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

        // ------------------------------------------------------------------------------------------------------------------

        stage('Deploy to QA') {
            // S'exécute uniquement pour les commits sur la branche 'develop'
            when {
                branch 'develop'
            }
            steps {
                sh """
                echo "Deploying to QA environment..."
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

        // ------------------------------------------------------------------------------------------------------------------

        stage('Deploy to Staging') {
            // S'exécute immédiatement après QA, car sur la branche 'develop'
            when {
                branch 'develop'
            }
            steps {
                // Une pause manuelle est recommandée avant le Staging/Prod pour inspection
                input(message: 'QA successful. Proceed to STAGING?', ok: 'Deploy')
                
                sh """
                echo "Deploying to STAGING environment..."
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

        // ------------------------------------------------------------------------------------------------------------------

        stage('Deploy to Production') {
            // S'exécute uniquement pour la branche 'master' (ou 'main')
            when {
                anyOf {
                    branch 'master'
                    branch 'main'
                }
            }
            steps {
                // Nécessite une approbation manuelle pour le déploiement en production
                input(message: 'STAGING successful. Deploy to PRODUCTION?', ok: 'Deploy')
                
                sh """
                echo "Deploying to PRODUCTION environment..."
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

    // ------------------------------------------------------------------------------------------------------------------

    post {
        always {
            // Toujours nettoyer l'espace de travail après le build, peu importe le résultat
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
