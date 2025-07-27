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

    stage('checkout') {
      steps {
        checkout scm
      }
    }

    stage('build docker images') {
      steps {
        script {
          def movieImageName = env.IMAGE_MOVIE
          def castImageName = env.IMAGE_CAST
          def tag = env.TAG

          def movieImage = docker.build("${movieImageName}:${tag}", "--label ci.tool=jenkins --label ci.job=${env.JOB_NAME} --label ci.build=${env.BUILD_NUMBER} movie-service")
          def castImage = docker.build("${castImageName}:${tag}", "--label ci.tool=jenkins --label ci.job=${env.JOB_NAME} --label ci.build=${env.BUILD_NUMBER} cast-service")

          // full_name
          env.MOVIE_IMAGE_TAGGED = "${movieImageName}:${tag}"
          env.CAST_IMAGE_TAGGED = "${castImageName}:${tag}"

          echo "Movie image built: ${env.MOVIE_IMAGE_TAGGED}"
          echo "Cast image built: ${env.CAST_IMAGE_TAGGED}"
        }
      }
    }

    stage('push to dockerHub') {
      steps {
        withDockerRegistry([credentialsId: 'DOCKER_HUB_CREDENTIAL', url: 'https://index.docker.io/v1/']) {
          script {
            def movieImage = docker.image(env.MOVIE_IMAGE_TAGGED)
            def castImage = docker.image(env.CAST_IMAGE_TAGGED)

            echo "pushing ${env.MOVIE_IMAGE_TAGGED} to Docker Hub"
            movieImage.push()

            echo "pushing ${env.CAST_IMAGE_TAGGED} to docker hub"
            castImage.push()
          }
        }
      }
    }

    stage('deploy to kubernetes') {
      when {
        expression {
          return env.BRANCH_NAME != 'master'
        }
      }
      steps {
        withCredentials([file(credentialsId: 'config', variable: 'KUBECONFIG')]) {
          script {
            sh """
              set -eux
              export KUBECONFIG=$KUBECONFIG

              helm upgrade --install movie ${CHART_PATH_MOVIE} \\
                --namespace ${NAMESPACE} --create-namespace \\
                --set image.repository=${IMAGE_MOVIE} \\
                --set image.tag=${TAG}

              helm upgrade --install cast ${CHART_PATH_CAST} \\
                --namespace ${NAMESPACE} --create-namespace \\
                --set image.repository=${IMAGE_CAST} \\
                --set image.tag=${TAG}
            """
          }
        }
      }
    }

    stage('check deployment') {
      steps {
        withCredentials([file(credentialsId: 'config', variable: 'KUBECONFIG')]) {
          script {
            sh """
              set -eux
              export KUBECONFIG=$KUBECONFIG

              echo "checking rollout status deployments in namespace ${NAMESPACE}..."
              kubectl rollout status deployment/movie-svc -n ${NAMESPACE} --timeout=120s
              kubectl rollout status deployment/cast-svc -n ${NAMESPACE} --timeout=120s

              echo "deployments are running!"
              echo "current pods status:"
              kubectl get pods -n ${NAMESPACE}
            """
          }
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
        withCredentials([file(credentialsId: 'config', variable: 'KUBECONFIG')]) {
          script {
            sh """
              set -eux
              export KUBECONFIG=$KUBECONFIG

              helm upgrade --install movie ${CHART_PATH_MOVIE} \\
                --namespace prod --create-namespace \\
                --set image.repository=${IMAGE_MOVIE} \\
                --set image.tag=${TAG}

              helm upgrade --install cast ${CHART_PATH_CAST} \\
                --namespace prod --create-namespace \\
                --set image.repository=${IMAGE_CAST} \\
                --set image.tag=${TAG}
            """
          }
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
