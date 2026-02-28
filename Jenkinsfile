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
            image: docker:29-dind
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
    IMAGE_TAG       = "${BUILD_NUMBER}-${GIT_COMMIT[0..6]}"
    DOCKER_HOST     = "tcp://localhost:2375" // Ensure global access for all stages
  }

  stages {
    stage('Prepare environment') {
      steps {
        container('tools') {
          sh "apk add --no-cache docker-cli aws-cli"
          
          // CRITICAL: Wait for Docker to be ready
          script {
            def dockerReady = false
            for (int i = 0; i < 10; i++) {
              try {
                sh "docker info"
                dockerReady = true
                break
              } catch (Exception e) {
                echo "Waiting for Docker daemon to start... (attempt ${i+1}/10)"
                sleep 5
              }
            }
            if (!dockerReady) {
              error "Docker daemon failed to start within 50 seconds."
            }
          }
        }
      }
    }

    stage('Build & Push Docker Image') {
      steps {
        container('tools') {
          sh """
            # Build
            docker build -t ${ECR_REPO}:${IMAGE_TAG} .
            docker tag ${ECR_REPO}:${IMAGE_TAG} ${ECR_REPO}:latest
            
            # Login & Push
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
          withCredentials([usernamePassword(
            credentialsId: 'github-token-niket', 
            usernameVariable: 'GIT_USER',
            passwordVariable: 'GIT_TOKEN'
          )]) {
            sh """
              git clone https://${GIT_USER}:${GIT_TOKEN}@github.com/niketvjoshi/nodejs-app-manifests.git
              cd nodejs-app-manifests

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
    success { echo "Pipeline succeeded." }
    failure { echo "Pipeline failed." }
  }
}
