pipeline {
    agent any

    environment {
        DOCKER_NAMESPACE = 'dalfang'
        IMAGE_TAG = "build-${BUILD_NUMBER}"
        MOVIE_IMAGE = 'dalfang/movie-service'
        CAST_IMAGE = 'dalfang/cast-service'
    }

    stages {
        stage('Compose smoke tests') {
            steps {
                sh '''
                    docker compose up --build -d
                    check_url() {
                      docker compose exec -T movie_service python - "$1" <<'PY'
import sys
import time
import urllib.request

url = sys.argv[1]
for attempt in range(30):
    try:
        with urllib.request.urlopen(url, timeout=2) as response:
            if response.status == 200:
                print("Healthy:", url)
                break
    except OSError:
        if attempt == 29:
            raise
        time.sleep(1)
PY
                    }
                    check_url http://127.0.0.1:8000/api/v1/checkapi
                    check_url http://cast_service:8000/api/v1/checkapi
                    check_url http://nginx:8080/api/v1/movies
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