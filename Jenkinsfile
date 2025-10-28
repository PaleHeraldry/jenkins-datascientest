pipeline {
  agent any
  stages {
    stage('Build Docker Images') {
      steps {
        script {
          docker.build("${MOVIE_IMAGE}:${IMAGE_TAG}", "./movie-service")
          docker.build("${CAST_IMAGE}:${IMAGE_TAG}", "./cast-service")
        }

      }
    }

    stage('Push Docker Images') {
      steps {
        script {
          docker.withRegistry("https://${DOCKER_REGISTRY}", "${DOCKER_CREDENTIALS_ID}") {
            docker.image("${MOVIE_IMAGE}:${IMAGE_TAG}").push()
            docker.image("${CAST_IMAGE}:${IMAGE_TAG}").push()
            docker.image("${MOVIE_IMAGE}:${IMAGE_TAG}").push('latest')
            docker.image("${CAST_IMAGE}:${IMAGE_TAG}").push('latest')
          }
        }

      }
    }

    stage('Deploy to Dev') {
      when {
        not {
          branch 'master'
        }

      }
      steps {
        script {
          sh """
          helm upgrade --install fastapiapp-dev ${HELM_CHART_PATH} \
          --namespace dev \
          --create-namespace \
          --set image.tag=${IMAGE_TAG} \
          --set replicaCount=1
          """
        }

      }
    }

    stage('Deploy to QA') {
      when {
        branch 'develop'
      }
      steps {
        script {
          sh """
          helm upgrade --install fastapiapp-qa ${HELM_CHART_PATH} \
          --namespace qa \
          --create-namespace \
          --set image.tag=${IMAGE_TAG} \
          --set replicaCount=2
          """
        }

      }
    }

    stage('Deploy to Staging') {
      when {
        branch 'develop'
      }
      steps {
        script {
          sh """
          helm upgrade --install fastapiapp-staging ${HELM_CHART_PATH} \
          --namespace staging \
          --create-namespace \
          --set image.tag=${IMAGE_TAG} \
          --set replicaCount=2
          """
        }

      }
    }

    stage('Deploy to Production') {
      when {
        branch 'master'
      }
      steps {
        input(message: 'Deploy to Production?', ok: 'Deploy')
        script {
          sh """
          helm upgrade --install fastapiapp-prod ${HELM_CHART_PATH} \
          --namespace prod \
          --create-namespace \
          --set image.tag=${IMAGE_TAG} \
          --set replicaCount=3
          """
        }

      }
    }

  }
  environment {
    DOCKER_REGISTRY = 'docker.io'
    DOCKER_CREDENTIALS_ID = 'rpadallery'
    GIT_CREDENTIALS_ID = 'PaleHeraldry'
    MOVIE_IMAGE = 'rpadallery/movie-service'
    CAST_IMAGE = 'rpadallery/cast-service'
    IMAGE_TAG = "${BUILD_NUMBER}"
    HELM_CHART_PATH = './charts'
  }
  post {
    always {
      cleanWs()
    }

    success {
      echo 'Pipeline completed successfully!'
    }

    failure {
      echo 'Pipeline failed!'
    }

  }
}
