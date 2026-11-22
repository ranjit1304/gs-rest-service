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
                // Practice setup only: opens the Docker socket for the agent user
                sh 'sudo -n chmod 666 /var/run/docker.sock'
                sh '''
                    docker run --rm \
                      -v "$WORKSPACE":/workspace \
                      -v maven-repo:/root/.m2 \
                      -w /workspace/${APP_DIR} \
                      maven:3.9-eclipse-temurin-17 \
                      mvn -B -ntp clean verify
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
RUN useradd -r -u 1001 appuser
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
                    docker
