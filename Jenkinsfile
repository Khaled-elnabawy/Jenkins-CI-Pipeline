pipeline {
  agent any
  environment {
    DOCKERHUB_REPO = "ahmedlebshten/helloapp"
    IMAGE_TAG = "${env.BUILD_NUMBER}"
    CD_REPO = "https://github.com/Ahmedlebshten/ArgoCD-Pipeline.git"
    CD_REPO_PATH = "app-cd/manifests"   // adjust if your manifest path differs
    DEPLOY_FILE = "deployment.yaml"     // file inside CD_REPO_PATH to update
  }
  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build Docker Image') {
      steps {
        sh "docker build -t ${DOCKERHUB_REPO}:${IMAGE_TAG} ."
        sh "docker tag ${DOCKERHUB_REPO}:${IMAGE_TAG} ${DOCKERHUB_REPO}:latest"
      }
    }

    stage('Docker Login') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials',
                                          usernameVariable: 'DH_USER',
                                          passwordVariable: 'DH_PASS')]) {
          sh 'echo $DH_PASS | docker login -u $DH_USER --password-stdin'
        }
      }
    }

    stage('Push Docker Image') {
      steps {
        sh "docker push ${DOCKERHUB_REPO}:${IMAGE_TAG}"
        sh "docker push ${DOCKERHUB_REPO}:latest"
      }
    }

    stage('Update CD repo (bump image tag)') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'git-cred',
                                          usernameVariable: 'GIT_USER',
                                          passwordVariable: 'GIT_PASS')]) {
          sh '''
            set -e
            rm -rf cd-repo
            # clone using credentials (token as password)
            git clone https://${GIT_USER}:${GIT_PASS}@github.com/Ahmedlebshten/ArgoCD-Pipeline.git cd-repo
            cd cd-repo || exit 2

            # go to path that contains the deployment file
            cd "${CD_REPO_PATH}" || (echo "Path ${CD_REPO_PATH} not found" && exit 3)

            # file to update
            if [ ! -f "${DEPLOY_FILE}" ]; then
              echo "ERROR: ${DEPLOY_FILE} not found under ${CD_REPO_PATH}"
              exit 4
            fi

            # update image tag (assumes line contains image: <repo>:<tag>)
            # safer: replace only the image line - adjust repo name if needed
            sed -i "s|\\(image:[[:space:]]${DOCKERHUB_REPO}:\\).|\\1${IMAGE_TAG}|g" "${DEPLOY_FILE}"

            # show diff for debug
            git --no-pager diff -- "${DEPLOY_FILE}" || true

            # commit & push if changed
            git config user.email "jenkins@ci.local"
            git config user.name "jenkins"
            if git diff --quiet; then
              echo "No changes in ${DEPLOY_FILE} (image already up-to-date)"
            else
              git add "${DEPLOY_FILE}"
              git commit -m "ci: bump image to ${IMAGE_TAG}"
              git push origin HEAD
            fi
          '''
        }
      }
    }
  }

  post {
    success { echo "Done: ${DOCKERHUB_REPO}:${IMAGE_TAG}" }
    failure { echo "Failed" }
  }
}
