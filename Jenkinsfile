pipeline {
    agent any

    environment {
        DOCKER_ID = 'dalfang'
        DOCKER_IMAGE = 'lioraapi'
        DOCKER_TAG = "v.${BUILD_ID}.0"
    }

    stages {
        stage('Docker Build') {
            steps {
                sh '''
                    docker rm -f jenkins-test || true
                    docker build -t $DOCKER_ID/$DOCKER_IMAGE:$DOCKER_TAG .
                '''
            }
        }

        stage('Docker Run') {
            steps {
                sh '''
                    docker run -d --rm -p 8081:80 --name jenkins-test \
                      $DOCKER_ID/$DOCKER_IMAGE:$DOCKER_TAG
                '''
            }
        }

        stage('Test Acceptance') {
            steps {
                sh '''
                    curl --fail --retry 10 --retry-connrefused \
                      http://localhost:8081/
                    docker rm -f jenkins-test
                '''
            }
        }

        stage('Docker Push') {
            environment {
                DOCKER_PASS = credentials('DOCKER_HUB_PASS')
            }
            steps {
                sh '''
                    printf '%s' "$DOCKER_PASS" | docker login \
                      --username "$DOCKER_ID" --password-stdin
                    docker push $DOCKER_ID/$DOCKER_IMAGE:$DOCKER_TAG
                    docker logout
                '''
            }
        }

        stage('Deployment dev') {
            environment {
                KUBECONFIG = credentials('config')
            }
            steps {
                sh '''
                    helm upgrade --install app fastapi \
                      --namespace dev \
                      --set image.repository=$DOCKER_ID/$DOCKER_IMAGE \
                      --set image.tag=$DOCKER_TAG
                '''
            }
        }

        stage('Deployment staging') {
            environment {
                KUBECONFIG = credentials('config')
            }
            steps {
                sh '''
                    helm upgrade --install app fastapi \
                      --namespace staging \
                      --set image.repository=$DOCKER_ID/$DOCKER_IMAGE \
                      --set image.tag=$DOCKER_TAG
                '''
            }
        }

        stage('Deployment prod') {
            input {
                message 'Deploy to production?'
                ok 'Yes'
            }
            environment {
                KUBECONFIG = credentials('config')
            }
            steps {
                sh '''
                    helm upgrade --install app fastapi \
                      --namespace prod \
                      --set image.repository=$DOCKER_ID/$DOCKER_IMAGE \
                      --set image.tag=$DOCKER_TAG
                '''
            }
        }
    }

    post {
        always {
            sh 'docker rm -f jenkins-test || true'
        }
    }
}
