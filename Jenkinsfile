```groovy
pipeline {
    agent any

    environment {
        DOCKER = 'C:\\Users\\ASUS\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin\\docker.exe'
        IMAGE_NAME = '24mis0111/student-placement'
        KUBE_DEPLOYMENT = 'student-placement'
        KUBE_CONTAINER = 'student-placement'
    }

    stages {
        stage('Checkout') {
            steps {
                echo 'Source code checked out from GitHub.'
            }
        }

        stage('Check Docker') {
            steps {
                bat '"%DOCKER%" --version'
                bat '"%DOCKER%" info'
            }
        }

        stage('Build Docker Image') {
            steps {
                bat '"%DOCKER%" build -t %IMAGE_NAME%:%BUILD_NUMBER% .'
                bat '"%DOCKER%" tag %IMAGE_NAME%:%BUILD_NUMBER% %IMAGE_NAME%:latest'
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credentials',
                    usernameVariable: 'DH_USER',
                    passwordVariable: 'DH_PASSWORD'
                )]) {
                    bat '''
                        echo %DH_PASSWORD% | "%DOCKER%" login -u "%DH_USER%" --password-stdin
                        if errorlevel 1 exit /b 1

                        "%DOCKER%" push %IMAGE_NAME%:%BUILD_NUMBER%
                        if errorlevel 1 exit /b 1

                        "%DOCKER%" push %IMAGE_NAME%:latest
                        if errorlevel 1 exit /b 1
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    try {
                        withCredentials([file(
                            credentialsId: 'kubernetes-config',
                            variable: 'KUBECONFIG_FILE'
                        )]) {
                            bat '''
                                set "KUBECONFIG=%KUBECONFIG_FILE%"

                                kubectl set image deployment/%KUBE_DEPLOYMENT% %KUBE_CONTAINER%=%IMAGE_NAME%:%BUILD_NUMBER%
                                if errorlevel 1 exit /b 1

                                kubectl rollout status deployment/%KUBE_DEPLOYMENT% --timeout=120s
                                if errorlevel 1 exit /b 1
                            '''
                        }
                    } catch (Exception e) {
                        echo 'Deployment failed. Attempting automatic rollback.'

                        withCredentials([file(
                            credentialsId: 'kubernetes-config',
                            variable: 'KUBECONFIG_FILE'
                        )]) {
                            bat '''
                                set "KUBECONFIG=%KUBECONFIG_FILE%"

                                kubectl rollout undo deployment/%KUBE_DEPLOYMENT%
                                if errorlevel 1 exit /b 1

                                kubectl rollout status deployment/%KUBE_DEPLOYMENT% --timeout=120s
                                if errorlevel 1 exit /b 1
                            '''
                        }

                        error('Deployment failed. Rollback attempted.')
                    }
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                withCredentials([file(
                    credentialsId: 'kubernetes-config',
                    variable: 'KUBECONFIG_FILE'
                )]) {
                    bat '''
                        set "KUBECONFIG=%KUBECONFIG_FILE%"

                        kubectl get deployments
                        kubectl get pods
                        kubectl get services
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'Student Placement Portal deployed successfully!'
        }

        failure {
            echo 'Pipeline failed. Check Console Output for the exact error.'
        }
    }
}
