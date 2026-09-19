pipeline {

    agent any

    options {
        skipDefaultCheckout(true)
        disableConcurrentBuilds()
        timestamps()
    }

    triggers {
        githubPush()
    }

    tools {
        jdk 'JDK-17'
        maven 'mymaven'
    }

    environment {

        // Application
        APP_NAME = 'sample-web-app'

        // Azure Container Registry
        ACR_NAME   = 'acrteja23'
        ACR_SERVER = 'acrteja23.azurecr.io'

        // JFrog
        JFROG_URL  = 'http://20.88.47.132:8082/artifactory'
        JFROG_REPO = 'maven-local'

        // Azure deployment VM
        RESOURCE_GROUP = 'teja-rg'
        DEPLOY_VM      = 'docker-vm'

        // JFrog CLI
        JF = '/var/lib/jenkins/tools/io.jenkins.plugins.jfrog.JfrogInstallation/jfrog-cli/jf'
    }

    stages {

        // =====================================================
        // 1. Checkout
        // =====================================================

        stage('Checkout') {

            steps {

                echo '===== Checkout Source Code ====='

                checkout scm
            }
        }


        // =====================================================
        // 2. Verify Jenkins Tools
        // =====================================================

        stage('Verify Tools') {

            steps {

                sh '''
                    set -e

                    echo "======================================"
                    echo "JAVA_HOME"
                    echo "======================================"

                    echo "${JAVA_HOME}"


                    echo "======================================"
                    echo "Java Version"
                    echo "======================================"

                    java -version


                    echo "======================================"
                    echo "Javac Version"
                    echo "======================================"

                    javac -version


                    echo "======================================"
                    echo "Maven Version"
                    echo "======================================"

                    mvn -version


                    echo "======================================"
                    echo "Docker Version"
                    echo "======================================"

                    docker --version


                    echo "======================================"
                    echo "Azure CLI Version"
                    echo "======================================"

                    az --version


                    echo "======================================"
                    echo "JFrog CLI Version"
                    echo "======================================"

                    test -x "${JF}"

                    "${JF}" --version
                '''
            }
        }


        // =====================================================
        // 3. Maven Build
        // =====================================================

        stage('Build') {

            steps {

                echo '===== Maven Build ====='

                sh '''
                    set -e

                    mvn clean package
                '''
            }
        }


        // =====================================================
        // 4. Get Version from pom.xml
        // =====================================================

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

                    if (!env.VERSION) {
                        error('Unable to read project version from pom.xml')
                    }

                    echo "======================================"
                    echo "Application Version = ${env.VERSION}"
                    echo "======================================"
                }
            }
        }


        // =====================================================
        // 5. Upload Artifact to JFrog
        // =====================================================

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

                        ARTIFACT="target/${APP_NAME}-${VERSION}.jar"


                        echo "======================================"
                        echo "Verify Maven Artifact"
                        echo "======================================"

                        test -f "${ARTIFACT}"

                        ls -lh "${ARTIFACT}"


                        echo "======================================"
                        echo "Upload to JFrog"
                        echo "======================================"

                        "${JF}" rt u \
                          "${ARTIFACT}" \
                          "${JFROG_REPO}/" \
                          --flat=true \
                          --url="${JFROG_URL}" \
                          --access-token="${JFROG_TOKEN}"


                        echo "JFrog upload completed."
                '''
                }
            }
        }


        // =====================================================
        // 6. Download Artifact from JFrog
        // =====================================================

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


                        echo "======================================"
                        echo "Remove Old Download"
                        echo "======================================"

                        rm -f \
                          "${APP_NAME}-${VERSION}.jar" \
                          app.jar


                        echo "======================================"
                        echo "Download from JFrog"
                        echo "======================================"

                        "${JF}" rt dl \
                          "${JFROG_REPO}/${APP_NAME}-${VERSION}.jar" \
                          "./" \
                          --flat=true \
                          --url="${JFROG_URL}" \
                          --access-token="${JFROG_TOKEN}"


                        echo "======================================"
                        echo "Verify Download"
                        echo "======================================"

                        test -f "${APP_NAME}-${VERSION}.jar"

                        ls -lh "${APP_NAME}-${VERSION}.jar"


                        echo "======================================"
                        echo "Prepare app.jar"
                        echo "======================================"

                        cp \
                          "${APP_NAME}-${VERSION}.jar" \
                          app.jar


                        test -f app.jar

                        ls -lh app.jar
                '''
                }
            }
        }


        // =====================================================
        // 7. Build Docker Image
        // =====================================================

        stage('Build Docker Image') {

            steps {

                echo '===== Build Docker Image ====='

                sh '''
                    set -e


                    echo "======================================"
                    echo "Verify Docker Inputs"
                    echo "======================================"

                    test -f Dockerfile
                    test -f app.jar


                    echo "======================================"
                    echo "Build Docker Image"
                    echo "======================================"

                    docker build \
                      -t "${APP_NAME}:${VERSION}" \
                      .


                    echo "======================================"
                    echo "Verify Docker Image"
                    echo "======================================"

                    docker image inspect \
                      "${APP_NAME}:${VERSION}" \
                      >/dev/null


                    docker images \
                      | grep "${APP_NAME}" \
                      || true
                '''
            }
        }


        // =====================================================
        // 8. Push Docker Image to ACR
        // =====================================================

        stage('Push to ACR') {

            steps {

                echo '===== Push Image to ACR ====='

                sh '''
                    set -e


                    echo "======================================"
                    echo "Jenkins Azure Managed Identity Login"
                    echo "======================================"

                    az login \
                      --identity \
                      --output none


                    echo "======================================"
                    echo "ACR Login"
                    echo "======================================"

                    az acr login \
                      --name "${ACR_NAME}"


                    echo "======================================"
                    echo "Tag Docker Image"
                    echo "======================================"

                    docker tag \
                      "${APP_NAME}:${VERSION}" \
                      "${ACR_SERVER}/${APP_NAME}:${VERSION}"


                    echo "======================================"
                    echo "Push Docker Image"
                    echo "======================================"

                    docker push \
                      "${ACR_SERVER}/${APP_NAME}:${VERSION}"


                    echo "======================================"
                    echo "ACR Push Completed"
                    echo "======================================"

                    echo "${ACR_SERVER}/${APP_NAME}:${VERSION}"
                '''
            }
        }


        // =====================================================
        // 9. Deploy to Docker VM
        // =====================================================

        stage('Deploy') {

            steps {

                echo "===== Deploy Version ${env.VERSION} ====="

                sh '''
                    set -e


                    az vm run-command invoke \
                      --resource-group "${RESOURCE_GROUP}" \
                      --name "${DEPLOY_VM}" \
                      --command-id RunShellScript \
                      --scripts "

                        set -e


                        echo '======================================'
                        echo 'Docker VM Azure Login'
                        echo '======================================'

                        az login \
                          --identity \
                          --output none


                        echo '======================================'
                        echo 'Docker VM ACR Login'
                        echo '======================================'

                        az acr login \
                          --name ${ACR_NAME}


                        echo '======================================'
                        echo 'Pull New Docker Image'
                        echo '======================================'

                        docker pull \
                          ${ACR_SERVER}/${APP_NAME}:${VERSION}


                        echo '======================================'
                        echo 'Remove Existing Container'
                        echo '======================================'

                        docker rm \
                          -f \
                          ${APP_NAME} \
                          2>/dev/null || true


                        echo '======================================'
                        echo 'Start New Container'
                        echo '======================================'

                        docker run \
                          -d \
                          --name ${APP_NAME} \
                          --restart unless-stopped \
                          -p 8080:8080 \
                          ${ACR_SERVER}/${APP_NAME}:${VERSION}


                        echo '======================================'
                        echo 'Container Status'
                        echo '======================================'

                        docker ps \
                          --filter name=${APP_NAME}


                        echo '======================================'
                        echo 'Deployed Docker Image'
                        echo '======================================'

                        docker inspect \
                          ${APP_NAME} \
                          --format='{{.Config.Image}}'
                      "
                '''
            }
        }


        // =====================================================
        // 10. Verify Deployment
        // =====================================================

        stage('Verify Deployment') {

            steps {

                echo '===== Verify Application Deployment ====='

                sh '''
                    set -e


                    az vm run-command invoke \
                      --resource-group "${RESOURCE_GROUP}" \
                      --name "${DEPLOY_VM}" \
                      --command-id RunShellScript \
                      --scripts "

                        set -e


                        echo '======================================'
                        echo 'Wait for Application Startup'
                        echo '======================================'

                        sleep 8


                        echo '======================================'
                        echo 'Verify Container Running'
                        echo '======================================'

                        RUNNING=\\$(docker inspect \
                          --format='{{.State.Running}}' \
                          ${APP_NAME})


                        echo \"Container Running: \\${RUNNING}\"


                        if [ \"\\${RUNNING}\" != 'true' ]
                        then

                            echo 'Container is not running.'

                            echo '===== Container Logs ====='

                            docker logs \
                              --tail 100 \
                              ${APP_NAME} || true

                            exit 1
                        fi


                        echo '======================================'
                        echo 'Running Container'
                        echo '======================================'

                        docker ps \
                          --filter name=${APP_NAME}


                        echo '======================================'
                        echo 'Running Image'
                        echo '======================================'

                        docker inspect \
                          ${APP_NAME} \
                          --format='{{.Config.Image}}'


                        echo '======================================'
                        echo 'Application HTTP Check'
                        echo '======================================'

                        curl \
                          --fail \
                          --silent \
                          --show-error \
                          --retry 10 \
                          --retry-connrefused \
                          --retry-delay 3 \
                          --max-time 10 \
                          http://localhost:8080 \
                          >/dev/null


                        echo 'Application is responding successfully.'
                      "
                '''
            }
        }
    }


    // =========================================================
    // Pipeline Result
    // =========================================================

    post {

        success {

            echo """
====================================================
PIPELINE SUCCESS
====================================================

Application:
${env.APP_NAME}

Version:
${env.VERSION}

Docker Image:
${env.ACR_SERVER}/${env.APP_NAME}:${env.VERSION}

Docker VM:
${env.DEPLOY_VM}

Status:
SUCCESS

====================================================
"""
        }


        failure {

            echo """
====================================================
PIPELINE FAILED
====================================================

Check the failed Jenkins stage
and Jenkins Console Output.

====================================================
"""
        }
    }
}
