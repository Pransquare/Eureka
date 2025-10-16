pipeline {
    agent any

    environment {
        EC2_HOST = "13.53.39.170"
        SSH_CREDENTIALS_ID = "ec2-ssh-key"
        EC2_USER = "ec2-user"
        DEPLOY_DIR = "/home/ec2-user/services/eureka-server" // Change per service
        SERVICE_NAME = "eureka-server"                       // Change per service
        SERVER_PORT = "8761"                                 // Change per service
        LOG_FILE = "${SERVICE_NAME}.log"
    }

    tools {
        jdk 'Java17'
        maven 'Maven3'
    }

    stages {

        stage('Checkout') {
            steps {
                echo "===== Cloning repository ====="
                git branch: 'master', url: 'https://github.com/Pransquare/Eureka.git' // Change repo
            }
        }

        stage('Build') {
            steps {
                echo "===== Building project ====="
                bat "mvn clean package -DskipTests"
            }
        }

        stage('Prepare Deploy Script') {
            steps {
                echo "===== Preparing deploy script ====="
                script {
                    writeFile file: 'deploy.sh', text: """
                        #!/bin/bash
                        mkdir -p ${DEPLOY_DIR}
                        pkill -f ${SERVICE_NAME}.jar || true
                        nohup java -jar ${DEPLOY_DIR}/${SERVICE_NAME}.jar --server.port=${SERVER_PORT} > ${DEPLOY_DIR}/${LOG_FILE} 2>&1 &
                        echo "Deployment completed for ${SERVICE_NAME}"
                    """
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                echo "===== Deploying to EC2 ====="
                sshPublisher(
                    publishers: [
                        sshPublisherDesc(
                            configName: 'ec2-server',
                            transfers: [
                                sshTransfer(
                                    sourceFiles: "target/${SERVICE_NAME}.jar, deploy.sh",
                                    remoteDirectory: "${DEPLOY_DIR}",
                                    execCommand: "chmod +x ${DEPLOY_DIR}/deploy.sh && ${DEPLOY_DIR}/deploy.sh"
                                )
                            ],
                            verbose: true
                        )
                    ]
                )
            }
        }
    }

    post {
        success {
            echo "${SERVICE_NAME} deployed successfully on port ${SERVER_PORT}!"
        }
        failure {
            echo "Deployment failed for ${SERVICE_NAME}. Check Jenkins logs."
        }
    }
}
