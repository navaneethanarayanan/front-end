pipeline {

    agent {
        label 'dind-agent'
    }


    environment {

        DOCKER_IMAGE = "navanee143/gen-ai-project"
        IMAGE_TAG = "${BUILD_NUMBER}"
        DOCKER_CREDENTIALS = "dockerhub-creds"

    }


    stages {


        stage('Install Dependencies') {

            steps {

                echo "Installing Node Dependencies"

                sh '''
                    npm install
                '''

            }

        }



        stage('Build React Application') {

            steps {

                echo "Building React Application"

                sh '''
                    npm run build
                '''

            }

        }



        stage('Docker Build') {

            steps {

                echo "Building Docker Image"

                sh '''

                    docker build \
                    -t ${DOCKER_IMAGE}:${IMAGE_TAG} .

                '''

            }

        }



        stage('Docker Login & Push') {

            steps {

                echo "Pushing Image to Docker Hub"


                withCredentials([

                    usernamePassword(
                        credentialsId: "${DOCKER_CREDENTIALS}",
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )

                ]) {


                    sh '''

                    echo $DOCKER_PASS | docker login \
                    -u $DOCKER_USER \
                    --password-stdin


                    docker push ${DOCKER_IMAGE}:${IMAGE_TAG}


                    '''

                }

            }

        }



        stage('Update Kubernetes Image') {

            steps {

                echo "Updating Kubernetes Deployment Image"


                sh '''

                kubectl set image deployment/gen-ai-project \
                gen-ai-project=${DOCKER_IMAGE}:${IMAGE_TAG}


                '''

            }

        }



        stage('Deploy to Kubernetes') {

            steps {

                echo "Deploying Application to Kubernetes"


                sh '''

                kubectl apply -f k8s/deployment.yaml

                kubectl apply -f k8s/service.yaml


                kubectl rollout status deployment/gen-ai-project


                '''

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