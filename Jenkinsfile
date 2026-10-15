
pipeline {
    agent any

    options {
        timestamps()
        timeout(time: 20, unit: 'MINUTES')
    }

    environment {
        DOCKER_BIN = 'C:\\Users\\srish\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin'
        IMAGE_NAME = 'hsri123/college-events'
        IMAGE_TAG = "${BUILD_NUMBER}"
        K8S_DEPLOYMENT = 'college-events-deployment'
        K8S_CONTAINER = 'college-events-container'
        DOCKER_CONFIG = "${WORKSPACE}\\.docker-jenkins"
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
                    "%DOCKER_BIN%\\docker.exe" version
                    if errorlevel 1 exit /b 1
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                bat '''
                    @echo off
                    echo Building College Event Management image...

                    "%DOCKER_BIN%\\docker.exe" build ^
                        -t "%IMAGE_NAME%:%IMAGE_TAG%" ^
                        -t "%IMAGE_NAME%:latest" .

                    if errorlevel 1 exit /b 1

                    echo Docker image built successfully.
                '''
            }
        }

        stage('Test Application Image') {
            steps {
                bat '''
                    @echo off
                    echo Checking the built image...

                    "%DOCKER_BIN%\\docker.exe" image inspect "%IMAGE_NAME%:%IMAGE_TAG%"

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
                    script {
                        bat '''
                            @echo off
                            if exist "%DOCKER_CONFIG%" rmdir /s /q "%DOCKER_CONFIG%"
                            mkdir "%DOCKER_CONFIG%"
                        '''

                        withEnv(["DOCKER_CONFIG=${env.DOCKER_CONFIG}"]) {
                            bat '''
                                @echo off
                                echo Authenticating with Docker Hub...

                                powershell -NoProfile -ExecutionPolicy Bypass -Command "$env:DOCKER_TOKEN | & (Join-Path $env:DOCKER_BIN 'docker.exe') login --username $env:DOCKER_USER --password-stdin"

                                if errorlevel 1 exit /b 1

                                echo Pushing versioned image...

                                "%DOCKER_BIN%\\docker.exe" push "%IMAGE_NAME%:%IMAGE_TAG%"

                                if errorlevel 1 exit /b 1

                                echo Pushing latest image...

                                "%DOCKER_BIN%\\docker.exe" push "%IMAGE_NAME%:latest"

                                if errorlevel 1 exit /b 1

                                echo Docker Hub push completed successfully.
                            '''
                        }
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
                    bat '''
                        @echo off
                        echo Checking Kubernetes connection...

                        "%DOCKER_BIN%\\kubectl.exe" --kubeconfig="%KUBECONFIG_FILE%" get nodes

                        if errorlevel 1 exit /b 1

                        echo Applying Kubernetes Deployment...

                        "%DOCKER_BIN%\\kubectl.exe" --kubeconfig="%KUBECONFIG_FILE%" apply -f deployment.yaml

                        if errorlevel 1 exit /b 1

                        echo Applying Kubernetes Service...

                        "%DOCKER_BIN%\\kubectl.exe" --kubeconfig="%KUBECONFIG_FILE%" apply -f service.yaml

                        if errorlevel 1 exit /b 1

                        echo Updating deployment to the new image...

                        "%DOCKER_BIN%\\kubectl.exe" --kubeconfig="%KUBECONFIG_FILE%" set image deployment/%K8S_DEPLOYMENT% %K8S_CONTAINER%=%IMAGE_NAME%:%IMAGE_TAG%

                        if errorlevel 1 exit /b 1

                        echo Waiting for deployment rollout...

                        "%DOCKER_BIN%\\kubectl.exe" --kubeconfig="%KUBECONFIG_FILE%" rollout status deployment/%K8S_DEPLOYMENT% --timeout=180s

                        if errorlevel 1 exit /b 1

                        echo Kubernetes deployment completed successfully.
                    '''
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
                        echo Checking deployment status...

                        "%DOCKER_BIN%\\kubectl.exe" --kubeconfig="%KUBECONFIG_FILE%" get deployment %K8S_DEPLOYMENT%

                        if errorlevel 1 exit /b 1

                        echo Checking application pods...

                        "%DOCKER_BIN%\\kubectl.exe" --kubeconfig="%KUBECONFIG_FILE%" get pods -l app=college-events

                        if errorlevel 1 exit /b 1

                        echo Checking Kubernetes Service...

                        "%DOCKER_BIN%\\kubectl.exe" --kubeconfig="%KUBECONFIG_FILE%" get service college-events-service

                        if errorlevel 1 exit /b 1

                        echo Deployment verification completed.
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'SUCCESS: College Event Management pipeline completed.'
            echo 'Docker image pushed and Kubernetes deployment verified.'
        }

        failure {
            echo 'FAILURE: Open Console Output and inspect the first failed stage.'
        }

        always {
            echo 'CI/CD pipeline execution finished.'
        }
    }
}
