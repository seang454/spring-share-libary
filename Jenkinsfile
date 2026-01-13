@Library('devop_itp_share_library@master') _

pipeline {
    agent any

    environment {
        REPO_NAME  = 'seang454'
        IMAGE_NAME = 'jenkins-itp-spring'
        TAG        = 'latest'
        NETWORK    = 'spring-net'
        DB_NAME    = 'postgres'
        DB_USER    = 'postgres'
        DB_PASS    = 'password'
    }

    stages {

        // 1️⃣ Clone Spring Boot Code
        stage('Clone Code') {
            steps {
                git 'https://github.com/seang454/spring-share-libary.git'
            }
        }

        // 2️⃣ Build & Test with H2 (no need for Postgres)
        stage('Build & Test with H2') {
            steps {
                script {
                    if (fileExists('pom.xml')) {
                        // Maven
                        withEnv(['SPRING_PROFILES_ACTIVE=test']) {
                            sh 'mvn clean test'
                        }
                    } else if (fileExists('build.gradle')) {
                        // Gradle
                        withEnv(['SPRING_PROFILES_ACTIVE=test']) {
                            sh 'chmod +x gradlew && ./gradlew clean test'
                        }
                    } else {
                        error "No build file found"
                    }
                }
            }
        }

        // 3️⃣ Prepare Dockerfile
        stage('Prepare Dockerfile') {
            steps {
                script {
                    // Prioritize project Dockerfile
                    if (fileExists('Dockerfile')) {
                        echo 'Using project Dockerfile.'
                    } else {
                        echo 'Dockerfile not found. Using shared library Dockerfile.'
                        def dockerfileContent = libraryResource 'springboot/dev.Dockerfile'
                        writeFile file: 'Dockerfile', text: dockerfileContent
                    }
                }
            }
        }

        // 4️⃣ Build Docker Image (with no-cache to ensure fresh build)
        stage('Build Image') {
            steps {
                sh 'docker build --no-cache -t ${REPO_NAME}/${IMAGE_NAME}:${TAG} .'
            }
        }

        // 5️⃣ Ensure Docker Hub Repo Exists
        stage('Ensure Docker Hub Repo Exists') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'DOCKERHUB-CREDENTIAL',
                    usernameVariable: 'DH_USERNAME',
                    passwordVariable: 'DH_PASSWORD'
                )]) {
                    sh '''
                    STATUS=$(curl -s -o /dev/null -w "%{http_code}" -u "$DH_USERNAME:$DH_PASSWORD" \
                      https://hub.docker.com/v2/repositories/$REPO_NAME/$IMAGE_NAME/)

                    if [ "$STATUS" -eq 404 ]; then
                      curl -s -u "$DH_USERNAME:$DH_PASSWORD" -X POST \
                        https://hub.docker.com/v2/repositories/ \
                        -H "Content-Type: application/json" \
                        -d "{\"name\":\"$IMAGE_NAME\",\"is_private\":false}"
                    fi
                    '''
                }
            }
        }

        // 6️⃣ Push Docker Image
        stage('Push Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'DOCKERHUB-CREDENTIAL',
                    usernameVariable: 'DH_USERNAME',
                    passwordVariable: 'DH_PASSWORD'
                )]) {
                    sh '''
                    echo "$DH_PASSWORD" | docker login -u "$DH_USERNAME" --password-stdin
                    docker push ${REPO_NAME}/${IMAGE_NAME}:${TAG}
                    docker logout
                    '''
                }
            }
        }

        // 7️⃣ Create Docker Network
        stage('Create Docker Network') {
            steps {
                sh '''
                docker network inspect ${NETWORK} >/dev/null 2>&1 || \
                docker network create ${NETWORK}
                '''
            }
        }

        // 8️⃣ Run PostgreSQL Container
        stage('Run PostgreSQL') {
            steps {
                sh '''
                docker rm -f postgres || true

                docker run -d \
                  --name postgres \
                  --network ${NETWORK} \
                  -e POSTGRES_DB=${DB_NAME} \
                  -e POSTGRES_USER=${DB_USER} \
                  -e POSTGRES_PASSWORD=${DB_PASS} \
                  -p 5432:5432 \
                  postgres:16

                # Wait for Postgres to be ready
                until docker exec postgres pg_isready -U ${DB_USER}; do
                  echo "Waiting for Postgres..."
                  sleep 2
                done
                '''
            }
        }

        // 9️⃣ Run Spring Boot Container
        stage('Run Spring Boot') {
            steps {
                sh '''
                docker rm -f spring-app || true

                docker run -d \
                  --name spring-app \
                  --network ${NETWORK} \
                  -p 8080:8080 \
                  ${REPO_NAME}/${IMAGE_NAME}:${TAG}
                '''
            }
        }
    }
}