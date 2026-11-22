pipeline {
    agent none

    options {
        timestamps()
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
        disableConcurrentBuilds()
    }

    environment {
        APP_DIR = 'complete'
        IMAGE   = 'practice-app'
        VERSION = '1.0.001'
    }

    stages {

        stage('Build & Test') {
            agent { label 'Slave1' }
            steps {
                // Practice setup only: remove after adding the agent user to the docker group
                sh 'sudo -n chmod 666 /var/run/docker.sock'
                sh '''
                    mkdir -p "$HOME/.m2-docker"
                    docker run --rm \
                      -u "$(id -u):$(id -g)" \
                      -e MAVEN_CONFIG=/var/maven/.m2 \
                      -v "$HOME/.m2-docker":/var/maven/.m2 \
                      -v "$WORKSPACE":/workspace \
                      -w /workspace/${APP_DIR} \
                      maven:3.9-eclipse-temurin-17 \
                      mvn -B -ntp -Duser.home=/var/maven clean verify
                '''
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: "${APP_DIR}/target/surefire-reports/*.xml"
                }
                success {
                    stash name: 'jar', includes: "${APP_DIR}/target/*.jar"
                }
            }
        }

        stage('Docker Image') {
            agent { label 'Slave1' }
            steps {
                unstash 'jar'
                sh 'cp ${APP_DIR}/target/*.jar app.jar'
                writeFile file: 'Dockerfile', text: '''FROM eclipse-temurin:17-jre
RUN useradd -m -u 1001 appuser
USER appuser
COPY app.jar /app/app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
'''
                sh 'docker build -t ${IMAGE}:${VERSION} .'
            }
        }

        stage('Smoke Test') {
            agent { label 'Slave1' }
            steps {
                sh '''
                    docker network inspect jenkins-practice_default >/dev/null 2>&1 || docker network create jenkins-practice_default

                    docker rm -f smoke-test || true
                    docker run -d --name smoke-test --network jenkins-practice_default ${IMAGE}:${VERSION}

                    for i in $(seq 1 30); do
                        code=$(docker run --rm --network jenkins-practice_default curlimages/curl \
                               -s -o /dev/null -w '%{http_code}' http://smoke-test:8080/greeting || true)
                        if [ "$code" = "200" ]; then
                            echo "App is up (attempt $i)"
                            exit 0
                        fi
                        echo "Attempt $i: got '$code', retrying..."
                        sleep 2
                    done

                    echo "App never answered with HTTP 200"
                    exit 1
                '''
            }
            post {
                always {
                    sh 'docker rm -f smoke-test || true'
                }
            }
        }

        stage('Release') {
            agent { label 'Slave1' }
            steps {
                input message: "Release ${IMAGE}:${VERSION}?", ok: 'Release'
                sh '''
                    docker tag ${IMAGE}:${VERSION} ${IMAGE}:release-${VERSION}
                    docker save -o ${IMAGE}-release-${VERSION}.tar ${IMAGE}:release-${VERSION}
                '''
                archiveArtifacts artifacts: '*.tar', fingerprint: true
            }
        }
    }

    post {
        failure {
            echo "Build failed: ${env.JOB_NAME} #${env.BUILD_NUMBER} - ${env.BUILD_URL} (in a real setup, send this via mail/Slack)"
        }
    }
}
