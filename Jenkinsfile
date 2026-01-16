pipeline {
    agent any

    stages {

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
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
                script {
                    try {
                        input message: 'Approve deployment of MySQL to Kubernetes'
                        echo 'MySQL deployment approved.'
                    } catch (err) {
                        error 'MySQL deployment aborted by user.'
                    }
                }
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

        stage('Approve Two-Tier App Deployment') {
            steps {
                script {
                    try {
                        input message: 'Approve deployment of Two-Tier Application to Kubernetes'
                        echo 'Two-Tier application deployment approved.'
                    } catch (err) {
                        error 'Two-Tier application deployment aborted by user.'
                    }
                }
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
        
        //stage('Minikube Service Start') {
            //steps {
                //sh 'minikube service two-tier-svc'
            //}
        //}
    }

    post {

        always {
            sh 'docker logout || true'
        }

        success {
            emailext(
                attachLog: true,
                subject: "SUCCESS | ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """
                <p>Hello Team,</p>

                <p>
                    The Jenkins job <b>${env.JOB_NAME}</b> completed
                    <b style="color:green;">SUCCESSFULLY</b>.
                </p>

                <table cellpadding="6">
                    <tr>
                        <td><b>Build Number:</b></td>
                        <td>${env.BUILD_NUMBER}</td>
                    </tr>
                    <tr>
                        <td><b>Build URL:</b></td>
                        <td><a href="${env.BUILD_URL}">${env.BUILD_URL}</a></td>
                    </tr>
                </table>

                <p>Security scan reports are attached.</p>

                <p>
                    Regards,<br/>
                    <b>Mradul Singh : DevOps Engineer</b>
                </p>
                """,
                to: 'vibhishh@gmail.com',
                attachmentsPattern: 'trivy-fs-report.txt,trivy-image-report.txt'
            )
        }

        failure {
            emailext(
                attachLog: true,
                subject: "FAILED | ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """
                <p>Hello Team,</p>

                <p>
                    The Jenkins job <b>${env.JOB_NAME}</b>
                    <b style="color:red;">FAILED</b>.
                </p>

                <table cellpadding="6">
                    <tr>
                        <td><b>Build Number:</b></td>
                        <td>${env.BUILD_NUMBER}</td>
                    </tr>
                    <tr>
                        <td><b>Build URL:</b></td>
                        <td><a href="${env.BUILD_URL}">${env.BUILD_URL}</a></td>
                    </tr>
                </table>

                <p>Please review the logs.</p>

                <p>
                    Regards,<br/>
                    <b>Mradul Singh : DevOps Engineer</b>
                </p>
                """,
                to: 'vibhishh@gmail.com'
            )
        }
    }
}
