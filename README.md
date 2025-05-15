# JenkinsfileMbala
pipeline {
  agent any

  environment {
    IMAGE_NAME = 'mon-app-python'
    IMAGE_TAG = "${env.BUILD_NUMBER}"
    DOCKER_REGISTRY = 'docker.io/monutilisateur'
  }

  tools {
    // Si vous utilisez Node ou autre, ajoutez ici
    // nodejs 'Node 16'
  }

  stages {
    stage('Checkout') {
      steps {
        git(
          url: 'https://github.com/mon-org/mon-app-Mbala.git',
          credentialsId: 'github-creds',
          branch: 'main'
        )
      }
    }

    stage('Install dependencies') {
      steps {
        sh 'pip install -r requirements.txt'
      }
    }

    stage('Run tests') {
      steps {
        sh 'pytest --junitxml=test-results/results.xml'
      }
      post {
        always {
          junit 'test-results/*.xml'
        }
      }
    }

    stage('Build Docker Image') {
      steps {
        sh "docker build -t ${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} ."
      }
    }

    stage('Push to Docker Hub') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
          sh '''
            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
            docker push ${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}
          '''
        }
      }
    }

    stage('Deploy (ex: via SSH)') {
      steps {
        sshagent(['ssh-server-creds']) {
          sh 'ssh user@serveur "docker pull ${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} && docker run -d -p 80:80 ${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}"'
        }
      }
    }
  }

  post {
    success {
      emailext to: 'dev@entreprise.com',
               subject: "✅ Build #${env.BUILD_NUMBER} SUCCESS",
               body: "Voir logs : ${env.BUILD_URL}"
    }
    failure {
      emailext to: 'dev@entreprise.com',
               subject: "❌ Build #${env.BUILD_NUMBER} FAILED",
               body: "Voir logs : ${env.BUILD_URL}"
    }
  }
}
