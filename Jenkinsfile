```groovy
pipeline {
    agent any

    options {
        timestamps()
        skipDefaultCheckout(true)
        disableConcurrentBuilds()
        timeout(time: 15, unit: 'MINUTES')
    }

    environment {
        DOCKER_IMAGE = 'hsri123/college-events'
        DEPLOYMENT_NAME = 'college-events-deployment'
        SERVICE_NAME = 'college-events-service'
        APP_LABEL = 'college-events'
        DOCKER_BIN = 'C:\\Users\\srish\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin'
    }

    stages {

        stage('Clone Code') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/SRISHTI-123-BYTE/Q3_College_Event_Management.git'

                bat '''
                    echo Repository checkout successful
                    git log -1 --oneline
                    if not exist index.html exit /b 1
                    if not exist Dockerfile exit /b 1
                    if not exist deployment.yaml exit /b 1
                    if not exist service.yaml exit /b 1
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                bat '''
                    docker version
                    docker build --no-cache -t %DOCKER_IMAGE%:%BUILD_NUMBER% .
                    if errorlevel 1 exit /b 1

                    docker tag %DOCKER_IMAGE%:%BUILD_NUMBER% %DOCKER_IMAGE%:latest
                    if errorlevel 1 exit /b 1
                '''
            }
        }

        stage('Test Application Image') {
            steps {
                bat '''
                    docker image inspect %DOCKER_IMAGE%:%BUILD_NUMBER%
                    if errorlevel 1 exit /b 1

                    if not exist index.html (
                        echo ERROR: index.html is missing
                        exit /b 1
                    )

                    findstr /I /C:"College" index.html
                    if errorlevel 1 (
                        echo WARNING: Expected College text was not found
                    )

                    echo Docker image and application files checked.
                '''
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
                    bat '''
                        @echo off
                        echo Logging in to Docker Hub...

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

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'kubeconfig',
                        variable: 'KUBECONFIG_FILE'
                    )
                ]) {
                    withEnv([
                        "PATH+KUBECTL=${env.DOCKER_BIN}"
                    ]) {
                        script {
                            try {
                                bat '''
                                    @echo off
                                    set KUBECONFIG=%KUBECONFIG_FILE%

                                    kubectl version --client
                                    if errorlevel 1 exit /b 1

                                    kubectl cluster-info
                                    if errorlevel 1 exit /b 1

                                    kubectl apply -f deployment.yaml
                                    if errorlevel 1 exit /b 1

                                    kubectl apply -f service.yaml
                                    if errorlevel 1 exit /b 1

                                    kubectl set image deployment/%DEPLOYMENT_NAME% *=%DOCKER_IMAGE%:%BUILD_NUMBER%
                                    if errorlevel 1 exit /b 1
                                '''

                                timeout(time: 3, unit: 'MINUTES') {
                                    bat '''
                                        set KUBECONFIG=%KUBECONFIG_FILE%
                                        kubectl rollout status deployment/%DEPLOYMENT_NAME% --timeout=150s
                                    '''
                                }
                            } catch (err) {
                                echo 'Deployment failed. Attempting rollback to the previous revision.'

                                bat '''
                                    @echo off
                                    set KUBECONFIG=%KUBECONFIG_FILE%
                                    kubectl rollout undo deployment/%DEPLOYMENT_NAME%
                                    kubectl rollout status deployment/%DEPLOYMENT_NAME% --timeout=150s
                                '''

                                error("New deployment failed. Rollback was attempted. Check Kubernetes status.")
                            }
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
                    withEnv([
                        "PATH+KUBECTL=${env.DOCKER_BIN}"
                    ]) {
                        bat '''
                            @echo off
                            set KUBECONFIG=%KUBECONFIG_FILE%

                            echo ===== DEPLOYMENT =====
                            kubectl get deployment %DEPLOYMENT_NAME%
                            if errorlevel 1 exit /b 1

                            echo ===== PODS =====
                            kubectl get pods -l app=%APP_LABEL% -o wide
                            if errorlevel 1 exit /b 1

                            echo ===== SERVICE =====
                            kubectl get service %SERVICE_NAME%
                            if errorlevel 1 exit /b 1

                            echo ===== DEPLOYED IMAGE =====
                            kubectl get deployment %DEPLOYMENT_NAME% -o jsonpath="{.spec.template.spec.containers[0].image}"
                            echo.

                            echo ===== ROLLOUT STATUS =====
                            kubectl rollout status deployment/%DEPLOYMENT_NAME% --timeout=120s
                            if errorlevel 1 exit /b 1

                            echo ===== READY REPLICA COUNT =====
                            kubectl get deployment %DEPLOYMENT_NAME% -o jsonpath="{.status.readyReplicas}"
                            echo.
                        '''
                    }
                }
            }
        }
    }

    post {
        success {
            echo 'SUCCESS: College Event Management application was built, pushed, deployed, and verified.'
        }

        failure {
            echo 'FAILURE: Check the first failed stage in the Console Output.'
        }

        always {
            echo 'CI/CD pipeline execution finished.'
        }
    }
}
```
