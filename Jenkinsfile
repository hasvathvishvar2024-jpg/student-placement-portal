pipeline {
    agent any

    environment {
        IMAGE_NAME = '24mis0111/student-placement'
        KUBE_DEPLOYMENT = 'student-placement'
        KUBE_CONTAINER = 'student-placement'
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/hasvathvishvar2024-jpg/student-placement-portal.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                bat 'docker build -t %IMAGE_NAME%:%BUILD_NUMBER% .'
                bat 'docker tag %IMAGE_NAME%:%BUILD_NUMBER% %IMAGE_NAME%:latest'
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credentials',
                    usernameVariable: 'DH_USER',
                    passwordVariable: 'DH_PASSWORD'
                )]) {
                    bat 'docker login -u %DH_USER% -p %DH_PASSWORD%'
                    bat 'docker push %IMAGE_NAME%:%BUILD_NUMBER%'
                    bat 'docker push %IMAGE_NAME%:latest'
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
                                set KUBECONFIG=%KUBECONFIG_FILE%
                                kubectl set image deployment/%KUBE_DEPLOYMENT% %KUBE_CONTAINER%=%IMAGE_NAME%:%BUILD_NUMBER%
                                if errorlevel 1 exit /b 1
                                kubectl rollout status deployment/%KUBE_DEPLOYMENT% --timeout=120s
                                if errorlevel 1 exit /b 1
                            '''
                        }
                    } catch (Exception e) {
                        echo 'Deployment failed. Rolling back to the previous revision.'

                        withCredentials([file(
                            credentialsId: 'kubernetes-config',
                            variable: 'KUBECONFIG_FILE'
                        )]) {
                            bat '''
                                set KUBECONFIG=%KUBECONFIG_FILE%
                                kubectl rollout undo deployment/%KUBE_DEPLOYMENT%
                                kubectl rollout status deployment/%KUBE_DEPLOYMENT% --timeout=120s
                            '''
                        }

                        error('Deployment failed. Rollback was attempted.')
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
                        set KUBECONFIG=%KUBECONFIG_FILE%
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
            echo 'Pipeline failed. Check the console output and rollback logs.'
        }
    }
}