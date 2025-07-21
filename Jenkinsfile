pipeline {
  agent any

  environment {
    REGISTRY = 'docker.io/belka75'
    IMAGE_MOVIE = "${REGISTRY}/movie-svc"
    IMAGE_CAST = "${REGISTRY}/cast-svc"
    CHART_PATH_MOVIE = 'charts/movie'
    CHART_PATH_CAST = 'charts/cast'
    NAMESPACE = "${env.BRANCH_NAME}" // dev, qa, staging, prod
    TAG = "${env.BRANCH_NAME}-${env.BUILD_NUMBER}"
  }

  options {
    skipStagesAfterUnstable()
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build Docker Images') {
      steps {
        script {
          docker.build("movie-svc:${TAG}", 'movie-service')
          docker.build("cast-svc:${TAG}", 'cast-service')
        }
      }
    }

    stage('Push to DockerHub') {
      steps {
        withDockerRegistry([ credentialsId: 'DOCKER_HUB_CREDENTIAL', url: '' ]) {
          script {
            docker.image("movie-svc:${TAG}").push()
            docker.image("cast-svc:${TAG}").push()
          }
        }
      }
    }

    stage('Deploy to Kubernetes') {
      when {
        expression {
          // deploy all exept "master"
          return env.BRANCH_NAME != 'master'
        }
      }
      steps {
        script {
          sh """
            helm upgrade --install movie ${CHART_PATH_MOVIE} \
              --namespace ${NAMESPACE} --create-namespace \
              --set image.repository=${IMAGE_MOVIE} \
              --set image.tag=${TAG}

            helm upgrade --install cast ${CHART_PATH_CAST} \
              --namespace ${NAMESPACE} --create-namespace \
              --set image.repository=${IMAGE_CAST} \
              --set image.tag=${TAG}
          """
        }
      }
    }

    stage('manual approval for production') {
      when {
        branch 'master'
      }
      steps {
        input message: 'would you like to deploy in prod ?'
      }
    }

    stage('deploy to production') {
      when {
        branch 'master'
      }
      steps {
        script {
          sh """
            helm upgrade --install movie ${CHART_PATH_MOVIE} \
              --namespace prod --create-namespace \
              --set image.repository=${IMAGE_MOVIE} \
              --set image.tag=${TAG}

            helm upgrade --install cast ${CHART_PATH_CAST} \
              --namespace prod --create-namespace \
              --set image.repository=${IMAGE_CAST} \
              --set image.tag=${TAG}
          """
        }
      }
    }
  }

  post {
    success {
      echo "deployment successful for ${env.BRANCH_NAME}"
    }
    failure {
      echo " pipeline failed"
    }
    always {
      sh 'docker system prune -f'
    }
  }
}
