pipeline {

    agent any

    options {
        // Checkout will be handled manually
        skipDefaultCheckout(true)

        // Prevent multiple deployments at the same time
        disableConcurrentBuilds()

        // Add timestamps to console logs
        timestamps()
    }

    // Trigger pipeline from GitHub webhook
    triggers {
        githubPush()
    }

    tools {
        maven 'mymaven'
        jdk 'JDK-17'
    }

    environment {

        // Application
        APP_NAME = 'sample-web-app'

        // Azure Container Registry
        ACR_NAME   = 'acrteja23'
        ACR_SERVER = 'acrteja23.azurecr.io'

        // JFrog
        JFROG_URL = 'http://20.88.47.132:8082/artifactory'

        // Azure deployment VM
        RESOURCE_GROUP = 'teja-rg'
        DEPLOY_VM      = 'docker-vm'

        // JFrog CLI
        JF = '/var/lib/jenkins/tools/io.jenkins.plugins.jfrog.JfrogInstallation/jfrog-cli/jf'
    }


    stages {

        // =========================================================
        // 1. Checkout Source Code
        // =========================================================

        stage('Checkout') {

            steps {

                echo '===== Checkout Source Code ====='

                checkout scm
            }
        }


        // =========================================================
        // 2. Verify Java / Maven
        // =========================================================

        stage('Verify Tools') {

            steps {

                echo '===== Verify Java and Maven ====='

                sh '''
                    set -e

                    echo "===== JAVA_HOME ====="
                    echo "${JAVA_HOME}"

                    echo "===== Java Version ====="
                    java -version

                    echo "===== Javac Version ====="
                    javac -version

                    echo "===== Maven Version ====="
                    mvn -version
                '''
            }
        }


        // =========================================================
        // 3. Maven Build
        // =========================================================

        stage('Build') {

            steps {

                echo '===== Maven Build ====='

                sh '''
                    set -e

                    mvn clean package
                '''
            }
        }


        // =========================================================
        // 4. Get Application Version from pom.xml
        // =========================================================

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


        // =========================================================
        // 5. Upload JAR to JFrog
        // =========================================================

        stage('Publish to JFrog') {

            steps {

                echo '===== Upload Artifact to JFrog ====='

                withCredentials([
                    string(
                        credentialsId: 'jfrog-access-token',
                        variable: 'JFROG_TOKEN'
                    )
                ]) {

                    sh '''
                        set -e

                        echo "Artifact:"
                        echo "target/${APP_NAME}-${VERSION}.jar"

                        test -f \
                          "target/${APP_NAME}-${VERSION}.jar"

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


        // =========================================================
        // 6. Download JAR from JFrog
        // =========================================================

        stage('Retrieve Artifact') {

            steps {

                echo '===== Download Artifact from JFrog ====='

                withCredentials([
                    string(
                        credentialsId: 'jfrog-access-token',
                        variable: 'JFROG_TOKEN'
                    )
                ]) {

                    sh '''
                        set -e

                        echo "===== Cleanup Previous Download ====="

                        rm -f \
                          "${APP_NAME}-${VERSION}.jar" \
                          app.jar


                        echo "===== Download from JFrog ====="

                        ${JF} rt dl \
                          "maven-local/${APP_NAME}-${VERSION}.jar" \
                          "./" \
                          --flat=true \
                          --url="${JFROG_URL}" \
                          --access-token="${JFROG_TOKEN}"


                        echo "===== Verify Download ====="

                        test -f \
                          "${APP_NAME}-${VERSION}.jar"

                        ls -lh \
                          "${APP_NAME}-${VERSION}.jar"


                        echo "===== Prepare JAR for Docker ====="

                        cp \
                          "${APP_NAME}-${VERSION}.jar" \
                          app.jar

                        test -f app.jar

                        ls -lh app.jar
                    '''
                }
            }
        }


        // =========================================================
        // 7. Build Docker Image
        // =========================================================

        stage('Build Docker Image') {

            steps {

                echo '===== Build Docker Image ====='

                sh '''
                    set -e

                    test -f Dockerfile
                    test -f app.jar

                    docker build \
                      -t "${APP_NAME}:${VERSION}" \
                      .

                    echo "===== Docker Image Created ====="

                    docker images \
                      | grep "${APP_NAME}" \
                      || true
                '''
            }
        }


        // =========================================================
        // 8. Push Docker Image to Azure Container Registry
        // =========================================================

        stage('Push to ACR') {

            steps {

                echo '===== Push Docker Image to ACR ====='

                sh '''
                    set -e


                    echo "===== Jenkins Azure Login ====="

                    az login \
                      --identity \
                      --output none


                    echo "===== ACR Login ====="

                    az acr login \
                      --name "${ACR_NAME}"


                    echo "===== Tag Docker Image ====="

                    docker tag \
                      "${APP_NAME}:${VERSION}" \
                      "${ACR_SERVER}/${APP_NAME}:${VERSION}"


                    echo "===== Push Docker Image ====="

                    docker push \
                      "${ACR_SERVER}/${APP_NAME}:${VERSION}"


                    echo "===== ACR Push Completed ====="

                    echo \
                      "${ACR_SERVER}/${APP_NAME}:${VERSION}"
                '''
            }
        }


        // =========================================================
        // 9. Deploy to Docker VM
        // =========================================================

        stage('Deploy') {

            steps {

                echo "===== Deploy ${env.VERSION} to Docker VM ====="

                sh '''
                    set -e

                    az vm run-command invoke \
                      --resource-group "${RESOURCE_GROUP}" \
                      --name "${DEPLOY_VM}" \
                      --command-id RunShellScript \
                      --scripts "

                        set -e


                        echo '================================='
                        echo 'Docker VM Azure Login'
                        echo '================================='

                        az login \
                          --identity \
                          --output none


                        echo '================================='
                        echo 'Docker VM ACR Login'
                        echo '================================='

                        az acr login \
                          --name ${ACR_NAME}


                        echo '================================='
                        echo 'Pull New Docker Image'
                        echo '================================='

                        docker pull \
                          ${ACR_SERVER}/${APP_NAME}:${VERSION}


                        echo '================================='
                        echo 'Remove Existing Container'
                        echo '================================='

                        docker rm \
                          -f \
                          ${APP_NAME} \
                          || true


                        echo '================================='
                        echo 'Start New Container'
                        echo '================================='

                        docker run \
                          -d \
                          --name ${APP_NAME} \
                          --restart unless-stopped \
                          -p 8080:8080 \
                          ${ACR_SERVER}/${APP_NAME}:${VERSION}


                        echo '================================='
                        echo 'Container Status'
                        echo '================================='

                        docker ps \
                          --filter name=${APP_NAME}


                        echo '================================='
                        echo 'Deployed Image'
                        echo '================================='

                        docker inspect \
                          ${APP_NAME} \
                          --format='{{.Config.Image}}'
                      "
                '''
            }
        }


        // =========================================================
        // 10. Verify Deployment
        // =========================================================

        stage('Verify Deployment') {

            steps {

                echo '===== Verify Application ====='

                sh '''
                    set -e

                    az vm run-command invoke \
                      --resource-group "${RESOURCE_GROUP}" \
                      --name "${DEPLOY_VM}" \
                      --command-id RunShellScript \
                      --scripts "

                        set -e


                        echo '================================='
                        echo 'Running Container'
                        echo '================================='

                        docker ps \
                          --filter name=${APP_NAME}


                        echo '================================='
                        echo 'Running Image'
                        echo '================================='

                        docker inspect \
                          ${APP_NAME} \
                          --format='{{.Config.Image}}'


                        echo '================================='
                        echo 'Application Health Check'
                        echo '================================='

                        curl \
                          --fail \
                          --retry 10 \
                          --retry-connrefused \
                          --retry-delay 3 \
                          http://localhost:8080
                      "
                '''
            }
        }
    }


    // =============================================================
    // Pipeline Result
    // =============================================================

    post {

        success {

            echo """
====================================================
DEPLOYMENT SUCCESSFUL
====================================================

Application : ${env.APP_NAME}
Version     : ${env.VERSION}

Docker Image:
${env.ACR_SERVER}/${env.APP_NAME}:${env.VERSION}

Docker VM:
${env.DEPLOY_VM}

====================================================
"""
        }


        failure {

            echo """
====================================================
PIPELINE FAILED
====================================================

Check the failed Jenkins stage
and Jenkins console output.

====================================================
"""
        }
    }
}
