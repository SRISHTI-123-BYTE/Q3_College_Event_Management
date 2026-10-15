
pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
        timestamps()
    }

    environment {
        DOCKER_IMAGE = 'hsri123/college-events'
        K8S_DEPLOYMENT = 'college-events-deployment'
        K8S_SERVICE = 'college-events-service'
        K8S_CONTAINER = 'college-events-container'
        DOCKER_BIN = 'C:\\Users\\srish\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin'
        KUBECTL = 'C:\\Users\\srish\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin\\kubectl.exe'
    }

    stages {
        stage('Clone Code') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/SRISHTI-123-BYTE/Q3_College_Event_Management.git'

                bat 'git log -1 --oneline'
            }
        }

        stage('Build Docker Image') {
            steps {
                withEnv(["PATH+DOCKER=${DOCKER_BIN}"]) {
                    bat '''
                        docker version
                        docker build -t %DOCKER_IMAGE%:%BUILD_NUMBER% .
                        docker tag %DOCKER_IMAGE%:%BUILD_NUMBER% %DOCKER_IMAGE%:latest
                    '''
                }
            }
        }

        stage('Test Application Image') {
            steps {
                withEnv(["PATH+DOCKER=${DOCKER_BIN}"]) {
                    bat '''
                        docker image inspect %DOCKER_IMAGE%:%BUILD_NUMBER%
                        if not exist index.html (
                            echo ERROR: index.html was not found in the workspace.
                            exit /b 1
                        )
                        findstr /I /C:"College" index.html
                    '''
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_TOKEN'
                    )
                ]) {
                    withEnv(["PATH+DOCKER=${DOCKER_BIN}"]) {
                        bat '''
                            @echo off
                            echo %DOCKER_TOKEN% | docker login -u %DOCKER_USER% --password-stdin
                            if errorlevel 1 exit /b 1

                            docker push %DOCKER_IMAGE%:%BUILD_NUMBER%
                            if errorlevel 1 exit /b 1

                            docker push %DOCKER_IMAGE%:latest
                            if errorlevel 1 exit /b 1

                            docker logout
                        '''
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'kubeconfig',
                        variable: 'KUBECONFIG_FILE'
                    )
                ]) {
                    script {
                        int deployResult = bat(
                            returnStatus: true,
                            script: '''
                                @echo off
                                set "KUBECONFIG=%KUBECONFIG_FILE%"

                                "%KUBECTL%" get nodes
                                if errorlevel 1 exit /b 1

                                "%KUBECTL%" set image deployment/%K8S_DEPLOYMENT% %K8S_CONTAINER%=%DOCKER_IMAGE%:%BUILD_NUMBER%
                                if errorlevel 1 exit /b 1

                                "%KUBECTL%" rollout status deployment/%K8S_DEPLOYMENT% --timeout=120s
                                if errorlevel 1 exit /b 1
                            '''
                        )

                        if (deployResult != 0) {
                            echo 'Deployment failed. Automatically rolling back.'

                            int rollbackResult = bat(
                                returnStatus: true,
                                script: '''
                                    @echo off
                                    set "KUBECONFIG=%KUBECONFIG_FILE%"

                                    "%KUBECTL%" rollout undo deployment/%K8S_DEPLOYMENT%
                                    if errorlevel 1 exit /b 1

                                    "%KUBECTL%" rollout status deployment/%K8S_DEPLOYMENT% --timeout=120s
                                    if errorlevel 1 exit /b 1
                                '''
                            )

                            if (rollbackResult != 0) {
                                error('Deployment failed and rollback could not be verified. Check Kubernetes events.')
                            }

                            error('Deployment failed. Previous release was restored.')
                        }
                    }
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'kubeconfig',
                        variable: 'KUBECONFIG_FILE'
                    )
                ]) {
                    withEnv(["PATH+DOCKER=${DOCKER_BIN}"]) {
                        bat '''
                            @echo off
                            set "KUBECONFIG=%KUBECONFIG_FILE%"

                            "%KUBECTL%" get deployment %K8S_DEPLOYMENT%
                            "%KUBECTL%" get pods -l app=college-events
                            "%KUBECTL%" get service %K8S_SERVICE%

                            "%KUBECTL%" rollout status deployment/%K8S_DEPLOYMENT% --timeout=120s
                            if errorlevel 1 exit /b 1

                            "%KUBECTL%" get deployment %K8S_DEPLOYMENT% -o jsonpath="{.status.readyReplicas}"
                            echo.
                        '''
                    }
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully.'
            echo 'Docker image built and pushed; Kubernetes deployment verified.'
        }

        failure {
            echo 'Pipeline failed. Review the first failed stage and its console output.'
        }

        always {
            echo 'CI/CD pipeline execution finished.'
        }
    }
}
