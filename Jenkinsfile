pipeline {
    agent any

    environment {
        DEPLOY_DIR = "/home/ec2-user/deployment"
        EC2_HOST = "13.60.33.154"
        SERVICE_NAME = "eureka-server"
        SERVER_PORT = "8761"
        LOG_FILE = "eureka-server.log"
        SSH_CREDENTIALS_ID = "ec2-key" // Jenkins SSH credentials
    }

    tools {
        jdk 'Java17'
        maven 'Maven3'
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'master', url: 'https://github.com/Pransquare/Eureka.git'
            }
        }

        stage('Build') {
            steps {
                script {
                    def mvnCmd = 'mvn clean package -DskipTests'
                    if (isUnix()) {
                        sh mvnCmd
                    } else {
                        bat mvnCmd
                    }
                }
            }
        }

        stage('Prepare Deploy Script') {
            steps {
                script {
                    // Write deploy.sh inside target/
                    writeFile file: "target/deploy.sh", text: """
#!/bin/bash
mkdir -p ${DEPLOY_DIR}
pkill -f ${SERVICE_NAME}.jar || true
nohup java -jar ${DEPLOY_DIR}/${SERVICE_NAME}.jar --server.port=${SERVER_PORT} > ${DEPLOY_DIR}/${LOG_FILE} 2>&1 &
echo "Deployment completed. Check logs at ${DEPLOY_DIR}/${LOG_FILE}"
"""
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                sshPublisher(
                    publishers: [
                        sshPublisherDesc(
                            configName: 'ec2-ssh-server',
                            transfers: [
                                sshTransfer(
                                    sourceFiles: "target/${SERVICE_NAME}.jar, target/deploy.sh",
                                    removePrefix: 'target',
                                    remoteDirectory: DEPLOY_DIR,
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
            echo " Deployment completed successfully! Check EC2 logs at ${DEPLOY_DIR}/${LOG_FILE}"
        }
        failure {
            echo " Deployment failed. Check Jenkins console logs for details."
        }
    }
}
