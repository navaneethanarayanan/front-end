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

  # React / Node.js Container
  - name: node
    image: node:22
    command:
    - cat
    tty: true
    workingDir: /home/jenkins/agent

    volumeMounts:
    - name: workspace-volume
      mountPath: /home/jenkins/agent


  # Docker CLI Container
  - name: docker
    image: docker:29.3.0-cli
    command:
    - cat
    tty: true
    workingDir: /home/jenkins/agent

    env:
    - name: DOCKER_HOST
      value: tcp://localhost:2376

    volumeMounts:
    - name: workspace-volume
      mountPath: /home/jenkins/agent


  # Docker-in-Docker Daemon
  - name: dind
    image: docker:29.3.0-dind

    securityContext:
      privileged: true

    command:
    - dockerd

    args:
    - "--host=tcp://0.0.0.0:2376"

    env:
    - name: DOCKER_TLS_CERTDIR
      value: ""

    volumeMounts:
    - name: dind-storage
      mountPath: /var/lib/docker


  # kubectl Container
  - name: kubectl
    image: alpine/k8s:1.31.0
    command:
    - cat
    tty: true
    workingDir: /home/jenkins/agent

    volumeMounts:
    - name: workspace-volume
      mountPath: /home/jenkins/agent


  # Jenkins Agent
  - name: jnlp
    image: jenkins/inbound-agent:latest

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
        K8S_NAMESPACE = "default"   // <-- set this to match the RBAC namespace above

    }


    stages {


        stage('Check Node') {

            steps {

                container('node') {

                    sh '''
                    echo "Node Version"
                    node --version

                    echo "NPM Version"
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


        stage('Check Docker Connection') {

            steps {

                container('docker') {

                    sh '''
                    echo "Checking Docker"

                    for i in $(seq 1 10); do
                        docker version && break
                        echo "Waiting for dind daemon to be ready..."
                        sleep 3
                    done
                    '''

                }

            }

        }



        stage('Docker Build') {

            steps {

                container('docker') {

                    echo "Building Docker Image"

                    sh '''
                    docker build \
                    -t ${DOCKER_IMAGE}:${IMAGE_TAG} .
                    '''

                }

            }

        }



        stage('Docker Login & Push') {

            steps {

                container('docker') {


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

                container('kubectl') {

                    sh '''

                    kubectl set image deployment/gen-ai-project \
                    gen-ai-project=${DOCKER_IMAGE}:${IMAGE_TAG} \
                    -n ${K8S_NAMESPACE}

                    '''

                }

            }

        }



        stage('Deploy Kubernetes') {

            steps {

                container('kubectl') {

                    sh '''

                    kubectl apply -f k8s/deployment.yaml -n ${K8S_NAMESPACE}

                    kubectl apply -f k8s/service.yaml -n ${K8S_NAMESPACE}


                    kubectl rollout status \
                    deployment/gen-ai-project \
                    -n ${K8S_NAMESPACE}

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