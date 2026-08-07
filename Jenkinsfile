pipeline {

    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: dind-agent

spec:
  containers:

    - name: node
      image: node:22
      command:
        - cat
      tty: true
      workingDir: /home/jenkins/agent
      volumeMounts:
        - name: workspace-volume
          mountPath: /home/jenkins/agent

    - name: dind-daemon
      image: docker:29.3.0-dind
      securityContext:
        privileged: true
      args:
        - "--host=tcp://0.0.0.0:2376"
        - "--host=unix:///var/run/docker.sock"
        - "--insecure-registry=nexus.shaktidb.iitmpravartak.net"
        - "--insecure-registry=sbomsandbox.shaktidb.iitmpravartak.net"
      env:
        - name: DOCKER_TLS_CERTDIR
          value: ""
      volumeMounts:
        - name: dind-storage
          mountPath: /var/lib/docker

    - name: jnlp
      image: nexus.shaktidb.iitmpravartak.net/repository/baseimage/jenkinsdindagent:29.3.0
      env:
        - name: DOCKER_HOST
          value: tcp://localhost:2376
      workingDir: /home/jenkins/agent
      volumeMounts:
        - name: workspace-volume
          mountPath: /home/jenkins/agent

  volumes:

    - name: dind-storage
      emptyDir: {}

    - name: workspace-volume
      emptyDir: {}
'''
        }
    }

    environment {

        DOCKER_IMAGE = "navanee143/gen-ai-project"
        IMAGE_TAG = "${BUILD_NUMBER}"
        DOCKER_CREDENTIALS = "dockerhub-creds"

    }

    stages {

        stage('Check Node') {

            steps {

                container('node') {

                    sh '''
                        echo "Node Version:"
                        node --version

                        echo "NPM Version:"
                        npm --version
                    '''

                }

            }

        }

        stage('Install Dependencies') {

            steps {

                container('node') {

                    echo "Installing Node Dependencies"

                    sh '''
                        npm install
                    '''

                }

            }

        }

        stage('Build React Application') {

            steps {

                container('node') {

                    echo "Building React Application"

                    sh '''
                        npm run build
                    '''

                }

            }

        }

        stage('Docker Build') {

            steps {

                container('jnlp') {

                    echo "Building Docker Image"

                    sh '''
                        docker version

                        docker build \
                        -t ${DOCKER_IMAGE}:${IMAGE_TAG} .
                    '''

                }

            }

        }

        stage('Docker Login & Push') {

            steps {

                container('jnlp') {

                    echo "Pushing Image to Docker Hub"

                    withCredentials([
                        usernamePassword(
                            credentialsId: "${DOCKER_CREDENTIALS}",
                            usernameVariable: 'DOCKER_USER',
                            passwordVariable: 'DOCKER_PASS'
                        )
                    ]) {

                        sh '''
                            echo "$DOCKER_PASS" | docker login \
                            -u "$DOCKER_USER" \
                            --password-stdin

                            docker push ${DOCKER_IMAGE}:${IMAGE_TAG}
                        '''

                    }

                }

            }

        }

        stage('Update Kubernetes Image') {

            steps {

                container('jnlp') {

                    echo "Updating Kubernetes Deployment Image"

                    sh '''
                        kubectl set image deployment/gen-ai-project \
                        gen-ai-project=${DOCKER_IMAGE}:${IMAGE_TAG}
                    '''

                }

            }

        }

        stage('Deploy to Kubernetes') {

            steps {

                container('jnlp') {

                    echo "Deploying Application to Kubernetes"

                    sh '''
                        kubectl apply -f k8s/deployment.yaml

                        kubectl apply -f k8s/service.yaml

                        kubectl rollout status deployment/gen-ai-project
                    '''

                }

            }

        }

    }

    post {

        success {

            echo "✅ CI/CD Pipeline Completed Successfully"

        }

        failure {

            echo "❌ CI/CD Pipeline Failed"

        }

        always {

            echo "Pipeline Execution Completed"

        }

    }

}