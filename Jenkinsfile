pipeline {
  agent any

  tools {
    maven 'Maven3'
  }

  environment {
    DOCKERHUB_USER   = 'nama7'
    APP_NAME         = 'springboot-app'
    DOCKER_IMAGE     = "${DOCKERHUB_USER}/${APP_NAME}"
    TAG              = "${BUILD_NUMBER}"
    KUBE_NAMESPACE   = 'prod'
    KUBE_CONTEXT     = 'kind-cilium-lab'
    
  }

  stages {
    stage('Commit Details') {
      steps {
        script {
          env.COMMIT_HASH = bat(script: '@git rev-parse --short HEAD', returnStdout: true).trim()
          def msg = bat(script: '@git log -1 --pretty=%%B', returnStdout: true).trim()
          echo "Commit: ${env.COMMIT_HASH}"
          echo "Message: ${msg}"
          currentBuild.displayName = "#${env.TAG}-${env.COMMIT_HASH}"
        }
      }
    }

    stage('Build App') {
      steps {
        bat 'mvn clean package -DskipTests'
        archiveArtifacts artifacts: 'target/*.jar'
      }
    }

    stage('Build Docker Image') {
      steps {
        bat 'docker build -t %DOCKER_IMAGE%:%TAG% --label commit=%COMMIT_HASH% .'
       
      }
    }

    stage('Push to Docker Hub') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: '30df0937-c64c-48a6-baaf-a594a04f86c4',
          usernameVariable: 'DOCKER_USER',
          passwordVariable: 'DOCKER_PASS'
        )]) {
          bat """
            echo %DOCKER_PASS%| docker login -u %DOCKER_USER% --password-stdin
            docker push ${DOCKER_IMAGE}:${TAG}
          """
        }
      }
    }

    stage('Deploy to Kubernetes') {
      steps {
        bat """
          kubectl config use-context %KUBE_CONTEXT%
          kubectl config set-context --current --namespace=%KUBE_NAMESPACE%
          sed 's|__TAG__|%TAG%|g' k8s/deployment.yaml > k8s/rendered-deployment.yaml
          kubectl apply -f k8s/rendered-deployment.yaml
          kubectl apply -f k8s/service.yaml
        """
      }
    }

//     stage('Verify Deployment') {
//       steps {
//         script {
//           sh "kubectl rollout status deployment/${APP_NAME} --timeout=300s"

//           def pods = sh(script: "kubectl get pods -l app=${APP_NAME} -o jsonpath='{.items[*].status.phase}'", returnStdout: true).trim()
//           echo "📦 Pod Status: ${pods}"

//           def servicePort = sh(script: "kubectl get svc ${APP_NAME} -o jsonpath='{.spec.ports[0].port}'", returnStdout: true).trim()
//           def serviceIP = sh(script: "kubectl get svc ${APP_NAME} -o jsonpath='{.spec.clusterIP}'", returnStdout: true).trim()
//           echo "🌐 Internal Service URL: http://${serviceIP}:${servicePort}"
//         }
//       }
//     }
//   }

//   post {
//     success {
//       echo "✅ Deployment successful! Tag: ${TAG}"
//     }
//     failure {
//       echo "❌ Deployment failed!"
//       script {
//         sh "kubectl rollout undo deployment/${APP_NAME} || true"
//       }
//     }
  }
}
