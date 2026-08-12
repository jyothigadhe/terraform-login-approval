pipeline {

    agent any

    environment {
        DOCKER_IMAGE = "gadhe/terraform-login-app:latest"
        DOCKER_CREDENTIALS = "dockerhub-creds"

        DEPLOY_HOST = "98.130.42.197"
        SSH_CREDENTIALS = "terraform-ec2-ssh"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    docker build -t $DOCKER_IMAGE .
                '''
            }
        }

        stage('Docker Push') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: "${DOCKER_CREDENTIALS}",
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {
                    sh '''
                        echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin
                        docker push $DOCKER_IMAGE
                        docker logout
                    '''
                }
            }
        }

        stage('Email Approval') {
            steps {

                emailext(
                    subject: "Deployment Approval Required - ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                    to: "gadhejyothi15@gmail.com",
                    body: """
Hello,

Docker image has been successfully built and pushed.

Application:
terraform-login-app

Docker Image:
${DOCKER_IMAGE}

Deployment EC2:
${DEPLOY_HOST}

Jenkins Build:
${env.BUILD_URL}

Please open Jenkins.

Click PROCEED to approve deployment.

Click ABORT to deny deployment.

Deployment will happen only after approval.

Thanks,
Jenkins
"""
                )

                timeout(time: 10, unit: 'MINUTES') {
                    input message: 'Approve deployment to EC2?', ok: 'Approve'
                }
            }
        }

        stage('Deploy to EC2') {
            steps {

                sshagent(credentials: ["${SSH_CREDENTIALS}"]) {

                    sh '''
                        ssh -o StrictHostKeyChecking=no ec2-user@$DEPLOY_HOST "
                            docker pull $DOCKER_IMAGE &&
                            docker stop terraform-login-app || true &&
                            docker rm terraform-login-app || true &&
                            docker run -d \
                                --name terraform-login-app \
                                -p 80:80 \
                                $DOCKER_IMAGE
                        "
                    '''
                }
            }
        }
    }

    post {

        success {
            emailext(
                subject: "Deployment Successful - ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                to: "gadhejyothi15@gmail.com",
                body: """
Deployment completed successfully.

Application:
terraform-login-app

EC2:
${DEPLOY_HOST}

Open:
http://${DEPLOY_HOST}
"""
            )
        }

        failure {
            emailext(
                subject: "Deployment Failed - ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                to: "gadhejyothi15@gmail.com",
                body: """
Deployment failed.

Job:
${env.JOB_NAME}

Build:
${env.BUILD_NUMBER}

Check Jenkins console output:
${env.BUILD_URL}
"""
            )
        }
    }
}
