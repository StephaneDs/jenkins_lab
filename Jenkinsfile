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
          def movieImageName = env.IMAGE_MOVIE
          def castImageName = env.IMAGE_CAST
          def tag = env.TAG

          def movieImage = docker.build("${movieImageName}:${tag}", 'movie-service')
          def castImage = docker.build("${castImageName}:${tag}", 'cast-service')

          echo "movie image created with ID: ${movieImage.id()}"
          echo "cast imge created with ID: ${castImage.id()}"

          env.MOVIE_IMAGE_TAGGED = "${movieImageName}:${tag}"
          env.CAST_IMAGE_TAGGED = "${castImageName}:${tag}"
        }
      }
    }

    stage('Push to DockerHub') {
      steps {
        withDockerRegistry([ credentialsId: 'DOCKER_HUB_CREDENTIAL', url: '' ]) {
          script {
            def movieImage = docker.image(env.MOVIE_IMAGE_TAGGED)
            def castImage = docker.image(env.CAST_IMAGE_TAGGED)
            

            echo"pushing ${env.MOVIE_IMAGE_TAGGED} to docker"
            movieImage.push()
            echo"pushing ${env.CAST_IMAGE_TAGGED} to docker"
            castImage.push()
          }
        }
      }
    }

    stage('Deploy to Kubernetes') {
      when {
        expression {
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
        input message: 'Would you like to deploy in production?'
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
      echo "Deployment successful for ${env.BRANCH_NAME}"
    }
    failure {
      echo "Pipeline failed"
    }
    always {
      sh 'docker system prune -f'
    }
  }
}
