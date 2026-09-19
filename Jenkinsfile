pipeline {
    agent any

    tools {
        maven 'mymaven'
    }

    environment {
        APP_NAME   = 'sample-web-app'

        ACR_NAME   = 'acrteja23'
        ACR_SERVER = 'acrteja23.azurecr.io'

        JFROG_URL  = 'http://20.88.47.132:8082/artifactory'

        RESOURCE_GROUP = 'teja-rg'
        DEPLOY_VM      = 'docker-vm'

        JF = '/var/lib/jenkins/tools/io.jenkins.plugins.jfrog.JfrogInstallation/jfrog-cli/jf'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh '''
                    mvn clean package
                '''
            }
        }

        stage('Get Version') {
            steps {
                script {
                    env.VERSION = sh(
                        script: '''
                            mvn help:evaluate \
                              -Dexpression=project.version \
                              -q \
                              -DforceStdout
                        ''',
                        returnStdout: true
                    ).trim()

                    echo "Application Version = ${env.VERSION}"
                }
            }
        }

        stage('Publish to JFrog') {
            steps {
                withCredentials([
                    string(
                        credentialsId: 'jfrog-access-token',
                        variable: 'JFROG_TOKEN'
                    )
                ]) {
                    sh '''
                        ${JF} rt u \
                          "target/${APP_NAME}-${VERSION}.jar" \
                          "maven-local/" \
                          --flat=true \
                          --url="${JFROG_URL}" \
                          --access-token="${JFROG_TOKEN}"
                    '''
                }
            }
        }

        stage('Retrieve Artifact') {
            steps {
                withCredentials([
                    string(
                        credentialsId: 'jfrog-access-token',
                        variable: 'JFROG_TOKEN'
                    )
                ]) {
                    sh '''
                        rm -f "${APP_NAME}-${VERSION}.jar"

                        ${JF} rt dl \
                          "maven-local/${APP_NAME}-${VERSION}.jar" \
                          "./" \
                          --flat=true \
                          --url="${JFROG_URL}" \
                          --access-token="${JFROG_TOKEN}"

                        test -f "${APP_NAME}-${VERSION}.jar"

                        cp "${APP_NAME}-${VERSION}.jar" app.jar
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build \
                      -t "${APP_NAME}:${VERSION}" .
                '''
            }
        }

        stage('Push to ACR') {
            steps {
                sh '''
                    az login --identity --output none

                    az acr login \
                      --name "${ACR_NAME}"

                    docker tag \
                      "${APP_NAME}:${VERSION}" \
                      "${ACR_SERVER}/${APP_NAME}:${VERSION}"

                    docker push \
                      "${ACR_SERVER}/${APP_NAME}:${VERSION}"
                '''
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                    az vm run-command invoke \
                      --resource-group "${RESOURCE_GROUP}" \
                      --name "${DEPLOY_VM}" \
                      --command-id RunShellScript \
                      --scripts "
                        set -e

                        az login --identity --output none

                        az acr login \
                          --name ${ACR_NAME}

                        docker pull \
                          ${ACR_SERVER}/${APP_NAME}:${VERSION}

                        docker rm -f \
                          ${APP_NAME} || true

                        docker run -d \
                          --name ${APP_NAME} \
                          --restart unless-stopped \
                          -p 8080:8080 \
                          ${ACR_SERVER}/${APP_NAME}:${VERSION}

                        docker ps \
                          --filter name=${APP_NAME}
                      "
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                    az vm run-command invoke \
                      --resource-group "${RESOURCE_GROUP}" \
                      --name "${DEPLOY_VM}" \
                      --command-id RunShellScript \
                      --scripts "
                        docker inspect \
                          ${APP_NAME} \
                          --format='{{.Config.Image}}'

                        curl --fail \
                          http://localhost:8080
                      "
                '''
            }
        }
    }

    post {
        success {
            echo "Deployment successful: ${ACR_SERVER}/${APP_NAME}:${VERSION}"
        }

        failure {
            echo "Pipeline failed. Check the failed stage."
        }
    }
}
