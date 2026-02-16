pipeline {
    agent any

    stages {

        stage('Clean Workspace') {
            steps { cleanWs() }
        }

        stage('Checkout the Code') {
            steps {
                git url: 'https://github.com/vibhishh/two-tier-flask-app.git', branch: 'master'
            }
        }

        stage('Setup Sonar Scanner') {
            steps {
                script {
                    env.SCANNER_HOME = tool(
                        name: 'sonar-scanner',
                        type: 'hudson.plugins.sonar.SonarRunnerInstallation'
                    )
                    env.PATH = "${env.SCANNER_HOME}/bin:${env.PATH}"
                    sh 'sonar-scanner --version'
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-scanner') {
                    sh '''
                        sonar-scanner \
                        -Dsonar.projectName=Two-Tier-App \
                        -Dsonar.projectKey=Two-Tier-App
                    '''
                }
            }
        }

        stage('Trivy File System Scan') {
            steps {
                sh 'trivy fs --severity HIGH,CRITICAL --format table --output trivy-fs-report.txt .'
                archiveArtifacts artifacts: 'trivy-fs-report.txt', fingerprint: true
            }
        }

        stage('Docker Image Build') {
            steps {
                sh 'docker build -t two-tier-app:latest .'
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh 'trivy image --severity HIGH,CRITICAL --format table --output trivy-image-report.txt two-tier-app:latest'
                archiveArtifacts artifacts: 'trivy-image-report.txt', fingerprint: true
            }
        }

        stage('Docker Login and Push Image') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-ID',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker tag two-tier-app:latest $DOCKER_USER/two-tier-app:latest
                        docker push $DOCKER_USER/two-tier-app:latest
                    '''
                }
            }
        }

        stage('Approve MySQL Deployment') {
            steps {
                input message: 'Deploy MySQL to Kubernetes?', ok: 'Deploy'
            }
        }

        stage('Deploy MySQL to Kubernetes') {
            steps {
                withKubeConfig(credentialsId: 'k8s') {
                    sh 'kubectl apply -f k8s/mysql-pv.yml'
                    sh 'kubectl apply -f k8s/mysql-pvc.yml'
                    sh 'kubectl apply -f k8s/mysql-deployment.yml'
                    sh 'kubectl apply -f k8s/mysql-svc.yml'
                }
            }
        }

        stage('Approve App Deployment') {
            steps {
                input message: 'Deploy Two-Tier App?', ok: 'Deploy'
            }
        }

        stage('Deploy Two-Tier App to Kubernetes') {
            steps {
                withKubeConfig(credentialsId: 'k8s') {
                    sh 'kubectl apply -f k8s/two-tier-app-deployment.yml'
                    sh 'kubectl apply -f k8s/two-tier-app-svc.yml'
                }
            }
        }
        stage('Expose Service') {
            steps {
                script {
                    sh 'minikube service two-tier-svc'
                }
            }
        }
    }

    post {

        always {
            sh 'docker logout || true'
        }

        success {
            emailext(
                attachLog: true,
                subject: "SUCCESS | ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: "Build Successful — Reports attached.",
                to: 'vibhishh@gmail.com',
                attachmentsPattern: 'trivy-fs-report.txt,trivy-image-report.txt'
            )
        }

        failure {
            emailext(
                attachLog: true,
                subject: "FAILED | ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: "Build Failed — Check logs.",
                to: 'vibhishh@gmail.com'
            )
        }
    }
}
