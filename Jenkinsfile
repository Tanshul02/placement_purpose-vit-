pipeline {
    agent any

    environment {
        DOCKER_HUB_USER = 'tanshul12'
        IMAGE_NAME      = 'placement-portal'
        IMAGE_TAG       = "${BUILD_NUMBER}"
        KUBECONFIG_CRED = 'k8s-kubeconfig'
        DOCKER_CRED     = 'dockerhub-cred'
    }

    stages {
        stage('1. Clone Code') {
            steps {
                checkout scm
            }
        }

        stage('2. Build Docker Image') {
            steps {
                bat "docker build -t %DOCKER_HUB_USER%/%IMAGE_NAME%:%IMAGE_TAG% ."
                bat "docker tag %DOCKER_HUB_USER%/%IMAGE_NAME%:%IMAGE_TAG% %DOCKER_HUB_USER%/%IMAGE_NAME%:latest"
            }
        }

        stage('3. Test Application Image') {
            steps {
                bat """
                    docker run -d --name temp-test -p 8085:80 %DOCKER_HUB_USER%/%IMAGE_NAME%:%IMAGE_TAG%
                    timeout /t 3 /nobreak >nul
                    curl -s -f http://localhost:8085 >nul || (docker rm -f temp-test && exit 1)
                    docker rm -f temp-test
                """
            }
        }

        stage('4. Push Image to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: DOCKER_CRED, usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
                    bat "echo %DH_PASS% | docker login -u %DH_USER% --password-stdin"
                    bat "docker push %DOCKER_HUB_USER%/%IMAGE_NAME%:%IMAGE_TAG%"
                    bat "docker push %DOCKER_HUB_USER%/%IMAGE_NAME%:latest"
                    bat "docker logout"
                }
            }
        }

        stage('5. Deploy to Kubernetes & Verify') {
            steps {
                withCredentials([file(credentialsId: KUBECONFIG_CRED, variable: 'KUBECONFIG_FILE')]) {
                    script {
                        bat "kubectl --kubeconfig=\"%KUBECONFIG_FILE%\" apply -f deployment.yaml"
                        bat "kubectl --kubeconfig=\"%KUBECONFIG_FILE%\" set image deployment/placement-portal-deployment placement-portal=%DOCKER_HUB_USER%/%IMAGE_NAME%:%IMAGE_TAG%"

                        def status = bat(
                            script: "kubectl --kubeconfig=\"%KUBECONFIG_FILE%\" rollout status deployment/placement-portal-deployment --timeout=60s",
                            returnStatus: true
                        )

                        if (status != 0) {
                            echo "Rollout failed! Automatically initiating rollback..."
                            bat "kubectl --kubeconfig=\"%KUBECONFIG_FILE%\" rollout undo deployment/placement-portal-deployment"
                            bat "kubectl --kubeconfig=\"%KUBECONFIG_FILE%\" rollout status deployment/placement-portal-deployment"
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
