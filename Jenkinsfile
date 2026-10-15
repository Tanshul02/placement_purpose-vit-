pipeline {
    agent any

    environment {
        DOCKER_HUB_USER = 'tanshul12'
        IMAGE_NAME      = 'placement-portal'
        IMAGE_TAG       = "${BUILD_NUMBER}"
        KUBECONFIG_CRED = 'k8s-kubeconfig'
        DOCKER_CRED     = 'dockerhub'
        DOCKER_BIN      = 'C:\\Users\\tanshul\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin'
    }

    stages {
        stage('1. Clone Code') {
            steps {
                checkout scm
            }
        }

        stage('2. Build Docker Image') {
            steps {
                bat "\"${DOCKER_BIN}\\docker.exe\" build -t %DOCKER_HUB_USER%/%IMAGE_NAME%:%IMAGE_TAG% ."
                bat "\"${DOCKER_BIN}\\docker.exe\" tag %DOCKER_HUB_USER%/%IMAGE_NAME%:%IMAGE_TAG% %DOCKER_HUB_USER%/%IMAGE_NAME%:latest"
            }
        }

        stage('3. Test Application Image') {
            steps {
                bat """
                    "${DOCKER_BIN}\\docker.exe" run -d --name temp-test -p 8085:80 %DOCKER_HUB_USER%/%IMAGE_NAME%:%IMAGE_TAG%
                    ping -n 4 127.0.0.1 >nul
                    curl -s -f http://localhost:8085 >nul || ("${DOCKER_BIN}\\docker.exe" rm -f temp-test && exit 1)
                    "${DOCKER_BIN}\\docker.exe" rm -f temp-test
                """
            }
        }

        stage('4. Push Image to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: DOCKER_CRED, usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
                    bat "echo %DH_PASS% | \"${DOCKER_BIN}\\docker.exe\" login -u %DH_USER% --password-stdin"
                    bat "\"${DOCKER_BIN}\\docker.exe\" push %DOCKER_HUB_USER%/%IMAGE_NAME%:%IMAGE_TAG%"
                    bat "\"${DOCKER_BIN}\\docker.exe\" push %DOCKER_HUB_USER%/%IMAGE_NAME%:latest"
                    bat "\"${DOCKER_BIN}\\docker.exe\" logout"
                }
            }
        }

        stage('5. Deploy to Kubernetes & Verify') {
            steps {
                withCredentials([file(credentialsId: KUBECONFIG_CRED, variable: 'KUBECONFIG_FILE')]) {
                    script {
                        bat "\"${DOCKER_BIN}\\kubectl.exe\" --kubeconfig=\"%KUBECONFIG_FILE%\" apply -f deployment.yaml"
                        bat "\"${DOCKER_BIN}\\kubectl.exe\" --kubeconfig=\"%KUBECONFIG_FILE%\" set image deployment/placement-portal-deployment placement-portal=%DOCKER_HUB_USER%/%IMAGE_NAME%:%IMAGE_TAG%"

                        def status = bat(
                            script: "\"${DOCKER_BIN}\\kubectl.exe\" --kubeconfig=\"%KUBECONFIG_FILE%\" rollout status deployment/placement-portal-deployment --timeout=60s",
                            returnStatus: true
                        )

                        if (status != 0) {
                            echo "Rollout failed! Automatically initiating rollback..."
                            bat "\"${DOCKER_BIN}\\kubectl.exe\" --kubeconfig=\"%KUBECONFIG_FILE%\" rollout undo deployment/placement-portal-deployment"
                            bat "\"${DOCKER_BIN}\\kubectl.exe\" --kubeconfig=\"%KUBECONFIG_FILE%\" rollout status deployment/placement-portal-deployment"
                            error("Deployment rollout timed out/failed. Rolled back to stable version.")
                        } else {
                            echo "Rollout completed successfully!"
                        }
                    }
                }
            }
        }
    }
}
