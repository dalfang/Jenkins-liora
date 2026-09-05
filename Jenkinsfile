pipeline {
    agent any

    environment {
        DOCKER_NAMESPACE = 'dalfang'
        IMAGE_TAG = "${env.BUILD_NUMBER}-${env.GIT_COMMIT ?: 'pending'}"
        MOVIE_IMAGE = "${DOCKER_NAMESPACE}/movie-service"
        CAST_IMAGE = "${DOCKER_NAMESPACE}/cast-service"
    }

    stages {
        stage('Compose smoke tests') {
            steps {
                sh '''
                    docker compose up --build -d
                    curl --fail --retry 12 --retry-connrefused http://127.0.0.1:8001/api/v1/checkapi
                    curl --fail --retry 12 --retry-connrefused http://127.0.0.1:8002/api/v1/checkapi
                    curl --fail --retry 12 --retry-connrefused http://127.0.0.1:8080/api/v1/movies
                    docker compose down -v
                '''
            }
        }

        stage('Build images') {
            steps {
                sh '''
                    docker build -t $MOVIE_IMAGE:$IMAGE_TAG movie-service
                    docker build -t $CAST_IMAGE:$IMAGE_TAG cast-service
                '''
            }
        }

        stage('Push images') {
            environment {
                DOCKER_HUB_TOKEN = credentials('DOCKER_HUB_PASS')
            }
            steps {
                sh '''
                    printf '%s' "$DOCKER_HUB_TOKEN" | docker login --username "$DOCKER_NAMESPACE" --password-stdin
                    docker push $MOVIE_IMAGE:$IMAGE_TAG
                    docker push $CAST_IMAGE:$IMAGE_TAG
                    docker logout
                '''
            }
        }

        stage('Deploy dev') {
            steps {
                script { deploy('dev') }
            }
        }

        stage('Deploy qa') {
            steps {
                script { deploy('qa') }
            }
        }

        stage('Deploy staging') {
            steps {
                script { deploy('staging') }
            }
        }

        stage('Approve production') {
            when {
                expression { env.BRANCH_NAME == 'master' || env.GIT_BRANCH == 'origin/master' }
            }
            input {
                message 'Deploy this image to production?'
                ok 'Deploy'
            }
            steps {
                echo 'Production deployment approved.'
            }
        }

        stage('Deploy production') {
            when {
                expression { env.BRANCH_NAME == 'master' || env.GIT_BRANCH == 'origin/master' }
            }
            steps {
                script { deploy('prod') }
            }
        }
    }

    post {
        always {
            sh 'docker compose down -v || true'
        }
    }
}

def deploy(environmentName) {
    withCredentials([file(credentialsId: 'config', variable: 'KUBECONFIG')]) {
        sh """
            helm upgrade --install movie-platform charts \\
              --namespace ${environmentName} \\
              --create-namespace \\
              --values charts/values-${environmentName}.yaml \\
              --set movieService.image.repository=${env.MOVIE_IMAGE} \\
              --set movieService.image.tag=${env.IMAGE_TAG} \\
              --set castService.image.repository=${env.CAST_IMAGE} \\
              --set castService.image.tag=${env.IMAGE_TAG} \\
              --wait --timeout 5m
        """
    }
}