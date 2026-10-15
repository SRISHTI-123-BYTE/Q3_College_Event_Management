
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
        CONTAINER_NAME = 'college-events-container'
        APP_LABEL = 'college-events'

        DOCKER_BIN = 'C:\\Users\\srish\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin'
    }

    stages {

        stage('Clone Code') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/SRISHTI-123-BYTE/Q3_College_Event_Management.git'

                bat '''
                    @echo off
                    echo Checking project files...
                    if not exist index.html exit /b 1
                    if not exist style.css exit /b 1
                    if not exist Dockerfile exit /b 1
                    if not exist deployment.yaml exit /b 1
                    if not exist service.yaml exit /b 1
                    echo All required project files found.
                '''
            }
        }

        stage('Check Docker') {
            steps {
                bat '''
                    @echo off

                    if not exist "%DOCKER_BIN%\\docker.exe" (
                        echo ERROR: docker.exe not found at %DOCKER_BIN%
                        exit /b 1
                    )

                    "%DOCKER_BIN%\\docker.exe" version
                    if errorlevel 1 exit /b 1

                    "%DOCKER_BIN%\\docker.exe" info
                    if errorlevel 1 exit /b 1
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                bat '''
                    @echo off

                    "%DOCKER_BIN%\\docker.exe" build -t %DOCKER_IMAGE%:%BUILD_NUMBER% .
                    if errorlevel 1 exit /b 1

                    "%DOCKER_BIN%\\docker.exe" tag %DOCKER_IMAGE%:%BUILD_NUMBER% %DOCKER_IMAGE%:latest
                    if errorlevel 1 exit /b 1

                    echo Docker image built successfully.
                '''
            }
        }

        stage('Test Application Image') {
            steps {
                bat '''
                    @echo off

                    "%DOCKER_BIN%\\docker.exe" image inspect %DOCKER_IMAGE%:%BUILD_NUMBER%
                    if errorlevel 1 exit /b 1

                    echo Application image exists.
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
                        setlocal DisableDelayedExpansion

                        set "TOKENFILE=%WORKSPACE%\\docker-token.txt"
                        <nul set /p "=%DOCKER_TOKEN%" > "%TOKENFILE%"

                        "%DOCKER_BIN%\\docker.exe" login --username "%DOCKER_USER%" --password-stdin < "%TOKENFILE%"
                        set "LOGIN_STATUS=%ERRORLEVEL%"
                        del "%TOKENFILE%" 2>nul

                        if not "%LOGIN_STATUS%"=="0" exit /b 1

                        "%DOCKER_BIN%\\docker.exe" push %DOCKER_IMAGE%:%BUILD_NUMBER%
                        if errorlevel 1 (
                            "%DOCKER_BIN%\\docker.exe" logout
                            exit /b 1
                        )

                        "%DOCKER_BIN%\\docker.exe" push %DOCKER_IMAGE%:latest
                        if errorlevel 1 (
                            "%DOCKER_BIN%\\docker.exe" logout
                            exit /b 1
                        )

                        "%DOCKER_BIN%\\docker.exe" logout
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
                    script {
                        int deployResult = 0

                        try {
                            deployResult = bat(
                                returnStatus: true,
                                script: '''
                                    @echo off
                                    set "KUBECONFIG=%KUBECONFIG_FILE%"

                                    if not exist "%DOCKER_BIN%\\kubectl.exe" (
                                        echo ERROR: kubectl.exe not found.
                                        exit /b 1
                                    )

                                    "%DOCKER_BIN%\\kubectl.exe" get nodes
                                    if errorlevel 1 exit /b 1

                                    "%DOCKER_BIN%\\kubectl.exe" apply -f deployment.yaml
                                    if errorlevel 1 exit /b 1

                                    "%DOCKER_BIN%\\kubectl.exe" apply -f service.yaml
                                    if errorlevel 1 exit /b 1

                                    "%DOCKER_BIN%\\kubectl.exe" set image deployment/%DEPLOYMENT_NAME% %CONTAINER_NAME%=%DOCKER_IMAGE%:%BUILD_NUMBER%
                                    if errorlevel 1 exit /b 1

                                    "%DOCKER_BIN%\\kubectl.exe" rollout status deployment/%DEPLOYMENT_NAME% --timeout=150s
                                    if errorlevel 1 exit /b 1
                                '''
                            )

                            if (deployResult != 0) {
                                echo 'Deployment failed. Attempting automatic rollback.'

                                int rollbackResult = bat(
                                    returnStatus: true,
                                    script: '''
                                        @echo off
                                        set "KUBECONFIG=%KUBECONFIG_FILE%"

                                        "%DOCKER_BIN%\\kubectl.exe" rollout undo deployment/%DEPLOYMENT_NAME%
                                        if errorlevel 1 exit /b 1

                                        "%DOCKER_BIN%\\kubectl.exe" rollout status deployment/%DEPLOYMENT_NAME% --timeout=150s
                                        if errorlevel 1 exit /b 1
                                    '''
                                )

                                if (rollbackResult != 0) {
                                    error('Deployment failed and rollback could not be verified.')
                                }

                                error('Deployment failed. Previous release restored.')
                            }
                        } catch (err) {
                            if (deployResult == 0) {
                                throw err
                            }

                            throw err
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
                    bat '''
                        @echo off
                        set "KUBECONFIG=%KUBECONFIG_FILE%"

                        echo ===== Deployment =====
                        "%DOCKER_BIN%\\kubectl.exe" get deployment %DEPLOYMENT_NAME%
                        if errorlevel 1 exit /b 1

                        echo ===== Pods =====
                        "%DOCKER_BIN%\\kubectl.exe" get pods -l app=%APP_LABEL% -o wide
                        if errorlevel 1 exit /b 1

                        echo ===== Service =====
                        "%DOCKER_BIN%\\kubectl.exe" get service %SERVICE_NAME%
                        if errorlevel 1 exit /b 1

                        echo ===== Rollout Status =====
                        "%DOCKER_BIN%\\kubectl.exe" rollout status deployment/%DEPLOYMENT_NAME% --timeout=120s
                        if errorlevel 1 exit /b 1

                        echo ===== Ready Replicas =====
                        "%DOCKER_BIN%\\kubectl.exe" get deployment %DEPLOYMENT_NAME% -o jsonpath="{.status.readyReplicas}"
                        echo.
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'SUCCESS: Application built, pushed, deployed and verified.'
        }

        failure {
            echo 'FAILURE: Inspect the first failed stage in Console Output.'
        }

        always {
            echo 'CI/CD pipeline execution finished.'
        }
    }
}
