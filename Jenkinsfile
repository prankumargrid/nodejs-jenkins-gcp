pipeline {
  agent any

  environment {
    APP_NAME = "nodejs-jenkins-gcp"
    GAR_REPO = "nodejs-repo"
    IMAGE_TAG = "${env.BUILD_NUMBER}"
  }

  options {
    timestamps()
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '20'))
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Install Dependencies') {
      agent {
        docker {
          image 'node:20-alpine'
          args '-u root:root'
        }
      }
      steps {
        sh 'npm ci'
      }
    }

    stage('Run Unit Tests') {
      agent {
        docker {
          image 'node:20-alpine'
          args '-u root:root'
        }
      }
      steps {
        sh 'npm test'
      }
    }

    stage('Build Docker Image') {
      steps {
        withCredentials([
          string(credentialsId: 'gcp-project-id', variable: 'GCP_PROJECT_ID'),
          string(credentialsId: 'artifact-registry-region', variable: 'GAR_REGION')
        ]) {
          script {
            env.FULL_IMAGE = "${GAR_REGION}-docker.pkg.dev/${GCP_PROJECT_ID}/${GAR_REPO}/${APP_NAME}:${IMAGE_TAG}"
            env.FULL_IMAGE_LATEST = "${GAR_REGION}-docker.pkg.dev/${GCP_PROJECT_ID}/${GAR_REPO}/${APP_NAME}:latest"
          }
          sh '''
            docker build -t $FULL_IMAGE .
            docker tag $FULL_IMAGE $FULL_IMAGE_LATEST
          '''
        }
      }
    }

    stage('Security Scan - Trivy') {
      steps {
        sh '''
          docker run --rm \
            -v /var/run/docker.sock:/var/run/docker.sock \
            -v $HOME/.cache:/root/.cache/ \
            aquasec/trivy image \
            --severity HIGH,CRITICAL \
            --exit-code 1 \
            --no-progress \
            $FULL_IMAGE
        '''
      }
    }

    stage('Authenticate to GCP') {
      steps {
        withCredentials([
          file(credentialsId: 'gcp-service-account-json', variable: 'GOOGLE_APPLICATION_CREDENTIALS'),
          string(credentialsId: 'artifact-registry-region', variable: 'GAR_REGION')
        ]) {
          sh '''
            gcloud auth activate-service-account --key-file="$GOOGLE_APPLICATION_CREDENTIALS"
            gcloud auth configure-docker ${GAR_REGION}-docker.pkg.dev --quiet
          '''
        }
      }
    }

    stage('Push Docker Image') {
      steps {
        sh '''
          docker push $FULL_IMAGE
          docker push $FULL_IMAGE_LATEST
        '''
      }
    }

    stage('Deploy to GKE') {
      when {
        branch 'main'
      }
      steps {
        withCredentials([
          file(credentialsId: 'gcp-service-account-json', variable: 'GOOGLE_APPLICATION_CREDENTIALS'),
          string(credentialsId: 'gcp-project-id', variable: 'GCP_PROJECT_ID'),
          string(credentialsId: 'gke-cluster-name', variable: 'GKE_CLUSTER'),
          string(credentialsId: 'gke-cluster-zone', variable: 'GKE_ZONE')
        ]) {
          sh '''
            gcloud auth activate-service-account --key-file="$GOOGLE_APPLICATION_CREDENTIALS"
            gcloud container clusters get-credentials "$GKE_CLUSTER" --zone "$GKE_ZONE" --project "$GCP_PROJECT_ID"

            sed "s|IMAGE_PLACEHOLDER|$FULL_IMAGE|g" k8s/deployment.yaml | kubectl apply -f -
            kubectl apply -f k8s/service.yaml
            kubectl rollout status deployment/nodejs-jenkins-gcp
          '''
        }
      }
    }
  }

  post {
    success {
      echo 'Build and deployment succeeded'
      // slackSend channel: '#deployments', color: 'good', message: "SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
    }
    failure {
      echo 'Build or deployment failed'
      // slackSend channel: '#deployments', color: 'danger', message: "FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
    }
    always {
      cleanWs()
    }
  }
}