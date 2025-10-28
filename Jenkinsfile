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
                            // Use withCredentials to inject the DOCKER_HUB_PASS secret as an environment variable (DOCKER_PASSWORD)
                            withCredentials([string(credentialsId: env.DOCKER_CREDENTIALS_ID, variable: 'DOCKER_PASSWORD')]) {
                                
                                // 1. Perform Docker Login using the injected secret
                                sh "echo \$DOCKER_PASSWORD | docker login -u ${env.DOCKER_USER} --password-stdin ${env.DOCKER_REGISTRY}"
        
                                // We must tag the images before pushing since we didn't capture the Groovy image object
                                // Tagging from the local image name to the remote target name
                                sh "docker tag ${env.MOVIE_IMAGE_BASE}:${env.IMAGE_TAG} ${env.MOVIE_IMAGE_BASE}:${env.IMAGE_TAG}"
                                sh "docker tag ${env.CAST_IMAGE_BASE}:${env.IMAGE_TAG} ${env.CAST_IMAGE_BASE}:${env.IMAGE_TAG}"
        
                                // 2. Push unique tags
                                sh "docker push ${env.MOVIE_IMAGE_BASE}:${env.IMAGE_TAG}"
                                sh "docker push ${env.CAST_IMAGE_BASE}:${env.IMAGE_TAG}"
        
                                // 3. Tag and Push 'latest'
                                sh "docker tag ${env.MOVIE_IMAGE_BASE}:${env.IMAGE_TAG} ${env.MOVIE_IMAGE_BASE}:latest"
                                sh "docker tag ${env.CAST_IMAGE_BASE}:${env.IMAGE_TAG} ${env.CAST_IMAGE_BASE}:latest"
                                
                                sh "docker push ${env.MOVIE_IMAGE_BASE}:latest"
                                sh "docker push ${env.CAST_IMAGE_BASE}:latest"
        
                                // 4. Log out (optional, good practice)
                                sh "docker logout ${env.DOCKER_REGISTRY}"
                            }
                        }
                    }
                }

        // ------------------------------------------------------------------------------------------------------------------

        stage('Deploy to Dev') {
            // ... when block ...
            steps {
                sh """
                echo "Deploying to DEV environment..."
                
                # 1. Write the Kubeconfig content to a temporary file
                # /tmp/kubeconfig_dev is the path where the file will be created
                echo "${config}" > /tmp/kubeconfig_dev
                
                # 2. Set the KUBECONFIG environment variable for the helm command
                export KUBECONFIG=/tmp/kubeconfig_dev
                
                # 3. Execute helm command (Helm will now use the Kubeconfig file)
                helm upgrade --install fastapiapp-dev ${HELM_CHART_PATH} \\
                --namespace dev \\
                --create-namespace \\
                --set movie.image.repository=${MOVIE_IMAGE_BASE} \\
                --set cast.image.repository=${CAST_IMAGE_BASE} \\
                --set image.tag=${IMAGE_TAG} \\
                --set replicaCount=1
                
                # 4. Clean up the temporary file (Good practice)
                rm /tmp/kubeconfig_dev
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
                
                # --- GESTION DU KUBECONFIG ---
                echo "${config}" > /tmp/kubeconfig_qa
                export KUBECONFIG=/tmp/kubeconfig_qa
                # -----------------------------
                
                helm upgrade --install fastapiapp-qa ${HELM_CHART_PATH} \\
                --namespace qa \\
                --create-namespace \\
                --set movie.image.repository=${MOVIE_IMAGE_BASE} \\
                --set cast.image.repository=${CAST_IMAGE_BASE} \\
                --set image.tag=${IMAGE_TAG} \\
                --set replicaCount=2
                
                # Nettoyage
                rm /tmp/kubeconfig_qa
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
                
                # --- GESTION DU KUBECONFIG ---
                echo "${config}" > /tmp/kubeconfig_staging
                export KUBECONFIG=/tmp/kubeconfig_staging
                # -----------------------------
                
                helm upgrade --install fastapiapp-staging ${HELM_CHART_PATH} \\
                --namespace staging \\
                --create-namespace \\
                --set movie.image.repository=${MOVIE_IMAGE_BASE} \\
                --set cast.image.repository=${CAST_IMAGE_BASE} \\
                --set image.tag=${IMAGE_TAG} \\
                --set replicaCount=2
                
                # Nettoyage
                rm /tmp/kubeconfig_staging
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
                
                # --- GESTION DU KUBECONFIG ---
                echo "${config}" > /tmp/kubeconfig_prod
                export KUBECONFIG=/tmp/kubeconfig_prod
                # -----------------------------
                
                helm upgrade --install fastapiapp-prod ${HELM_CHART_PATH} \\
                --namespace prod \\
                --create-namespace \\
                --set movie.image.repository=${MOVIE_IMAGE_BASE} \\
                --set cast.image.repository=${CAST_IMAGE_BASE} \\
                --set image.tag=${IMAGE_TAG} \\
                --set replicaCount=3
                
                # Nettoyage
                rm /tmp/kubeconfig_prod
                """
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
