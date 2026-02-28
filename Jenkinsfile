pipeline {
  agent {
    kubernetes {
      yaml """
        apiVersion: v1
        kind: Pod
        spec:
          serviceAccountName: jenkins
          containers:
          - name: docker
            image: docker:24-dind
            securityContext:
              privileged: true
            env:
            - name: DOCKER_TLS_CERTDIR
              value: ""
            volumeMounts:
            - name: docker-graph-storage
              mountPath: /var/lib/docker
          - name: tools
            image: alpine/git:latest
            command: ['sleep', 'infinity']
            env:
            - name: DOCKER_HOST
              value: tcp://localhost:2375
          volumes:
          - name: docker-graph-storage
            emptyDir: {}
      """
    }
  }

  environment {
    AWS_REGION      = 'ap-south-1'
    AWS_ACCOUNT_ID  = '196549506578'
    ECR_REPO        = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/nodejs-app"
    // Short commit hash for cleaner tagging
    IMAGE_TAG       = "${BUILD_NUMBER}-${GIT_COMMIT[0..6]}"
  }

  stages {
    stage('Install Tools') {
      steps {
        container('tools') {
          sh """
            apk add --no-cache docker-cli aws-cli
          """
        }
      }
    }

    stage('Build Docker Image') {
      steps {
        container('tools') {
          sh """
            docker build -t ${ECR_REPO}:${IMAGE_TAG} .
            docker tag ${ECR_REPO}:${IMAGE_TAG} ${ECR_REPO}:latest
          """
        }
      }
    }

    stage('Push to ECR') {
      steps {
        container('tools') {
          sh """
            aws ecr get-login-password --region ${AWS_REGION} | \
              docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com

            docker push ${ECR_REPO}:${IMAGE_TAG}
            docker push ${ECR_REPO}:latest
          """
        }
      }
    }

    stage('Update Manifests Repo') {
      steps {
        container('tools') {
          // Ensure 'github-credentials' matches the ID you created in Jenkins
          withCredentials([usernamePassword(
            credentialsId: 'github-credentials', 
            usernameVariable: 'GIT_USER',
            passwordVariable: 'GIT_TOKEN'
          )]) {
            sh """
              git clone https://${GIT_USER}:${GIT_TOKEN}@github.com/niketvjoshi/nodejs-app-manifests.git
              cd nodejs-app-manifests

              # Update the image tag in rollout.yaml
              sed -i "s|image: ${ECR_REPO}:.*|image: ${ECR_REPO}:${IMAGE_TAG}|" k8s/rollout.yaml

              git config user.email "jenkins@ci.local"
              git config user.name "Jenkins"
              git add k8s/rollout.yaml
              git commit -m "ci: update image to ${IMAGE_TAG} [skip ci]"
              git push origin main
            """
          }
        }
      }
    }
  }

  post {
    success {
      echo "Pipeline succeeded. ArgoCD will detect the manifest change and trigger rollout."
    }
    failure {
      echo "Pipeline failed. Check logs above."
    }
  }
}
