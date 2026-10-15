pipeline {
    agent any

    environment {
        DOCKER_IMAGE = '24mis0111/student-placement'
        DOCKER = 'C:\\Users\\ASUS\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin\\docker.exe'
        KUBECTL = 'C:\\Users\\ASUS\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin\\kubectl.exe'
    }

    stages {
        stage('Clone Code') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/hasvathvishvar2024-jpg/student-placement-portal.git'
            }
        }

        stage('Check Docker') {
            steps {
                bat '''
                    echo Checking Docker...
                    "%DOCKER%" --version
                    if errorlevel 1 exit /b 1
                '''
            }
        }

        stage('Build New Docker Image') {
            steps {
                bat '''
                    echo Building Student Placement Portal image...

                    "%DOCKER%" build -t %DOCKER_IMAGE%:%BUILD_NUMBER% .
                    if errorlevel 1 exit /b 1

                    "%DOCKER%" tag %DOCKER_IMAGE%:%BUILD_NUMBER% %DOCKER_IMAGE%:latest
                    if errorlevel 1 exit /b 1

                    echo Docker image built successfully.
                '''
            }
        }

        stage('Push New Image') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {
                    bat '''
                        echo Logging in to Docker Hub...

                        echo %DOCKER_PASSWORD% | "%DOCKER%" login -u "%DOCKER_USER%" --password-stdin
                        if errorlevel 1 exit /b 1

                        echo Pushing versioned image...

                        "%DOCKER%" push %DOCKER_IMAGE%:%BUILD_NUMBER%
                        if errorlevel 1 exit /b 1

                        echo Pushing latest image...

                        "%DOCKER%" push %DOCKER_IMAGE%:latest
                        if errorlevel 1 exit /b 1

                        echo Docker image pushed successfully.
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    try {
                        withCredentials([
                            file(
                                credentialsId: 'kubernetes-config',
                                variable: 'KUBECONFIG_FILE'
                            )
                        ]) {
                            bat '''
                                echo Applying Kubernetes configuration...

                                "%KUBECTL%" --kubeconfig "%KUBECONFIG_FILE%" apply -f deployment.yaml
                                if errorlevel 1 exit /b 1

                                echo Updating Student Placement Portal image...

                                "%KUBECTL%" --kubeconfig "%KUBECONFIG_FILE%" set image deployment/student-placement student-placement=%DOCKER_IMAGE%:%BUILD_NUMBER%
                                if errorlevel 1 exit /b 1

                                echo Waiting for rollout...

                                "%KUBECTL%" --kubeconfig "%KUBECONFIG_FILE%" rollout status deployment/student-placement --timeout=120s
                                if errorlevel 1 exit /b 1

                                echo Deployment completed successfully.
                            '''
                        }
                    } catch (Exception e) {
                        echo 'Deployment failed. Starting automatic rollback.'

                        withCredentials([
                            file(
                                credentialsId: 'kubernetes-config',
                                variable: 'KUBECONFIG_FILE'
                            )
                        ]) {
                            bat '''
                                echo Rolling back to the previous stable version...

                                "%KUBECTL%" --kubeconfig "%KUBECONFIG_FILE%" rollout undo deployment/student-placement
                                if errorlevel 1 exit /b 1

                                echo Waiting for rollback to complete...

                                "%KUBECTL%" --kubeconfig "%KUBECONFIG_FILE%" rollout status deployment/student-placement --timeout=120s
                                if errorlevel 1 exit /b 1

                                echo Previous stable version restored.
                            '''
                        }

                        error('New deployment failed. Automatic rollback was attempted.')
                    }
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'kubernetes-config',
                        variable: 'KUBECONFIG_FILE'
                    )
                ]) {
                    bat '''
                        echo ======================================
                        echo STUDENT PLACEMENT PORTAL VERIFICATION
                        echo ======================================

                        echo Deployment status:

                        "%KUBECTL%" --kubeconfig "%KUBECONFIG_FILE%" get deployment student-placement
                        if errorlevel 1 exit /b 1

                        echo Pod status:

                        "%KUBECTL%" --kubeconfig "%KUBECONFIG_FILE%" get pods -o wide
                        if errorlevel 1 exit /b 1

                        echo Service status:

                        "%KUBECTL%" --kubeconfig "%KUBECONFIG_FILE%" get service student-placement-service
                        if errorlevel 1 exit /b 1

                        echo ======================================
                        echo VERIFICATION COMPLETED
                        echo ======================================
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'Student Placement Portal pipeline completed successfully.'
        }

        failure {
            echo 'Pipeline failed. Check Console Output for details.'
        }
    }
}